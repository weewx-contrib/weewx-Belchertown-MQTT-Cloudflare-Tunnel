# weewx-Belchertown-MQTT-Cloudflare-Tunnel

# Live Weather Updates on the WeeWX Belchertown Skin — with Zero Open Ports
## Mosquitto MQTT over WebSockets · Apache reverse proxy · Cloudflare Tunnel

**Author:** Ian Millard · ianmillard@icloud.com
**Version:** 1.0 · 12 June 2026

This guide builds, from scratch, a WeeWX website running the Belchertown skin with real-time ("live") data updates, published to the internet through a **Cloudflare Tunnel** — meaning **no inbound ports are opened or forwarded anywhere**. Your router stays closed, your home IP never appears in DNS, and visitors get HTTPS and live WebSocket updates via Cloudflare's edge.

It is based on a verified working installation using **two machines on the same LAN**:

- **Webserver Pi** — runs WeeWX (pip install), Apache, and the `cloudflared` tunnel daemon
- **Broker Pi** — runs the Mosquitto MQTT broker

A single-machine layout (everything on one box) works identically — wherever the guide says `<broker-ip>`, use `127.0.0.1` instead, and the LAN firewall steps become unnecessary.

Every code block below is labelled with **which machine it runs on** — in a two-machine setup, running the right command on the wrong box is the easiest mistake to make.

## How it works

The Belchertown skin's live updates work by having each visitor's browser subscribe to an MQTT broker over WebSockets. Because the site is served over HTTPS, the WebSocket must be encrypted too (`wss://`). Rather than exposing the broker — or anything else — to the internet, everything is reached through one outbound connection:

```
        Webserver Pi                                      Broker Pi
┌────────────────────────────────┐               ┌──────────────────────────┐
│ WeeWX ──────────────────────── MQTT/TCP :1883 ──► Mosquitto               │
│                                │               │  ├ listener 1883 (TCP)   │
│ Apache :443 ◄─── cloudflared   │               │  └ listener 9001 (WS)    │
│    │                  ▲        │               └─────────────▲────────────┘
│    └─ ProxyPass /mqtt │ ───────┼─────────── plain ws (LAN) ──┘
└───────────────────────┼────────┘
                        │ outbound tunnel (no inbound ports)
                  Cloudflare edge
                        ▲
                        │ https / wss (443, TLS)
                     Browser
```

- **WeeWX** publishes each loop packet to Mosquitto over the LAN (port 1883).
- **The visitor's browser** connects to `wss://your-domain/mqtt` — terminated at Cloudflare's edge.
- **cloudflared**, running on the webserver Pi, maintains an outbound-only connection to Cloudflare and delivers the traffic to Apache locally.
- **Apache** serves the site and reverse-proxies the `/mqtt` path to Mosquitto's WebSocket listener on the Broker Pi (port 9001, LAN only).

| Role | Software | Runs on | Connects to |
|---|---|---|---|
| Publisher | WeeWX + `weewx-mqtt` extension | Webserver Pi | Broker Pi, TCP 1883 (LAN) |
| Broker (server) | Mosquitto | Broker Pi | Listens on 1883 (TCP) and 9001 (WebSockets), LAN only |
| Web/proxy | Apache + cloudflared | Webserver Pi | Apache serves the site; `/mqtt` proxied to Broker Pi 9001; cloudflared connects out to Cloudflare |
| Subscriber | Visitor's browser (Paho JS, built into Belchertown pages) | — | `wss://your-domain/mqtt` via Cloudflare edge → tunnel → Apache |

> **Naming warning:** several similarly named products exist. **Mosquitto** and **NanoMQ** are MQTT *brokers* (servers) — either can fill the broker role. **NanoMQTT** (DigiCert TrustCore SDK) is an embedded MQTT *client library* and **cannot** act as the broker; its own documentation says you still need a broker such as Mosquitto. This guide uses Mosquitto, the Debian/Raspberry Pi OS default.

> **No Cloudflare?** The same architecture works with ordinary port forwarding instead of a tunnel: forward 80/443 on your router to Apache, use certbot's normal HTTP-01 certificates, and skip Steps 5–6. Everything else in this guide (Steps 1–4, the verification logic, the troubleshooting) is identical. The tunnel route is recommended: it removes the attack surface entirely.

