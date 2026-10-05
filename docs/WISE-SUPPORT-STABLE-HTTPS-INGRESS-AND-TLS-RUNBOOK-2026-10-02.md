# WISE Support — Stable HTTPS Ingress & Let's Encrypt Runbook

## Purpose

This document records the **proven OCI validation setup for the stable WISE Support hostname** and the repeatable procedure used to replace the temporary IP/self-signed HTTPS endpoint with a trusted Let's Encrypt certificate.

It is intentionally operational and environment-specific. It complements:

- `docs/WISE-SUPPORT-ENVIRONMENT-REFERENCE.md`
- `docs/WISE-SUPPORT-DRESS-REHEARSAL-ENVIRONMENT-CHANGELOG-2026-10-02.md`
- `docs/WISE_SUPPORT_CURRENT_ENVIRONMENT_AND_MAINTENANCE_RUNBOOK.md`

Do not treat the current public IP as a permanent production infrastructure decision.

---

## 1. Stable Support hostname

### DNS

The stable WISE Support hostname is:

```
support.wisehealth.in
```

DNS record:

| Field | Value |
|---|---|
| Type | A |
| Name | `support` |
| Value | `140.245.237.47` |
| Purpose | Stable public hostname for the OCI Chatwoot validation ingress |

The DNS record was externally verified and returned:

```
support.wisehealth.in -> 140.245.237.47
```

Recommended DNS comment:

> WISE Support — Chatwoot public HTTPS ingress. Used by Telegram webhook and WISE web-chat; replaces temporary IP-based validation endpoint.

### Architectural role

The same hostname is intended to front the Chatwoot installation for multiple support channels:

```
support.wisehealth.in
        |
        +--> Chatwoot HTTPS origin / portal
        |
        +--> Telegram webhook / channel ingress
        |
        +--> Future WISE Web Chat integration
```

The WISE application entry point remains conceptually separate:

```
https://wisehealth.in/support
        |
        +--> WISE Support web-chat page
                |
                +--> Chatwoot Web Widget
```

Therefore:

- `support.wisehealth.in` = support infrastructure / Chatwoot origin
- `wisehealth.in/support` = WISE Health patient-facing support entry point

---

## 2. OCI validation host

Current validation VM:

- OCI Compute
- Region: `ap-hyderabad-1`
- AD: AD-1
- Instance: `wise-support-chat-oci-validation`
- OS: Oracle Linux Server 9.8 x86_64
- Public IP at time of validation: `140.245.237.47`
- Nginx: `1.20.1`
- Chatwoot: v4.14.2 lineage

The IP is a validation-environment value and may change. Patient-facing links should use the hostname, never the raw IP.

---

## 3. Required network path

Let's Encrypt HTTP-01 validation requires public TCP/80 access.

The proven ingress path is:

```
Internet
   |
   v
support.wisehealth.in
   |
   v
OCI public IP :80
   |
   v
OCI VCN security rule TCP/80
   |
   v
Oracle Linux firewalld HTTP
   |
   v
Nginx :80
   |
   v
/usr/share/nginx/html
```

HTTPS uses the corresponding TCP/443 path.

### OCI ingress rules

The public subnet/security list must permit:

| Protocol | Port | Source | Purpose |
|---|---:|---|---|
| TCP | 80 | 0.0.0.0/0 | HTTP / Let's Encrypt HTTP-01 validation |
| TCP | 443 | 0.0.0.0/0 | HTTPS Support ingress |

Do not expose PostgreSQL/Redis ports publicly.

### firewalld

The validated VM firewall services are:

```
dhcpv6-client http https ssh
```

Check with:

```bash
sudo firewall-cmd --list-services
```

If HTTP is missing:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

---

## 4. Nginx HTTP webroot

The active HTTP server block in `/etc/nginx/nginx.conf` uses:

```text
listen 80;
listen [::]:80;
server_name _;
root /usr/share/nginx/html;
```

The active Chatwoot-specific configuration is:

```text
/etc/nginx/conf.d/chatwoot.conf
```

The webroot used for ACME validation is therefore:

```text
/usr/share/nginx/html
```

---

## 5. Important lesson: Certbot Nginx plugin

The initial attempt used:

```bash
certbot certonly --nginx -d support.wisehealth.in
```

The Certbot Nginx plugin was installed and visible, but the plugin repeatedly failed with:

```
NoInstallationError:
Could not find a usable 'nginx' binary.
```

Even when Certbot was invoked with an expanded PATH, the Nginx plugin continued to fail.

Do **not** repeat this as the primary procedure on this validation VM.

The successful procedure uses the Certbot **webroot** authenticator. It does not ask Certbot to rewrite or inspect the Nginx configuration.

---

## 6. Certbot installation

