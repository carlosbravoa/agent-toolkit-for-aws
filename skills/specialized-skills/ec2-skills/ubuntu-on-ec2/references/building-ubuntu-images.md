# Building Ubuntu and Ubuntu Pro golden AMIs

This file covers only what is Ubuntu-specific. The `amazon-ec2-image-builder` skill owns the general flow (build role, recipe, infrastructure configuration, pipeline, distribution, troubleshooting). Follow its `creating-images.md`, and substitute the values below.

Contents: [Parent image](#1-parent-image) · [Recipe](#2-recipe-root-volume-on-devsda1) · [Components](#3-components) · [Writing apt components](#4-writing-apt-based-components) · [Pro-specific steps](#5-ubuntu-pro-golden-images) · [Hardening](#6-hardening-cis-and-disa-stig) · [Patching](#7-keeping-the-image-current) · [Manual create-image](#8-without-image-builder-manual-create-image) · [Troubleshooting](#9-ubuntu-specific-build-failures)

## 1. Parent image

Pick a parent that tracks new releases, so scheduled pipelines pick up Canonical's refreshed images (Canonical publishes updated images regularly, including for security fixes).

| Image | `--parent-image` value |
|---|---|
| Ubuntu Server LTS, Amazon-managed Image Builder image | `arn:aws:imagebuilder:${region}:aws:image/ubuntu-server-24-lts-x86/x.x.x` (also `-arm64`; 26.04 is `ubuntu-server-26-04-lts-x86` / `-arm64`; 22.04 is `ubuntu-server-22-lts-x86` / `-arm64`) |
| Ubuntu Server, straight from Canonical's SSM parameter | `ssm:/aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id` |
| **Ubuntu Pro** | `ssm:/aws/service/canonical/ubuntu/pro-server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id` (there is no Amazon-managed Ubuntu Pro server image) |
| Ubuntu Pro Minimal | `ssm:/aws/service/canonical/ubuntu/pro-minimal/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id` |
| Marketplace listing (FIPS, CIS-hardened, and others) | The listing's AMI ID or Marketplace product ID, after subscribing (see [marketplace.md](marketplace.md)) |

- Confirm the Amazon-managed names in the target Region before using them: `aws imagebuilder list-images --owner Amazon --filters "name=platform,values=Linux" --query "imageVersionList[?contains(name, 'Ubuntu')].[name,arn]" --output text`. Names containing a vendor suffix (for example `ubuntu-2404-lts-noble-...`) are Marketplace-backed and need a subscription.
- **Prefer the Canonical SSM parameter when you need Pro, minimal, or a release the Amazon-managed catalog hasn't added yet.** Public `/aws/service/*` parameters are readable by Image Builder at schedule time, so `EXPRESSION_MATCH_AND_DEPENDENCY_UPDATES_AVAILABLE` pipelines detect new Canonical images.
- Keep architecture consistent: `amd64` parent → x86_64 instance types and components; `arm64` parent → Graviton types.

## 2. Recipe: root volume on `/dev/sda1`

Ubuntu AMIs use `/dev/sda1` as the root device and default to an 8 GiB root volume. The Image Builder skill's example uses `/dev/xvda` (Amazon Linux). On an Ubuntu parent, that adds a second, empty volume and leaves the root volume unencrypted and unresized.

```bash
aws imagebuilder create-image-recipe --name ${name}-recipe --semantic-version 1.0.0 \
  --parent-image "ssm:/aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id" \
  --components "componentArn=${component_arn_1}" "componentArn=${component_arn_2}" \
  --block-device-mappings '[{"deviceName":"/dev/sda1","ebs":{"volumeSize":20,"volumeType":"gp3","encrypted":true,"kmsKeyId":"${kms_key_arn}","deleteOnTermination":true}}]' \
  --region ${region}
```

Drop `kmsKeyId` to use the default EBS key. Keep it if the AMI will be shared across accounts.

## 3. Components

Check Amazon-managed components first: `aws imagebuilder list-components --owner Amazon --region ${region}`. Check each component's `supportedOsVersions` for the target Ubuntu release; new LTS releases are added over time.

| Need | Amazon-managed component | Ubuntu notes |
|---|---|---|
| OS patching during the build | `update-linux` | Runs apt upgrades and handles its own reboot. Add it to pick up updates published since the parent image's serial. |
| AWS CLI v2 | `aws-cli-version-2-linux` | Only if `command -v aws` is empty on the parent; skip it when the CLI is already present |
| CloudWatch agent | `amazon-cloudwatch-agent-linux` | |
| Docker | `docker-ce-ubuntu` | |
| DISA-STIG | `stig-build-linux` | Lists Ubuntu 18, 20, 22, and 24 in `supportedOsVersions`; on Pro, prefer USG (section 6) |
| SCAP compliance scan | `scap-compliance-checker-linux` | |
| apt repository check (test phase) | `apt-repository-test-linux` | |

SSM Agent ships preinstalled as a snap on Ubuntu server images, so Image Builder doesn't install it and doesn't remove it after the build (`systemsManagerAgent.uninstallAfterBuild` only affects an agent that Image Builder installed itself).

## 4. Writing apt-based components

```yaml
name: ${component_name}
description: Installs ${software} on Ubuntu
schemaVersion: 1.0
phases:
  - name: build
    steps:
      - name: WaitForBoot
        action: ExecuteBash
        inputs:
          commands:
            - cloud-init status --wait || true
            - systemd-run --property="After=apt-daily.service apt-daily-upgrade.service" --wait /bin/true
      - name: Install
        action: ExecuteBash
        inputs:
          commands:
            - export DEBIAN_FRONTEND=noninteractive NEEDRESTART_MODE=a
            - apt-get update
            - apt-get install -y -o Dpkg::Options::=--force-confold ${packages}
  - name: validate
    steps:
      - name: Verify
        action: ExecuteBash
        inputs:
          commands:
            - dpkg -s ${main_package} >/dev/null
```

- The `WaitForBoot` step prevents the most common Ubuntu build failure: `unattended-upgrades` holding the dpkg lock right after boot.
- `NEEDRESTART_MODE=a` and `--force-confold` stop interactive prompts that otherwise hang the step until it times out.
- For third-party apt repositories, install the signing key under `/etc/apt/keyrings/` and reference it with `signed-by=`. `apt-key` is deprecated.
- Snaps work in builds (`snap install ...`). They update themselves on running instances unless you pin or hold them (`snap refresh --hold`).
- To reboot mid-build (for example after a kernel update), exit with code `194` and guard the step with a marker file, as described in the Image Builder skill.

## 5. Ubuntu Pro golden images

**From a Pro parent (recommended):**

- Every instance launched from the output AMI is billed as Ubuntu Pro (`UsageOperation` `RunInstances:0g00`, platform "Ubuntu Pro Linux") and attaches itself on first boot. Confirm this on the output AMI with `aws ec2 describe-images --image-ids ${ami_id} --query "Images[0].[PlatformDetails,UsageOperation]"`.
- The build instance can use Pro services during the build (ESM packages, `pro enable fips-updates`, `pro enable usg`) if the build subnet has outbound access to the Canonical endpoints listed in [ubuntu-pro.md](ubuntu-pro.md). Check `pro status` early in the build and fail clearly if the instance isn't attached.

**Detach before the image is captured.** Add this as the last build step so clones don't inherit the build instance's Pro machine identity:

```yaml
      - name: PrepareForImaging
        action: ExecuteBash
        inputs:
          commands:
            - pro detach --assume-yes || true
            - cloud-init clean --logs
```

Instances launched from a Pro AMI attach again automatically. Image Builder already resets `machine-id` and SSH host keys during its own cleanup.

**From a standard parent with a contract token:**

- Fetch the token from Secrets Manager inside the component, never inline (pattern in [ubuntu-pro.md](ubuntu-pro.md), path C). The build role then needs `secretsmanager:GetSecretValue` on that one secret.
- Detach as the final step (same as above). The output AMI must not contain the token; instances attach at boot using the same Secrets Manager pattern in user data.
- Build logs capture command output, so never echo the token.

**FIPS images:** run `pro enable fips-updates --assume-yes`, exit `194` to reboot into the FIPS kernel (guard with a marker file), then validate `cat /proc/sys/crypto/fips_enabled` returns `1` in a test phase. Alternatively, start from a Canonical FIPS Marketplace listing ([marketplace.md](marketplace.md)).

## 6. Hardening: CIS and DISA-STIG

| Option | When |
|---|---|
| **USG on Ubuntu Pro** (`pro enable usg`, then `usg fix <profile>`) | Canonical's supported tooling for CIS and DISA-STIG on Ubuntu. Profiles are maintained per release. |
| `stig-build-linux` Amazon-managed component | DISA-STIG without Pro; check `supportedOsVersions` |
| CIS-hardened Marketplace image as the parent | An already-hardened, CIS-published starting point |

USG build step (Pro parent):

```yaml
      - name: Harden
        action: ExecuteBash
        inputs:
          commands:
            - export DEBIAN_FRONTEND=noninteractive
            - pro enable usg
            - apt-get install -y usg
            - usg fix cis_level1_server
            - usg audit cis_level1_server
```

- Hardening can mount `/tmp` with `noexec`, which breaks AWSTOE (`Unable to bootstrap TOE`). Set the recipe's `--working-directory` to an executable path such as `/opt/imagebuilder`.
- Hardening can also stop SSM Agent from connecting, which fails the test stage. Keep a test-phase step that confirms the instance is still managed, and check `usg` tailoring options if a rule must be excluded.
- Archive the `usg audit` report (written under `/var/lib/usg/`) to S3 in the test phase if auditors need evidence.

## 7. Keeping the image current

- A scheduled pipeline with an SSM or `x.x.x` Ubuntu parent rebuilds when Canonical publishes a new image. Add `update-linux` to also pick up updates published since that image's serial.
- Distribute through a launch template or an SSM parameter under `/imagebuilder/`, as described in the Image Builder skill, so Auto Scaling groups roll onto each new AMI.
- Plan the move to the next LTS as a new recipe (new parent path), not an in-place `do-release-upgrade` inside the build.

## 8. Without Image Builder: manual create-image

For a one-off image from a configured instance:

```bash
# on the instance
sudo pro detach --assume-yes        # only if a contract token was attached
sudo cloud-init clean --logs --machine-id
sudo rm -f /etc/ssh/ssh_host_*      # regenerated on next boot
history -c
# from your workstation
aws ec2 create-image --instance-id ${instance_id} --name "${name}-$(date +%Y%m%d)" \
  --tag-specifications "ResourceType=image,Tags=[{Key=os,Value=ubuntu-24.04}]"
```

Leave `--no-reboot` off so the file system is consistent. Instances launched from an image whose source was billed as Pro inherit Pro billing. Sharing, deprecating, and retiring the AMI belong to the `aws-compute` skill.

## 9. Ubuntu-specific build failures

| Symptom in `/aws/imagebuilder/<image-name>` logs | Cause | Fix |
|---|---|---|
| `Could not get lock /var/lib/dpkg/lock-frontend` | `unattended-upgrades` running after boot | Add the `WaitForBoot` step (section 4) |
| Step hangs until timeout during apt install | Interactive prompt (`needrestart`, a conffile question, debconf) | `DEBIAN_FRONTEND=noninteractive NEEDRESTART_MODE=a` and `--force-confold` |
| Root volume still 8 GiB or unencrypted; an extra volume appears | Block device mapping used `/dev/xvda` | Use `/dev/sda1` (section 2) |
| `Unable to bootstrap TOE` after hardening | `/tmp` mounted `noexec` | Set the recipe working directory to an executable path |
| `pro enable` fails: "not attached" or connection errors | Standard parent without a token, or no route to Canonical endpoints | Use a Pro parent or attach a token; add a NAT route or proxy for the build subnet |
| Launch fails with `OptInRequired` | Marketplace parent without a subscription | Subscribe to the listing in this account |
| Output instances aren't billed as Pro | Parent was a standard image | Use the `pro-server` SSM parent, or convert instances (see [ubuntu-pro.md](ubuntu-pro.md)) |
