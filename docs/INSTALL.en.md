# Add vGPU Management to Cockpit on RHEL KVM

Use `vgpu-web-cockpit-addon-rhel-kvm-en.zip` to add the **vGPU Management** page, a host configuration helper, and a boot-time restore service to an existing RHEL KVM host. Use the host's existing Cockpit URL, TLS certificate, and sign-in. [Cockpit package documentation](https://docs.cockpit-project.org/cockpit-guide/364/guide/packages.html)

## Prerequisites

- Cockpit is running on the target host. Users who configure vGPUs can switch to **Administrative access** in Cockpit.
- Python 3, libvirt (`qemu:///system`), NVIDIA vGPU Manager, and the NVIDIA driver are already installed and configured.
- The add-on supports time-sliced vGPUs on non-MIG GPUs in equal-size mode using SR-IOV VFs with NVIDIA vendor-specific VFIO. Check the support matrix for your NVIDIA vGPU release and RHEL KVM combination. [NVIDIA RHEL KVM support matrix](https://docs.nvidia.com/vgpu/latest/product-support-matrix/red-hat-el-kvm.html)

## 1. Install the add-on

Copy the ZIP to `~/Downloads` on the target host. Sign in with a regular user account that can run `sudo`, then run the following commands.

```bash
vgpu_root="$HOME/Downloads/vgpu-web-cockpit-addon-rhel-kvm-en"
python3 -m zipfile -e "$vgpu_root.zip" "$vgpu_root" &&
bash "$vgpu_root/vgpu-web/deploy/install-cockpit.sh"
```

To include VFs and vGPU types that were already configured on the host, check the active GPU configuration and add `--adopt-current` on the first installation. A standard first installation saves configuration as you make changes in the add-on. Reinstalling on a host with an existing state file preserves its saved configuration.

## 2. Open vGPU Management

1. Open the target host's Cockpit URL. If you already have a session open, sign out and sign in again.
2. Select **vGPU Management** in the sidebar. Choose a physical GPU to see its VFs, profiles, and VM references.
3. Switch to **Administrative access** to make changes. Use an available VF and a stopped VM when assigning a vGPU. The page can enable VFs, set or clear vGPU types, attach a vGPU with `managed=no`, and detach a VF from all referencing VMs. [Cockpit process execution and privileges](https://docs.cockpit-project.org/cockpit-guide/368/guide/cockpit-spawn.html)

## 3. Verify saved configuration and boot-time restore

The add-on saves VFs and vGPU types to `/var/lib/vgpu-web/gpu-state.json`. Clearing a vGPU type removes it from the saved configuration. The VF is enabled at the next boot with its type unset. libvirt stores VM device assignments and `managed=no` in the persistent VM definition. [libvirt device management](https://libvirt.org/manpages/virsh.html#attach-device)

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
