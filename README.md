# OpenClaw on Azure Linux VM with Native GitHub Copilot Provider

Deploy OpenClaw with the native GitHub Copilot provider (`github-copilot`) on a single Azure Linux VM using Azure Resource Manager (ARM) templates, with Azure Bastion-first administration and SSH key-based access.

This implementation is designed for enterprise Azure environments that typically require:

1. Controlled admin access paths to VM via Azure Bastion SSH (no public SSH exposure).
2. Repeatable infrastructure deployment via ARM.
3. Operationally simple single-VM rollout before scaling complexity.

## Architecture

1. A single Azure resource group containing all resources.
2. Network:
   - VNet + VM subnet.
   - Azure Bastion Standard host with native client tunneling enabled + Azure Bastion subnet.
   - Network Security Group (NSG) policy:
     - deny SSH from Internet
     - deny SSH from VirtualNetwork
     - allow SSH from Azure Bastion subnet only
3. Compute: Ubuntu LTS VM.
4. Storage: Managed OS disk.
5. Access pattern:
   - VM admin via Azure Bastion SSH.
   - OpenClaw UI via Azure Bastion tunnel to local loopback.

## Repository Contents

1. `infra/azuredeploy.json` - ARM template.
2. `infra/azuredeploy.parameters.json` - deployment parameters.
3. `scripts/bootstrap-openclaw.sh` - bootstrap script to install OpenClaw.

## Prerequisites

### Required Tools and Access

