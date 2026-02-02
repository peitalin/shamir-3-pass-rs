# Security Policy

## Supported Versions

Security fixes are provided on a best-effort basis for the latest released version and the `main` branch.

## Reporting a Vulnerability

Please do not open a public GitHub issue for potential security vulnerabilities.

- Preferred: report privately to `dev@web3authn.org` or `n6378056@gmail.com`

When reporting, include:

- Affected version(s)
- A clear description of the issue and impact
- A minimal reproduction (code or steps)
- Any suggested fix/mitigation, if you have one

## Disclosure Process

This project aims to:

- Acknowledge receipt of reports as soon as practical
- Work with the reporter on a fix and coordinated disclosure timeline
- Publish a release and advisory once a fix is available

## Cryptography Disclaimer

Cryptographic software is easy to misuse. This crate:

- Has not been professionally audited
- Uses big integer operations that are not guaranteed to be constant-time
- Does not currently prove that a user-supplied modulus `p` is prime; use `generate_shamir_p(_b64u)` or validate `p` externally

If you have questions about safe use or threat modeling, consider opening a discussion with non-sensitive details.
