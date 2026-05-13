# Microsoft Intune Deployment Guide

Deploy the Apple Intelligence restriction profile via Microsoft Intune.

## Important: Untested by the Author — Treat as a Starter Profile

This profile is built from Apple's published MDM restriction keys. **I do not have a Microsoft Intune tenant and have not deployed this against a supervised, DEP-enrolled Mac.** Review the payload, regenerate the UUIDs and identifiers (see the parent [mitigations README](../README.md)), and assign to a small device group for piloting before broad rollout. If you find an issue with the payload or the keys behave differently than expected, please open an issue — the open questions in the parent README are the specific tests that would be most useful to validate against Intune.

**Use at your own risk.** Provided as-is, with no warranty of any kind. See [LICENSE](../../LICENSE).

This profile disables Apple Intelligence **feature surfaces** via managed
preferences and the documented restriction keys. It does **not** prevent any
process from loading `FoundationModels.framework` directly; no
framework-level MDM control exists. The preference
side of the profile can also be bypassed by an attacker using `defaults write`
and a Darwin notification on unsupervised devices; supervised-device behavior
under managed-preference precedence is one of the open community-test questions.
**Deploy EDR detection rules alongside this profile** regardless of supervision
state. See [detection/](../../detection/) for rules covering the bypass.

## Deployment Steps

### 1. Upload the Profile

1. Sign in to the [Microsoft Intune admin center](https://intune.microsoft.com)
2. Navigate to **Devices** > **Configuration profiles**
3. Click **+ Create profile**
4. Set:
   - **Platform**: macOS
   - **Profile type**: Templates
   - **Template name**: Custom
5. Click **Create**

### 2. Configure the Profile

1. **Basics** tab:
   - **Name**: `Disable Apple Intelligence - apfelbauer`
   - **Description**: `Disables Apple Intelligence features. Part of apfelbauer defense-in-depth. Does not prevent FoundationModels framework access.`
2. **Configuration settings** tab:
   - **Custom configuration profile name**: `Disable Apple Intelligence`
   - **Deployment channel**: **Device channel** (recommended)
   - Click **Select a file** and upload `disable-apple-intelligence.mobileconfig`
3. Click **Next**

### 3. Scope and Assign

1. **Scope tags** tab: Add any relevant scope tags for your organization
2. **Assignments** tab:
   - Under **Included groups**, click **Add groups**
   - Select the Azure AD / Entra ID group containing target Mac devices
   - Optionally add exclusion groups for test devices or developer machines
3. **Review + create** tab: Review settings and click **Create**

### 4. Verify Deployment

1. Navigate to the profile in **Devices** > **Configuration profiles**
2. Click the profile and check the **Device status** tab
3. Verify devices show **Succeeded** status
4. On a target device, open **System Settings** > **Apple Intelligence & Siri** and verify features are managed/disabled

## Troubleshooting

| Status | Action |
|--------|--------|
| **Pending** | Wait for the next device check-in (up to 8 hours, or trigger sync from Company Portal) |
| **Error** | Check the error code in the profile status; common issues include unsupported macOS version |
| **Not applicable** | Verify the device is running macOS Tahoe 26.x and is in the assigned group |

## Removal

1. Navigate to the profile in **Devices** > **Configuration profiles**
2. Either:
   - Remove the group assignment to stop deploying to those devices, or
   - Delete the profile entirely
3. The profile will be removed from devices on next check-in

## Notes

- Intune custom profiles for macOS are deployed as MDM-managed configuration profiles.
- Device check-in interval is approximately 8 hours by default; users can force sync via Company Portal.
- The profile applies at the device level and cannot be removed by the end user.
