# RHEL KVM の Cockpit に「vGPU 管理」を追加する

新規ホストには `vgpu-web-cockpit-addon-rhel-kvm.zip` を使用します。追加するのは Cockpit の画面、GPU 構成用ヘルパー、ホスト起動時の復元サービスです。接続先、HTTPS 証明書、ログインは対象ホストの Cockpit を使用します。 [Cockpit のパッケージ仕様](https://docs.cockpit-project.org/cockpit-guide/364/guide/packages.html)

## 前提

- RHEL KVM ホストの Cockpit にログインでき、構成を変更するユーザーは Cockpit の「管理者アクセス」を利用できること。
- ホストに Python 3、libvirt（`qemu:///system`）、稼働中の NVIDIA vGPU Manager とドライバーがあること。
- 対象は NVIDIA vendor-specific VFIO の SR-IOV VF を使う、非 MIG・標準 GPU モードのタイムスライス vGPU です。GPU とドライバーの対応は、利用中のリリースのサポート表で確認できます。 [NVIDIA の RHEL KVM サポート表](https://docs.nvidia.com/vgpu/latest/product-support-matrix/red-hat-el-kvm.html)

## 1. 導入

ZIP を対象ホストの `~/Downloads` に転送します。sudo を使える一般ユーザーでホストにログインし、次を実行します。

```bash
vgpu_root="$HOME/Downloads/vgpu-web-cockpit-addon-rhel-kvm"
python3 -m zipfile -e "$vgpu_root.zip" "$vgpu_root" &&
bash "$vgpu_root/vgpu-web/deploy/install-cockpit.sh"
```

すでに手動で構成した VF と vGPU タイプを初回から復元対象にしたい場合は、現在の構成がホスト上に反映されていることを確認し、上記インストーラーに `--adopt-current` を付けて実行します。通常の初回導入では、以後この画面で設定する構成が保存対象になります。アドオンの保存ファイルがあるホストで再実行すると、その保存内容を引き継ぎます。

## 2. 接続と操作

1. ブラウザーで対象ホストの Cockpit を開き、ログインします。既存のセッションを開いていた場合はログインし直します。
2. 左メニューの **vGPU 管理** を選択します。物理 GPU を切り替え、VF、プロファイル、VM の参照を確認します。
3. 変更するときは Cockpit 上部の **管理者アクセス** を有効にします。操作対象は空き VF と停止中の VM を選びます。VF の有効化、タイプ設定・クリア、`managed=no` での割り当て、VF 単位の一括解除を実行できます。 [Cockpit の権限付きプロセス実行](https://docs.cockpit-project.org/cockpit-guide/368/guide/cockpit-spawn.html)

## 3. 保存内容と起動時復元を確認する

VF とタイプは `/var/lib/vgpu-web/gpu-state.json` に保存されます。タイプをクリアした VF は、次回起動時も VF が有効で、タイプは未設定です。VM の割り当てと `managed=no` は libvirt が永続定義に保存します。 [libvirt の VM デバイス管理](https://libvirt.org/manpages/virsh.html#attach-device)

```bash
systemctl is-enabled vgpu-web-restore.service
sudo cat /var/lib/vgpu-web/gpu-state.json
sudo journalctl -t vgpu-web-cockpit -o cat
```

復元サービスが `enabled` であることを確認します。ホスト再起動後は、Cockpit の vGPU 画面と次のログで復元結果を照合します。保存したタイプを割り当てた検証 VM があれば、ゲスト OS での vGPU 認識も確認します。

```bash
sudo journalctl -b -u vgpu-web-restore.service --no-pager
```

## メニューが表示されない場合

ホスト上で `cockpit-bridge --packages` を実行し、一覧に `nvidia-vgpu` があることを確認して、Cockpit にログインし直します。 [Cockpit のパッケージ確認方法](https://docs.cockpit-project.org/cockpit-guide/364/guide/packages.html#layout-of-package-files)
