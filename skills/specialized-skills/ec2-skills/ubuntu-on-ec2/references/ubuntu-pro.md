# Ubuntu Pro on EC2

Contents: [What Pro adds](#what-pro-adds) · [Pick the path](#pick-the-path) · [Pro AMIs](#path-a-ubuntu-pro-amis) · [Conversion](#path-b-convert-a-running-ubuntu-lts-instance-with-license-manager) · [Token attach](#path-c-attach-a-canonical-contract-token) · [Services](#enable-and-manage-pro-services) · [Patching](#patching-with-pro) · [Troubleshooting](#troubleshooting)

## What Pro adds

| Service | What it does | Enabled by default on Pro AMIs |
|---|---|---|
| `esm-infra` | Security updates for `main` beyond the 5 years of standard support (up to 10 years) | Yes |
| `esm-apps` | Security updates for the ~23,000 packages in `universe` | Yes |
| `livepatch` | Applies critical kernel security fixes without a reboot | Yes (where the kernel is supported) |
| `fips-updates` / `fips` | FIPS 140 certified cryptographic modules | No |
| `usg` | Ubuntu Security Guide: automated CIS benchmark and DISA-STIG audit and remediation | No |
| `landscape`, `realtime-kernel`, others | Fleet management and specialized kernels | No |

Run `pro status --all` on an instance to see what its release supports. Availability varies by release and architecture.

## Pick the path

| Situation | Path | Token? | Billing |
|---|---|---|---|
| New instances or fleets | **A. Ubuntu Pro AMI** | No | Per-hour, on the EC2 bill |
| Existing Ubuntu LTS instances that must keep running (no relaunch) | **B. License Manager conversion** | No | Switches to the Pro per-hour rate |
| The organization already has a Canonical contract | **C. Token attach** on a standard image | Yes | Paid to Canonical |
| FIPS-certified image, or buying through Marketplace | Marketplace listing ([marketplace.md](marketplace.md)) | No | Marketplace line item |

Recommend path A for anything new. Before launching or converting, state the per-hour price difference and get approval. Check the EC2 pricing page, or run `aws pricing get-products --service-code AmazonEC2` with the filter `operatingSystem=Ubuntu Pro`. Ubuntu Pro is **not available as a Reserved Instance**; use Savings Plans to discount it.

**Network requirement for every path:** to activate and receive updates, the instance needs outbound TCP 443 to these Canonical endpoints, through a NAT gateway or proxy. A private subnet with only VPC endpoints is not enough.

- `contracts.canonical.com` (activation)
- `esm.ubuntu.com` (ESM apt repositories)
- `livepatch.canonical.com`
- `api.snapcraft.io`, `dashboard.snapcraft.io`, `login.ubuntu.com`, and `cloudfront.cdn.snapcraftcontent.com` (snaps, including the Livepatch client)

The full list is in the [Ubuntu Pro Client network requirements](https://canonical-ubuntu-pro-client.readthedocs-hosted.com/en/latest/references/network_requirements/).

## Path A: Ubuntu Pro AMIs

Resolve the AMI from `/aws/service/canonical/ubuntu/pro-server/<release>/stable/current/<arch>/hvm/<volume>/ami-id` (or `pro-minimal/`). The rest of the launch is identical to a standard Ubuntu launch ([finding-and-launching-ubuntu-amis.md](finding-and-launching-ubuntu-amis.md)).

On first boot, the Pro client attaches itself automatically using the instance's AWS identity document. No token and no Canonical account are needed.

Verify:

```bash
aws ec2 describe-instances --instance-ids ${instance_id} --query "Reservations[0].Instances[0].[PlatformDetails,UsageOperation]" --output text
# Expect: Ubuntu Pro Linux    RunInstances:0g00
```

```bash
# on the instance
pro status            # "Subscription: Ubuntu Pro (cloud)" and services enabled
```

## Path B: Convert a running Ubuntu LTS instance with License Manager

AWS License Manager can move a running Ubuntu LTS instance to Ubuntu Pro without relaunching it. There are two destinations:

| Destination | Destination license context | Billing |
|---|---|---|
| **Ubuntu Pro provided by AWS** (default recommendation) | `UsageOperation=RunInstances:0g00` | AWS, on the EC2 bill |
| **Ubuntu Pro sold by Canonical in AWS Marketplace** | `UsageOperation=RunInstances,ProductCodes=[{ProductCodeType=marketplace,ProductCodeId=<code>}]` | Canonical, through the Marketplace subscription, which must be active before converting (see [marketplace.md](marketplace.md)) |

**This is one-way.** License type conversion only goes from Ubuntu LTS to Ubuntu Pro. Going back to Ubuntu LTS requires an AWS Support case. Say this when asking for approval.

Before converting:

1. Confirm the instance runs Ubuntu LTS and isn't already Pro. Check with `aws ec2 describe-instances --instance-ids ${instance_id} --query "Reservations[0].Instances[0].[PlatformDetails,UsageOperation,ProductCodes]"`: expect `Linux/UNIX`, `RunInstances`, and no Ubuntu Pro product code.
2. Confirm the instance is not attached to a Pro token (`pro status` shows "not attached"). If it is, detach it first so it isn't paid for twice.
3. Make sure the Pro client is installed and current (`pro --version`). If it is missing or old, run `sudo apt-get update && sudo apt-get install -y ubuntu-pro-client`.
4. Confirm outbound access to the Canonical endpoints listed above.
5. Check the current [conversion prerequisites](https://docs.aws.amazon.com/license-manager/latest/userguide/conversion-prerequisites.html), which change over time. The CLI path works on running instances. The License Manager console lists only instances that are **stopped** and reported by Systems Manager Inventory.
6. Get explicit approval, covering the new hourly price and that the conversion is one-way.

Convert to Ubuntu Pro provided by AWS:

```bash
instance_arn="arn:aws:ec2:${region}:${account_id}:instance/${instance_id}"
aws license-manager create-license-conversion-task-for-resource --region ${region} \
  --resource-arn "${instance_arn}" \
  --source-license-context UsageOperation=RunInstances \
  --destination-license-context UsageOperation=RunInstances:0g00
aws license-manager get-license-conversion-task --region ${region} --license-conversion-task-id ${task_id}
```

Then verify on the instance with `pro status`, which should show an active Ubuntu Pro subscription. If it still shows "not attached" after a few minutes, run `sudo pro auto-attach`.

The conversion attaches the entitlement only. It **does not install flavor-specific packages** such as FIPS, USG/CIS hardening, or the real-time kernel, even when converting to a flavor-specific Marketplace listing. Enable those afterwards with `pro enable` (see below).

For a fleet, convert instances one at a time and keep a record of the task IDs (`aws license-manager list-license-conversion-tasks`). For new capacity, switch the launch template to a Pro AMI rather than converting every new instance.

## Path C: Attach a Canonical contract token

Use this only when the organization already pays Canonical directly. Pro is also free for personal use on a limited number of machines with a token from ubuntu.com/pro.

- **Never put the token in user data, AMIs, component YAML, or shell history.** Store it in AWS Secrets Manager, grant the instance role `secretsmanager:GetSecretValue` on that one secret, and attach at boot using an attach config file so the token never appears on a command line:

```bash
#!/bin/bash
set -euo pipefail
umask 077
token=$(aws secretsmanager get-secret-value --secret-id ${secret_id} --query SecretString --output text)
printf 'token: %s\nenable_services:\n  - esm-infra\n  - esm-apps\n  - livepatch\n' "$token" > /run/pro-attach.yaml
pro attach --attach-config /run/pro-attach.yaml
rm -f /run/pro-attach.yaml
```

- The token counts machines against the contract. Detach before terminating instances you don't need (`sudo pro detach --assume-yes`), and never bake an attached instance into an AMI (see [building-ubuntu-images.md](building-ubuntu-images.md)).

## Enable and manage Pro services

```bash
pro status --all
sudo pro enable livepatch
sudo pro enable esm-apps
canonical-livepatch status --verbose     # confirm kernel patches applied
pro security-status                       # package counts by support channel
sudo pro fix CVE-2024-XXXX                # check and fix a specific CVE
```

**FIPS:**

- Prefer `fips-updates` over `fips`. It keeps the certified modules but also receives security fixes.
- Enabling FIPS installs a FIPS kernel and requires a reboot. Treat it as **one-way**: disabling it isn't a supported clean operation. Test on a disposable instance first.
- For new FIPS workloads, prefer a FIPS image from Marketplace or a golden AMI built with FIPS already enabled, rather than enabling FIPS on running production instances.
- FIPS availability lags new LTS releases. Check `pro status --all` for the target release.

```bash
sudo pro enable fips-updates --assume-yes
sudo reboot
# after reboot
cat /proc/sys/crypto/fips_enabled   # 1
```

**USG (CIS and DISA-STIG hardening):**

```bash
sudo pro enable usg
sudo apt-get install -y usg
sudo usg list                              # profiles available for this release
sudo usg audit cis_level1_server           # report only; writes an HTML/XML report under /var/lib/usg
sudo usg fix cis_level1_server             # applies remediation
```

- **Audit before fixing.** Remediation can change SSH, the firewall, mount options (`/tmp` `noexec`), and services. After a fix, confirm SSM Agent still connects and the application still works.
- Apply hardening in an image build with a test stage ([building-ubuntu-images.md](building-ubuntu-images.md)), not ad hoc on production instances.
- USG profiles cover CIS Level 1/2 server and workstation, and DISA-STIG where published. `usg list` is authoritative for each release.

## Patching with Pro

- `unattended-upgrades` installs security updates daily. With Pro attached, that includes the ESM pockets. Kernel updates still need a reboot unless Livepatch covers them. Schedule reboots with `/var/run/reboot-required` as the signal.
- For fleets, AWS Systems Manager Patch Manager works with Ubuntu (predefined baseline `AWS-UbuntuDefaultPatchBaseline`) and uses the apt sources configured on the instance. On attached Pro instances, those sources include ESM. If Patch Manager owns patching, set the baseline to match the update policy, and decide whether `unattended-upgrades` stays enabled so the two don't race for the dpkg lock.
- For immutable fleets, rebuild the golden AMI on a schedule instead (see [building-ubuntu-images.md](building-ubuntu-images.md)).

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Pro AMI instance shows "not attached" | No outbound HTTPS to Canonical, or the first-boot attach hasn't finished | Check the NAT gateway or proxy route, then run `sudo pro auto-attach`; logs in `/var/log/ubuntu-advantage.log` |
| `pro auto-attach` fails on a standard (non-Pro) instance | The instance isn't billed as Pro | Convert it (path B) or attach a token (path C) |
| Conversion task `FAILED` | Prerequisites unmet: not LTS, already Pro, no active Marketplace subscription for a Marketplace destination, or Pro client missing | Read `StatusMessage` from `get-license-conversion-task`, fix the prerequisite, retry |
| Need to go back from Pro to LTS billing | License type conversion is one-way | Open an AWS Support case |
| Livepatch "kernel not supported" | Kernel flavor or version not covered | `canonical-livepatch status`; update to the current `linux-aws` kernel and reboot |
| apt errors from `esm.ubuntu.com` | Instance cannot reach ESM endpoints | Allow outbound HTTPS to Canonical endpoints through the NAT gateway or proxy |
| A cloned instance shares the original's machine identity | An AMI was baked from an attached instance | Rebuild the AMI after `pro detach` and `cloud-init clean` ([building-ubuntu-images.md](building-ubuntu-images.md)) |
