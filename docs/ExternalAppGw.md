# Connecting an Internal-LB Moodle Deployment to an Externally-Managed Application Gateway

This is a runbook for the scenario where Moodle is deployed with an **internal** Load
Balancer (`internalLoadBalancerSwitch: true`, e.g. via the `*-internal-lb.json` wrapper
templates) and fronted by an **Application Gateway that lives outside this template**
(a different VNet, peered to Moodle's VNet, managed separately).

It captures the issues we actually hit setting this up end-to-end, so the same mistakes
aren't repeated on the next deployment.

## Before you deploy: set `siteURL` up front

The internal-LB wrapper templates (`azuredeploy-minimal-internal-lb.json`,
`azuredeploy-small2mid-noha-internal-lb.json`) now expose a `siteURL` parameter — **set
it to the real public hostname that the Application Gateway will serve** (e.g.
`upskill.example.com`) at deploy time.

If you leave it at the default, `azuredeploy.json`'s fallback logic auto-generates
`lb-<resourceprefix>.<region>.cloudapp.azure.com` as the site URL — a private-LB DNS
name with **no actual public DNS record**, since there's no public IP to attach a DNS
label to in the internal-LB scenario. Moodle will then embed that broken hostname in
every link (`wwwroot`, login, theme/JS/CSS asset URLs, etc.), and no amount of App
Gateway configuration will fix that — you have to fix it at the Moodle layer either way
(see "If you didn't set `siteURL` at deploy time" below).

This one setting is also what determines the **CN of the self-signed TLS cert** nginx
generates on each node (`install_moodle.sh` / `setup_webserver.sh`), which matters a lot
for the App Gateway wiring below.

## Architecture recap

- Each Moodle node (controller + every VMSS web instance) runs its own nginx + Varnish
  stack and terminates real TLS itself (`httpsTermination: VMSS`, the default).
- With `fileServerType: nfs` (used in this deployment), `/moodle` — including
  `/moodle/certs/nginx.crt`/`nginx.key` — is **shared**: the controller acts as the NFS
  server, and every VMSS instance mounts `/moodle` from it. Only `install_moodle.sh`
  (controller) ever generates the self-signed cert; `setup_webserver.sh` (VMSS
  instances) just reads whatever's already there over NFS. This means there's only
  ever **one** cert in play across the whole farm, and scaling out the VMSS does *not*
  generate a new one — confirm this holds for your `fileServerType` if it differs
  (e.g. `azurefiles`/`gluster`), since the sharing mechanism differs.
- The real risk isn't per-instance divergence — it's that `install_moodle.sh` used to
  regenerate the self-signed cert (with a brand new random key) **unconditionally**
  every time it ran, which would silently invalidate whatever cert an external
  Application Gateway/proxy had already trusted, on any controller re-provisioning
  event (extension re-run, VM reimage/repair — not just first deploy). This has been
  fixed to skip regeneration if a cert/key pair already exists at
  `/moodle/certs/nginx.crt`/`.key`.
- The internal Load Balancer's frontend private IP is always the `.10` host in the web
  subnet (`lbPrivateIP` in `nested/network.json`) — this is what the App Gateway's
  backend pool should point at, not an individual instance IP.
- The LB passes both `:80` (Varnish) and `:443` (nginx TLS) straight through to
  whichever instance answers — it does not terminate TLS itself.
- The **controller is not part of the VMSS backend pool** the LB balances across, so it
  never actually receives production traffic through the LB/App Gateway path — but
  since it's the NFS server hosting `/moodle/certs`, it's still the machine you need to
  touch if you ever need to rotate the cert deliberately.

## App Gateway configuration checklist

1. **Backend pool**: target the internal LB's private IP (`.10` in the web subnet),
   not a specific VM IP.
2. **Backend HTTP settings**: Protocol `HTTPS`, Port `443`.
3. **Trusted root certificate**: since nginx uses a self-signed cert by default, the
   cert *is* its own root — extract and upload it:
   ```bash
   openssl s_client -connect <internal-LB-IP>:443 -servername <siteURL> </dev/null 2>/dev/null \
     | openssl x509 > appgw-trusted-root.pem
   ```
4. **Host name override**: set to the **same value as `siteURL`** (see "Why the
   hostname override matters" below) — do **not** use a different value here than what
   Moodle's `wwwroot` is set to.
5. **Health probe**: use a custom probe, not the default. `/` returns a `303` redirect
   which the default probe treats as unhealthy; probe `/login/index.php` instead (a
   clean `200`), with the same host name as above, and success codes `200-399` to be
   safe.
6. **NSG**: the VMSS NSG (`nested/webvmss.json`) already allows `*` as source on ports
   `80`/`443`, so cross-VNet peered traffic is allowed by default — no NSG change
   needed, though you may want to tighten this to the App Gateway's subnet CIDR later.

Verify with:
```bash
az network application-gateway show-backend-health -g <rg> -n <appgw-name> -o json
```

## Why the hostname override matters (and why it must match `siteURL`)

The App Gateway's backend "Host name override" setting controls **both** the SNI sent
during the backend TLS handshake **and** the `Host` HTTP header forwarded to nginx —
these are the same setting and cannot be split. (Azure explicitly rejects trying to
combine hostname override with a rewrite rule on the `Host` header — you'll get
`ApplicationGatewayHostOverrideAndHostHeaderRewriteNotSupported`.)

Moodle checks that the incoming request's `Host` header matches `$CFG->wwwroot`. If the
override value differs from `wwwroot` (e.g. override is the internal LB's private DNS
name while `wwwroot` is the real public domain), Moodle will issue a redirect to the
"correct" canonical URL on **every single request** — and since the browser's follow-up
request goes through the same App Gateway with the same override, it loops forever
(`ERR_TOO_MANY_REDIRECTS`). `curl -I` won't show this because it doesn't follow
redirects by default — you'll see a single, seemingly-fine `303` unless you check where
its `Location` header actually points.

**The fix is to make everything use the same hostname**: cert CN/SAN, `wwwroot`, and the
App Gateway hostname override should all be identical. There is no need to decouple SNI
from the Host header if the cert is generated for the real public domain in the first
place.

## If you didn't set `siteURL` at deploy time (fixing it live, no redeploy)

1. **Find where `config.php` lives.** With `htmlLocalCopySwitch: true` (the default),
   each node may have its own local copy, but in practice this deployment used the
   shared mount `/moodle/html/moodle/config.php` on every node — check both:
   ```bash
   ls -la /var/www/html/moodle/config.php 2>/dev/null
   ls -la /moodle/html/moodle/config.php 2>/dev/null
   ```
2. **Update `wwwroot`** and purge caches (cached theme/JS/CSS URLs embed the old
   domain):
   ```bash
   sudo sed -i "s#\$CFG->wwwroot.*=.*'.*';#\$CFG->wwwroot   = 'https://<real-domain>';#" /moodle/html/moodle/config.php
   sudo -u www-data php /moodle/html/moodle/admin/cli/purge_caches.php
   sudo systemctl restart php8.1-fpm   # adjust version; OPcache can otherwise keep serving the old compiled config.php
   ```
3. **Regenerate the TLS cert with the correct CN/SAN on every node that's actually in
   the LB backend pool** (not just the controller — see architecture recap above):
   ```bash
   sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
     -keyout /moodle/certs/nginx.key -out /moodle/certs/nginx.crt \
     -subj "/C=US/ST=WA/L=Redmond/O=IT/CN=<real-domain>" \
     -addext "subjectAltName=DNS:<real-domain>"
   sudo chown www-data:www-data /moodle/certs/nginx.*
   sudo chmod 0400 /moodle/certs/nginx.*
   sudo systemctl reload nginx
   ```
   If SSH from the controller to the VMSS instance isn't set up, use
   `az vmss run-command invoke` instead of SSH.
4. **Re-extract the cert and update the App Gateway** (trusted root cert + hostname
   override), per the checklist above.

## Quick verification commands

From the controller, bypassing the LB to test a specific instance directly:
```bash
curl -v -H "X-Forwarded-Proto: https" -H "Host: <real-domain>" http://<instance-ip>:81/
```
`X-Redirect-By: Moodle` in the response headers confirms PHP-FPM/Moodle is actually
handling the request (not just nginx serving a static error page).

Through the internal LB (exercises the whole path, including Varnish):
```bash
openssl s_client -connect <internal-LB-IP>:443 -servername <real-domain> </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```
Confirm the `subject`/SAN match `<real-domain>` and there's no more `CN=lb-<prefix>...`.