---

## Prerequisites

- Two Raspberry Pis (or equivalent Linux machines) on the same LAN; give the Broker Pi a **static IP or DHCP reservation** — if its address changes, everything breaks silently
- WeeWX installed and generating the Belchertown skin successfully on the webserver Pi, with Apache serving the generated site locally
- A domain on a **Cloudflare account** (free plan is sufficient — it includes Tunnel and WebSockets), using Cloudflare's nameservers
- Root/sudo access on both machines
- Replace `weather.example.com` with your real domain and `<broker-ip>` with the Broker Pi's LAN IP throughout

---

## Step 1 — Broker Pi: install and configure Mosquitto

**Broker Pi · bash**
```bash
sudo apt update
sudo apt install mosquitto mosquitto-clients
```

Edit `/etc/mosquitto/mosquitto.conf` (or create a file in `/etc/mosquitto/conf.d/`) and ensure these listeners exist:

**Broker Pi · /etc/mosquitto/mosquitto.conf**
```
# --- Listener 1: Standard MQTT for the WeeWX extension ---
listener 1883 0.0.0.0
protocol mqtt

# --- Listener 2: WebSockets for the Belchertown dashboard (via Apache proxy) ---
listener 9001 0.0.0.0
protocol websockets

# --- Security (home/private network setup) ---
allow_anonymous true
```

**Important details:**

- The moment any `listener` line exists, Mosquitto's implicit default listener is disabled — so the explicit `listener 1883` line is required, otherwise WeeWX loses its connection.
- Both listeners bind `0.0.0.0` because the webserver Pi must reach them across the LAN. (Single-machine layout: bind both to `127.0.0.1` instead — nothing then needs to leave the box.)
- `allow_anonymous true` is acceptable for a home LAN. See the security notes at the end for hardening.

