# Infrastructure Templates

ARM (Azure Resource Manager) templates for the three lab VMs, exported from the live Azure
resource group via **Export template** and sanitized before being committed here.

| File | VM | Role |
|---|---|---|
| `wazuh-server.json` | `soc-lab-wazuh` | SIEM |
| `ubuntu-server.json` | `soc-lab-UbuntuServer` | Linux server + Zeek network sensor |
| `windows-host.json` | `soc-lab-windows` | Victim endpoint |

## What's in them

Each template documents the VM's actual configuration: size (`Standard_D4s_v3` /
`Standard_D2s_v3`), OS image, disk setup, and network interface attachment — see
`README.md` at the repo root for the full architecture table. They do **not** contain the
VM's disk contents, installed software configuration, or any data from inside the VM.

## What was redacted, and why

Azure's raw export embeds real account details directly in the JSON. Before committing,
the following were replaced with parameters or placeholders:

| Removed | Replaced with | Why |
|---|---|---|
| Subscription ID | `<SUBSCRIPTION_ID>` | Identifies and can help target the specific Azure account. |
| Resource group name | `<RESOURCE_GROUP>` | Reveals internal Azure account structure. |
| Admin usernames (`soc-lab-admin`, `soc-lab-UbuntuAdmin`, `azureuser`) | `adminUsername` parameter, default `REPLACE_ME` | No reason to publish real login usernames alongside hostnames. |
| SSH public key (tied to a real institutional email) | `sshPublicKey` parameter, default `REPLACE_ME_WITH_YOUR_SSH_PUBLIC_KEY` | The key itself is meant to be public, but publishing it tied to an identifiable email is unnecessary exposure. |

No `adminPassword` was present in the exported templates — Azure's VM export doesn't
include plaintext secrets. If you ever export a template that *does* include one (e.g. a
full resource-group export with a `parameters.json` side file), **do not commit it** — see
the root `.gitignore`, which already excludes `parameters.json` and anything matching
`*secrets*` / `*password*` / `*.key` / `*.pem`.

## Limitation: these are not a from-scratch deploy

Each template exports a single VM resource, referencing its OS disk and network interface
by existing Azure resource ID (`managedDisk.id`, `networkProfile.networkInterfaces[].id`)
rather than defining them inline. That means these templates document *how the VM is
configured*, but deploying them requires the referenced disk and NIC to already exist in
the target resource group — they won't stand up the vnet, subnet, NSG, disks, or NICs from
nothing. A full from-scratch redeploy would need `az group export` at the resource-group
level (which pulls in the network and disk resources too) rather than per-VM exports.

## Redeploying against an existing resource group

```bash
az deployment group create \
  --resource-group <your-resource-group> \
  --template-file infra/wazuh-server.json \
  --parameters adminUsername=<your-username> sshPublicKey="<your-ssh-public-key>" \
  --parameters disks_soc_lab_wazuh_OsDisk_1_5623dd2ee14a4836acf5e0036d3462ae_externalid=<your-disk-resource-id> \
  --parameters networkInterfaces_soc_lab_wazuh940_externalid=<your-nic-resource-id>
```

Same pattern for `ubuntu-server.json` and `windows-host.json` (the latter has no SSH
parameter — it uses Windows password auth, set via the Azure Portal/CLI at creation time,
not stored in the template).
