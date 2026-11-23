# NVIDIA vGPU Management for Cockpit on RHEL KVM

Manage NVIDIA vGPUs from an existing Cockpit instance on RHEL KVM. Choose **Equal-Size**, **Mixed-Size**, or **MIG**, configure SR-IOV VFs, and restore saved configurations at host startup.

RHEL KVM の Cockpit から NVIDIA vGPU を構成するアドオンです。物理 GPU ごとに **Equal-Size**、**Mixed-Size**、**MIG** を選び、SR-IOV VF の設定、VM への割り当て、起動時の復元を管理できます。

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

<img width="1733" height="907" alt="sample" src="https://github.com/user-attachments/assets/7fc2809a-1d88-4ae7-ac5f-d142e29df1ff" />
<img width="1755" height="896" alt="sample2" src="https://github.com/user-attachments/assets/308cc4b6-abb0-4145-a7a9-02390e3a3914" />