Restart, **enable at boot** (easily forgotten — without this, live updates die at the Broker Pi's next reboot), and verify both ports:

**Broker Pi · bash**
```bash
sudo systemctl restart mosquitto
sudo systemctl enable mosquitto
ss -tln | grep -E "1883|9001"
```

You should see LISTEN lines for both ports. If Mosquitto fails to start, check `journalctl -u mosquitto -n 30`.

### Ensure ports 1883 and 9001 are open to the LAN

These ports stay **LAN-only** — they are never exposed to the internet in this design. But the `0.0.0.0` bind only makes Mosquitto listen; a host firewall on the Broker Pi can still block the webserver Pi. If `ufw` (or similar) is active:

**Broker Pi · bash**
```bash
sudo ufw allow from 192.168.1.0/24 to any port 1883 proto tcp   # adjust to your LAN subnet
sudo ufw allow from 192.168.1.0/24 to any port 9001 proto tcp
```

(If no firewall is running, nothing to do.) Then verify reachability **from the webserver Pi**:

**Webserver Pi · bash**
```bash
mosquitto_sub -h <broker-ip> -p 1883 -t '$SYS/broker/version' -C 1
```

A version string back means the path is clear; a hang or "connection refused" means a firewall or listener bind is in the way.

---

## Step 2 — Webserver Pi: WeeWX publishes loop data to the broker

Install the `weewx-mqtt` extension (matthewwall's MQTT uploader), then configure it in `weewx.conf`:

**Webserver Pi · weewx.conf**
```ini
[StdRESTful]
    [[MQTT]]
        server_url = mqtt://<broker-ip>:1883/
        topic = weather
        unit_system = METRIC        # or US — must match your site's units
        binding = archive, loop
        aggregation = aggregate
        retain = true
```

- `aggregation = aggregate` publishes a single JSON payload to `weather/loop`, which is the format the skin expects.
- `retain = true` means a freshly loaded page receives the most recent reading immediately, instead of waiting for the next loop packet.
- Confirm `user.mqtt.MQTT` appears in `restful_services` under `[Engine]` (the extension installer normally adds it).

Restart WeeWX, then verify data is flowing — run this from the webserver Pi so it also re-proves the LAN path:

**Webserver Pi · bash**
```bash
mosquitto_sub -h <broker-ip> -t "weather/loop" -v
```

JSON should appear every loop interval (roughly every 2.5–16 seconds depending on the station). **Do not continue until this works** — everything downstream depends on it.

---

## Step 3 — Webserver Pi: Apache reverse-proxies `/mqtt` to the broker

Enable the proxy modules:

**Webserver Pi · bash**
```bash
sudo a2enmod proxy proxy_http proxy_wstunnel
```

Edit your **HTTPS (port 443) virtual host** — on a certbot setup this is typically `/etc/apache2/sites-available/000-default-le-ssl.conf` — and add inside the `<VirtualHost *:443>` block:

**Webserver Pi · 000-default-le-ssl.conf**
```apache
ProxyPass        /mqtt ws://<broker-ip>:9001/mqtt
ProxyPassReverse /mqtt ws://<broker-ip>:9001/mqtt
```

**Important details:**

- It goes in the **443** vhost: cloudflared will deliver traffic to `https://localhost:443`, so that's the vhost that must carry the proxy rule.
- The target scheme is lowercase `ws://` — that is what tells Apache to tunnel WebSocket traffic.
- The `/mqtt` path is deliberate: Belchertown's JavaScript client connects to the `/mqtt` path by convention.
- There must be only **one** `ProxyPass /mqtt` directive in the entire effective configuration — the first match wins, so a stale duplicate silently overrides your edit.
- Apache needs *a* certificate on the 443 vhost, but with the tunnel it does **not** need to be valid (cloudflared will be told not to verify it). If you're setting up fresh and have no certificate yet, either generate a self-signed one (`sudo make-ssl-cert generate-default-snakeoil` + the `default-ssl` vhost) or jump ahead to the **DNS-01 certificates** section to get a real Let's Encrypt cert without opening any ports — then return here.

Test syntax and restart:

**Webserver Pi · bash**
```bash
sudo apachectl configtest && sudo systemctl restart apache2
```

### Nginx equivalent

If your site runs on nginx instead, the matching block inside the existing `server { listen 443 ssl; }` is:

**Webserver · nginx site config**
```nginx
location /mqtt {
    proxy_pass http://<broker-ip>:9001/mqtt;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 3600s;    # loop packets are seconds apart; default 60s drops idle connections
    proxy_send_timeout 3600s;
}
```

---

## Step 4 — Webserver Pi: Belchertown skin settings

In `weewx.conf`, under the Belchertown report (preferred over editing `skin.conf`, so skin upgrades don't overwrite it):

**Webserver Pi · weewx.conf**
```ini
[StdReport]
    [[Belchertown]]
        skin = Belchertown
        [[[Extras]]]
            mqtt_websockets_enabled = 1
            mqtt_websockets_host = "weather.example.com"
            mqtt_websockets_port = 443
            mqtt_websockets_ssl = 1
            mqtt_websockets_topic = "weather/loop"
```

**Critical points — these are the most common mistakes:**

- `mqtt_websockets_host` is what the **visitor's browser** connects to. It is therefore your public **website hostname** — not the Broker Pi's IP, not `localhost`, and not port 9001. The browser goes to Cloudflare's edge; only Apache talks to the broker.
- Port **443** and ssl **1** are correct because the visitor's connection terminates at Cloudflare's edge over TLS, regardless of what happens behind the tunnel.
- Check `skin.conf` doesn't carry conflicting old values — settings in `weewx.conf` override `skin.conf`, but only if they're actually present there.
- **Regenerate the site after any change** — these values are baked into the published JavaScript. The command depends on how WeeWX was installed:

**Webserver Pi · bash**
```bash
# pip install (WeeWX runs in a virtual environment) — no sudo needed:
source ~/weewx-venv/bin/activate
weectl report run

# Debian/package install:
sudo weectl report run

# Older WeeWX (4.x and earlier):
sudo wee_reports
```

If you skip the regeneration, the deployed pages keep trying the old host/port no matter what you've fixed elsewhere.

---

## Step 5 — Webserver Pi: connect Cloudflare Tunnel

### 5a — Install cloudflared

**Webserver Pi · bash**
```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install cloudflared
```

### 5b — Create the tunnel and connect the Pi

In **Cloudflare One** (one.dash.cloudflare.com) go to **Networks → Connectors → Cloudflare Tunnels → Create a tunnel** → connector type **Cloudflared** → name it (e.g. `weather-pi`) → save. The dashboard shows an install command containing a token — run it on the webserver Pi:

**Webserver Pi · bash**
```bash
sudo cloudflared service install <token-from-dashboard>
```

Within seconds the tunnel should show **HEALTHY** in the dashboard. Verify locally with `systemctl status cloudflared`.

> **Run the install command exactly once.** Running it twice (or creating two tunnels) enrols the Pi under two different tunnel UUIDs, and DNS pointing at the wrong one produces error 1033 later. If in doubt, the UUID the running service actually registered is in the logs: `sudo journalctl -u cloudflared | grep tunnelID | tail -1`

### 5c — Publish the application (the route)

On the tunnel's page: **Published application routes → Add**:

- Subdomain: *(blank)* · Domain: your domain · Path: *(blank)*
- Service: **HTTPS** · URL: `localhost:443`
- **Origin configurations → TLS → No TLS Verify: ON**

The No TLS Verify toggle is required: cloudflared connects to `localhost:443` and would otherwise reject Apache's certificate, which (if valid at all) names your domain, not `localhost`. The hop is localhost-only, so nothing is lost. If you prefer a *verified* origin instead, set **Origin Server Name** to your domain and keep a valid certificate on Apache via the **DNS-01 certificates** section below.

**Sanity check:** back on the tunnel list, the **Routes** column must now show your hostname. If it reads `--`, the route didn't save — the tunnel is connected but routing nothing, and the site cannot work.

### 5d — DNS: point the domain at the tunnel

The route save normally creates the DNS record automatically (it appears in DNS → Records as a proxied "Tunnel"-type record). If it doesn't, create it manually: **Type CNAME · Name `@` · Target `<tunnel-id>.cfargotunnel.com` · Proxied (orange cloud — mandatory)**, where `<tunnel-id>` is the UUID from the tunnel's overview page (or the `journalctl` command above — the *running* connector's UUID is the truth).

> **Ordering trap (migrations only):** if an old A record for the bare domain points at your home IP, the tunnel record can't be created until it's removed — but deleting it *before* the replacement exists plants a "no such record" answer in public resolvers' negative caches for up to **30 minutes** (the zone's 1800s negative TTL). Do the delete and the create back-to-back, and don't panic if resolution takes a while to return. The cache-proof way to check the zone's true state is to ask the authoritative nameserver directly:
>
> **Any machine · bash**
> ```bash
> sudo apt install dnsutils
> dig weather.example.com @<your-zone-ns>.ns.cloudflare.com A +norecurse
> ```
> (your two assigned nameservers are shown on the domain's DNS page). An ANSWER section with Cloudflare edge IPs (104.x / 172.6x) means the zone is correct and the resolvers will catch up. You can hurry 1.1.1.1 along at `https://one.one.one.one/purge-cache/`.

Also confirm the zone-level **Network → WebSockets** toggle is **On** (main dashboard → your domain → Network) — it defaults on, but verify.

---

## Step 6 — Verify the whole chain, in dependency order

Each test isolates one link. Run them in order; the first failure tells you exactly which layer to fix.

**a) Broker listening:**

**Broker Pi · bash**
```bash
ss -tln | grep -E "1883|9001"
```

**b) Data flowing in, across the LAN:**

**Webserver Pi · bash**
```bash
mosquitto_sub -h <broker-ip> -t "weather/loop" -v
```

**c) WebSocket handshake directly against the broker (bypasses Apache and the tunnel):**

**Webserver Pi · bash** — expected: `HTTP/1.1 101 Switching Protocols`
```bash
curl -i -N http://<broker-ip>:9001/mqtt \
  -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: dGVzdDEyMzQ1Njc4OTA=" \
  -H "Sec-WebSocket-Protocol: mqtt"
```

**d) WebSocket handshake via Apache locally (bypasses the tunnel):**

