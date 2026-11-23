# NVIDIA vGPU Management for Cockpit on RHEL KVM

## Overview / 概要

Configuring NVIDIA vGPU on Red Hat Enterprise Linux (RHEL) with KVM involves several steps across the host and the virtual machine. With NVIDIA vendor-specific VFIO, an administrator typically uses the command line to inspect the GPU’s physical function (PF) and SR-IOV virtual functions (VFs), enable VFs, and configure a vGPU profile on a selected VF. The VF is then attached to a VM using `managed='no'` on the PCI `hostdev` entry in its libvirt definition. Because NVIDIA disables the VFs on host reboot, a startup process must re-enable them and reapply the saved vGPU configuration. [NVIDIA vGPU configuration guide](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-user-guide/installing-configuring-grid-vgpu.html#installing-and-configuring-the-nvidia-virtual-gpu-manager-for-red-hat-enterprise-linux-kvm)

This project adds a **vGPU Management** page to Cockpit, the RHEL web console that Red Hat documents for browser-based VM management. Once the add-on is installed, administrators can inspect GPUs and VFs, configure vGPUs in **Equal-Size**, **Mixed-Size**, or **MIG** mode, and assign them to stopped VMs from the browser. The add-on sets `managed='no'` when attaching a vGPU and restores saved host-side settings at startup. These day-to-day configuration tasks can be done without entering Linux commands or editing VM XML by hand. [Red Hat documentation for VM management in the web console](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/deploy-setting_up_the_web_console_to_manage_virtual_machines)

Red Hat Enterprise Linux（RHEL）の KVM 環境で NVIDIA vGPU を構成するには、ホストと仮想マシン（VM）の両方で設定が必要です。NVIDIA vendor-specific VFIO を使う場合、通常は Linux のコマンドラインで GPU の物理機能（PF）と SR-IOV の仮想機能（VF）の状態を確認し、VF を有効化して vGPU プロファイルを設定します。VM への割り当てでは、libvirt の VM 定義にある PCI デバイス（`hostdev`）に `managed='no'` を指定します。ホストを再起動すると VF は無効になるため、起動時に VF を再有効化し、保存した vGPU 構成を適用する仕組みも必要です。 [NVIDIA vGPU 構成ガイド](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-user-guide/installing-configuring-grid-vgpu.html#installing-and-configuring-the-nvidia-virtual-gpu-manager-for-red-hat-enterprise-linux-kvm)

これらの作業を一つの画面で行えるように、Red Hat がブラウザーでの VM 管理手段として案内している RHEL Web コンソール（Cockpit）に **vGPU 管理**を追加しました。アドオンの導入後は、GPU と VF の状態確認、**Equal-Size**、**Mixed-Size**、**MIG** での vGPU 構成、停止中の VM への割り当てをブラウザーで行えます。割り当て時の `managed='no'` の指定と、ホスト起動時の構成復元はアドオンが行うため、日常の設定作業で Linux コマンドを入力したり、VM の XML を手作業で編集したりする必要がありません。 [Red Hat の Web コンソールによる VM 管理手順](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/deploy-setting_up_the_web_console_to_manage_virtual_machines)

| Language | Installation guide | Add-on ZIP |
| --- | --- | --- |
| 日本語 | [導入・更新手順](docs/INSTALL.ja.md) | [日本語版 ZIP](dist/vgpu-web-cockpit-addon-rhel-kvm.zip) |
| English | [Install and upgrade](docs/INSTALL.en.md) | [English ZIP](dist/vgpu-web-cockpit-addon-rhel-kvm-en.zip) |

## Supported vGPU modes

| Mode | What the add-on manages |
| --- | --- |
| **Equal-Size** | Time-sliced vGPUs with the same framebuffer size on a physical GPU. |
| **Mixed-Size** | Time-sliced vGPUs with different framebuffer sizes on a capable GPU, such as L40S. Cockpit shows the profiles the driver reports as available for the current configuration. |
| **MIG** | Graphics-capable GPU instances (`+gfx`) and their default compute instances on RTX PRO 6000 / 4500 Blackwell Server Edition. Create a GI and CI, then configure a MIG-backed vGPU on a VF and select the parent GI. |

The add-on configures SR-IOV VFs using NVIDIA vendor-specific VFIO. It hides the MIG option on GPUs such as L40S. Mixed-Size in this add-on applies to the physical GPU with MIG disabled. The NVIDIA support matrix lists eligible GPU, driver, and RHEL KVM combinations. [NVIDIA vGPU 21.0 RHEL KVM support matrix](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-release-notes-red-hat-el-kvm/validated-platforms.html)

## Features

- Select a physical GPU to inspect its active mode, PCI addresses, SR-IOV VFs, available profiles, and VM references. Cockpit uses the host's existing sign-in and TLS configuration.
- Use **Administrative access** to apply a mode change, enable VFs, configure or clear a vGPU type, and attach to a stopped persistent VM with `managed=no`. You can update eligible PCI `hostdev` entries and detach a VF from all referencing stopped persistent VMs.
- In MIG mode, view existing GIs and CIs, including GIs without `+gfx` that occupy candidate placements. Create or delete GIs and CIs and see the parent GI for each configured VF.
- Use **Startup restore settings** to manage saved modes, GI layouts, VFs, and vGPU types. Checkbox changes apply at the next host boot. libvirt stores VM assignments in the persistent VM definition.
- Inspect warnings and diagnostic JSON in Cockpit. The system journal records the operation, target, and result under `vgpu-web-cockpit`.

## 対応する vGPU モード

| モード | このアドオンでできること |
| --- | --- |
| **Equal-Size** | 物理 GPU 上に、同じフレームバッファ容量のタイムスライス vGPU を構成します。 |
| **Mixed-Size** | L40S などの対応 GPU 上に、異なるフレームバッファ容量のタイムスライス vGPU を混在させます。ドライバーが現在の構成で作成可能と報告したプロファイルから選択します。 |
| **MIG** | RTX PRO 6000 / 4500 Blackwell Server Edition で、グラフィックス対応の GI（`+gfx`）と既定 CI を作成し、VF 上の MIG-backed vGPU を親 GI に配置します。 |

構成対象は NVIDIA vendor-specific VFIO の SR-IOV VF です。L40S などでは MIG の選択肢を非表示にします。アドオンでの Mixed-Size 切り替えは、MIG 無効時の物理 GPU が対象です。GPU・ドライバー・RHEL KVM の組み合わせは NVIDIA のサポート表で確認してください。[NVIDIA vGPU 21.0 RHEL KVM 対応表](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-release-notes-red-hat-el-kvm/validated-platforms.html)

## 対応機能

- 物理 GPU ごとに現在の構成方式、PCI アドレス、SR-IOV VF、作成可能なプロファイル、VM 参照を確認できます。接続には既設 Cockpit のログインと TLS 設定を使用します。
- **管理者アクセス**でモード変更、VF の有効化、vGPU タイプの設定・クリア、停止中の永続 VM への `managed=no` での割り当てができます。対象の PCI `hostdev` 定義の更新や、VF を参照する停止中 VM からの一括解除にも対応します。
- MIG のときは GI/CI を作成・削除でき、VF の親 GI を確認できます。配置を占有する `+gfx` なしの既存 GI も表示します。
- **再起動後の復元設定**でモード、GI 配置、VF、vGPU タイプを管理できます。チェックの変更は次回起動時の復元に反映され、VM の割り当ては libvirt の永続定義に保存されます。
- Cockpit で警告と診断 JSON を確認できます。操作、対象、結果は `vgpu-web-cockpit` としてシステムジャーナルに記録されます。

## Install or upgrade / 導入と更新

The host needs Cockpit, Python 3, libvirt, and a compatible NVIDIA vGPU Manager. Copy the ZIP to the host, extract it, run the bundled `install-cockpit.sh` as a regular user with `sudo` access, and sign back in to Cockpit. Rerun the installer from the updated ZIP to upgrade and keep the saved restore configuration. [English installation steps](docs/INSTALL.en.md)

ホストに Cockpit、Python 3、libvirt、対応する NVIDIA vGPU Manager を用意します。ZIP をホストへ転送・展開し、sudo を使える一般ユーザーで同梱の `install-cockpit.sh` を実行してから、Cockpit にログインし直します。更新時も新しい ZIP で同じ手順を実行すると、既存の復元設定を引き継ぎます。[日本語の導入手順](docs/INSTALL.ja.md)

## Screenshots / 画面例

These screenshots show an earlier version of the inventory. 画像は以前の版の画面例です。
<img width="1575" height="999" alt="MIG sample" src="https://github.com/user-attachments/assets/99117ec9-f3cb-45bd-9353-6a315e8f6877" />
<img width="1694" height="929" alt="Mixed-Size sample" src="https://github.com/user-attachments/assets/623bce01-ddd9-4200-b160-4780ff2946d1" />
<img width="1719" height="915" alt="Equal-Size sample" src="https://github.com/user-attachments/assets/f40aec29-8ede-4006-b7de-78d54c61ff72" />

