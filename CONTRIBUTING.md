# Contributing to apfelbauer

Contributions that help defenders are welcome.

## What We're Looking For

- **Detection rules** for additional EDR/SIEM platforms — especially Jamf Protect, Microsoft Defender for Endpoint, Carbon Black, and Wazuh
- **MDM profiles** for additional management platforms
- **Reproduction results** on different macOS versions or hardware
- **Corrections** to technical claims (with evidence)

## How to Contribute

1. Fork the repository
2. Create a feature branch (`git checkout -b detection/carbon-black`)
3. Include the platform, version tested, and evidence format
4. Submit a pull request with a clear description

## Guidelines

- Detection rules should include the vendor, query language version, and testing notes
- MDM profiles should be validated with `plutil -lint` before submission
- Reproduction results should specify exact macOS version and build number
- Do not submit credentials, DSID values, or other sensitive data

## Code of Conduct

This is a security research project. Be professional, accurate, and responsible.
