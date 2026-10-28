---
name: ubuntu-on-ec2
description: Runs Ubuntu and Ubuntu Pro on Amazon EC2 - finding the official Canonical AMIs (server, minimal, Pro, EKS) through SSM public parameters, choosing an LTS release, launching with Ubuntu-specific defaults (/dev/sda1 root, IMDSv2-only images, the ubuntu user, snap-based SSM Agent), enabling Ubuntu Pro (Pro AMIs, token attach, License Manager conversion of running Ubuntu LTS instances, ESM, Livepatch, FIPS, USG CIS/DISA-STIG hardening), building Ubuntu and Ubuntu Pro golden AMIs with EC2 Image Builder, and using Ubuntu listings from AWS Marketplace (FIPS images, private offers). Applies when the user names Ubuntu, Ubuntu Pro, Canonical, ESM, Livepatch, FIPS on Ubuntu, CIS-hardened Ubuntu, or apt-based golden images on EC2. Not for generic EC2 fleet management, Amazon Linux images, or EKS cluster setup.
version: 1
---

# Ubuntu and Ubuntu Pro on Amazon EC2

## Overview

Domain expertise for running Canonical's Ubuntu images on EC2: finding the right AMI, launching it with Ubuntu-specific settings, turning on Ubuntu Pro, baking golden images, and using AWS Marketplace listings. Canonical publishes the official images under AWS account `099720109477` and indexes them in SSM public parameters under `/aws/service/canonical/ubuntu/`.