**Webserver Pi · bash** — expected: `HTTP/1.1 101 Switching Protocols`
```bash
curl -i -N -k https://localhost/mqtt -H "Host: weather.example.com" \
  -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: dGVzdDEyMzQ1Njc4OTA=" \
  -H "Sec-WebSocket-Protocol: mqtt"
```

**e) Site via the full public path (Cloudflare edge → tunnel → Apache):**

**Any machine · bash** — expected: `HTTP/2 200` with `server: cloudflare`
```bash
curl -I https://weather.example.com/
```

**f) WebSocket via the full public path:**

**Any machine · bash** — expected: `HTTP/1.1 101 Switching Protocols`
```bash
curl -i -N --http1.1 https://weather.example.com/mqtt \
  -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: dGVzdDEyMzQ1Njc4OTA=" \
  -H "Sec-WebSocket-Protocol: mqtt"
```

> **`--http1.1` is mandatory through Cloudflare.** The edge negotiates HTTP/2 with curl by default, and `Connection: Upgrade` is illegal in HTTP/2 — the edge rejects the request with a 502 *before it ever reaches the tunnel* (cloudflared logs nothing). Real browsers handshake correctly, so the site can be working perfectly while the un-flagged curl claims 502. This only affects the synthetic test, not visitors.

**g) Browser:** hard-refresh the site (Ctrl+Shift+R), open developer tools (F12) → Network tab → filter "WS". You should see a connection to `wss://weather.example.com/mqtt` with status **101**, frames arriving on each loop packet, and the page's connection indicator turning green with values updating live.

