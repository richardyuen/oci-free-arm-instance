---
title: "Convert OCI Always Free A1 retry workflow to Feishu notifications"
date: 2026-05-23
category: workflow-issues
module: oci-free-arm-instance
problem_type: workflow_issue
component: development_workflow
severity: medium
applies_when:
  - "Adapting heykapil/oci-free-arm-instance for OCI Always Free A1 provisioning"
  - "Using GitHub Actions with OCI secrets and Feishu/Lark custom bot notifications"
  - "Suppressing expected OCI capacity misses while preserving actionable alerts"
symptoms:
  - "Initial fork target lacked the runnable GitHub Actions workflow"
  - "OCI CLI failed on malformed tenancy OCID before reaching instance creation"
  - "OCI rejected incomplete SSH public key and x86 image values during launch"
  - "Expected Out of host capacity retries would have spammed Feishu every five minutes"
root_cause: incomplete_setup
resolution_type: workflow_improvement
tags: [oci, github-actions, feishu, lark, always-free, a1-flex, webhook]
---

# Convert OCI Always Free A1 retry workflow to Feishu notifications

## Context

The goal was to fork `oci-free-arm-instance`, keep its GitHub Actions retry loop for creating an OCI Always Free `VM.Standard.A1.Flex` instance, and replace Discord notifications with Feishu/Lark custom bot notifications.

The first fork target, `maoucodes/oci-free-arm-instance`, looked related but had deleted the workflow. The durable base had to be the original `heykapil/oci-free-arm-instance` repo, then the local and remote fork were recreated from that source.

After the Feishu workflow existed, live GitHub Actions runs exposed the real setup sequence:

- `tenancy | malformed | this must be an OCID` meant `OCI_CLI_TENANCY` was not the raw `ocid1.tenancy...` value.
- `Invalid ssh public key type` meant `SSH_PUBLIC_KEY` contained only the trailing machine-name comment, not the full `ssh-rsa` or `ssh-ed25519` public key line.
- `Shape VM.Standard.A1.Flex is not valid for image` meant `IMAGE_ID` was an x86 image; A1.Flex requires an Arm/aarch64 image in the same region.
- `Out of host capacity` meant the configuration was finally working and OCI simply lacked current A1.Flex capacity.

## Guidance

Start from the original workflow-bearing repository, then keep deployment-specific OCI values in GitHub Actions secrets instead of hardcoding region, image, subnet, or availability domain.

The workflow environment should read the live setup from repository secrets:

```yaml
env:
  COMPARTMENT_ID: ${{ secrets.OCI_COMPARTMENT_ID }}
  IMAGE_ID: ${{ secrets.IMAGE_ID }}
  SUBNET_ID: ${{ secrets.OCI_SUBNET_ID }}
  AD_NAME: ${{ secrets.AD_NAME }}
  OCI_CLI_REGION: ${{ secrets.OCI_CLI_REGION }}
  OCI_CLI_USER: ${{ secrets.OCI_CLI_USER }}
  OCI_CLI_TENANCY: ${{ secrets.OCI_CLI_TENANCY }}
  OCI_CLI_FINGERPRINT: ${{ secrets.OCI_CLI_FINGERPRINT }}
  OCI_CLI_KEY_CONTENT: ${{ secrets.OCI_CLI_KEY_CONTENT }}
  SUPPRESS_LABEL_WARNING: "True"
```

Replace the Discord action with a Python standard-library Feishu webhook step. This avoids adding a third-party Feishu action or external relay service and supports both unsigned and signed Feishu bots:

```python
if secret:
    timestamp = str(int(time.time()))
    string_to_sign = f"{timestamp}\n{secret}".encode("utf-8")
    sign = base64.b64encode(
        hmac.new(string_to_sign, digestmod=hashlib.sha256).digest()
    ).decode("utf-8")
    payload["timestamp"] = timestamp
    payload["sign"] = sign
```

Keep expected capacity misses silent while preserving important notifications:

```yaml
if: always() && (steps.create_vm.outcome == 'success' || !contains(steps.create_vm.outputs.log_data, 'Out of host capacity'))
```

That condition keeps the five-minute retry loop running, records every attempt in GitHub Actions, and sends Feishu only for successful provisioning or non-capacity failures that need attention.

## Why This Matters

For OCI Always Free A1.Flex, `Out of host capacity` is the normal waiting state, not a bug. Treating it like an alert creates notification fatigue and makes the eventual success message easier to miss.

The opposite mistake is also risky: suppressing all failures would hide configuration problems. The workflow should separate expected retry noise from actionable failures:

- Capacity unavailable: keep retrying, no Feishu message.
- Bad OCID, wrong SSH key, wrong image, webhook error, or other setup failure: send Feishu.
- `PROVISIONING` success: send Feishu and disable the workflow immediately.

## When to Apply

- When adapting `heykapil/oci-free-arm-instance` rather than writing a fresh OCI retry workflow.
- When the notification channel is Feishu/Lark instead of Discord.
- When repository owners configure OCI values through GitHub Actions secrets.
- When `Out of host capacity` is expected and should not page or spam the user.
- When success must still be visible because the workflow should be disabled after the VM is created.

## Examples

Required secrets for this workflow:

```text
OCI_COMPARTMENT_ID
OCI_SUBNET_ID
AD_NAME
IMAGE_ID
OCI_CLI_REGION
OCI_CLI_USER
OCI_CLI_TENANCY
OCI_CLI_FINGERPRINT
OCI_CLI_KEY_CONTENT
SSH_PUBLIC_KEY
FEISHU_WEBHOOK_URL
```

Optional Feishu signing secret:

```text
FEISHU_WEBHOOK_SECRET
```

Correct `OCI_CLI_TENANCY` shape:

```text
ocid1.tenancy.oc1..xxxxxxxx
```

Correct `SSH_PUBLIC_KEY` shape:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... nemo@MacBook
```

Correct image rule:

```text
IMAGE_ID must be an Arm/aarch64 image OCID for the same region as OCI_CLI_REGION.
Do not use a normal x86 Ubuntu image for VM.Standard.A1.Flex.
```

Success handling:

```text
If the log contains "lifecycle-state": "PROVISIONING", disable the workflow immediately.
Otherwise it may keep trying every five minutes and attempt another VM.
```

## Related

- `.github/workflows/create-vm.yml`
- `README.md`
- Upstream source: `heykapil/oci-free-arm-instance`
- Initial wrong fork target: `maoucodes/oci-free-arm-instance`
- No prior `docs/solutions/` entries existed in this repo when this learning was written.
