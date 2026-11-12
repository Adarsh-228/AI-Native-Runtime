# Security Policy

## Supported Versions

During early pre-1.0 research and prototyping stages, security updates are applied to the `main` branch.

| Version | Supported |
| :--- | :--- |
| `0.1.x` (Pre-alpha) | Yes |

## Reporting a Vulnerability

The `ai-native-runtime` team takes security vulnerabilities seriously, especially concerning:
- Cross-node execution privilege escalation
- Unintended data egress violating `LocalOnly` policies
- Cryptographic identity or session spoofing
- Remote code execution in sandboxed tool or inference environments

**Please do not report security vulnerabilities through public GitHub issues.**

Instead, report vulnerabilities privately by emailing the security contact specified in the repository settings or opening a GitHub Private Security Advisory.

Please include:
- A description of the vulnerability and potential impact.
- Steps to reproduce or a minimal proof of concept.
- Affected components (crates, SDKs, or protocols).

We acknowledge reports within 48 hours and work with the reporter on responsible coordinated disclosure.

