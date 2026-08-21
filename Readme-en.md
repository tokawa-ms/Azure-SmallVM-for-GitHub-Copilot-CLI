# Azure VM for GitHub Copilot CLI

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Azure ARM Template](https://img.shields.io/badge/Azure-ARM%20Template-0078D4?logo=microsoftazure&logoColor=white)](azuredeploy.json)
[![Ubuntu 24.04 LTS](https://img.shields.io/badge/Ubuntu-24.04%20LTS-E95420?logo=ubuntu&logoColor=white)](https://releases.ubuntu.com/noble/)
[![Microsoft Entra ID SSH](https://img.shields.io/badge/Entra%20ID-SSH-5E5E5E?logo=microsoft&logoColor=white)](https://learn.microsoft.com/entra/identity/devices/howto-vm-sign-in-azure-ad-linux)
[![Defender for Cloud JIT](https://img.shields.io/badge/Defender%20for%20Cloud-JIT-5C2D91?logo=microsoftazure&logoColor=white)](https://learn.microsoft.com/azure/defender-for-cloud/just-in-time-access-overview)
[![GitHub Copilot CLI](https://img.shields.io/badge/GitHub%20Copilot-CLI-000000?logo=githubcopilot&logoColor=white)](https://docs.github.com/copilot/concepts/agents/about-copilot-cli)

[日本語](Readme.md) | **English**

Deploy a small Azure Linux VM for long-running GitHub Copilot CLI sessions, with
SSH access authenticated by Microsoft Entra ID.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Ftokawa-ms%2FAzure-SmallVM-for-GitHub-Copilot-CLI%2Fmain%2Fazuredeploy.json)

## Highlights

| Secure access | Long-running work | Ready-to-code tooling | Reusable deployment |
|---|---|---|---|
| Entra ID SSH + JIT | tmux survives disconnects | Copilot CLI + zsh + Git tools | ARM template + cloud-init |

## Architecture

The template creates and configures:

- Ubuntu Server 24.04 LTS Gen2
- `Standard_B2als_v2` by default (2 vCPU / 4 GiB)
- 64-GiB Standard SSD LRS
- Trusted Launch, Secure Boot, and vTPM
- A Standard static public IPv4 address
- An NSG with no permanent inbound access from the public internet
- A system-assigned managed identity for Microsoft Entra SSH
- The `AADSSHLoginForLinux` VM extension
- A VM Login role assignment for a specified Entra user or group
- A Defender for Servers Plan 2 JIT policy for TCP/22, limited to one hour
- Cloud-init that installs GitHub Copilot CLI, Git, tig, tmux, zsh, and Oh My Zsh
- The `ja_JP.UTF-8` locale and the Nerd Font-aware `agnoster` theme
- fzf, zoxide, command suggestions, syntax highlighting, and persistent history

TCP/22 remains closed during normal operation. Before connecting, request JIT
access for only your current public IP address. SSH authentication uses a
short-lived certificate issued through Microsoft Entra ID.

## Prerequisites

- Microsoft Defender for Servers Plan 2 is enabled on the target subscription.
- The deploying identity can create resources, role assignments, and JIT
  network access policies.
- The user or group allowed to sign in exists in the subscription's Entra tenant.
- Azure CLI and OpenSSH are available on the client.
- `HackGen Console NF` is installed on the client terminal when Nerd Font
  glyphs and Japanese text are required.
- The user has an active GitHub Copilot license.
- The organizational network permits HTTPS access to GitHub Copilot, GitHub,
  and Microsoft Entra ID.

## Deployment parameters

| Parameter | Description | Default |
|---|---|---|
| `vmName` | Virtual machine name | `copilot-cli-vm` |
| `vmSize` | Virtual machine SKU | `Standard_B2als_v2` |
| `adminUsername` | Break-glass local administrator | `azureuser` |
| `adminSshPublicKey` | Break-glass SSH public key | Required |
| `entraPrincipalId` | Object ID of the user or group allowed to sign in | Required |
| `entraPrincipalType` | `User` or `Group` | `User` |
| `vmLoginRole` | Privilege granted at VM scope | `Administrator` |
| `osDiskSizeGB` | OS disk size | `64` |
| `jitMaxRequestAccessDuration` | Maximum JIT access window | `PT1H` |

Get the signed-in user's object ID:

```bash
az ad signed-in-user show --query id -o tsv
```

For a group:

```bash
az ad group show --group "<group-name>" --query id -o tsv
```

Create a break-glass SSH key if needed:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/azure-copilot-cli-vm
```

## JIT access and SSH

In the Azure portal, open the VM and request JIT access from **Connect**. Select
port 22, **My IP**, and the shortest practical duration. Never select `Any`.

After access is approved, connect from a local terminal:

```bash
az login --tenant "<tenant-id>"
az account set --subscription "<subscription-id>"
az extension add --name ssh
az ssh vm --resource-group "<resource-group>" --name "copilot-cli-vm"
```

`Owner` and `Contributor` do not grant Entra SSH sign-in by themselves. The
template assigns one of these roles at VM scope:

- `Virtual Machine Administrator Login`
- `Virtual Machine User Login`

## GitHub Copilot CLI

Cloud-init installs:

- GitHub Copilot CLI
- Git, tig, curl, jq, and ripgrep
- tmux
- zsh and Oh My Zsh with the `agnoster` theme and `git` plugin
- fzf, zoxide, zsh-autosuggestions, zsh-syntax-highlighting, and bat
- The Japanese language pack and `ja_JP.UTF-8` locale
- build-essential
- Python 3, pip, and venv
- unzip, zip, and ca-certificates

After the first SSH connection, verify provisioning:

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

Copilot CLI authentication is separate from Entra authentication to the VM.
Start the GitHub OAuth device flow:

```bash
copilot login
```

Approve the displayed code in a browser. If the GitHub organization uses SAML
SSO, authorize access to the organization as well.

## zsh and Japanese text

Interactive SSH sessions automatically start zsh. Oh My Zsh is installed at a
pinned revision under `/opt/oh-my-zsh`, with the `agnoster` theme and a small
set of built-in plugins. The configuration also enables command suggestions,
syntax highlighting, fzf history search, zoxide directory navigation, and a
deduplicated 50,000-entry history.

Useful shortcuts:

| Command | Expansion or purpose |
|---|---|
| `g` / `gst` / `gl` | Git, status, and graphical log |
| `t` | tig |
| `ta <name>` / `tls` | Attach to tmux / list sessions |
| `cpl` | GitHub Copilot CLI |
| `z <keyword>` | Navigate with zoxide |
| `Ctrl-R` | Search command history with fzf |

To disable the automatic switch for subsequent connections, create this file
in your home directory:

```bash
touch ~/.disable-auto-zsh
```

SSH text and glyphs are rendered by the client terminal, not by the VM.
Installing HackGen Nerd Font on the VM therefore does not affect the display.
On the Windows client, download `HackGen_NF_*.zip` from
[HackGen Releases](https://github.com/yuru7/HackGen/releases), install the TTF
files, and select `HackGen Console NF` as the font face for the relevant
Windows Terminal profile.

Verify the VM locale and Japanese text:

```bash
locale
printf 'Japanese: 日本語表示テスト\n'
```

## Long-running sessions

Run Copilot CLI inside tmux so that it is independent of the SSH connection and
the JIT access window:

```bash
tmux new -s copilot
copilot
```

Press `Ctrl-b`, then `d` to detach. The VM and processes inside tmux continue
after JIT expires. After requesting JIT access and reconnecting:

```bash
tmux attach -t copilot
```

A VM reboot or host maintenance ends the tmux process. Commit or push work to
Git regularly.

## Security notes

- Limit every JIT request to the requestor's `/32` source IP.
- Do not add a permanent TCP/22 allow rule.
- Do not enable local password authentication.
- Never store the break-glass private key in the repository, cloud-init, or ARM
  parameters.
- Never embed GitHub tokens in the ARM template or cloud-init.
- Keep the VM single-user and do not share its OS disk or snapshots.
- Audit JIT requests and NSG changes in Azure Activity Log.
- Follow organizational Azure Policy, egress filtering, and data handling rules.

If no Linux keychain is available, Copilot CLI might ask to store credentials in
`~/.copilot/config.json`. If approved, restrict access:

```bash
chmod 700 ~/.copilot
chmod 600 ~/.copilot/config.json
```

## Estimated cost

Illustrative retail pricing for 730 hours per month in Japan East:

| Item | Estimated monthly cost before tax |
|---|---:|
| `Standard_B2als_v2` | About JPY 5,862 |
| 64-GiB Standard SSD | About JPY 787 |
| Standard public IPv4 | About JPY 598 |
| Total | **About JPY 7,247** |

Pricing, exchange rates, and contract discounts vary. Existing Defender for
Servers Plan 2, the GitHub Copilot license, outbound data, logs, and backups are
not included.

## Files

| File | Purpose |
|---|---|
| [`azuredeploy.json`](azuredeploy.json) | ARM template |
| [`cloud-init.yaml`](cloud-init.yaml) | VM bootstrap, also embedded in the template |
| [`Readme.md`](Readme.md) | Japanese documentation |

## References

- [Sign in to a Linux VM by using Microsoft Entra ID](https://learn.microsoft.com/entra/identity/devices/howto-vm-sign-in-azure-ad-linux)
- [Just-in-time VM access](https://learn.microsoft.com/azure/defender-for-cloud/just-in-time-access-overview)
- [Install GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/install-copilot-cli)
- [Authenticate GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/authenticate-copilot-cli)

## Contributing

Issues and pull requests are welcome. When making changes:

- Keep `cloud-init.yaml` synchronized with the `cloudInit` value embedded in
  `azuredeploy.json`.
- Never commit private keys, tokens, or environment-specific parameter files.
- Confirm that the ARM JSON and cloud-init YAML remain syntactically valid.
- Preserve the secure defaults: disabled password authentication and no
  permanent inbound SSH rule.

## License

This project is available under the [MIT License](LICENSE).