**Works best with** the [AWS MCP server](https://docs.aws.amazon.com/aws-mcp/). All guidance also works with standard AWS CLI access.

## First decision: which Ubuntu image

Ask this before anything else. The answer determines the AMI source, the billing, and whether a subscription or token is needed.

| The user needs | Use | Billing |
|---|---|---|
| Ubuntu with standard support (security updates for `main` for 5 years per LTS) | Canonical **server** AMI (SSM `server/`) | EC2 Linux rate only |
| Smaller footprint, fewer packages (containers hosts, appliances) | Canonical **minimal** AMI (SSM `server-minimal/`) | EC2 Linux rate only |
| Expanded security updates (ESM for `main` + `universe`), Livepatch, FIPS, USG hardening, or up to 10 years of support | Canonical **Ubuntu Pro** AMI (SSM `pro-server/` or `pro-minimal/`) - no token, no subscription | Per-hour Pro charge on the EC2 bill (platform "Ubuntu Pro Linux") |
| Ubuntu Pro on instances that already run Ubuntu LTS | **License Manager conversion** to Ubuntu Pro (no relaunch; one-way) | Pro per-hour rate on the EC2 bill, or a Canonical Marketplace subscription |
| Pro under an existing Canonical contract (token) | Standard server AMI + `pro attach` with the contract token | Paid to Canonical, EC2 Linux rate on AWS |
| FIPS 140 certified Ubuntu, a Marketplace private offer, or purchasing through Marketplace | **AWS Marketplace** listing from Canonical | Marketplace line item |

Default to the **current LTS** with the **standard server image**, unless the user needs the support or compliance features in the Pro rows. Never default to an interim release (non-LTS, 9 months of support) for production.

## Routing (references in this skill)

Read the matching reference before answering. The SSM paths, billing codes, and Pro commands are exact, and recalling them from general knowledge is how they get subtly wrong.

| User need | Read |
|---|---|
| Find the AMI ID for a release, architecture, or variant; launch an instance; Ubuntu launch gotchas | [finding-and-launching-ubuntu-amis.md](references/finding-and-launching-ubuntu-amis.md) |
| Ubuntu Pro: choose Pro AMI vs token vs conversion, enable ESM/Livepatch/FIPS/USG, verify, patch | [ubuntu-pro.md](references/ubuntu-pro.md) |
| Build an Ubuntu or Ubuntu Pro golden AMI (EC2 Image Builder or manual `create-image`) | [building-ubuntu-images.md](references/building-ubuntu-images.md) |
| FIPS images, Marketplace subscriptions, private offers, third-party hardened Ubuntu | [marketplace.md](references/marketplace.md) |

## Guardrails (every workflow)

- **Only trust Canonical-owned images.** Resolve AMIs through `/aws/service/canonical/ubuntu/...` SSM parameters or `describe-images --owners 099720109477`. A name filter like `*ubuntu*` without an owner also returns community AMIs that anyone can publish. Marketplace images are owned by the shared `aws-marketplace` account, so verify those by the listing's product code and seller (see [marketplace.md](references/marketplace.md)).
- **The root device is `/dev/sda1`**, not `/dev/xvda`. Block device mappings that use `/dev/xvda` (the Amazon Linux convention used elsewhere in this toolkit) add a second empty volume instead of resizing or encrypting the root volume.
- **Ubuntu 24.04 and later AMIs are IMDSv2-only** (`ImdsSupport: v2.0`). Tools on the instance that still call IMDSv1 get `401`. Set `HttpTokens=required` in launch templates for every release anyway.
- **Never put a Pro token in user data, component YAML, or AMIs.** User data is readable through IMDS and logged by cloud-init. Store the token in AWS Secrets Manager and fetch it at boot, or use Pro AMIs or License Manager conversion, which need no token.
- **Don't bake an attached Pro token into an image.** Detach before imaging (see [building-ubuntu-images.md](references/building-ubuntu-images.md)). Instances launched from Pro AMIs attach themselves automatically; they need no cleanup beyond the standard one.
- **Wait for cloud-init and apt before running apt.** `unattended-upgrades` runs on first boot and holds the dpkg lock, so an early `apt-get` fails with `Could not get lock /var/lib/dpkg/lock-frontend`. Run `cloud-init status --wait` first and use `DEBIAN_FRONTEND=noninteractive`.
- **Confirm costs before switching to Pro.** Pro AMIs, conversion, and Marketplace listings change the hourly price. State the billing impact and get explicit approval before launching or converting.

## Ubuntu facts agents commonly get wrong

| Topic | Correct behavior |
|---|---|
| Default SSH user | `ubuntu` (not `ec2-user` or `root`) |
| SSM Agent | Preinstalled as a snap in server images. Service: `snap.amazon-ssm-agent.amazon-ssm-agent.service`. Manage it with `sudo snap restart amazon-ssm-agent`, not `systemctl restart amazon-ssm-agent`. Confirm presence on minimal images before relying on Session Manager. |
| Package manager | `apt`. Security updates arrive automatically each day through `unattended-upgrades`. A kernel update needs a reboot (`/var/run/reboot-required` exists) unless Livepatch covers it. |
| AWS CLI | Not guaranteed on the image. Check `command -v aws`; install with `sudo snap install aws-cli --classic` or the AWS installer. |
| Boot mode | `uefi-preferred` (amd64) or `uefi` (arm64 on newer releases). |
| Release names | SSM paths accept the version number (`24.04`) and AMI names use the codename (`noble`). |
| Support lifecycle | LTS: 5 years of standard support, extended with Ubuntu Pro (ESM) to 10 years. Interim releases: 9 months. Check `pro security-status` on an instance, or the [Ubuntu release cycle](https://ubuntu.com/about/release-cycle). |

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Permission denied (publickey)` | Wrong user | Connect as `ubuntu` |
| Extra empty volume, root volume not resized or encrypted | Mapping used `/dev/xvda` | Use `/dev/sda1` |
| App on 24.04+ gets `401` from `169.254.169.254` | IMDSv1 call on an IMDSv2-only image | Update the client to IMDSv2 (token PUT); for containers set hop limit 2 |
| `Could not get lock /var/lib/dpkg/lock-frontend` in user data or a build | `unattended-upgrades` running at first boot | `cloud-init status --wait`, then retry; avoid killing apt |
| Instance not an SSM managed node | No instance profile, or the snap agent is stopped | Attach `AmazonSSMManagedInstanceCore`; `sudo snap services amazon-ssm-agent` |
| `OptInRequired` on `RunInstances` | Marketplace AMI without a subscription | Subscribe to the listing (see [marketplace.md](references/marketplace.md)) |
| `pro status` shows "not attached" on a Pro AMI or converted instance | Auto-attach couldn't reach Canonical or hasn't run yet | `sudo pro auto-attach`; check outbound HTTPS (see [ubuntu-pro.md](references/ubuntu-pro.md)) |

## Related skills

| Use | When |
|---|---|
| **launching-ec2-instance-with-best-practices** | The full guided launch procedure (networking, IAM, security groups, tags). Take the Ubuntu AMI and settings from this skill. |
| **amazon-ec2-image-builder** | Image Builder mechanics (roles, pipelines, distribution, troubleshooting). This skill covers the Ubuntu-specific parts. |
| **aws-compute** | Launch templates, Auto Scaling, Systems Manager, AMI sharing and retirement |
| **aws-containers** | EKS clusters. This skill only covers finding Ubuntu EKS node AMIs. |

## Security considerations

Use Canonical-owned or Canonical-sold images only, require IMDSv2, prefer Session Manager over SSH, encrypt the root volume (`/dev/sda1`) with a customer-managed KMS key when the AMI will be shared, and keep Pro tokens in Secrets Manager. For hardening, prefer the Ubuntu Security Guide (USG) CIS or DISA-STIG profiles on Ubuntu Pro, applied in an image build and tested before rollout. Enable CloudTrail to audit `RunInstances`, `CreateLicenseConversionTaskForResource`, and AMI sharing changes. References: [Ubuntu on AWS documentation](https://documentation.ubuntu.com/aws/), [EC2 security best practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security.html).
