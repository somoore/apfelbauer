# Kandji Deployment Guide

Deploy the Apple Intelligence restriction profile via Kandji.

## Important: Untested by the Author — Treat as a Starter Profile

This profile is built from Apple's published MDM restriction keys. **I do not have a Kandji tenant and have not deployed this against a supervised, DEP-enrolled Mac.** Review the payload, regenerate the UUIDs and identifiers (see the parent [mitigations README](../README.md)), and pilot on a staging Blueprint before deploying to production. If you find an issue with the payload or the keys behave differently than expected on your devices, please open an issue with what you observed — the open questions in the parent README are the specific tests that would be most useful to validate against Kandji.

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

1. Log in to the Kandji web app
2. Navigate to **Library** in the left sidebar
3. Click **Add New** > **Custom Profile**
4. Click **Upload Profile** and select `disable-apple-intelligence.mobileconfig`
5. Kandji will parse and display the payload contents for review

### 2. Configure the Library Item

1. Set the **Name** to `Disable Apple Intelligence - apfelbauer`
2. Optionally add a **Description**: `Disables Apple Intelligence features. Part of apfelbauer defense-in-depth. Does not prevent FoundationModels framework access.`
3. Under **Install Type**, select **Device Level** (recommended for consistent enforcement)
4. Click **Save**

### 3. Scope to a Blueprint

1. Navigate to **Blueprints** in the left sidebar
2. Select the target Blueprint (or create a new one for testing)
3. Click **Add Library Item**
4. Search for `Disable Apple Intelligence` and add it
5. The profile will deploy to all Macs assigned to this Blueprint on next check-in

### 4. Verify Deployment

1. Navigate to a target device in **Devices**
2. Click the device and go to the **Library Items** tab
3. Confirm the profile shows as **Installed**
4. On the device, open **System Settings** > **Apple Intelligence & Siri** and verify features are managed/disabled

## Removal

To remove the profile:
1. Remove the Library Item from the Blueprint, or
2. Delete the Custom Profile from the Library

Kandji will remove the profile from devices on next check-in.

## Notes

- Kandji Custom Profiles are installed as MDM-managed profiles and cannot be removed by the user.
- Profile changes propagate on the next device check-in (typically within 15 minutes, or force a sync from the device record).
- Test on a staging Blueprint before broad deployment.