When all of a–g pass, you're done — and at no point did anything require an open inbound port. If you migrated from a port-forwarded setup, this is the moment to **delete the old 80/443 forwards on your router** and re-run e–g (results should be identical, since nothing has used the forwards since DNS switched to the tunnel).

---

## Certificates with zero open ports — DNS-01 via the Cloudflare plugin

With no inbound ports, certbot's default **HTTP-01** challenge can never reach your server, so certificates can't be issued or renewed that way. You have three options:

1. **No TLS Verify (the default in this guide)** — let Apache's certificate be self-signed or lapsed; cloudflared ignores it, visitors only ever see Cloudflare's edge certificate. Zero maintenance.
2. **Cloudflare Origin Certificate** — a free 15-year cert issued by Cloudflare, valid only between Cloudflare and your origin. Generate under SSL/TLS → Origin Server and install in Apache.
3. **A real Let's Encrypt certificate via DNS-01** — proves domain control by publishing a DNS TXT record instead of answering an inbound HTTP request. Since your DNS is on Cloudflare, certbot can automate this completely with the Cloudflare plugin. This is the right choice if you want a publicly valid certificate on the origin (e.g. for verified-origin TLS, other LAN services, or simply tidiness).

Here's option 3 in full.

### C1 — Install the plugin

**Webserver Pi · bash**
```bash
sudo apt install python3-certbot-dns-cloudflare
```

(Confirm certbot itself is installed: `certbot --version`.)

### C2 — Create a scoped Cloudflare API token

In the Cloudflare dashboard: click the profile icon (top right) → **My Profile → API Tokens → Create Token** → use the **"Edit zone DNS"** template:

