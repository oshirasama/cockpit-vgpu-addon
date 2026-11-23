# Add vGPU Management to Cockpit on RHEL KVM

Use `vgpu-web-cockpit-addon-rhel-kvm-en.zip` to add the **vGPU Management** page, a host configuration helper, and a boot-time restore service to an existing RHEL KVM host. Use the host's existing Cockpit URL, TLS certificate, and sign-in. [Cockpit package documentation](https://docs.cockpit-project.org/cockpit-guide/364/guide/packages.html)

## Prerequisites

- Cockpit is running on the target host. Users who configure vGPUs can switch to **Administrative access** in Cockpit.
- Python 3, libvirt (`qemu:///system`), NVIDIA vGPU Manager, and the NVIDIA driver are already installed and configured.
- The add-on manages **Equal-Size**, **Mixed-Size** on capable GPUs, and **MIG** on RTX PRO 6000 and 4500 Blackwell Server Edition. All use SR-IOV VFs with NVIDIA vendor-specific VFIO. For RHEL 10.2 KVM, use a compatible NVIDIA vGPU Manager. [NVIDIA vGPU 21.0 RHEL KVM support matrix](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-release-notes-red-hat-el-kvm/validated-platforms.html)

## 1. Install the add-on

Copy the ZIP to `~/Downloads` on the target host. Sign in with a regular user account that can run `sudo`, then run the following commands.

```bash
vgpu_root="$HOME/Downloads/vgpu-web-cockpit-addon-rhel-kvm-en"
python3 -m zipfile -e "$vgpu_root.zip" "$vgpu_root" &&
bash "$vgpu_root/vgpu-web/deploy/install-cockpit.sh"
```

To update, extract the new ZIP and rerun the same installer. It preserves the existing restore settings. To capture the active configuration on the first installation, verify the host configuration and pass `--adopt-current` to the installer.

## 2. Open vGPU Management

1. Open the target host's Cockpit URL. If you already have a session open, sign out and sign in again.
2. Select **vGPU Management** in the sidebar. Choose a physical GPU to see its VFs, profiles, and VM references.
3. Switch to **Administrative access** to make changes. Use an available VF and a stopped VM when assigning a vGPU. The page can enable VFs, set or clear vGPU types, attach a vGPU with `managed=no`, and detach a VF from all referencing VMs. [Cockpit process execution and privileges](https://docs.cockpit-project.org/cockpit-guide/368/guide/cockpit-spawn.html)

### Equal-Size / Mixed-Size / MIG

The mode selector uses **Equal-Size / Mixed-Size / MIG**. GPUs without supported MIG management, such as L40S, show only Equal-Size and Mixed-Size. Select a mode and use **Change mode** to apply it. Selecting a radio option alone never changes the host. Remove VM references, existing and saved vGPU types, and remaining GIs before switching modes.

While a mode change is pending, only cleanup operations are available; new GI/vGPU creation and VM attachments are hidden. Profile options are filtered by the active GPU mode: for example, DC-2Q in Equal-Size / Mixed-Size and DC-1-2Q in MIG. The host helper also rejects profiles from the wrong mode.

Mixed-Size is available when the driver confirms support. Configure different-size profiles on available VFs and attach them to stopped VMs. The driver assigns placement automatically. Startup restore enables VFs, applies the saved size mode, then creates vGPUs. Mixed-Size within MIG GIs is outside this update's scope. [NVIDIA Mixed-Size configuration](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-user-guide/installing-configuring-grid-vgpu.html#putting-a-gpu-into-mixed-size-mode-on-a-linux-with-kvm-hypervisor)

### Create a MIG-backed vGPU

Switch to **Administrative access** in Cockpit and select **Refresh** to view or configure GIs and CIs. Existing instances include non-graphics GIs so they can be identified and deleted when no longer needed. Creation and vGPU placement use graphics-enabled `+gfx` profiles. [NVIDIA SMI MIG management commands](https://docs.nvidia.com/deploy/nvidia-smi/index.html#mig-management)

When no GIs exist, **Manage GI / CI** opens automatically. Create a GI and default CI before configuring the VF profile. GI creation also works when MIG was enabled from the CLI and registers MIG mode and the new GI for startup restore.

1. Shut down the VMs using the target GPU, detach its VFs, and clear their vGPU types. Stop host workloads using that GPU.
2. Select **MIG**, then **Change mode**. Blackwell can enable MIG without a host reboot. Verify that the current and pending modes both show `enabled`. If enabled from the command line, check **Restore MIG mode** in Startup restore settings. [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/latest/getting-started-with-mig.html)
3. Open **Manage GI / CI**, pick a GI profile and placement, then select **Create GI and default CI**. To adopt an existing GI, check its restore option. The add-on manages the default host CI that spans the GI.
4. Select **Enable VFs** if needed. Choose an available VF, a vGPU profile, and a target GI, then select **Set MIG-backed vGPU**. To adopt an existing vGPU, check its restore option.
5. Attach the VF to a stopped test VM. Start the VM and verify it with `nvidia-smi vgpu -q` on the host and `nvidia-smi` in the guest. For a 1:1 Q-series test, use `1g.24gb+gfx` with `DC-1-24Q` on the 6000, or `1g.16gb+gfx` with `DC-1-16Q` on the 4500. [NVIDIA vGPU 21.0 configuration guide](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-user-guide/installing-configuring-grid-vgpu.html) · [vGPU profiles](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-user-guide/virtual-gpu-types-grid-reference.html)

To return to time-sliced mode, stop the VMs, detach their VFs, clear the vGPU types, delete the GIs and CIs, then select **Equal-Size** or **Mixed-Size** and apply **Change mode**. Record the original profile and VM assignments if you plan to recreate them.

## 3. Verify saved configuration and boot-time restore

**Startup restore settings** shows the saved mode, GIs and default CIs, VFs, and vGPU types. Changing a checkbox updates what is restored at the next boot and leaves the running configuration in place. The add-on saves these settings to `/var/lib/vgpu-web/gpu-state.json`. In MIG mode it recreates the GIs and default CIs and maps each vGPU to the new GI ID. libvirt stores VM device assignments and `managed=no` in the persistent VM definition. [NVIDIA RHEL KVM deployment guide](https://docs.nvidia.com/vgpu/deployment/rhel/latest/vgpu-config.html) · [libvirt device management](https://libvirt.org/manpages/virsh.html#attach-device)

```bash
systemctl is-enabled vgpu-web-restore.service
sudo cat /var/lib/vgpu-web/gpu-state.json
sudo journalctl -t vgpu-web-cockpit -o cat
```

Check that the restore service is `enabled`. After rebooting the host, compare the configuration shown in **vGPU Management** with the restore log. If you assigned a restored type to a test VM, verify that the guest OS detects the vGPU.

```bash
sudo journalctl -b -u vgpu-web-restore.service --no-pager
```

## If the menu entry is missing

Run `cockpit-bridge --packages` on the host and check for `nvidia-vgpu` in the output. Sign out of Cockpit and sign in again. [Cockpit package discovery](https://docs.cockpit-project.org/cockpit-guide/364/guide/packages.html#layout-of-package-files)
