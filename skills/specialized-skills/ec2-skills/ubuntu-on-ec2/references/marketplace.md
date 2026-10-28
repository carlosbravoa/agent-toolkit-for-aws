# Ubuntu from AWS Marketplace

The Canonical-owned public AMIs ([finding-and-launching-ubuntu-amis.md](finding-and-launching-ubuntu-amis.md)) are the default. Use Marketplace when one of these applies:

| Reason | Example listings |
|---|---|
| **FIPS 140 certified images** ready to launch | Ubuntu Pro FIPS 20.04 LTS, Ubuntu Pro FIPS 22.04 LTS (FIPS with updates) |
| Purchasing through Marketplace: private offers, annual pricing, spend commitments, a single procurement channel | Canonical Ubuntu Pro listings, including the "Ubuntu Pro License Manager Upgrade" general listings |
| Specialized Canonical images | Ubuntu Pro with real-time kernel, Ubuntu Pro Minimal with CIS hardening, Ubuntu Pro for EKS |
| Third-party hardened or packaged Ubuntu | CIS-hardened images published by CIS, vendor appliances built on Ubuntu |

Sellers can change listings, versions, and prices. Confirm everything against the listing page.

## How Marketplace Ubuntu differs from Canonical's public AMIs

| | Canonical public AMIs | Marketplace listings |
|---|---|---|
| Owner | `099720109477` | `aws-marketplace` (`679593333241`), shared by all sellers |
| Billing | Standard: EC2 Linux rate. Pro: `RunInstances:0g00` on the EC2 bill. | Product code on the AMI; software charges billed by the seller through Marketplace (`UsageOperation` usually stays `RunInstances`) |
| Subscription | None | Accept the listing's terms once per AWS account before launching |
| Lookup | SSM parameters under `/aws/service/canonical/ubuntu/` | AMI ID from the listing's launch page, or by product code |

## 1. Identify a legitimate listing

Because every Marketplace AMI shares the `aws-marketplace` owner, **the owner ID alone doesn't prove who published an image.** Verify by:

1. Opening the listing in the AWS Marketplace console and confirming the seller (Canonical listings are sold by "Canonical Group Limited").
2. Taking the **product code** from the listing (or from your subscription in the console under **Manage subscriptions**).
3. Resolving AMIs by that product code, not by name:

```bash
aws ec2 describe-images --region ${region} --owners aws-marketplace \
  --filters "Name=product-code,Values=${product_code}" "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].[ImageId,Name,Architecture]" --output text
```

Product codes are per listing and often per architecture (x86_64 and Arm listings are separate). AWS documents the Canonical Ubuntu Pro product codes that support License Manager conversion in its [Linux conversion procedures](https://docs.aws.amazon.com/license-manager/latest/userguide/conversion-procedures-linux.html); for example, the general "Ubuntu Pro License Manager Upgrade" listing is `561bki7b434ls3uekcvfuzzph` (x86_64) and `3cuwhptbszkqo4x83mut8ri41` (Arm).

Name-based searches are fine for exploration (for example `"Name=name,Values=ubuntu-pro-fips-updates-server/*jammy*amd64*"`), but resolve the final AMI by product code.

## 2. Subscribe

- Subscribe in the AWS Marketplace console: open the listing, choose **Continue to Subscribe**, and accept the terms. This is a purchasing decision, so the user (or their procurement admin) must do it. Never subscribe on their behalf without explicit approval. If the organization uses a Private Marketplace, an administrator may need to approve the product first.
- The subscription is per AWS account. In AWS Organizations, every account that launches the AMI needs its own subscription (or the organization's license-sharing setup through AWS License Manager).
- Launching before subscribing fails with `OptInRequired`. That's the expected signal; don't try another AMI to get around it.

## 3. Launch and verify

Launch exactly like a Canonical AMI (root device `/dev/sda1`, IMDSv2 required, `ubuntu` user), using the AMI ID from step 1. Then verify:

```bash
aws ec2 describe-instances --region ${region} --instance-ids ${instance_id} \
  --query "Reservations[0].Instances[0].[ImageId,ProductCodes,PlatformDetails,UsageOperation]" --output json
```

```bash
# on the instance, for Canonical Ubuntu Pro listings
pro status               # should show attached through the cloud subscription
cat /proc/sys/crypto/fips_enabled   # FIPS listings: expect 1
```

## 4. Converting existing instances to a Marketplace Pro subscription

Running Ubuntu LTS instances can be moved onto a Canonical Marketplace Pro subscription with License Manager, without relaunching. The conversion attaches the product code; the usage operation stays `RunInstances`.

```bash
aws license-manager create-license-conversion-task-for-resource --region ${region} \
  --resource-arn "arn:aws:ec2:${region}:${account_id}:instance/${instance_id}" \
  --source-license-context "UsageOperation=RunInstances" \
  --destination-license-context "UsageOperation=RunInstances,ProductCodes=[{ProductCodeType=marketplace,ProductCodeId=${product_code}}]"
```

- The Marketplace subscription must be active first, and the listing must support the instance's architecture and instance type. Convert x86_64 and Arm instances in separate tasks with the matching listing.
- The conversion only attaches billing and entitlement. Flavor-specific setup (FIPS, CIS hardening, real-time kernel) must still be enabled with `pro enable`.
- It's one-way, like the AWS-billed conversion (see [ubuntu-pro.md](ubuntu-pro.md)).

## 5. Golden images from Marketplace AMIs

- A Marketplace AMI ID (or product ID) is a valid Image Builder parent image. The build account must be subscribed, or the build fails at launch with `OptInRequired`.
- **Derived AMIs keep the product code.** Every instance launched from them is billed for the listing, and every account that launches them needs its own subscription.
- AMIs with Marketplace product codes can't be made public. Share only with specific accounts that are subscribed.
- A pinned Marketplace AMI ID never updates. For a recurring pipeline, re-resolve the latest AMI by product code on a schedule (for example, publish it to an SSM parameter under `/imagebuilder/` and use that as the parent).

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `OptInRequired` on launch or Image Builder build | Account isn't subscribed | Subscribe in the Marketplace console (the user or an admin) |
| Subscription succeeds but the product is blocked | Private Marketplace doesn't allow the product | Ask the Private Marketplace admin to add it |
| `describe-images` by name returns unexpected publishers | Many sellers share the `aws-marketplace` owner | Resolve by product code from the verified listing |
| Unsupported instance type at launch | The listing restricts instance types or architecture | Check the listing's usage information; use the matching x86_64 or Arm listing |
| Unexpected charges on a derived AMI | The product code carries into derived images | Expected. Rebuild from a Canonical public AMI if Marketplace billing isn't wanted |