- Permissions: **Zone · DNS · Edit** (the template sets this)
- Zone Resources: **Include · Specific zone · `weather.example.com`** — scope it to this one zone, never "All zones"
- Create the token and **copy it immediately** (it's shown once)

Do not use your Global API Key — the scoped token can only edit this zone's DNS, which is exactly the blast radius you want for a credential sitting on a Pi.

### C3 — Store the token securely

**Webserver Pi · bash**
```bash
sudo mkdir -p /root/.secrets/certbot
sudo nano /root/.secrets/certbot/cloudflare.ini
```

**Webserver Pi · /root/.secrets/certbot/cloudflare.ini**
```ini
# Cloudflare API token with Zone:DNS:Edit on the weather zone only
dns_cloudflare_api_token = <paste-your-token-here>
```

**Webserver Pi · bash**
```bash
sudo chmod 600 /root/.secrets/certbot/cloudflare.ini
```

Certbot refuses world-readable credential files — the `chmod 600` is mandatory, not cosmetic.

### C4 — Issue (or re-issue) the certificate via DNS-01

**Webserver Pi · bash**
```bash
sudo certbot certonly --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/certbot/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 30 \
  -d weather.example.com -d www.weather.example.com
```

What happens: certbot asks Cloudflare's API to create a `_acme-challenge` TXT record, waits the propagation delay, Let's Encrypt verifies it over DNS, the cert is issued, and the TXT record is removed — no inbound connection at any point. If a certificate for the same names already exists (from the HTTP-01 days), certbot asks whether to replace it; answering yes also **rewrites the renewal configuration** in `/etc/letsencrypt/renewal/weather.example.com.conf` to use DNS-01 from now on — that single run *is* the switch-over.

Drop the `-d www...` if you don't serve a www name. If validation is flaky, raise `--dns-cloudflare-propagation-seconds` to 60.

### C5 — Verify renewals work hands-off

**Webserver Pi · bash**
```bash
sudo certbot renew --dry-run
```

A clean dry-run means renewals will keep working forever with no ports open. Certbot's systemd timer (installed by default on Debian) handles the schedule. One Apache-specific note: `certonly` doesn't reload Apache after a real renewal, so add a deploy hook once:

**Webserver Pi · bash**
```bash
sudo sh -c 'printf "#!/bin/sh\nsystemctl reload apache2\n" > /etc/letsencrypt/renewal-hooks/deploy/reload-apache.sh'
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-apache.sh
```

### C6 — (Optional) switch the tunnel to verified-origin TLS

With a valid cert on Apache you can replace No TLS Verify with proper verification: on the tunnel route's **Origin configurations → TLS**, turn **No TLS Verify OFF** and set **Origin Server Name** to `weather.example.com` (so cloudflared validates the cert against your domain rather than `localhost`). Re-run verification test (e) afterwards — an `x509` error in `journalctl -u cloudflared` means the cert/name don't match and you should revert to No TLS Verify while investigating.

---

## Troubleshooting

All of these were encountered (or anticipated) while building the reference installation. Work out **which hop fails first** using the Step 6 ladder, then find the symptom below.

### Broker layer

**Direct broker curl (test c) fails** — Either Mosquitto's WebSocket listener isn't healthy, or the LAN path is blocked. On the Broker Pi: check `journalctl -u mosquitto -n 30` for a config error, confirm `protocol websockets` sits directly under the `listener 9001` line, confirm the bind is `0.0.0.0` (not `127.0.0.1`, which is unreachable from another machine), and confirm the running process was started *after* the config was written (`sudo systemctl restart mosquitto`). Then re-check the firewall rules from Step 1.

**Live updates work, then die after a reboot** — The broker service isn't enabled on the Broker Pi. `systemctl status mosquitto` shows `disabled` in the `Loaded:` line. Fix: `sudo systemctl enable mosquitto`.

**Two brokers fighting over port 1883** — If both Mosquitto and another broker (e.g. NanoMQ) are installed on the same machine, only one can bind 1883. Disable the one you're not using: `sudo systemctl disable --now <other-broker>`. Note that configuring a broker that was never installed achieves nothing — a `command not found` for the broker binary means the config file you wrote has no process reading it. `which <binary>` and `ps aux | grep <name>` confirm what's actually installed and running.

### Apache layer

**Local Apache curl (test d) returns 404** — Apache isn't matching the `/mqtt` location: ProxyPass missing, in the wrong vhost, or Apache wasn't restarted.

**Local Apache curl (test d) returns 503** — Apache matched the rule but cannot connect to the broker. The error log names the target: `sudo tail /var/log/apache2/error.log` → `(111)Connection refused: AH00957: ws: attempt to connect to <ip>:<port> failed`. Check the broker is listening (`ss -tln` on the Broker Pi), the firewall (Step 1), and the ProxyPass IP/port. Also hunt for a **duplicate** `ProxyPass /mqtt` winning over your edit (first match applies): `grep -Rn "ProxyPass" /etc/apache2/ | grep -v "#"` — note the **capital `-R`**: lowercase `-r` does *not* follow the symlinks in `sites-enabled/`, and will hide directives from you. Finally, beware the file being **edited and then overwritten** — an editor window still holding the old version, saved after a `sed` fix, silently reverts it; always re-`grep` immediately before restarting Apache.

**Local Apache curl returns 400** — The upgrade headers aren't being tunnelled: confirm `proxy_wstunnel` is enabled (`apachectl -M | grep wstunnel`) and the target scheme is `ws://`.

### Tunnel / DNS layer

**`Could not resolve host`, or DNS returns only an SOA (no Answer)** — The tunnel's DNS record isn't published. Check three things: the **Routes** column on the tunnel isn't `--` (Step 5c), the record exists and is **Proxied**, and you aren't just seeing a stale negative cache — the SOA serial not changing between queries means the zone genuinely hasn't changed; use the authoritative `dig` from Step 5d to bypass all caches.

**HTTP 530, body `error code: 1033`** — Cloudflare can't find a connected tunnel for the UUID in DNS. Compare the record's `<uuid>.cfargotunnel.com` against `sudo journalctl -u cloudflared | grep tunnelID | tail -1`. A mismatch is the double-enrolment trap from Step 5b — point DNS at the running UUID.

**502 on the WebSocket curl with *nothing* in `journalctl -u cloudflared`** — The edge rejected it before the tunnel: almost always the missing `--http1.1` flag (Step 6f); also confirm the zone's **Network → WebSockets** toggle is On.

**502 *with* `ERR` lines in the cloudflared journal** — cloudflared reached Apache and failed: an `x509` message means No TLS Verify isn't actually enabled on the route (or, after C6, the cert/Origin Server Name don't match); `connection refused` means Apache is down (`systemctl status apache2`).

