# RHEL KVM の Cockpit に「vGPU 管理」を追加する

新規導入と更新には `vgpu-web-cockpit-addon-rhel-kvm.zip` を使用します。追加するのは Cockpit の画面、GPU 構成用ヘルパー、ホスト起動時の復元サービスです。接続先、HTTPS 証明書、ログインは対象ホストの Cockpit を使用します。 [Cockpit のパッケージ仕様](https://docs.cockpit-project.org/cockpit-guide/364/guide/packages.html)

## 前提

- RHEL KVM ホストの Cockpit にログインでき、構成を変更するユーザーは Cockpit の「管理者アクセス」を利用できること。
- ホストに Python 3、libvirt（`qemu:///system`）、稼働中の NVIDIA vGPU Manager とドライバーがあること。
- 対象は NVIDIA vendor-specific VFIO の SR-IOV VF を使う **Equal-Size**、対応 GPU での **Mixed-Size**、RTX PRO 6000 / 4500 Blackwell Server Edition での **MIG** です。RHEL 10.2 KVM の場合は、対応する NVIDIA vGPU Manager を使用してください。 [NVIDIA vGPU 21.0 の RHEL KVM サポート表](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-release-notes-red-hat-el-kvm/validated-platforms.html)

## 1. 導入

ZIP を対象ホストの `~/Downloads` に転送します。sudo を使える一般ユーザーでホストにログインし、次を実行します。

```bash
vgpu_root="$HOME/Downloads/vgpu-web-cockpit-addon-rhel-kvm"
python3 -m zipfile -e "$vgpu_root.zip" "$vgpu_root" &&
bash "$vgpu_root/vgpu-web/deploy/install-cockpit.sh"
```

更新時も上記の手順で新しい ZIP を再展開し、インストーラーを実行します。保存済みの復元設定は引き継がれます。初回導入で既存構成を復元対象に取り込む場合は、ホストの構成を確認し、インストーラーに `--adopt-current` を付けて実行します。

## 2. 接続と操作

1. ブラウザーで対象ホストの Cockpit を開き、ログインします。既存のセッションを開いていた場合はログインし直します。
2. 左メニューの **vGPU 管理** を選択します。物理 GPU を切り替え、VF、プロファイル、VM の参照を確認します。
3. 変更するときは Cockpit 上部の **管理者アクセス** を有効にします。操作対象は空き VF と停止中の VM を選びます。VF の有効化、タイプ設定・クリア、`managed=no` での割り当て、VF 単位の一括解除を実行できます。 [Cockpit の権限付きプロセス実行](https://docs.cockpit-project.org/cockpit-guide/368/guide/cockpit-spawn.html)

### Equal-Size / Mixed-Size / MIG

構成方式の名称は **Equal-Size / Mixed-Size / MIG** です。L40S など MIG 非対応 GPU では MIG の選択肢を表示しません。現在の構成方式を確認して希望する方式を選択し、**構成方式を変更**で適用します。選択だけではホストを変更しません。対象 GPU/VF の VM 参照・vGPU タイプ・復元対象、MIG の GI が残っている場合は解除が必要です。

適用待ちの間は GI/vGPU の新規作成・VM への新規割り当てを表示せず、現在の構成の解除操作だけを表示します。プロファイル一覧は実際の GPU モードで絞り込み、Equal-Size / Mixed-Size では `DC-2Q` など、MIG では `DC-1-2Q` などを表示します。モードに合わないプロファイルの設定はホスト側でも拒否します。

Mixed-Size はドライバーが対応を報告する場合に選択できます。適用後は VF ごとに異なるサイズのプロファイルを設定し、停止中の VM に割り当てます。配置はドライバーが自動決定します。起動時は VF → サイズモード → vGPU の順で復元します。GI 内の Mixed-Size 切り替えは今回の対象外です。[NVIDIA Mixed-Size 構成手順](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-user-guide/installing-configuring-grid-vgpu.html#putting-a-gpu-into-mixed-size-mode-on-a-linux-with-kvm-hypervisor)

### MIG-backed vGPU を作成する

GI/CI を表示・構成するには、Cockpit 上部の **管理者アクセス** を有効にして **再取得** を押します。既存 GI は `+gfx` なしも表示し、配置を占有する非グラフィックス GI を確認・削除できます。新規作成と vGPU の配置は `+gfx` が対象です。[NVIDIA SMI の MIG 管理コマンド](https://docs.nvidia.com/deploy/nvidia-smi/index.html#mig-management)

GI がまだないときは **GI / CI の管理**が自動で開きます。まず GI と既定 CI を作成してから、VF のプロファイルを設定します。CLI で MIG を有効にした状態でも GI を作成でき、作成時に MIG モードと GI を復元対象に登録します。

1. 対象 GPU の VM を停止し、VF の割り当てを解除して既存の vGPU タイプをクリアします。その GPU を使用するホスト上の処理も停止します。
2. 構成方式で **MIG** を選び、**構成方式を変更**で適用します。Blackwell ではホストの再起動は不要です。現在と保留の両方が `enabled` になったことを確認します。CLI ですでに有効化済みなら復元設定の **MIG モードを復元**にチェックします。[NVIDIA MIG ユーザーガイド](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/latest/getting-started-with-mig.html)
3. **GI / CI の管理**を開き、GI プロファイルと配置を選んで **GI と既定 CI を作成**します。既存 GI を取り込む場合は復元設定の該当 GI にチェックします。ホスト側の CI は、GI 全体を使う既定 CI を管理します。
4. VF が表示されていなければ、**VF を有効化**します。空き VF で vGPU プロファイルと GI を選び、**MIG vGPU を設定**します。既存 vGPU を取り込む場合は復元設定の該当 vGPU にチェックします。
5. VF を停止中の検証 VM に割り当て、VM を起動します。ホストの `nvidia-smi vgpu -q` とゲストの `nvidia-smi` で認識を確認します。1:1 の Q 系の例は、6000 が `1g.24gb+gfx` と `DC-1-24Q`、4500 が `1g.16gb+gfx` と `DC-1-16Q` の組み合わせです。 [NVIDIA vGPU 21.0 構成ガイド](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-user-guide/installing-configuring-grid-vgpu.html) · [対応プロファイル表](https://docs.nvidia.com/vgpu/21.0/grid-vgpu-user-guide/virtual-gpu-types-grid-reference.html)

元の GPU モードへ戻すには、VM を停止して割り当てを解除し、vGPU タイプをクリアしてから GI/CI を削除し、**Equal-Size** または **Mixed-Size** を選んで **構成方式を変更**を適用します。以前の構成を再設定する場合は、切り替え前にプロファイルと VM の対応を控えてください。

## 3. 保存内容と起動時復元を確認する

**再起動後の復元設定**で保存済みの構成方式、GI/CI、VF、vGPU タイプを確認できます。チェックの変更は次回起動時の復元に反映され、稼働中の構成は維持されます。GPU 構成の保存先は `/var/lib/vgpu-web/gpu-state.json` です。MIG では GI と既定 CI を再作成し、新しい GI ID に vGPU を配置します。VM の割り当てと `managed=no` は libvirt が永続定義に保存します。 [NVIDIA RHEL KVM デプロイメントガイド](https://docs.nvidia.com/vgpu/deployment/rhel/latest/vgpu-config.html) · [libvirt の VM デバイス管理](https://libvirt.org/manpages/virsh.html#attach-device)

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
