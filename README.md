# browser-leak-audit

A checklist skill for auditing a browser for leaks while using a VPN — WebRTC, DNS, IPv6, canvas fingerprinting — with a tested Chrome Secure DNS fix.

Built from a breakdown of the [Habr leak-survey (Nov 2026)](https://habr.com/ru/articles/1111276/) and a real audit run.

## The problem

A VPN changes only the network path (IP). The browser keeps reporting the timezone, language, WebRTC candidates, DNS resolvers, and canvas/WebGL fingerprint. A site can compare the IP geography against these parameters — even when the VPN itself works perfectly.

Six leak categories:

| # | Type | What happens |
|---|------|--------------|
| 1 | IP | Traffic exits via an unexpected interface |
| 2 | DNS | Name resolution bypasses the VPN tunnel |
| 3 | WebRTC | Browser uses its own network mechanisms |
| 4 | IPv6 | VPN covers IPv4 only |
| 5 | Environment mismatch | Language/timezone don't match IP geolocation |
| 6 | Fingerprinting | Characteristic combo recognizes you between visits |

**Mismatch ≠ leak.** A Russian language at a US IP is normal (traveler, emigrant, remote worker). Two distinct strategies exist: *alignment* (tune the browser to the VPN geo) and *standardization* (make everyone look the same, e.g. Mullvad Browser). For a single personal server, chasing full geo-alignment is unnecessary.

## The tests

Open these in the browser under test:

| Test | What to look at | Good | Bad |
|------|-----------------|------|-----|
| `browserleaks.com/webrtc` | public IP, local IP | IP = server IP, local = 127.0.0.1 / mDNS | Real ISP IP exposed |
| `test-ipv6.com` | IPv6 address, score | No v6 (0/10) = no hole | Has IPv6 the VPN doesn't cover |
| `browserleaks.com/dns` | resolver list | One provider, one location | Many providers/locations = leak |
| `ipleak.net` | IP, WebRTC, DNS, fallback | Everything on server IP | Local IP, foreign resolvers |
| `browserleaks.com/canvas` | fingerprint uniqueness | — | 100% unique = recognizable (not a leak) |

Reading screenshots via OCR (more reliable than vision models for these pages):

```bash
tesseract screenshot.jpeg /tmp/ocr_out -l rus+eng && cat /tmp/ocr_out.txt
```

## Local diagnostics (macOS)

```bash
scutil --dns | grep nameserver        # system DNS
ifconfig | grep inet6                 # global IPv6 (ignore fe80/::1)
sudo -n true                          # passwordless sudo available?
```

## Fix: pin Cloudflare DoH in Chrome

Symptom: `browserleaks.com/dns` shows **Cloudflare + Google** (the Secure DNS auto-select pattern) or local-provider resolvers while a VPN is on.

1. Open `chrome://settings/security`
2. "Use secure DNS" → **"With"** (custom provider)
3. Enter: `https://cloudflare-dns.com/dns-query`
4. **Fully restart Chrome** (not just the tab)
5. Re-run the test: it should show **Cloudflare only**, with Google gone

A managed policy (`DnsOverHttpsMode` / `DnsOverHttpsTemplates` in the Chrome plist under `/Library/Managed Preferences`) needs root; on a personal machine the UI path above is simpler.

**Why Cloudflare isn't a problem under national DNS blocking:** DoH travels inside the tunnel and exits from the VPN server, so domestic blocks never see the path. If the VPN drops, resolution simply fails (fail-closed) and Chrome falls back to the system resolver.

## What NOT to do

- **Don't stack anti-fingerprinting extensions.** Chameleon + CanvasBlocker + a UA spoofer fight over the same parameters and make the fingerprint *more* unique. The number of overridden parameters is a poor measure of protection.
- Don't chase full geo-alignment of every parameter on a single personal server.
- Don't blindly disable WebRTC — it breaks video calls. If needed, use `webRTCIPHandlingPolicy=disable_non_proxied_udp`.
- Don't turn off Secure DNS in favor of the local router — that sends queries to the local provider instead.

## Limitations

The DNS test shows **which** resolvers answered, not the **path** the query took. A clean IP/WebRTC result is strong circumstantial evidence for the path. For certainty, watch whether local-provider resolvers appear while the VPN is on.

## License

MIT
