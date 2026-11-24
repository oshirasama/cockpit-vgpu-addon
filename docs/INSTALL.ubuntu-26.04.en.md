# Add vGPU Management to Cockpit on Ubuntu 26.04 LTS KVM

Use this add-on to inspect physical GPUs (PFs) and SR-IOV virtual functions (VFs), configure vGPU profiles, assign VFs to stopped VMs, and select which settings to restore at host startup. The add-on is designed for SR-IOV-capable GPUs using NVIDIA vendor-specific VFIO. VM assignments use the VF's PCI BDF and `managed=no`. [NVIDIA vGPU 21.0 configuration guide](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-user-guide/installing-configuring-grid-vgpu.html#adding-one-or-more-vgpus-to-a-linux-with-kvm-hypervisor-vm-by-using-virsh)

## 1. Prepare the Ubuntu host

- Set up Python 3 and libvirt's system connection (`qemu:///system`) on the Ubuntu 26.04 LTS KVM host. Use a regular account with sudo access. [Ubuntu libvirt installation](https://ubuntu.com/server/docs/how-to/virtualisation/libvirt/#install-libvirt)
- Obtain the **NVIDIA vGPU Manager `.deb` for Ubuntu** that matches your host release, and install it using NVIDIA's instructions. [NVIDIA Ubuntu support matrix](https://docs.nvidia.com/vgpu/21.0/product-support-matrix/ubuntu.html) · [NVIDIA vGPU Manager installation](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-user-guide/installing-configuring-grid-vgpu.html#installing-the-virtual-gpu-manager-package-for-ubuntu)
- If Cockpit is missing, install it with the commands below. The `cockpit-machines` package provides the VM creation, start, and stop controls. [Ubuntu Cockpit package](https://packages.ubuntu.com/resolute/cockpit) · [Ubuntu cockpit-machines package](https://packages.ubuntu.com/eu/resolute/admin/cockpit-machines) · [Cockpit startup](https://cockpit-project.org/guide/latest/startup.html)

```bash
sudo apt update
sudo apt install cockpit cockpit-machines
sudo systemctl enable --now cockpit.socket
```

Verify that the host detects the NVIDIA GPUs, the vGPU Manager service is active, and libvirt can list VMs. If `nova_core` has bound to the GPU and prevents vGPU Manager from starting, follow NVIDIA's workaround. [NVIDIA NovaCore known issue](https://docs.nvidia.com/vgpu/21.0/known-issues/bug-6103562-nova-core-prevents-vgpu-manager-service-from-starting.html) · [libvirt virsh reference](https://libvirt.org/manpages/virsh.html#list)

```bash
nvidia-smi -L
systemctl is-active nvidia-vgpu-mgr.service
sudo virsh --connect qemu:///system list --all
```

## 2. Install the add-on

Copy `vgpu-web-cockpit-addon-ubuntu-26.04-en.zip` to `~/Downloads` on the Ubuntu host. Run the installer from a regular account with sudo access.

```bash
vgpu_root="$HOME/Downloads/vgpu-web-cockpit-addon-ubuntu-26.04-en"
python3 -m zipfile -e "$vgpu_root.zip" "$vgpu_root" &&
bash "$vgpu_root/vgpu-web/deploy/install-cockpit-ubuntu.sh"
```

The installer checks the Ubuntu release and host prerequisites, then registers the Cockpit page, host configuration helper, and startup restore service. On a first installation, the restore configuration starts empty. To capture the live GPU configuration during installation, verify the current settings and pass `--adopt-current` to the installer. To update an existing installation, extract the new ZIP and rerun the installer. Saved startup restore settings carry over.

## 3. Configure vGPUs in Cockpit

1. Open the Ubuntu host's Cockpit HTTPS URL in a browser. The default port is `9090`. If a Cockpit session is already open, sign out and sign in again. [Cockpit connection instructions](https://cockpit-project.org/running.html)
2. Open **vGPU Management** from the sidebar. To make changes, switch to **Administrative access**, then select **Refresh**.
3. Select the physical GPU and check its configuration mode. Before changing modes, stop VMs using that GPU, detach its VFs, and clear the existing vGPU types. When leaving MIG mode, remove the existing GPU instances (GIs) and compute instances (CIs).
4. Select **Enable VFs** if needed. Choose an available VF and vGPU profile, apply the profile, and assign the VF to a stopped VM. The assignment is saved in the VM's persistent libvirt definition.

| Mode | Configuration |
| --- | --- |
| **Equal-Size** | Configure vGPUs with the same frame buffer size on one GPU. |
| **Mixed-Size** | Configure vGPUs with different frame buffer sizes on one GPU when the driver reports support. |
| **MIG** | On supported GPUs, enable MIG and use **Manage GI / CI** to create a graphics-enabled `+gfx` GI and its default CI. Select the vGPU profile and target GI for the VF. |

NVIDIA lists Ubuntu 26.04 LTS as a validated platform for vGPU 21.0. The Mixed-Size entries for RTX PRO 6000 Blackwell Server Edition, RTX PRO 4500 Blackwell Server Edition, and L40S omit Ubuntu 26.04, so support for that combination is unconfirmed in the published table. Console VNC is unsupported for SR-IOV vGPUs created with vendor-specific VFIO on Ubuntu. [NVIDIA validated Ubuntu platforms](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-release-notes-ubuntu/validated-platforms.html) · [NVIDIA Console VNC limitation](https://docs.nvidia.com/vgpu/21.0/known-issues/bug-3856353-vendor-specific-vfio-vgpus-do-not-support-console-vnc.html)

## 4. Verify startup restore

In **Startup restore settings**, check the saved configuration mode, GIs and default CIs, VFs, and vGPU types. The page also shows settings saved when items were enabled or created through the add-on. To include a configuration created at the command line, select the corresponding restore checkboxes.

The add-on saves settings to `/var/lib/vgpu-web/gpu-state.json`. The installer orders the restore service before libvirt's VM autostart. Verify that the service is enabled. After a host reboot, check the restored GPU configuration in Cockpit and inspect the service log.

```bash
systemctl is-enabled vgpu-web-restore.service
sudo journalctl -b -u vgpu-web-restore.service --no-pager
```

If **vGPU Management** is missing from the sidebar, run `cockpit-bridge --packages` on the host and check for `nvidia-vgpu`. Then sign out of Cockpit and sign in again. [Cockpit package discovery](https://docs.cockpit-project.org/cockpit-guide/364/guide/packages.html)
