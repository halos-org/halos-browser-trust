# halos-browser-trust - Agent Instructions

## Repository Purpose

Debian package that makes on-device browsers on a HaLOS desktop trust CAs
installed via `update-ca-certificates` — most importantly the device's own
auto-CA. Two mechanisms, because the two browser families resolve trust
differently:
- **Chromium / system-NSS apps:** bridge `libnssckbi.so` → `p11-kit-trust.so`.
- **Firefox:** bundles its own NSS/root store, so an enterprise policy installs
  the system CA file instead.

## Background

On Debian, NSS trusts only the static Mozilla bundle in `libnssckbi.so` and
ignores `/etc/ssl/certs`. Fedora and Arch symlink `libnssckbi.so` to
`p11-kit-trust.so` by default; Debian does not. This package adds that bridge.
It is the browser-side complement to halos-core-containers' `halos-manage-certs`,
which installs the device CA into the system trust store (issue #158).

Use `p11-kit-trust.so`, never `p11-kit-proxy.so` (the proxy module breaks Firefox
HTTPS — see p11-kit#172).

## For Agentic Coding: Use the HaLOS Workspace

Work from the halos workspace repository for full context across all HaLOS repositories.

## Quick Start

```bash
./run lint         # shellcheck the maintainer scripts
./run build-deb    # Build .deb package
./run clean        # Remove build artifacts
```

## How It Works

1. `debian/postinst` (`configure`) preserves libnss3's real `libnssckbi.so` as
   `libnssckbi.so.distrib` and replaces it with a symlink to the sibling
   `pkcs11/p11-kit-trust.so`.
2. `debian/triggers` declares `interest-noawait` on the `libnssckbi.so` path. A
   `libnss3` upgrade re-unpacks its own file over the symlink, then dpkg runs the
   trigger at the end of the transaction (`postinst triggered`), which re-asserts
   the symlink and refreshes `.distrib` with libnss3's current module. This
   self-heals across NSS upgrades — a plain `dpkg-divert` does **not** (its
   `--rename` artifact flips the symlink to `.distrib` on libnss3 re-unpack;
   verified on-device).
3. `debian/postrm` (`remove`/`purge`) restores stock NSS by renaming `.distrib`
   back, guarded so a `purge` after `remove` is a no-op.
4. HaLOS is arm64-only, so the multiarch path is hardcoded. The only shipped
   file is the Firefox policy (below); everything else is maintainer scripts +
   trigger, so the package stays `Architecture: all`.

### Firefox

Firefox does not use the system `libnssckbi.so` — it bundles its own NSS and
root store — so the bridge above does not reach it. The package ships
`/etc/firefox/policies/policies.json` with `Certificates.Install` pointing at
the system CA file (`/usr/local/share/ca-certificates/halos-ca.crt`, maintained
by `halos-manage-certs`), which Firefox reads at startup. This is Mozilla's
documented Linux mechanism — `Certificates.ImportEnterpriseRoots` is
Windows/macOS-oriented and does not read the system store on Linux (verified
on-device). Because the policy points at the CA *file*, it tracks rotation with
no per-user state.

## Verification

Verify with a real on-device browser, not `curl`/`openssl` — those use the system
store and would mask a browser-side failure. Check **both** families:
- Chromium: headless `--dump-dom https://<device>.local/` loads the dashboard.
- Firefox: headless `--screenshot` of a static device page (e.g. `/ca/`) renders
  instead of hanging on the cert-error interstitial.
Stock (no package) yields a cert error in both.

## Version Management

Use `./run bumpversion [patch|minor|major]`. Never edit VERSION or debian/changelog manually.

## CI/CD

Uses shared-workflows for Debian package building:
- **pr.yml**: PR checks (shellcheck, lintian)
- **main.yml**: Builds and publishes to apt.halos.fi unstable on push to main
- **release.yml**: Publishes to apt.halos.fi stable when a release is published

## Related

- **halos-metapackages**: the `halos-desktop` metapackage depends on this package
- **halos-core-containers**: `halos-manage-certs` installs the device CA into the
  system trust store (issue #158); this package makes the browser honor it

Part of the [HaLOS](https://github.com/halos-org/halos) distribution.
