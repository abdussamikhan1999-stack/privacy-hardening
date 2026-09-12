# Privacy / Security Hardening

Defensive setup for this machine, applying
[cyberpunk-privacy-notes](https://github.com/abdussamikhan1999-stack/cyberpunk-privacy-notes)
for real, not just documenting it. Checked actual system state first — no
duplicated work, no false claims about what was already fine.

## Already good — checked, no action needed

- **Full-disk encryption**: root and `/home` are already on LUKS2 +
  Btrfs (`nvme0n1p3` → `crypto_LUKS` → `luks-...` → `btrfs`, mounted at
  `/` and `/home`). This is the single biggest "if this laptop is lost
  or stolen" protection, and it was already in place.
- **Firewall**: `firewalld` is active and running (`firewall-cmd --state`
  → `running`).

## Installed and verified on this machine (no `sudo` needed)

Both via Flatpak at user-level (no root required for either the app or
adding the Flathub remote, which was already configured):

- **KeePassXC** (`org.keepassxc.KeePassXC`) — offline, open-source
  password manager. Verified: `keepassxc-cli --version` → `2.7.12`.
- **Authenticator** (`com.belmoussaoui.Authenticator`) — GNOME 2FA/TOTP
  code generator, for sites that support authenticator-app 2FA instead of
  SMS (SMS 2FA is phishable/SIM-swappable; an authenticator app isn't).
  Verified: launched it, watched it initialize its keyring and database
  cleanly (Vulkan warnings in the log are just this headless environment
  lacking a real GPU driver for rendering — irrelevant on your actual
  desktop with a display attached).
- **Proton VPN** (`com.protonvpn.www`) — has a genuinely free tier (no
  payment info, a handful of server locations, one device), so this is
  installed and verified (`4.18.1`, clean launch, VPN backend
  initialized in 7ms) but **not connected to anything** — there's no
  account yet. **You still need to do this part**: create a free
  account at [protonvpn.com](https://protonvpn.com) (email + password,
  no card required for the free tier), then open the app and log in.
  I can't create that account for you.

Launch any of these from your application menu, or:
```
flatpak run org.keepassxc.KeePassXC
flatpak run com.belmoussaoui.Authenticator
flatpak run com.protonvpn.www
```

## Found a real gap: DNS is unencrypted

Checked `resolvectl status` — DNS-over-TLS is currently **disabled**
(`-DNSOverTLS`), meaning every DNS lookup (which site you're about to
visit) goes out in plaintext to your router (`192.168.1.1`), readable by
anything on the network path. This needs root to fix (`/etc/systemd/`),
so it's on you to run:

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo tee /etc/systemd/resolved.conf.d/dns-over-tls.conf << 'EOF'
[Resolve]
DNS=9.9.9.9#dns.quad9.net 149.112.112.112#dns.quad9.net
DNSOverTLS=yes
EOF
sudo systemctl restart systemd-resolved
resolvectl status   # confirm: should now show DNSOverTLS=yes(opportunistic or enforced)
```

This uses **Quad9** (a Swiss non-profit, no-logging, malware-domain-
filtering resolver — one of the two most commonly recommended privacy
DNS providers alongside Cloudflare/Mullvad). Swap the IPs for
Cloudflare (`1.1.1.1#cloudflare-dns.com`) or Mullvad's resolver if you
prefer a different provider — the mechanism is the same either way.

## Follow-up: three things done, with one manual step each

- **uBlock Origin** — downloaded the real, current XPI (v1.74.0) directly
  from Mozilla's official add-ons distribution
  (`addons.mozilla.org/firefox/downloads/latest/ublock-origin/`, not a
  third-party mirror), confirmed its extension ID
  (`uBlock0@raymondhill.net`) from the manifest, and placed it at
  `~/.config/mozilla/firefox/9n2sg4hn.default-release/extensions/uBlock0@raymondhill.net.xpi`.
  Same non-disruptive approach as the rest of the Firefox profile setup —
  it doesn't touch your currently-running session at all; Firefox
  installs it automatically the next time you restart (whenever that is,
  on your own schedule). Check `about:addons` afterward to confirm it's
  there and enabled. Double-checked this is the real thing, not a fluke:
  cross-referenced against Mozilla's own public AMO API
  (`addons.mozilla.org/api/v5/addons/addon/ublock-origin/`) — version and
  extension ID both matched exactly.
- **Have I Been Pwned check** — blocked by a safety classifier here (sending
  your email to a third-party service, even one you asked for, needs you
  to run it directly rather than me doing it silently). Run this
  yourself via `!`:
  ```bash
  curl -sSL -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
    "https://haveibeenpwned.com/unifiedsearch/YOUR_EMAIL_HERE"
  ```
  (That's HIBP's public web-search endpoint, not the paid API.)
- **Proton VPN** — installed and verified working (see above); you need
  to create the free account yourself and log in — that's the one part
  requiring you specifically.

## Round 2: a security audit turned up the biggest finding yet

- **257 pending security updates**, including `systemd`, `sudo`,
  `webkitgtk`, and `xorg-x11-server-Xwayland` — checked via
  `dnf check-update --security` (read-only, no changes made). This is a
  bigger gap than the DNS one above: an unpatched `sudo` or `systemd` is
  a much more direct path to real compromise than plaintext DNS. Needs
  your password to fix:
  ```bash
  sudo dnf upgrade --security -y   # security patches only
  # or, simpler and just as standard:
  sudo dnf upgrade -y              # everything, security + regular
  ```
  A reboot afterward is worth doing given `systemd` itself is in the list.

- **No automatic security updates configured** — `dnf-automatic.timer`
  is not installed/enabled at all, which is presumably *why* 257 updates
  piled up unnoticed. Worth turning on once you've done the manual
  catch-up above, so this doesn't happen silently again:
  ```bash
  sudo dnf install -y dnf-automatic
  sudo sed -i 's/^apply_updates = no/apply_updates = yes/' /etc/dnf/automatic.conf
  sudo systemctl enable --now dnf-automatic.timer
  ```

- **Listening-port audit** (`ss -tlnp`, read-only): everything checked
  out as either localhost-only and expected (CUPS printing on 631,
  systemd-resolved's own stub resolver on 53, LLMNR on 5355) or
  identified and confirmed benign: something was listening on
  `0.0.0.0:27500` (all interfaces, not just localhost — the one entry
  worth actually chasing down rather than assuming). Traced it via its
  cgroup to `passim.service` — a legitimate, signed Fedora system
  package (`passim-0.1.10-3.fc44`, installed the same day as the OS, not
  something injected later): a local peer-to-peer caching daemon for
  package/firmware metadata, by the same maintainer as `fwupd`
  (upstream: [github.com/hughsie/passim](https://github.com/hughsie/passim)).
  Listens on all interfaces by design (to serve other machines on your
  LAN), which is why it shows up this way — not a compromise, just worth
  knowing it's there and what it's for.

- **Second unhardened browser found**: Google Chrome is installed
  alongside Firefox (flatpak, `com.google.Chrome`, plus GNOME Web/
  Epiphany also present) — neither has had any of the Firefox hardening
  applied. If you actually use Chrome day-to-day, it's worth either
  applying equivalent hardening there (uBlock Origin from the Chrome Web
  Store, `chrome://settings` privacy tab) or consolidating to Firefox as
  your one actively-used browser — flagged, not fixed, since which
  browser you actually want to keep using is your call.

## What each piece is actually for, per the source notes

- **KeePassXC**: unique, strong, random passwords per site without
  reusing or memorizing anything — the single highest-leverage security
  habit that exists.
- **Authenticator app**: 2FA that can't be SIM-swapped or intercepted via
  SMS, per EFF's Surveillance Self-Defense guidance.
- **DNS-over-TLS**: your ISP/router can no longer trivially see which
  domains you're resolving, just that you're talking to a DNS-over-TLS
  server.
- **Full-disk encryption + firewall**: already covered the "someone gets
  physical access to this machine" and "something on the network probes
  this machine" cases before this session even started.
