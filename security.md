# Security Policy

This document defines the **security assumptions, boundaries, and reporting process** for this Copyparty deployment reference.

This repository contains **deployment guidance only** — not Copyparty source code.

---

## Security assumptions

This setup is secure **only if all of the following are true**:

- Copyparty runs **behind a reverse proxy**
- Authentication is handled **entirely by Apache**
- Copyparty **trusts auth headers only from localhost**
- Copyparty is **never exposed directly to the internet**

If any assumption is violated, the deployment is insecure.

---

## Trust boundaries

Internet
↓
Apache (TLS + Auth)
↓ (trusted headers, localhost only)
Copyparty
↓
Filesystem (/srv/copyparty/users)

yaml
Copy code

- Apache is the **security boundary**
- Copyparty is treated as an internal service
- Filesystem permissions provide final isolation

---

## Mandatory security requirements

The following are **non-negotiable**:

- ❌ Do not expose Copyparty directly to the internet
- ❌ Do not bind Copyparty to `0.0.0.0`
- ❌ Do not trust auth headers from non-local IPs
- ❌ Do not commit secrets to Git

- ✅ Run Copyparty as an unprivileged user
- ✅ Terminate TLS at the reverse proxy
- ✅ Restrict filesystem paths explicitly
- ✅ Keep Apache, OpenSSL, and the OS patched

---

## Secrets handling

This repository **must never contain**:

- passwords
- API keys or tokens
- private keys or certificates
- real infrastructure secrets

Use environment variables, external config includes, or a secret manager.

If a secret enters Git history, **rotate it immediately**.

---

## Common failure modes

Most real-world incidents come from:

- broken Apache authentication
- header spoofing due to bad `xff-src`
- exposed internal ports
- incorrect filesystem permissions

These are **operator errors**, not Copyparty bugs.

---

## Vulnerability reporting

If you discover a security issue **in this deployment guidance**:

1. **Do not open a public issue**
2. Contact the maintainer privately with:
   - impact summary
   - affected configuration
   - reproduction steps



You can contact: anishjawalkar2004@gmail.com




## Final note

This setup is secure **only when operated correctly**.

Most Copyparty compromises happen because:
> ports were exposed, proxies were broken, or headers were trusted blindly.

Don’t do that.