1. Azure CLI installed. See [Azure CLI install steps](https://learn.microsoft.com/cli/azure/install-azure-cli) if needed.
2. SSH public key available at `~/.ssh/id_ed25519.pub`. To generate one, run the `ssh-keygen -t ed25519 -a 100 -f ~/.ssh/id_ed25519 -C "yourname@company"` command.

### Azure Login

```bash
az login
az account show -o table
```

### Azure CLI SSH Extension

```bash
az extension add -n ssh
```

### Azure Resource Provider Registration

```bash
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.Network

az provider show --namespace Microsoft.Compute --query registrationState -o tsv
az provider show --namespace Microsoft.Network --query registrationState -o tsv
```

## Deploy Azure Resources

### Set Azure Deployment Variables

```bash
RG="rg-openclaw"
LOCATION="westus2"
TEMPLATE_URI="https://raw.githubusercontent.com/johnsonshi/openclaw-azure-github-copilot/main/infra/azuredeploy.json"
PARAMS_URI="https://raw.githubusercontent.com/johnsonshi/openclaw-azure-github-copilot/main/infra/azuredeploy.parameters.json"
SSH_PUB_KEY="$(cat ~/.ssh/id_ed25519.pub)"
```

### Create Azure Resource Group

```bash
az group create -n "${RG}" -l "${LOCATION}"
```

### Validate and Preview Azure Deployment

```bash
az deployment group validate \
  -g "${RG}" \
  --template-uri "${TEMPLATE_URI}" \
  --parameters "${PARAMS_URI}" \
  --parameters sshPublicKey="${SSH_PUB_KEY}"

az deployment group what-if \
  -g "${RG}" \
  --template-uri "${TEMPLATE_URI}" \
  --parameters "${PARAMS_URI}" \
  --parameters sshPublicKey="${SSH_PUB_KEY}"
```

### Create Azure Deployment

```bash
az deployment group create \
  -g "${RG}" \
  --template-uri "${TEMPLATE_URI}" \
  --parameters "${PARAMS_URI}" \
  --parameters sshPublicKey="${SSH_PUB_KEY}"
```

> [!NOTE]
> You can override any template parameter in the CLI command.

```bash
az deployment group create \
  -g "${RG}" \
  --template-uri "${TEMPLATE_URI}" \
  --parameters location="${LOCATION}" \
  --parameters vmName="vm-openclaw" \
  --parameters vmSize="Standard_B2as_v2" \
  --parameters adminUsername="openclaw" \
  --parameters sshPublicKey="${SSH_PUB_KEY}" \
  --parameters vnetName="vnet-openclaw" \
  --parameters vnetAddressPrefix="10.40.0.0/16" \
  --parameters vmSubnetName="snet-openclaw-vm" \
  --parameters vmSubnetPrefix="10.40.2.0/24" \
  --parameters bastionSubnetPrefix="10.40.1.0/26" \
  --parameters nsgName="nsg-openclaw-vm" \
  --parameters nicName="nic-openclaw-vm" \
  --parameters bastionName="bas-openclaw" \
  --parameters bastionPublicIpName="pip-openclaw-bastion" \
  --parameters osDiskSizeGb=64
```

## Setup OpenClaw on Azure VM

### Set Post-Deployment Variables

```bash
RG="rg-openclaw"
VM_NAME="vm-openclaw"
BASTION_NAME="bas-openclaw"
ADMIN_USERNAME="openclaw"
VM_ID="$(az vm show -g "${RG}" -n "${VM_NAME}" --query id -o tsv)"
```

### Install OpenClaw on the VM

Run host provisioning first from local shell to install OpenClaw on the VM (no interactive Azure Bastion session required):

```bash
az vm run-command invoke \
  -g "${RG}" \
  -n "${VM_NAME}" \
  --command-id RunShellScript \
  --scripts "curl -fsSL https://raw.githubusercontent.com/johnsonshi/openclaw-azure-github-copilot/main/scripts/bootstrap-openclaw.sh | bash"
```

### Connect to VM Through Azure Bastion (SSH)

Create an Azure Bastion SSH session for use in configuring OpenClaw:

```bash
az network bastion ssh \
  --name "${BASTION_NAME}" \
  --resource-group "${RG}" \
  --target-resource-id "${VM_ID}" \
  --auth-type ssh-key \
  --username "${ADMIN_USERNAME}" \
  --ssh-key ~/.ssh/id_ed25519
```

### OpenClaw Configuration (Provider / Model)

Configure OpenClaw in the **same Azure Bastion SSH VM shell**:

```bash
openclaw onboard --install-daemon
```

During onboarding wizard, use:

1. Continue security prompt: `Yes`
2. Onboarding mode: `QuickStart`
3. Model/auth provider: `Copilot`
4. Copilot auth method: `GitHub Copilot (GitHub device login)`

QuickStart selections expected:

1. Gateway port: `18789`
2. Gateway bind: `Loopback (127.0.0.1)`
3. Gateway auth: `Token (default)`
4. Tailscale exposure: `Off`
5. Channel mode: `Direct to chat channels`

After selecting Copilot device login, OpenClaw will print a device code and prompt you to authorize at `https://github.com/login/device`.
Keep the VM shell open while completing device authorization.

You can run provider auth directly as well:

```bash
openclaw models auth login-github-copilot
```

Set a default model:

```bash
openclaw models set github-copilot/<model-id>
```

`model-id` guidance:

1. Example: `openclaw models set github-copilot/gpt-5.4`
2. You can also try various OpenClaw-supported model ids by running:

```bash
openclaw models list
```

3. If rejected for your Copilot plan, try another supported model id or contact your Copilot plan admin.

For more information on authenticating OpenClaw with the GitHub Copilot Provider:
`https://docs.openclaw.ai/providers/github-copilot`

### OpenClaw Runtime Validation

Run the following in the **same Azure Bastion SSH VM shell**:

```bash
node -v
npm -v
openclaw --version
openclaw gateway status
openclaw logs --limit 200
```

### OpenClaw Control UI Access

Create a local Azure Bastion tunnel to securely reach the VM's OpenClaw UI port without public exposure.

```bash
az network bastion tunnel \
  --name "${BASTION_NAME}" \
  --resource-group "${RG}" \
  --target-resource-id "${VM_ID}" \
  --resource-port 18789 \
  --port 18789
```

Then browse to `http://127.0.0.1:18789`.

## Operations

Before running the commands in this section, connect to the VM using [Connect to VM Through Azure Bastion (SSH)](#connect-to-vm-through-azure-bastion-ssh).

### OS Patching

```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo reboot
```

### OpenClaw Service Operations

```bash
openclaw gateway status
openclaw gateway restart
openclaw logs --follow
```

### OpenClaw Upgrade / Rollback

```bash
sudo npm install -g openclaw@<target-version>
# rollback:
sudo npm install -g openclaw@<previous-known-good-version>
openclaw gateway restart
```