**Everything intermittently broken right after switching over** — Give your router a minute if it was just reconfigured, and hard-refresh the browser; stale error pages and settling network gear produce convincing false alarms.

### Page layer

**Page loads, connects, but shows no data** — The browser reaches the broker but nothing is published on the topic it subscribed to. Confirm test (b), and confirm `mqtt_websockets_topic` matches `<topic>/loop` from the WeeWX `[[MQTT]]` section (`weather/loop` in this guide). Mismatched units between `unit_system` and the site's configuration cause wrong-looking numbers rather than no numbers.

**Browser error `NS_ERROR_WEBSOCKET_CONNECTION_REFUSED` (Firefox) or failed WS connection** — The handshake didn't return 101 somewhere along the chain. Run the Step 6 ladder from (c) upward and find the first failing hop.

**Changed a Belchertown setting but the page behaves as before** — You didn't regenerate the site (Step 4) — the old values are still baked into the published JavaScript. Re-run the report and hard-refresh.

---

## Security notes

- **No inbound ports exist anywhere in this design.** The router forwards nothing; the only public entry point is Cloudflare's edge, reached by cloudflared's outbound connection. Your home IP appears nowhere in DNS.
- Ports 1883 and 9001 are reachable on the LAN only and must **never** be forwarded on the router. For tighter LAN control, restrict the Broker Pi's firewall rules to the webserver Pi's IP only (`sudo ufw allow from <webserver-ip> to any port 9001 proto tcp`, and likewise 1883 if WeeWX is the only publisher).
- `allow_anonymous true` means anything on the LAN can publish. For hardening, set it to `false`, create a password file (`mosquitto_passwd -c /etc/mosquitto/passwd weewx`), reference it with `password_file /etc/mosquitto/passwd`, and give WeeWX the credentials: `server_url = mqtt://weewx:PASSWORD@<broker-ip>:1883/`. Browser subscribers come through the proxy and can be left anonymous or given a read-only ACL.
- The Cloudflare API token (DNS-01 section) should be scoped to **Zone:DNS:Edit on the single zone** and stored root-only (`chmod 600`). If it ever leaks, revoke it in the dashboard — it cannot touch anything but this zone's DNS.
- Origin TLS: No TLS Verify is safe here because the cloudflared→Apache hop never leaves the machine. Use the DNS-01 section if you want a verified origin instead.

---

*Reference installation: WeeWX (pip install, Belchertown skin) + Apache 2.4 + cloudflared on one Raspberry Pi; Mosquitto (Debian packages) on a second Raspberry Pi on the same LAN; public access via Cloudflare Tunnel with no open inbound ports.*

*Ian Millard · ianmillard@icloud.com · Version 1.0 · 12 June 2026*

*An HTML version of this guide (with copy-to-clipboard buttons on each code block) is available alongside this file.*