Certbot is installed in an isolated Python environment:

```text
/opt/certbot/bin/certbot
```

The currently validated version is:

```
certbot 5.8.0
```

Use the absolute path in operational commands rather than relying on shell PATH:

```bash
sudo /opt/certbot/bin/certbot --version
```

A system-wide `certbot` command is not assumed to be available.

### Python note

Certbot reports that Python 3.9 support will be dropped in a future Certbot release. The current certificate is working, but the VM's Python/Certbot runtime should be reviewed before a future OS/package maintenance cycle.

---

## 7. ACME webroot validation procedure

### Create the challenge directory

```bash
sudo mkdir -p /usr/share/nginx/html/.well-known/acme-challenge
```

### Create a test file

```bash
echo "wise-support-acme-test" | sudo tee /usr/share/nginx/html/.well-known/acme-challenge/test.txt
```

### Verify local/external reachability

The test file must be reachable through the public hostname:

```bash
curl http://support.wisehealth.in/.well-known/acme-challenge/test.txt
```

Expected:

```
wise-support-acme-test
```

This test must succeed before requesting the certificate.

### Obtain the certificate

The proven command is:

```bash
sudo /opt/certbot/bin/certbot certonly \
  --webroot \
  -w /usr/share/nginx/html \
  -d support.wisehealth.in
```

Successful issuance produced:

```text
/etc/letsencrypt/live/support.wisehealth.in/fullchain.pem
/etc/letsencrypt/live/support.wisehealth.in/privkey.pem
```

The certificate was successfully issued. At activation verification on 2026-10-02, the served certificate reported:

```
subject=CN=support.wisehealth.in
issuer=C=US, O=Let's Encrypt, CN=YE1
notBefore=Oct  2 08:04:48 2026 GMT
notAfter=Dec 31 08:04:47 2026 GMT
```

The exact expiry should always be checked from the current certificate rather than copied from this historical record.

---

## 8. Renewal setup

Certbot successfully validated the renewal configuration with:

```bash
sudo /opt/certbot/bin/certbot renew --dry-run
```

Observed result:

```
Congratulations, all simulated renewals succeeded:
  /etc/letsencrypt/live/support.wisehealth.in/fullchain.pem (success)
```

No Certbot systemd timer was initially present, so a dedicated renewal service/timer was created.

### Renewal service

File:

```text
/etc/systemd/system/certbot-renew.service
```

Content:

```ini
[Unit]
Description=Renew Let's Encrypt certificates

[Service]
Type=oneshot
ExecStart=/opt/certbot/bin/certbot renew
```

### Renewal timer

File:

```text
/etc/systemd/system/certbot-renew.timer
```

Content:

```ini
[Unit]
Description=Daily Let's Encrypt certificate renewal check

[Timer]
OnCalendar=daily
Persistent=true
RandomizedDelaySec=1h

[Install]
WantedBy=timers.target
```

Enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now certbot-renew.timer
```

Verify:

```bash
systemctl status certbot-renew.timer --no-pager
systemctl list-timers --all | grep certbot
```

The timer checks daily. Certbot itself only renews when renewal is actually due.

---

## 9. Renewal deploy hook

Certificate files can be renewed without automatically making a running Nginx process serve the new certificate. Therefore a deploy hook reloads Nginx after a successful certificate deployment.

Directory:

```text
/etc/letsencrypt/renewal-hooks/deploy/
```

Hook:

```text
/etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

Content:

```bash
#!/bin/bash
/usr/sbin/nginx -t && /usr/bin/systemctl reload nginx
```

Permissions:

```bash
sudo chmod 750 /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

This deliberately uses:

```text
nginx -t -> reload
```

rather than restarting Nginx.

That minimizes disruption to the Chatwoot validation service.

---

## 10. TLS activation — completed 2026-10-02

The trusted certificate was activated in the live Nginx Chatwoot server block.

### Previous temporary configuration

```nginx
server {
    listen 443 ssl;
    server_name 140.245.237.47;

    ssl_certificate     /etc/nginx/ssl/chatwoot-ip.crt;
    ssl_certificate_key /etc/nginx/ssl/chatwoot-ip.key;
```

### Activated configuration

```nginx
server {
    listen 443 ssl;
    server_name support.wisehealth.in;

    ssl_certificate     /etc/letsencrypt/live/support.wisehealth.in/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/support.wisehealth.in/privkey.pem;
```

The existing Chatwoot reverse-proxy `location /` and `proxy_pass http://127.0.0.1:3000` configuration was preserved unchanged.

A backup was created before editing:

```text
/etc/nginx/conf.d/chatwoot.conf.bak-20261002
```

Validation before reload succeeded:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Nginx was then gracefully reloaded.

### Live HTTPS verification

```text
HTTP/1.1 200 OK
Server: nginx/1.20.1
Content-Type: text/html; charset=utf-8
```

The served certificate was verified directly with SNI:

```text
subject=CN=support.wisehealth.in
issuer=C=US, O=Let's Encrypt, CN=YE1
notBefore=Oct  2 08:04:48 2026 GMT
notAfter=Dec 31 08:04:47 2026 GMT
```

**Checkpoint status: TRUSTED HTTPS ACTIVATION PROVEN.**

The temporary self-signed IP certificate is no longer the active Nginx certificate. It is retained as a historical/rollback artifact and should not be deleted until the subsequent hostname/Telegram regression checkpoint is complete.

---

## 11. Final validation checklist

### DNS

- [x] `support.wisehealth.in` resolves to current OCI validation IP.
- [x] External DNS verification performed.

### HTTP/ACME

- [x] OCI TCP/80 ingress opened.
- [x] firewalld HTTP enabled.
- [x] Nginx listens on TCP/80.
- [x] ACME webroot is publicly reachable.
- [x] Let's Encrypt certificate successfully issued.

### Renewal

- [x] Certbot renewal configuration created.
- [x] `certbot renew --dry-run` succeeds.
- [x] Daily systemd renewal timer configured.
- [x] Nginx deploy hook configured.
- [ ] Perform a future live renewal/reload observation when the certificate becomes renewal-eligible.

### HTTPS activation

- [x] Nginx switched from IP/self-signed certificate to Let's Encrypt certificate.
- [x] `nginx -t` passes.
- [x] Nginx reload succeeds.
- [x] `https://support.wisehealth.in/` serves HTTP 200 through Chatwoot.
- [x] Served certificate subject is `support.wisehealth.in`.
- [x] Served certificate issuer is Let's Encrypt.
- [ ] Browser verification without certificate warning (not separately recorded yet).

### Telegram CTA

- [ ] Existing rehearsal CTA changed from IP-based URL to `https://support.wisehealth.in/`.
- [ ] Two-way Telegram text E2E rerun after hostname migration.
- [x] Existing `@wescura_support_bot` CTA remains unchanged.
- [x] POC `@wise_chatwoot_poc_bot` remains isolated until the replacement CTA is proven.

### Web Support

- [ ] `https://wisehealth.in/support` web-chat page implemented separately.
- [ ] Chatwoot-specific widget implementation remains in `wise-support-chat`, not embedded as provider-specific core architecture in `quick-chat-landing`.
- [ ] Web chat E2E tested separately.
- [ ] Third/replacement CTA proven before retiring the old first CTA.

---

## 12. Operational cautions

- Never expose Telegram bot tokens in this document.
- Never commit private keys or Certbot account credentials.
- The current OCI public IP is validation infrastructure and may change.
- `support.wisehealth.in` is the stable hostname; patient-facing links should not contain the raw IP.
- HTTP/80 must remain reachable if HTTP-01 renewal is used.
- Do not mix the POC Telegram bot with the existing WISE Support bot.
- Do not treat the OCI validation topology as the final production architecture without a separate production readiness decision.
- Attachment/media failure remains a separate backlog item; the proven two-way text E2E is not to be reclassified as failing because attachments remain unresolved.

---

## 13. Historical validation sequence

The stable-hostname work was performed in this order:

1. Added DNS A record `support.wisehealth.in -> 140.245.237.47`.
2. Confirmed DNS resolution.
3. Found Nginx still serving the temporary self-signed IP certificate.
4. Confirmed Nginx listened on both 80 and 443.
5. Added HTTP to OCI/firewalld ingress for ACME validation.
6. Confirmed external HTTP/80 reachability.
7. Installed Certbot in `/opt/certbot`.
8. Nginx Certbot plugin was tested but failed to locate the Nginx binary reliably.
9. Identified active Nginx HTTP webroot as `/usr/share/nginx/html`.
10. Switched to Certbot webroot authentication.
11. Successfully issued the trusted certificate for `support.wisehealth.in`.
12. Confirmed renewal with `certbot renew --dry-run`.
13. Added persistent daily systemd renewal checking.
14. Added a deploy hook to validate and reload Nginx after successful renewal.
15. Backed up `/etc/nginx/conf.d/chatwoot.conf`.
16. Changed the live Chatwoot TLS server block from the IP/self-signed certificate to `support.wisehealth.in` and the Let's Encrypt certificate.
17. `nginx -t` passed.
18. Nginx was gracefully reloaded.
19. `https://support.wisehealth.in/` returned HTTP 200.
20. SNI certificate verification confirmed the Let's Encrypt certificate for `support.wisehealth.in`.

This sequence is the repeatable reference for future recreation of the hostname/TLS validation setup.
