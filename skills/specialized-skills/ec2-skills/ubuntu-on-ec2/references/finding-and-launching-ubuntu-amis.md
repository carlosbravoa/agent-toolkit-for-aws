# Finding and launching Ubuntu AMIs

Contents: [SSM parameters](#1-resolve-the-ami-through-ssm-public-parameters) · [describe-images](#2-alternative-describe-images-with-the-canonical-owner) · [Release choice](#3-choose-the-release) · [Launch settings](#4-launch-settings-that-differ-from-amazon-linux) · [User data](#5-user-data-on-ubuntu) · [Verify](#6-verify-after-launch)

## 1. Resolve the AMI through SSM public parameters

Canonical publishes the current AMI ID of every image family as SSM public parameters. This is the preferred lookup: it is always current, works the same way in every commercial Region, and can be referenced directly from launch templates and CloudFormation.

```
/aws/service/canonical/ubuntu/<family>/<release>/stable/current/<arch>/hvm/<volume>/ami-id
```

| Segment | Values |
|---|---|
| `<family>` | `server`, `server-minimal`, `pro-server`, `pro-minimal` (EKS node images: see below) |
| `<release>` | Version number such as `24.04` or `26.04` (codename paths like `noble` also exist for some families) |
| `<arch>` | `amd64` (x86_64) or `arm64` (Graviton) |
| `<volume>` | `ebs-gp3` for 23.10 and later; `ebs-gp2` for 23.04 and earlier (22.04, 20.04) |

Getting the volume segment wrong returns `ParameterNotFound`. 22.04 uses `ebs-gp2` and 24.04 uses `ebs-gp3`.

```bash
# Current Ubuntu 24.04 LTS server, x86_64
aws ssm get-parameter --region ${region} \
  --name /aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id \
  --query Parameter.Value --output text

# Current Ubuntu Pro 24.04 LTS, Graviton
aws ssm get-parameter --region ${region} \
  --name /aws/service/canonical/ubuntu/pro-server/24.04/stable/current/arm64/hvm/ebs-gp3/ami-id \
  --query Parameter.Value --output text
```

Discover which releases and families exist in the Region rather than assuming (the path tree also holds dated, non-`current` entries):

```bash
aws ssm get-parameters-by-path --region ${region} --recursive \
  --path /aws/service/canonical/ubuntu/server \
  --query "Parameters[?contains(Name, '/stable/current/')].Name" --output text | tr '\t' '\n' | sort
```

**Use the parameter in place of a hardcoded AMI ID** wherever possible, so the image updates without code changes:

- Launch template / `run-instances`: `--image-id resolve:ssm:/aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id`
- CloudFormation: a parameter of type `AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>` defaulting to the path
- CDK: `ec2.MachineImage.fromSsmParameter('/aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id')`

Note the trade-off for Auto Scaling groups: `resolve:ssm` gives a new AMI on each launch, so a fleet can run mixed image versions until an instance refresh. For controlled rollouts, resolve the ID once, pin it in a numbered launch template version, and roll with instance refresh (see the `aws-compute` skill).

**EKS worker nodes:** Canonical publishes Ubuntu EKS node images per Kubernetes version:

```
/aws/service/canonical/ubuntu/eks/<release>/<k8s-version>/stable/current/<arch>/hvm/ebs-gp3/ami-id
/aws/service/canonical/ubuntu/eks-pro/<release>/<k8s-version>/stable/current/<arch>/hvm/ebs-gp3/ami-id
```

Match `<k8s-version>` to the cluster's version exactly. Cluster and node-group setup belong to the `aws-containers` skill.

## 2. Alternative: describe-images with the Canonical owner

Use this when you need image metadata or older serials. **Always pass the owner.** Without it, `*ubuntu*` filters also match community AMIs that anyone can publish.

```bash
aws ec2 describe-images --region ${region} --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*" "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].[ImageId,Name,UsageOperation]" --output text
```

AMI name patterns (the date serial at the end sorts correctly by `CreationDate`):

| Family | Name pattern |
|---|---|
| server | `ubuntu/images/hvm-ssd-gp3/ubuntu-<codename>-<version>-<arch>-server-<serial>` (`hvm-ssd/` for 22.04 and earlier) |
| minimal | `ubuntu-minimal/images/hvm-ssd-gp3/ubuntu-<codename>-<version>-<arch>-minimal-<serial>` |
| Pro server | `ubuntu-pro-server/images/hvm-ssd-gp3/ubuntu-<codename>-<version>-<arch>-pro-server-<serial>` |
| Pro minimal | `ubuntu-pro-minimal/images/hvm-ssd-gp3/ubuntu-<codename>-<version>-<arch>-pro-minimal-<serial>` |

Codenames: `jammy` (22.04), `noble` (24.04), `resolute` (26.04).

Owner account IDs: `099720109477` in commercial Regions. AWS GovCloud (US) and China Regions use different Canonical owner IDs. Resolve through the SSM parameters there instead of hardcoding an owner.

**How to tell standard from Pro from the AMI itself:** `UsageOperation` is `RunInstances` with `PlatformDetails` `Linux/UNIX` for standard images, and `RunInstances:0g00` with `PlatformDetails` `Ubuntu Pro Linux` for Pro images. The same fields on a running instance (`describe-instances`) show how it is billed.

## 3. Choose the release

- Default to the **newest LTS** that the user's software supports. List the available LTS releases from SSM (step 1) rather than assuming the latest one. Releases with an even year and `.04` are LTS.
- Use the previous LTS only when the software stack or vendor certification requires it.
- **Never use interim releases (`.10`, odd-year `.04`) for production.** They get 9 months of support, and their SSM parameters disappear from `current` after end of life.
- For a release past its standard support window (for example, 20.04 since May 2025), the instance stops getting security updates unless Ubuntu Pro is enabled. Recommend Pro or an upgrade, and say which.
- Check an existing instance's status with `pro security-status` (works without Pro attached) and `lsb_release -a`.

## 4. Launch settings that differ from Amazon Linux

The `launching-ec2-instance-with-best-practices` procedure applies unchanged except for these values:

| Setting | Ubuntu value |
|---|---|
| Image ID | SSM parameter from step 1 (or the resolved ID) |
| Root device in block device mappings | `/dev/sda1`, for example `[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":20,"VolumeType":"gp3","Encrypted":true}}]` |
| Root volume size | The images default to 8 GiB. Set 20 GiB or more for anything beyond a test instance, because apt caches, snaps, and kernels fill 8 GiB quickly. |
| Metadata options | `HttpTokens=required`; `HttpPutResponseHopLimit=2` if containers run on the host |
| SSH user | `ubuntu` |
| Instance types | Any current-generation type. Pick `amd64` images for Intel/AMD types and `arm64` images for Graviton types (`*g`, `*gd`, `*gn`). |
| Systems Manager | SSM Agent is preinstalled as a snap in server and Pro server images; attach an instance profile with `AmazonSSMManagedInstanceCore` (or use account-level DHMC) |

```bash
aws ec2 run-instances --region ${region} \
  --image-id resolve:ssm:/aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id \
  --instance-type m7i.large \
  --iam-instance-profile Name=${instance_profile} \
  --subnet-id ${subnet_id} --security-group-ids ${sg_id} \
  --metadata-options HttpTokens=required,HttpEndpoint=enabled,HttpPutResponseHopLimit=2 \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":20,"VolumeType":"gp3","Encrypted":true}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=${name}},{Key=os,Value=ubuntu-24.04}]'
```

## 5. User data on Ubuntu

cloud-init processes user data on Ubuntu. Prefer `#cloud-config` over shell scripts for package work: cloud-init's `packages` module handles apt locking and runs in the right order.

```yaml
#cloud-config
package_update: true
package_upgrade: true
packages:
  - nginx
  - unzip
runcmd:
  - systemctl enable --now nginx
```

When you must use a shell script:

```bash
#!/bin/bash
set -euo pipefail
export DEBIAN_FRONTEND=noninteractive NEEDRESTART_MODE=a
apt-get update
apt-get install -y -o Dpkg::Options::=--force-confold nginx
```

- `NEEDRESTART_MODE=a` stops `needrestart` (22.04 and later) from prompting interactively about service restarts.
- Logs: `/var/log/cloud-init-output.log`; status: `cloud-init status --long`.
- Never put secrets or Pro tokens in user data (see the SKILL.md guardrails). For Pro, see [ubuntu-pro.md](ubuntu-pro.md).

## 6. Verify after launch

```bash
aws ec2 describe-instances --region ${region} --instance-ids ${instance_id} \
  --query "Reservations[0].Instances[0].[State.Name,ImageId,PlatformDetails,UsageOperation,MetadataOptions.HttpTokens]" --output text
aws ssm describe-instance-information --region ${region} \
  --filters "Key=InstanceIds,Values=${instance_id}" --query "InstanceInformationList[0].[PingStatus,PlatformName,PlatformVersion]" --output text
```

On the instance (through Session Manager: `aws ssm start-session --target ${instance_id}`):

```bash
cloud-init status --long
lsb_release -ds
pro status          # "Ubuntu Pro Linux" instances should show attached services
```
