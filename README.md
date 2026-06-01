# halos-browser-trust

Debian package that makes NSS-based applications (Chromium, Firefox) on a HaLOS desktop trust the system CA store.

## Why

On Debian, NSS loads its trust roots from the static Mozilla bundle in `libnssckbi.so` and ignores `/etc/ssl/certs`. So a HaLOS device that has installed its own auto-CA into the system trust store (via `update-ca-certificates`) is still not trusted by the on-device browser — opening the local dashboard on the device's own screen shows a certificate warning.

Fedora and Arch ship `libnssckbi.so` as a symlink to `p11-kit-trust.so`, which exposes the system trust store to NSS. Debian doesn't. This package adds that bridge.

## How it works

On install, the package replaces `/usr/lib/<triplet>/libnssckbi.so` with a symlink to `p11-kit-trust.so` (preserving the original as `.distrib`). NSS applications then trust everything in the system p11-kit store: the Mozilla roots (from `ca-certificates`) plus any CA installed via `update-ca-certificates`.

- A dpkg file trigger on the `libnssckbi.so` path re-asserts the symlink whenever `libnss3` is upgraded, so the bridge self-heals across NSS updates.
- Tracks CA rotation for free — no per-user `nssdb`, no resync.
- Purge/remove restores stock NSS trust.

## Part of HaLOS

Pulled in by the `halos-desktop` metapackage, so it lands only on desktop variants. Part of the [HaLOS](https://github.com/halos-org/halos) distribution.

## License

MIT
