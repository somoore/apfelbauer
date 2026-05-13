# JAMF Pro Deployment Guide

Deploy the Apple Intelligence restriction profile via JAMF Pro.

## Important: Untested by the Author — Treat as a Starter Profile

This profile is built from Apple's published MDM restriction keys. **I do not have a Jamf Pro instance and have not deployed this against a supervised, DEP-enrolled Mac.** Review the payload, regenerate the UUIDs and identifiers (see the parent [mitigations README](../README.md)), and scope to a Smart Group for piloting before organization-wide rollout. If you find an issue with the payload or the keys behave differently than expected, please open an issue — the open questions in the parent README are the specific tests that would be most useful to validate against Jamf, since Jamf admins are the largest community in a position to answer the managed-domain precedence question.

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

1. Log in to your JAMF Pro instance
2. Navigate to **Computers** > **Configuration Profiles**
3. Click **+ New**
4. Click **Upload** and select `disable-apple-intelligence.mobileconfig`
5. JAMF Pro will parse the profile and display the payload for review

### 2. Configure the Profile

1. **General** tab:
   - **Name**: `Disable Apple Intelligence - apfelbauer`
   - **Description**: `Disables Apple Intelligence features. Part of apfelbauer defense-in-depth. Does not prevent FoundationModels framework access.`
   - **Category**: Select or create `Security` (optional)
   - **Distribution Method**: `Install Automatically`
   - **Level**: `Computer Level`
2. Review the payload tabs to confirm the restriction keys are set correctly

### 3. Scope to Smart Groups

1. Click the **Scope** tab
2. Under **Targets**, click **+ Add**
3. Select the target scope:
   - **All Computers** for organization-wide deployment, or
   - A **Smart Group** (recommended) for staged rollout
4. Example Smart Group criteria for macOS Tahoe:
   - `Operating System Version` is `like` `26.`
   - `Architecture Type` is `Apple Silicon (ARM64)`
5. Optionally add **Exclusions** for developer machines or test devices

### 4. Deploy

1. Click **Save**
2. The profile will deploy to scoped devices on the next JAMF check-in
3. To force immediate deployment: **Computers** > select device > **Management** > **Send MDM Command** > **Install Configuration Profile**

### 5. Verify Deployment

1. Navigate to a target device in **Computers** > **Search Inventory**
2. Click the device and go to the **Configuration Profiles** tab
3. Confirm the profile shows as installed
4. On the device, open **System Settings** > **Apple Intelligence & Siri** and verify features are managed/disabled

## Smart Group Examples

### All macOS Tahoe Macs

| Criteria | Operator | Value |
|----------|----------|-------|
| Operating System Version | like | 26. |

### Apple Silicon Macs on Tahoe (required for Apple Intelligence)

| Criteria | Operator | Value |
|----------|----------|-------|
| Operating System Version | like | 26. |
| Architecture Type | is | Apple Silicon (ARM64) |

## Removal

1. Navigate to the Configuration Profile in **Computers** > **Configuration Profiles**
2. Either:
   - Modify the Scope to remove target groups, or
   - Delete the profile entirely
3. JAMF Pro will remove the profile from devices on next check-in

## Notes

- JAMF Pro configuration profiles are MDM-managed and cannot be removed by the end user.
- Default check-in frequency is every 15 minutes (configurable in **Settings** > **Computer Management** > **Check-In**).
- If using JAMF Connect or JAMF Protect alongside JAMF Pro, consider adding the EDR detection rules from [detection/](../../detection/) as Jamf Protect analytics.
- JAMF Protect customers can import threat detections directly; see the Splunk rules in [detection/splunk/](../../detection/splunk/) as a reference for Jamf Protect custom analytics.
