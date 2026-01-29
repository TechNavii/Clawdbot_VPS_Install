# VPS 上で Clawdbot を安全にセットアップする手順（推奨）

このガイドは、VPS（仮想専用サーバー）上で Clawdbot Gateway を **外部公開せず**（loopback のみで待受）、必要なときだけ **SSH トンネル**で手元の PC から接続する構成を前提にしています。

> 公式ドキュメントでも、Gateway は loopback（通常 `127.0.0.1`）にバインドし、リモート利用は SSH ポートフォワードで行う構成が説明されています。

## 事前準備

開始前に以下を用意してください。

- VPS（任意のプロバイダで可）
- OS: Ubuntu 24.04 LTS
- サーバーの IP アドレス
- ターミナルアプリ（Mac: Terminal / Windows: PowerShell など）

### プレースホルダの置き換え

- `YOUR_SERVER_IP`: VPS の IP アドレス（例: `143.198.45.123`）

## 0. 推奨セキュリティ方針（要点）

- `root` で常用しない（専用ユーザーで運用）
- SSH は **鍵認証**を前提にし、可能なら **root ログイン/パスワード認証を無効化**
- Gateway は **`--bind loopback`** で待受し、`18789` を **インターネットに公開しない**

以降の手順では、まず初回だけ管理者（root もしくはプロバイダ既定ユーザー）で入り、最低限の初期設定を行います。

## 1. サーバーへ初回接続する（SSH）

プロバイダが案内する初期ユーザーでログインします（例: root）。

```bash
ssh root@YOUR_SERVER_IP
```

## 2. OS を最新化する（推奨）

```bash
apt update
apt upgrade -y
```

（`root` でない場合は `sudo` を付けてください。）

## 3. 専用ユーザーを作成する

以降の運用用に専用ユーザー（例: `clawd`）を作成します。

```bash
adduser clawd
usermod -aG sudo clawd
```

### 3-1. SSH 鍵を `clawd` に設定する（推奨）

手元の PC に公開鍵がある前提で、以下のいずれかで `clawd` に登録します。

- `ssh-copy-id` が使える場合（推奨）

```bash
ssh-copy-id clawd@YOUR_SERVER_IP
```

- 手動で行う場合（公開鍵を貼り付け）
  1) ローカルで `~/.ssh/id_ed25519.pub` などの中身を確認し、
  2) VPS 側で `/home/clawd/.ssh/authorized_keys` に追記します。

## 4. ファイアウォールを有効化する（任意・推奨）

Gateway を loopback バインドで運用する場合、**18789 を開ける必要はありません**。

```bash
ufw allow OpenSSH
ufw enable
ufw status
```

## 5. Clawdbot をインストールする（より安全な実行方法）

`clawd` に切り替えてから作業します。

```bash
su - clawd
```

インストーラーを **ダウンロード → 内容確認 → 実行** の順で行います（`curl | bash` を避けます）。

```bash
curl -fsSL --proto '=https' --tlsv1.2 https://molt.bot/install.sh -o /tmp/moltbot_install.sh
sed -n '1,200p' /tmp/molt_install.sh
bash /tmp/molt_install.sh
```

インストール後に、`clawdbot` が見えることを確認します。

```bash
command -v clawdbot
clawdbot --help
```

もしコマンドが見えなかったら以下の手順を実施します。

```bash
echo 'export PATH="/home/clawd/.npm-global/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

## 6. 初期セットアップ（オンボーディング）

インストーラーの案内に従ってオンボーディング/初期設定を完了してください。

補足: 公式ドキュメント上、Gateway は既定で `gateway.mode=local` が設定されていないと起動を拒否する場合があります（初期設定で書き込まれる想定です）。

## 7. Gateway を loopback で起動する（外部公開しない）

```bash
clawdbot gateway --bind loopback --port 18789 --verbose
```

`--bind` は `loopback|lan|tailnet|auto|custom` 等が利用できますが、**セキュリティ目的なら loopback を推奨**します。

## 8. ローカル PC から接続する（SSH トンネル）

ローカル PC で **新しいターミナル**を開き、次を実行します。

```bash
ssh -N -L 18789:127.0.0.1:18789 -o ExitOnForwardFailure=yes clawd@YOUR_SERVER_IP
```

このターミナルを開いたままにすると、ローカルから `127.0.0.1:18789` 経由で Gateway にアクセスできます。

## 9. 自動起動（systemd）にする（任意・推奨）

### 10-1. `clawdbot` の絶対パスを確認する

`systemd` は対話シェルの `PATH` に依存しないため、`ExecStart` は **絶対パス**にします。

```bash
command -v clawdbot
```

例: `/home/clawd/.npm-global/bin/clawdbot`

### 10. サービスを作成する

以下の例では、`ExecStart=` のパスをあなたの環境に合わせて置き換えてください。

`/etc/systemd/system/clawdbot-gateway.service`:

```ini
[Unit]
Description=Clawdbot Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=clawd
Group=clawd
WorkingDirectory=/home/clawd

# command -v clawdbot の出力に置き換える
ExecStart=/home/clawd/.npm-global/bin/clawdbot gateway --bind loopback --port 18789 --verbose

Restart=on-failure
RestartSec=3

# 壊れにくい範囲のハードニング（必要なら段階的に調整）
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ProtectControlGroups=true
ProtectKernelTunables=true
ProtectKernelModules=true
RestrictSUIDSGID=true
LockPersonality=true

[Install]
WantedBy=multi-user.target
```

反映して起動します。

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now clawdbot-gateway
sudo systemctl status clawdbot-gateway
```

ログ:

```bash
sudo journalctl -u clawdbot-gateway -f
```
