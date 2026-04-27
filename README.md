# trusttunnel.sh

Installer and manager script for [TrustTunnel](https://github.com/TrustTunnel/TrustTunnel)
VPN endpoint, pinned to upstream **v1.0.33** and rebranded for the
`thekhabaroff` fork.

Single-file bash entrypoint: it installs the `trusttunnel_endpoint` binary
via the upstream installer, generates the four TOML configs
(`vpn.toml`, `hosts.toml`, `credentials.toml`, `rules.toml`), wires up the
`trusttunnel.service` systemd unit, and provides an interactive management
menu on subsequent runs.

## Quick install

```shell
curl -fsSL https://raw.githubusercontent.com/thekhabaroff/trusttunnel/devin/1777241942-followup-review-fixes/trusttunnel.sh \
  -o /root/trusttunnel.sh && chmod +x /root/trusttunnel.sh && sudo /root/trusttunnel.sh
```

Subsequent runs (after configuration) drop straight into the management menu.

## Layout

| Item | Path |
| --- | --- |
| Install dir / config dir | `/opt/trusttunnel` |
| Service unit | `/etc/systemd/system/trusttunnel.service` |
| Marker file | `/opt/trusttunnel/.trusttunnel_configured` |
| Manager script | `/root/trusttunnel.sh` |
| Upstream installer | <https://raw.githubusercontent.com/TrustTunnel/TrustTunnel/refs/heads/master/scripts/install.sh> |
| Self-update URL | <https://raw.githubusercontent.com/thekhabaroff/trusttunnel/master/trusttunnel.sh> |

## What changed compared to the upstream `tt-installer`

Rebased on top of `deathline94/tt-installer/main/installer.sh` and brought
in line with TrustTunnel **v1.0.33**:

- **Pinned endpoint version** — `install_trusttunnel` now calls the
  upstream installer with `-V 1.0.33 -o /opt/trusttunnel`, so the binary
  drop matches the config schema this script generates.
- **Diagnostic handlers** (added in TrustTunnel 1.0.17):
  `vpn.toml` now declares `ping_enable`, `ping_path`, `speedtest_enable`,
  `speedtest_path`. The wizard asks before enabling them, both default to
  `false`.
- **Auth failure status code** (1.0.17): `auth_failure_status_code = 407`
  is written explicitly so behavior is stable across upgrades.
- **Per-client connection limits** (1.0.7): the wizard prompts for global
  defaults `default_max_http2_conns_per_client` /
  `default_max_http3_conns_per_client`, and the `Add User` menu accepts
  per-user `max_http2_conns` / `max_http3_conns` overrides.
- **`client_random_prefix` rules** (1.0.1 / 1.0.28):
    - `rules.toml` is generated with header comments documenting
      `cidr` / `client_random_prefix` matchers (exact and `prefix/mask`
      bitwise format).
    - `show_client_config` invokes
      `trusttunnel_endpoint -c USER -a ADDR --client-random-prefix` so the
      endpoint auto-generates the prefix, appends the matching `allow` rule
      to `rules.toml`, and embeds the value in the exported client
      config / deep-link. Falls back to the old invocation if the flag
      isn't supported.
- **Deep-link `tt://?...`** (1.0.13): the post-install banner advertises
  the new format; the legacy `tt://` form is still accepted by clients.
- **Systemd unit reuse**: when the upstream installer drops
  `trusttunnel.service.template`, `setup_systemd_service` copies it
  verbatim to `/etc/systemd/system/trusttunnel.service` instead of
  shipping its own copy, so any `ExecStart` changes between releases are
  picked up automatically. An embedded fallback unit is still used if
  the template is absent.
- **Status/configuration view** now reports the diagnostic handler
  toggles, the per-client connection caps, the auth failure status code,
  and the actual installed binary version (`trusttunnel_endpoint --version`).
- **Marker file** persists the new fields (`PING_ENABLED`,
  `SPEEDTEST_ENABLED`, `DEFAULT_MAX_HTTP2_CONNS`,
  `DEFAULT_MAX_HTTP3_CONNS`, `TRUSTTUNNEL_VERSION`).
- **Branding/path changes** to match this fork:
  `INSTALLER_URL`, `MANAGER_SCRIPT=/root/trusttunnel.sh`, banner.

## Requirements

- Linux on `x86_64` or `aarch64`.
- `root` (`sudo`).
- `bash`, `curl`, `openssl`, `systemctl`.

## Management menu

After the first run the script becomes a state-aware menu:

| # | Action |
| --- | --- |
| 1 | Start service |
| 2 | Stop service |
| 3 | Restart service |
| 4 | View live `journalctl` logs |
| 5 | Show status |
| 6 | Edit one of the four TOML files |
| 7 | Add user (with optional per-user H2/H3 caps) |
| 8 | Show client config / deep-link |
| 9 | Reinstall endpoint (preserves configs) |
| 10 | Uninstall (removes everything under `/opt/trusttunnel`) |

## License

Same MIT license as the rest of the repository.
