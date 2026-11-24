# Ubuntu 26.04 LTS KVM の Cockpit に「vGPU 管理」を追加する

物理 GPU（PF）と SR-IOV 仮想機能（VF）の確認、vGPU プロファイルの設定、停止中の VM への割り当て、ホスト再起動後の復元設定を Cockpit から操作できます。VM への割り当てには NVIDIA の手順に沿って VF の PCI BDF と `managed=no` を使用します。対象は **NVIDIA vendor-specific VFIO を使用する SR-IOV 対応 GPU** です。 [NVIDIA vGPU 21.0 構成ガイド](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-user-guide/installing-configuring-grid-vgpu.html#adding-one-or-more-vgpus-to-a-linux-with-kvm-hypervisor-vm-by-using-virsh)

## 1. Ubuntu ホストを準備する

- Ubuntu 26.04 LTS の KVM ホストに、Python 3、libvirt のシステム接続（`qemu:///system`）、sudo を使えるユーザーを用意します。KVM/libvirt の新規導入は Ubuntu の手順に従います。 [Ubuntu の libvirt 導入手順](https://ubuntu.com/server/docs/how-to/virtualisation/libvirt/#install-libvirt)
- Ubuntu 26.04 LTS に対応する **NVIDIA vGPU Manager の Ubuntu 用 `.deb`** を NVIDIA から入手し、公式ガイドに沿って導入します。VF の vGPU 設定にはこのホスト用パッケージが必要です。 [NVIDIA の Ubuntu 対応表](https://docs.nvidia.com/vgpu/latest/product-support-matrix/ubuntu.html) · [NVIDIA vGPU Manager の導入手順](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-user-guide/installing-configuring-grid-vgpu.html#installing-the-virtual-gpu-manager-package-for-ubuntu)
- Cockpit の導入が済んでいない場合は、次を実行します。`cockpit-machines` は VM の作成や起動・停止を Cockpit から行う場合に使用します。 [Ubuntu の Cockpit パッケージ](https://packages.ubuntu.com/resolute/cockpit) · [Ubuntu の cockpit-machines パッケージ](https://packages.ubuntu.com/eu/resolute/admin/cockpit-machines) · [Cockpit の起動手順](https://cockpit-project.org/guide/latest/startup.html)

```bash
sudo apt update
sudo apt install cockpit cockpit-machines
sudo systemctl enable --now cockpit.socket
```

次のコマンドで、ホストの NVIDIA GPU、vGPU Manager の稼働、libvirt への接続を確認します。Ubuntu 26.04 で `nova_core` が GPU にバインドされて vGPU Manager が起動しない場合は、NVIDIA の回避手順を参照してください。 [Ubuntu の GPU 仮想化ガイド](https://ubuntu.com/server/docs/gpu-virtualization-with-qemu-kvm/) · [NVIDIA の NovaCore 既知問題](https://docs.nvidia.com/vgpu/21.0/known-issues/bug-6103562-nova-core-prevents-vgpu-manager-service-from-starting.html) · [libvirt の virsh リファレンス](https://libvirt.org/manpages/virsh.html#list)

```bash
nvidia-smi -L
systemctl is-active nvidia-vgpu-mgr.service
sudo virsh --connect qemu:///system list --all
```

## 2. アドオンを導入する

`vgpu-web-cockpit-addon-ubuntu-26.04-ja.zip` を Ubuntu ホストの `~/Downloads` に転送します。sudo を使える一般ユーザーで実行します。

```bash
vgpu_root="$HOME/Downloads/vgpu-web-cockpit-addon-ubuntu-26.04-ja"
python3 -m zipfile -e "$vgpu_root.zip" "$vgpu_root" &&
bash "$vgpu_root/vgpu-web/deploy/install-cockpit-ubuntu.sh"
```

インストーラーは Ubuntu のリリースと必要なコマンドを確認し、Cockpit のメニュー、GPU 操作用ヘルパー、起動時の復元サービスを登録します。初回の復元設定は空で作成します。既存の GPU 構成を初回から復元対象に含める場合は、構成を確認してからインストーラーに `--adopt-current` を指定します。更新時は新しい ZIP を展開して同じ手順を実行すると、保存済みの復元設定が引き継がれます。

## 3. Cockpit で vGPU を設定する

1. ブラウザーで Ubuntu ホストの Cockpit（標準ポートは HTTPS の `9090`）を開き、ログインします。既存の Cockpit セッションはログインし直します。 [Cockpit の接続と起動](https://cockpit-project.org/guide/latest/startup.html)
2. 左メニューの **vGPU 管理**を開きます。構成を変更する場合は上部の**管理者アクセス**を有効にして **再取得**を押します。
3. 対象の物理 GPU を選び、構成方式を確認します。方式を切り替える前に、対象 GPU を使う VM を停止し、VF の割り当てと vGPU タイプを解除します。MIG から切り替える場合は既存 GI/CI も削除します。
4. VF が未作成なら **VF を有効化**します。空き VF と vGPU プロファイルを選んで設定し、停止中の VM に割り当てます。割り当ては libvirt の永続定義に反映されます。

| 構成方式 | Cockpit での操作 |
| --- | --- |
| **Equal-Size** | 同じフレームバッファーサイズの vGPU を同一 GPU 上に構成します。 |
| **Mixed-Size** | ドライバーが対応を報告する場合に選択できます。異なるフレームバッファーサイズの vGPU を同一 GPU 上に構成します。 |
| **MIG** | 対応 GPU で表示されます。MIG が有効になったら、**GI / CI の管理**で `+gfx` の GI と既定 CI を作成し、VF に vGPU プロファイルと配置先 GI を指定します。 |

**公式資料で確認済み**。NVIDIA vGPU 21.0 の Ubuntu リリースノートに Ubuntu 26.04 LTS が掲載されています。RTX PRO 6000 Blackwell Server Edition、RTX PRO 4500 Blackwell Server Edition、L40S の **Mixed-Size 欄には 26.04 の記載がなく**、これらの GPU での Mixed-Size の公式サポートは同表からは確認できません。Ubuntu の vendor-specific VFIO で作成した SR-IOV vGPU のコンソール VNC は NVIDIA のサポート対象外です。 [NVIDIA の Ubuntu 検証済みプラットフォーム一覧](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-release-notes-ubuntu/validated-platforms.html) · [NVIDIA の VM 割り当て手順](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-user-guide/installing-configuring-grid-vgpu.html#adding-one-or-more-vgpus-to-a-linux-with-kvm-hypervisor-vm-by-using-virsh)

## 4. 再起動後の復元を確認する

**再起動後の復元設定**で構成方式、GI と既定 CI、VF、vGPU タイプの保存状況を確認します。画面から有効化・作成した構成の復元状況もここに表示されます。CLI で構成した項目を取り込む場合は、該当するチェックボックスを選びます。チェックの変更は次回起動時の復元対象に反映されます。

設定は `/var/lib/vgpu-web/gpu-state.json` に保存されます。インストーラーは利用中の libvirt サービスに合わせて起動順を設定し、vGPU の復元処理が libvirt による VM の自動起動に先行するようにします。ホストを再起動したら、Cockpit 上の GPU 構成と以下のログで復元結果を確認します。

```bash
systemctl is-enabled vgpu-web-restore.service
sudo journalctl -b -u vgpu-web-restore.service --no-pager
```

左メニューに **vGPU 管理**が見当たらない場合は、`cockpit-bridge --packages` で `nvidia-vgpu` が表示されることを確認し、ログインし直します。 [Cockpit のパッケージ仕様](https://docs.cockpit-project.org/cockpit-guide/364/guide/packages.html)
