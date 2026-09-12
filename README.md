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

Launch either from your application menu, or:
```
flatpak run org.keepassxc.KeePassXC
flatpak run com.belmoussaoui.Authenticator
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

## Deliberately not done without asking first

- **Browser (uBlock Origin, etc.)**: your real Firefox profile already
  has Betterfox's `user.js` applied (see
  [firefox-customization-notes](https://github.com/abdussamikhan1999-stack/firefox-customization-notes)),
  which covers a lot of the tracking-protection ground already
  (`browser.contentblocking.category = "strict"`, etc.). Installing an
  extension into your *live, currently-running* browser is a further step
  I didn't take unprompted — say the word and I'll do it the same
  verified way as the rest of that profile setup.
- **Have I Been Pwned check**: this needs your email address sent to a
  third-party service. I won't do that without you explicitly asking —
  tell me which email(s) to check and I'll look them up.
- **VPN client**: no VPN provider account exists to configure one against.
  If you have (or want) a specific provider (Mullvad/ProtonVPN/IVPN are
  the ones cyberpunk-privacy-notes names), tell me which and I'll set up
  the client.

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
