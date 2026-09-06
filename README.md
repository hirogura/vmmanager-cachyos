# vmmanager-cachyos

libvirt / QEMU 上の仮想マシンを Web ブラウザから管理するための Web UI の **CachyOS / Arch Linux 対応版**です。
上流の [hirogura/vmmanager](https://github.com/hirogura/vmmanager) (Debian/Ubuntu 向け) を CachyOS でもそのままインストールできるようにしたフォークです。

## 概要

- Python (Flask) + libvirt ベースの Web アプリケーション
- systemd サービス (`vm-manage`) として動作
- [Tailscale serve](https://tailscale.com) により HTTPS 化し、**Tailnet 内からのみ**アクセス可能
- アプリは `127.0.0.1:8090` のみで待ち受けるため、**LAN からは直接アクセス不可**

![ロゴ画像](image-ph.png)

## インストール方法 (CachyOS)

インストールスクリプトを GitHub からダウンロードして、root で実行します。

```bash
curl -fsSL -o /tmp/install-vmmanager-cachyos.sh \
  https://raw.githubusercontent.com/hirogura/vmmanager-cachyos/main/install-vmmanager-cachyos.sh
chmod +x /tmp/install-vmmanager-cachyos.sh
sudo /tmp/install-vmmanager-cachyos.sh
```

### インストールスクリプトが行うこと

1. システムパッケージのインストール (`pacman`: python, libvirt, QEMU, edk2-ovmf, swtpm, dnsmasq など)
2. `libvirtd` サービスの有効化 (+ 既定NATネットワーク `default` の自動起動)
3. ストレージプールの設定
   - Btrfs 上では事前に `/opt/vm` をサブボリュームとして作成します
     (snapper の親スナップショットから除外して肥大化を防ぐ + `chattr +C` / `compression none` で COW・圧縮を無効化し qcow2/raw の断片化を防ぐ)
     - 既に通常ディレクトリとして存在する場合: 空なら置き換え、非空なら `/opt/vm.bak.YYYYMMDDHHMMSS` に退避してから作成します
     - ネストしたサブボリュームのため `/etc/fstab` の追記は不要です
   - デフォルトプール `default` を `/opt/vm` に向けます (`/opt/vm` が無ければ作成します)
   - `/iso` ディレクトリが存在する場合は `iso` プールとして追加します
4. Tailscale のインストール (未導入の場合, `pacman -S tailscale`)
5. GitHub リポジトリからアプリ本体を `/opt/vm-manage` に取得
6. Python 仮想環境と Flask のセットアップ (`--system-site-packages` で `libvirt-python` を共有)
7. `websockify` (pip) と `noVNC` (GitHub から `/usr/share/novnc` へ) のセットアップ
8. systemd サービス (`vm-manage.service`) の作成・起動
9. `tailscale serve` でポート `8090` を HTTPS 公開 (Tailnet 内のみ)
   - さらに `/websockify` を VNC コンソール (WebSocket) 用に同じ `8090` 上で公開

### アクセス方法

インストール完了時に表示される URL からアクセスします。

```
https://<マシン名>.<テイルネット名>.ts.net:8090
```

例: `https://myhost.my-tailnet.ts.net:8090`

- アクセスできるのは **同じ Tailnet にログインしている端末のみ** です
- HTTPS 証明書は Tailscale が自動で発行します
- Tailscale に未ログインの場合は、初回実行時に `tailscale up` の認証が必要です
  (表示される URL をブラウザで開いてログインしてください)

## アンインストール方法

サービスとアプリ本体を削除します。

```bash
sudo systemctl stop vm-manage.service
sudo systemctl disable vm-manage.service
sudo rm -f /etc/systemd/system/vm-manage.service
sudo systemctl daemon-reload
sudo rm -rf /opt/vm-manage
```

Tailscale serve の公開設定も削除する場合:

```bash
sudo tailscale serve --https=8090 off
sudo tailscale serve --https=8090 --set-path=/websockify off
```

Tailscale 自体をアンインストールする場合 (CachyOS):

```bash
sudo tailscale logout
sudo pacman -R tailscale
```

※ VM 本体 (libvirt で管理されている仮想マシンやディスク) は削除されません。仮想マシン自体を削除する場合は別途 `virsh` などを使用してください。

## 上流 (Debian/Ubuntu 版) からの変更点

- `apt-get` → `pacman` への置き換え (`install-vmmanager-cachyos.sh` を新設)
  - `qemu-system-x86`, `qemu-desktop`, `qemu-img`, `edk2-ovmf`, `swtpm`, `dnsmasq`, `iptables-nft` 等の Arch パッケージ名に対応
  - `libvirt-python` はシステムパッケージ + venv の `--system-site-packages` で利用
  - `novnc` / `websockify` は Arch 公式リポジトリに無いため、noVNC は GitHub から `/usr/share/novnc` に clone、websockify は venv の pip で導入 (+ `/usr/local/bin` に symlink)
- `app.py` のパス自動解決
  - OVMF: Debian (`/usr/share/OVMF/OVMF_CODE_4M*.fd`) と Arch (`/usr/share/edk2/x64/OVMF_CODE*.4m.fd`) の両方から実在ファイルを検出。無ければ `firmware='efi'` の自動解決に任せる
  - noVNC: `/usr/share/novnc`, `/usr/share/webapps/novnc`, `/opt/vm-manage/novnc` から検出
  - websockify: `PATH` 上の `websockify` → `/usr/local/bin/websockify` → `venv/bin/websockify` の順に検出
  - `<seclabel model='apparmor'>` は `/etc/apparmor.d` が存在する場合のみ付与 (CachyOS では省略し libvirt の自動付与に任せる)
  - ボリュームの `chown` は `libvirt-qemu:kvm` → `libvirt-qemu:libvirt` → `qemu:kvm` → `root:kvm` の順にフォールバック
  - アプリ内アップデート機能の参照先を本リポジトリの `install-vmmanager-cachyos.sh` に変更

## 開発

```bash
cd /opt/vm-manage
python3 -m venv --system-site-packages venv
venv/bin/pip install flask flask-sock simple-websocket websockify
venv/bin/python app.py   # http://127.0.0.1:8090
```

## ライセンス

このプロジェクトは [MIT License](LICENSE) の下で公開されています (上流リポジトリより継承)。
