# NVIDIA vGPU Management for Cockpit on RHEL and Ubuntu KVM

## Downloads / ダウンロード

Choose the package for your KVM host and preferred language. Each ZIP includes its installation guide.

KVM ホストの OS と利用する言語に合わせて ZIP を選択してください。各 ZIP に導入手順を同梱しています。

| KVM host / ホスト | Language / 言語 | Add-on ZIP / 配布パッケージ | Installation guide / 導入手順 |
| --- | --- | --- | --- |
| RHEL KVM | 日本語 | [Download ZIP](dist/vgpu-web-cockpit-addon-rhel-kvm.zip?raw=true) | [導入・更新手順](docs/INSTALL.ja.md) |
| RHEL KVM | English | [Download ZIP](dist/vgpu-web-cockpit-addon-rhel-kvm-en.zip?raw=true) | [Install and upgrade](docs/INSTALL.en.md) |
| Ubuntu 26.04 LTS KVM | 日本語 | [Download ZIP](dist/vgpu-web-cockpit-addon-ubuntu-26.04-ja.zip?raw=true) | [導入・更新手順](docs/INSTALL.ubuntu-26.04.ja.md) |
| Ubuntu 26.04 LTS KVM | English | [Download ZIP](dist/vgpu-web-cockpit-addon-ubuntu-26.04-en.zip?raw=true) | [Install and upgrade](docs/INSTALL.ubuntu-26.04.en.md) |

## Overview / 概要

This project adds a **vGPU Management** page to Cockpit on **RHEL KVM** and **Ubuntu 26.04 LTS KVM**. It uses NVIDIA vendor-specific VFIO to inspect physical GPUs and SR-IOV VFs, set vGPU profiles, and assign VFs to stopped VMs. The add-on writes `managed='no'` to the VM's libvirt definition and restores the saved host-side configuration at startup, including re-enabling the VFs. [Cockpit setup](https://cockpit-project.org/running.html) · [NVIDIA vGPU 21.0 configuration guide](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-user-guide/installing-configuring-grid-vgpu.html)

このプロジェクトは、**RHEL KVM** と **Ubuntu 26.04 LTS KVM** の Cockpit に **vGPU 管理**ページを追加します。NVIDIA vendor-specific VFIO を使用し、物理 GPU と SR-IOV VF の確認、vGPU プロファイル設定、停止中の VM への割り当てをブラウザーから操作できます。割り当て時は libvirt の VM 定義に `managed='no'` を設定し、ホスト起動時には VF を再有効化して保存済みの構成を復元します。 [Cockpit の設定](https://cockpit-project.org/running.html) · [NVIDIA vGPU 21.0 構成ガイド](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-user-guide/installing-configuring-grid-vgpu.html)

## Supported vGPU modes

| Mode | What the add-on manages |
| --- | --- |
| **Equal-Size** | Time-sliced vGPUs with the same framebuffer size on a physical GPU. |
| **Mixed-Size** | Time-sliced vGPUs with different framebuffer sizes on a capable GPU, such as L40S. Cockpit shows the profiles the driver reports as available for the current configuration. |
| **MIG** | Graphics-capable GPU instances (`+gfx`) and their default compute instances on RTX PRO 6000 / 4500 Blackwell Server Edition. Create a GI and CI, then configure a MIG-backed vGPU on a VF and select the parent GI. |

The add-on configures SR-IOV VFs using NVIDIA vendor-specific VFIO. It hides the MIG option on GPUs such as L40S. Mixed-Size in this add-on applies to the physical GPU with MIG disabled. Check NVIDIA's platform-specific matrices for the GPU, host release, and mode combination. On Ubuntu 26.04, the vGPU 21.0 Mixed-Size table omits that host release for RTX PRO 6000 / 4500 Blackwell Server Edition and L40S. [RHEL KVM validated platforms](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-release-notes-red-hat-el-kvm/validated-platforms.html) · [Ubuntu validated platforms](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-release-notes-ubuntu/validated-platforms.html)

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

構成対象は NVIDIA vendor-specific VFIO の SR-IOV VF です。L40S などでは MIG の選択肢を非表示にします。アドオンでの Mixed-Size 切り替えは、MIG 無効時の物理 GPU が対象です。GPU・ホスト OS・構成方式の組み合わせは、プラットフォーム別の NVIDIA 対応表で確認してください。vGPU 21.0 の Ubuntu 向け Mixed-Size 表では、RTX PRO 6000 / 4500 Blackwell Server Edition と L40S の対象 OS に Ubuntu 26.04 の記載がありません。 [RHEL KVM 対応表](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-release-notes-red-hat-el-kvm/validated-platforms.html) · [Ubuntu 対応表](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-release-notes-ubuntu/validated-platforms.html)

## 対応機能

- 物理 GPU ごとに現在の構成方式、PCI アドレス、SR-IOV VF、作成可能なプロファイル、VM 参照を確認できます。接続には既設 Cockpit のログインと TLS 設定を使用します。
- **管理者アクセス**でモード変更、VF の有効化、vGPU タイプの設定・クリア、停止中の永続 VM への `managed=no` での割り当てができます。対象の PCI `hostdev` 定義の更新や、VF を参照する停止中 VM からの一括解除にも対応します。
- MIG のときは GI/CI を作成・削除でき、VF の親 GI を確認できます。配置を占有する `+gfx` なしの既存 GI も表示します。
- **再起動後の復元設定**でモード、GI 配置、VF、vGPU タイプを管理できます。チェックの変更は次回起動時の復元に反映され、VM の割り当ては libvirt の永続定義に保存されます。
- Cockpit で警告と診断 JSON を確認できます。操作、対象、結果は `vgpu-web-cockpit` としてシステムジャーナルに記録されます。

## Install or upgrade / 導入と更新

Prepare Cockpit, Python 3, libvirt, and a compatible NVIDIA vGPU Manager on the KVM host. Copy and extract the ZIP, then run the bundled installer as a regular user with sudo access. The installer is `vgpu-web/deploy/install-cockpit.sh` on RHEL and `vgpu-web/deploy/install-cockpit-ubuntu.sh` on Ubuntu. Sign in to Cockpit again after installation. For upgrades, rerun the installer from the updated ZIP to carry forward the saved restore settings. See the installation guide in the download table for the selected host.

KVM ホストに Cockpit、Python 3、libvirt、対応する NVIDIA vGPU Manager を用意します。ZIP を展開し、sudo を使える一般ユーザーで、RHEL は `vgpu-web/deploy/install-cockpit.sh`、Ubuntu は `vgpu-web/deploy/install-cockpit-ubuntu.sh` を実行します。導入後、Cockpit にログインし直します。更新時は新しい ZIP から同じインストーラーを実行すると、保存済みの復元設定を引き継ぎます。詳細はダウンロード表の各導入手順を参照してください。

## Screenshots / 画面例
<img width="1575" height="999" alt="MIG sample" src="https://github.com/user-attachments/assets/99117ec9-f3cb-45bd-9353-6a315e8f6877" />
<img width="1694" height="929" alt="Mixed-Size sample" src="https://github.com/user-attachments/assets/623bce01-ddd9-4200-b160-4780ff2946d1" />
<img width="1719" height="915" alt="Equal-Size sample" src="https://github.com/user-attachments/assets/f40aec29-8ede-4006-b7de-78d54c61ff72" />

