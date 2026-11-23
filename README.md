<img width="1755" height="896" alt="sample2" src="https://github.com/user-attachments/assets/7537aa70-2de9-48ab-87a8-bb4ba8995d3e" />
# NVIDIA vGPU Management for Cockpit on RHEL KVM

Add a vGPU Management page and restore saved SR-IOV VFs and vGPU types at host boot. The add-on uses the host's existing Cockpit sign-in and TLS configuration.

| Language | Installation guide | Add-on ZIP |
| --- | --- | --- |
| 日本語 | [導入手順](docs/INSTALL.ja.md) | [日本語版 ZIP](dist/vgpu-web-cockpit-addon-rhel-kvm.zip) |
| English | [Installation guide](docs/INSTALL.en.md) | [English ZIP](dist/vgpu-web-cockpit-addon-rhel-kvm-en.zip) |

The host needs Cockpit, libvirt, NVIDIA vGPU Manager, and the NVIDIA driver. See the installation guides for the supported vGPU configuration and the reboot verification procedure.

既設の Cockpit、libvirt、NVIDIA vGPU Manager、NVIDIA ドライバーを使います。対応する vGPU 構成と再起動後の確認方法は、導入手順書を参照してください。

## Supported features

- **Cockpit access** — Browse the inventory in read-only mode. Switch to Administrative access to change the configuration.
- **Per-GPU inventory** — Select a physical GPU to view its PCI address, SR-IOV VFs, current and available vGPU profiles, and VM references.
- **SR-IOV VF enablement** — Enable VFs on the selected physical GPU.
- **vGPU type configuration** — Apply an available profile to a VF or clear the configured vGPU type.
- **VM device assignments** — Attach a vGPU to a stopped persistent VM with `managed=no`, update eligible PCI `hostdev` entries to `managed=no`, and detach a VF from all referencing stopped persistent VMs in one operation.
- **Boot-time restore** — Save VF enablement and vGPU types, then restore them at host startup. Use `--adopt-current` on first installation to save an already active configuration.
- **Diagnostics and operation history** — View inventory warnings and diagnostic JSON in Cockpit. The system journal records the operation, target, and result under `vgpu-web-cockpit`.

Configuration and restore cover time-sliced vGPUs in equal-size mode on non-MIG GPUs using NVIDIA vendor-specific VFIO with SR-IOV VFs. See the [English installation guide](docs/INSTALL.en.md) for prerequisites.

## 対応機能

- **Cockpit の権限管理** — 読み取り専用で構成を確認し、変更時は「管理者アクセス」を使います。
- **GPU ごとの構成表示** — 物理 GPU を切り替え、PCI アドレス、SR-IOV VF、設定済みと利用可能な vGPU プロファイル、VM からの参照を確認できます。
- **SR-IOV VF の有効化** — 選択した物理 GPU で VF を有効化します。
- **vGPU タイプの設定** — VF に利用可能なプロファイルを設定し、設定済みのタイプをクリアできます。
- **VM への割り当てと解除** — 停止中の永続 VM へ `managed=no` で割り当て、対象の PCI `hostdev` 定義を `managed=no` に更新できます。選択した VF を参照する停止中の永続 VM から一括解除できます。
- **構成の保存と起動時復元** — VF の有効化と vGPU タイプを保存し、ホスト起動時に復元します。初回導入時の `--adopt-current` で稼働中の構成も保存できます。
- **診断と操作履歴** — Cockpit で取得時の警告と診断 JSON を確認できます。操作、対象、結果は `vgpu-web-cockpit` としてシステムジャーナルに記録します。

構成変更と復元は、NVIDIA vendor-specific VFIO の SR-IOV VF を使用する非 MIG GPU 上の、等サイズモードのタイムスライス vGPU が対象です。前提条件は[日本語の導入手順](docs/INSTALL.ja.md)を参照してください。

Sample Image
<img width="1733" height="907" alt="sample" src="https://github.com/user-attachments/assets/7fc2809a-1d88-4ae7-ac5f-d142e29df1ff" />
<img width="1755" height="896" alt="sample2" src="https://github.com/user-attachments/assets/308cc4b6-abb0-4145-a7a9-02390e3a3914" />
