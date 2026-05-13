<p align="center">
  <img src="assets/apfelbauer-logo.png" alt="apfelbauer" width="220">
</p>

<h1 align="center">apfelbauer</h1>
<p align="center"><em>Living off the land in the Apple orchard.</em></p>

---

A small toolkit for macOS fleet admins thinking about Apple Intelligence enablement state. Three unprivileged commands toggle the master Apple Intelligence opt-in (on, off, and a status check). The same commands work whether or not a user, admin, or MDM has tried to turn the feature off.

Verified on macOS Tahoe 26.3 through 26.5 developer beta 25F71 (Apple Silicon).

**Prerequisite:** an Apple Account must be signed in (System Settings → Apple Account). The toggle is keyed by the account's DSID, which is read from `~/Library/Preferences/MobileMeAccounts.plist`. On a Mac with no Apple Account signed in, `defaults read MobileMeAccounts` returns `Domain MobileMeAccounts does not exist` and the chain does not apply. Apple Intelligence itself is unavailable in that state, so there is no toggle to flip.

---

## Enable Apple Intelligence

```bash
DSID=$(defaults read MobileMeAccounts | grep "AccountDSID" | grep -v "Alternate" | head -1 | sed 's/.*= *//; s/;.*//' | tr -d ' ')
defaults write com.apple.CloudSubscriptionFeatures.optIn "$DSID" -int 1
defaults write com.apple.CloudSubscriptionFeatures.optIn device -int 1
defaults write com.apple.CloudSubscriptionFeatures.optIn opted_in_buddy -int 1
defaults write com.apple.CloudSubscriptionFeatures.optIn auto_opt_in -int 1
notifyutil -p com.apple.CloudSubscriptionFeatures.OptIn.Changed
```

Wait ~8–12 seconds for the model to load (longer on first-time enablement; the on-device model has to download).

## Disable Apple Intelligence

```bash
DSID=$(defaults read MobileMeAccounts | grep "AccountDSID" | grep -v "Alternate" | head -1 | sed 's/.*= *//; s/;.*//' | tr -d ' ')
defaults write com.apple.CloudSubscriptionFeatures.optIn "$DSID" -int 0
defaults write com.apple.CloudSubscriptionFeatures.optIn device -int 0
notifyutil -p com.apple.CloudSubscriptionFeatures.OptIn.Changed
```

Takes effect within ~5 seconds.

## Check status

```bash
swift -e 'import FoundationModels; if #available(macOS 26.0, *) { print("\(SystemLanguageModel.default.availability)") }'
```

Prints `available` when Apple Intelligence is on, or an `unavailable(...)` reason otherwise.

---

## Why this matters

The Apple Intelligence on/off state lives in a user-domain `UserDefaults` preference (`~/Library/Preferences/com.apple.CloudSubscriptionFeatures.optIn`). Posting the public Darwin notification `com.apple.CloudSubscriptionFeatures.OptIn.Changed` makes the daemon re-read the preference without a logout or restart. Both surfaces are reachable from any unprivileged process running as the user.

That has three operational consequences for a managed fleet:

1. **The on/off state is not observable from the management plane.** An MDM console reports what it pushed, not what the daemon currently believes. There is no system log when the master toggle flips and no MDM event for an opt-in plist change.
2. **The on/off state is not enforceable against the user's own software on unsupervised Macs.** A user-domain preference is writable by any user-context process. The per-feature restriction keys (`allowWritingTools`, `allowMailSummary`, etc.) gate UI surfaces; they do not change the master toggle's state.
3. **On supervised, DEP-enrolled Macs the answer is unverified.** Whether managed-preference-domain values take precedence over user-domain values for this key on supervised devices is the test that decides whether MDM holds the state on a managed fleet. If you run Jamf, Kandji, Intune, or Workspace ONE and can test it, please open an issue or share results.

Apple's position is that the enablement preference is a feature-rollout control, not a security or privacy boundary. That's a defensible product framing. The transparency gap (no audit event, no MDM event, no runtime telemetry) is the part that matters for regulated postures (HIPAA, PCI, FedRAMP, government endpoint policies that distinguish AI-processed from non-AI-processed data).

One real mitigation worth noting: on Macs where no Apple Account is signed in, Apple Intelligence is unavailable and this chain does not apply. For hardened or kiosk endpoints where on-device generative AI is not part of the workflow, simply not signing in to iCloud closes the entire surface. For everything else (the default consumer configuration, BYOD fleets, anyone using iCloud Mail / Drive / Photos / Continuity), the chain applies as soon as the account is signed in.

---

## Defender artifacts

- [`detection/`](detection/) — starter EDR rules for SentinelOne, CrowdStrike, Elastic Security, and Splunk. Untested at production scale.
- [`mitigations/`](mitigations/) — starter MDM configuration profiles for Kandji, Intune, and JAMF Pro. Untested on supervised/DEP-enrolled fleets.

Both are provided as-is. Review and tune in your own environment before deploying. See per-directory READMEs for caveats and open questions.

---

## Responsible use

Defensive security research and authorized testing only. Do not use against systems you do not own or have explicit authorization to test.

---

<p align="center">
  <strong>Author:</strong> Scott Moore — Independent Security Researcher<br>
  <strong>License:</strong> <a href="LICENSE">MIT</a>
</p>
