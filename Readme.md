# Azure VM for GitHub Copilot CLI

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Azure ARM Template](https://img.shields.io/badge/Azure-ARM%20Template-0078D4?logo=microsoftazure&logoColor=white)](azuredeploy.json)
[![Ubuntu 24.04 LTS](https://img.shields.io/badge/Ubuntu-24.04%20LTS-E95420?logo=ubuntu&logoColor=white)](https://releases.ubuntu.com/noble/)
[![Microsoft Entra ID SSH](https://img.shields.io/badge/Entra%20ID-SSH-5E5E5E?logo=microsoft&logoColor=white)](https://learn.microsoft.com/entra/identity/devices/howto-vm-sign-in-azure-ad-linux)
[![Defender for Cloud JIT](https://img.shields.io/badge/Defender%20for%20Cloud-JIT-5C2D91?logo=microsoftazure&logoColor=white)](https://learn.microsoft.com/azure/defender-for-cloud/just-in-time-access-overview)
[![GitHub Copilot CLI](https://img.shields.io/badge/GitHub%20Copilot-CLI-000000?logo=githubcopilot&logoColor=white)](https://docs.github.com/copilot/concepts/agents/about-copilot-cli)

**日本語** | [English](Readme-en.md)

Microsoft Entra ID で SSH ログインし、GitHub Copilot CLI を長時間実行するための
小規模な Azure Linux VM をデプロイします。

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Ftokawa-ms%2Fazure-copilot-cli-vm%2Fmain%2Fazuredeploy.json)

## Highlights

| セキュアな接続 | 長時間実行 | 開発環境を自動構築 | 再利用可能 |
|---|---|---|---|
| Entra ID SSH + JIT | tmux で切断後も継続 | Copilot CLI + zsh + Gitツール | ARM Template + cloud-init |

## 設計

このテンプレートは次のリソースと設定を作成します。

- Ubuntu Server 24.04 LTS Gen2
- `Standard_B2als_v2`（既定値、2 vCPU / 4 GiB）
- 64 GiB Standard SSD LRS
- Trusted Launch、Secure Boot、vTPM
- Standard Static Public IPv4
- Public Internet からの恒久的な受信許可を持たない NSG
- Microsoft Entra SSH ログイン用のシステム割り当てマネージド ID
- `AADSSHLoginForLinux` VM 拡張
- 指定した Entra ユーザーまたはグループへの VM Login ロール割り当て
- Defender for Servers Plan 2 の JIT ポリシー（TCP/22、最大1時間）
- GitHub Copilot CLI、Git、tig、tmux、zsh、Oh My Zsh を導入する cloud-init
- `ja_JP.UTF-8` ロケールと Nerd Font 対応の `agnoster` テーマ
- fzf、zoxide、コマンド候補、構文ハイライト、永続コマンド履歴

通常時は TCP/22 が閉じています。接続時だけ JIT で要求元の Public IP に対して
TCP/22 を開き、Microsoft Entra ID の短期 SSH 証明書でログインします。

## 前提条件

- 対象サブスクリプションで Microsoft Defender for Servers Plan 2 が有効
- デプロイ実行者にリソース作成、ロール割り当て、JIT ポリシー作成の権限がある
- ログイン対象のユーザーまたはグループが、サブスクリプションと同じ
  Microsoft Entra テナントに存在する
- 接続元で Azure CLI と OpenSSH を利用できる
- Nerd Font の記号と日本語を表示する場合、接続元ターミナルに
  `HackGen Console NF` がインストールされている
- GitHub Copilot の有効なライセンスがある
- 組織のネットワークから GitHub Copilot、GitHub、Microsoft Entra ID への
  HTTPS 通信が許可されている

## デプロイパラメーター

| パラメーター | 内容 | 既定値 |
|---|---|---|
| `vmName` | VM 名 | `copilot-cli-vm` |
| `vmSize` | VM SKU | `Standard_B2als_v2` |
| `adminUsername` | 緊急用ローカル管理者 | `azureuser` |
| `adminSshPublicKey` | 緊急用 SSH 公開鍵 | 必須 |
| `entraPrincipalId` | ログインを許可するユーザーまたはグループの Object ID | 必須 |
| `entraPrincipalType` | `User` または `Group` | `User` |
| `vmLoginRole` | Entra ログイン時の権限 | `Administrator` |
| `osDiskSizeGB` | OS ディスク容量 | `64` |
| `jitMaxRequestAccessDuration` | JIT の最大開放時間 | `PT1H` |

ユーザーの Object ID は次のように確認できます。

```bash
az ad signed-in-user show --query id -o tsv
```

グループを使う場合:

```bash
az ad group show --group "<group-name>" --query id -o tsv
```

緊急用 SSH 鍵がない場合:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/azure-copilot-cli-vm
```

## JIT アクセスと SSH

Azure Portal で対象 VM を開き、**Connect** から JIT アクセスを要求します。
ポート22、接続元は **My IP**、必要最小限の時間を指定してください。
`Any` は選択しないでください。

JIT が承認された後、ローカル端末から接続します。

```bash
az login --tenant "<tenant-id>"
az account set --subscription "<subscription-id>"
az extension add --name ssh
az ssh vm --resource-group "<resource-group>" --name "copilot-cli-vm"
```

`Owner` や `Contributor` だけでは Entra SSH ログイン権限になりません。
テンプレートは指定したプリンシパルに以下のいずれかを VM スコープで付与します。

- `Virtual Machine Administrator Login`
- `Virtual Machine User Login`

## GitHub Copilot CLI

cloud-init は以下を自動的にインストールします。

- GitHub Copilot CLI
- Git、tig、curl、jq、ripgrep
- tmux
- zsh、Oh My Zsh（`agnoster` テーマ、`git` プラグイン）
- fzf、zoxide、zsh-autosuggestions、zsh-syntax-highlighting、bat
- 日本語 language pack と `ja_JP.UTF-8` ロケール
- build-essential
- Python 3、pip、venv
- unzip、zip、ca-certificates

初回 SSH 接続時に準備状況を確認します。

```bash
cloud-init status --wait
cat /var/log/copilot-vm-init.log
copilot --version
git --version
tig --version
zsh --version
fzf --version
zoxide --version
```

Copilot CLI への認証は VM の Entra ID ログインとは別です。
GitHub OAuth デバイスフローを開始します。

```bash
copilot login
```

表示されたコードを手元のブラウザで承認します。GitHub Organization が SAML SSO
を使っている場合は、対象 Organization へのアクセスも承認してください。

## zsh と日本語表示

対話 SSH セッションは自動的に zsh を起動します。Oh My Zsh は
`/opt/oh-my-zsh` に固定リビジョンで共有インストールされ、`agnoster` テーマと
軽量な標準プラグインが有効です。入力候補、構文ハイライト、fzf による履歴検索、
zoxide によるディレクトリ移動、5万件の重複抑制付き履歴も構成されます。

主な短縮コマンド:

| コマンド | 展開または用途 |
|---|---|
| `g` / `gst` / `gl` | Git、status、グラフ付きlog |
| `t` | tig |
| `ta <name>` / `tls` | tmux attach / セッション一覧 |
| `cpl` | GitHub Copilot CLI |
| `z <keyword>` | zoxide によるディレクトリ移動 |
| `Ctrl-R` | fzf によるコマンド履歴検索 |

一時的に自動切り替えを無効化する場合は、ホームに次のファイルを作成して再接続します。

```bash
touch ~/.disable-auto-zsh
```

SSH の文字や記号を描画するフォントは VM ではなく接続元ターミナルで使用されます。
そのため HackGen Nerd Font を VM にインストールしても表示は変わりません。接続元の
Windows には [HackGen Releases](https://github.com/yuru7/HackGen/releases) から
`HackGen_NF_*.zip` を取得して TTF をインストールし、Windows Terminal の対象
プロファイルでフォント フェイスを `HackGen Console NF` に設定してください。

VM 側のロケールは次のコマンドで確認できます。

```bash
locale
printf '日本語表示テスト\n'
```

## 長時間実行

SSH や JIT の期限から処理を独立させるため、Copilot CLI は tmux 内で起動します。

```bash
tmux new -s copilot
copilot
```

`Ctrl-b`、続けて `d` でデタッチします。JIT の期限が切れても、VM と tmux 内の
プロセスは継続します。再接続後は次のコマンドで戻ります。

```bash
tmux attach -t copilot
```

VM の再起動やホストメンテナンスでは tmux も終了します。成果物は定期的に Git
へコミットまたはプッシュしてください。

## セキュリティ上の注意

- JIT 要求時は必ず要求元 IP の `/32` のみを許可する
- 恒久的な TCP/22 Allow 規則を追加しない
- ローカルパスワード認証を有効にしない
- 緊急用秘密鍵をリポジトリ、cloud-init、ARM パラメーターへ保存しない
- GitHub トークンを ARM Template や cloud-init に埋め込まない
- VM は個人専用とし、OS ディスクやスナップショットを共有しない
- JIT 要求と NSG 変更を Azure Activity Log で監査する
- 組織の Azure Policy、送信通信制御、データ取り扱い規則を優先する

ヘッドレス Linux でキーチェーンが使えない場合、Copilot CLI は認証情報を
`~/.copilot/config.json` に保存する確認を表示することがあります。保存する場合は
ファイルを本人だけが読める状態にします。

```bash
chmod 700 ~/.copilot
chmod 600 ~/.copilot/config.json
```

## 概算コスト

Japan East で 730 時間連続稼働した場合の参考値です。価格、為替、契約割引により
変動します。

| 項目 | 月額概算（税別） |
|---|---:|
| `Standard_B2als_v2` | 約 ¥5,862 |
| 64 GiB Standard SSD | 約 ¥787 |
| Standard Public IPv4 | 約 ¥598 |
| 合計 | **約 ¥7,247** |

既存の Defender for Servers Plan 2、GitHub Copilot ライセンス、送信データ、
ログ保存、バックアップは上記に含みません。

## ファイル

| ファイル | 内容 |
|---|---|
| [`azuredeploy.json`](azuredeploy.json) | ARM Template |
| [`cloud-init.yaml`](cloud-init.yaml) | VM 初期化処理。テンプレートにも同じ内容を内包 |
| [`Readme-en.md`](Readme-en.md) | English documentation |

## 参考資料

- [Microsoft Entra ID を使用した Linux VM へのサインイン](https://learn.microsoft.com/entra/identity/devices/howto-vm-sign-in-azure-ad-linux)
- [Just-in-time VM access](https://learn.microsoft.com/azure/defender-for-cloud/just-in-time-access-overview)
- [GitHub Copilot CLI のインストール](https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/install-copilot-cli)
- [GitHub Copilot CLI の認証](https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/authenticate-copilot-cli)

## Contributing

Issue と Pull Request を歓迎します。変更時は以下を守ってください。

- `cloud-init.yaml` と `azuredeploy.json` 内の `cloudInit` を同じ内容に保つ
- 秘密鍵、トークン、実環境のパラメーターファイルをコミットしない
- ARM JSON と cloud-init YAML が構文上有効であることを確認する
- セキュリティ既定値（パスワード認証無効、恒久的なSSH許可なし）を維持する

## License

このプロジェクトは [MIT License](LICENSE) の下で公開されています。
