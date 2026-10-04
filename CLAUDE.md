# openwrt-addon-feed — guide for Claude

The add-on half of our OpenWrt package world, github.com/ddimension/openwrt-addon-feed:
AP management, multiroom audio, the wpad variants, monitoring, Lua bindings.
The modem stack and the `ddimension-feed` bootstrap package live in
github.com/ddimension/openwrt-repo (`~/projects/ddimension-openwrt-repo`); the
two feeds are independent and share one signing key. Reasoning and the package
table are in `README.md`, CI in `.github/ci/README.md`. Everything is English.
Commit/push only when asked.

## Branches are channels

| Branch | Publishes | Moves by |
|---|---|---|
| `main` | `…/main/<release>/<arch>/` — development | every push; this is where you work |
| `stable` | `…/stable/<release>/<arch>/` — releases | `scripts/stable-take.sh`, cherry-picks; published **only by a release tag** |

- Commit on `main`. Check `git branch --show-current` first.
- A push to `stable` builds and publishes nothing. A release is an annotated tag
  `YYYY.MM.DD[.N]` made by `scripts/release-stable.sh` on a commit that is on
  origin/stable and has a green build.
- `stable` is a line of its own, not a pointer onto main: take what is ready
  (`scripts/stable-take.sh <pkg>…`, `--ci` for `.github/`, `--all` to merge
  main), leave the rest. A fix may be made on stable and merged up into main.

## Bumping a package

| Package | How |
|---|---|
| git-source packages (`apman`, `snapcast-mptcp`, `luacurl`, `lua-mosquitto`, `nsca-ng`, `usb-relay-hid`, `libubus-lua-async`) | bump `PKG_SOURCE_VERSION` (+ `PKG_VERSION`/`PKG_SOURCE_DATE` as the Makefile uses them), `PKG_RELEASE`+1, then `scripts/update-hashes.sh <pkg>` |
| `apman` | from apman-agent: `contrib/release.sh` ([apman/README.md](apman/README.md)) |
| built from `files/` in this repo (`homesync`, `wpad-ieee8021x`) | edit, bump `PKG_RELEASE` |
| `wpad-saeradh2e` | OpenWrt `hostapd` + our patches; version is `<date>~<commit>` of the tracked hostapd commit |
| `heatingrod` | git-archive snapshot in `files/`, `PKG_HASH` ([heatingrod/README.md](heatingrod/README.md)) |

- **`PKG_MIRROR_HASH` only from the SDK** (`scripts/update-hashes.sh`). It is
  all-or-nothing: on `FAILED` it touches no Makefile — read `$LOGDIR/hashes.txt`.
- One commit per bump, Makefile and hash together: the old Makefile breaks on
  the new tarball.
- `heatingrod` is not built by CI (rust host toolchain, ~35-45 GB build dir);
  `pcie_mhi` and `python3-edlclient` do not live here at all.

## CI facts that bite

- The feed name here is **`ddaddon`**, not `wwand` (`FEEDNAME` in `build.yml`,
  `src-link` in `scripts/sdk-inner.sh`/`update-hashes.sh`). A build tree that
  carries both feeds must not have two entries of the same name.
- A push to `main` builds and publishes main only. `cancel-in-progress` is per
  ref: **one push, then wait**. `.md`-only pushes build nothing (path filters do
  not apply to tags).
- New package: add it to `.github/ci/packages` (the one list for CI and
  `scripts/local-build.sh`), or say in its README why not.
- gh-pages is written only by `.github/ci/publish-pages.sh`. Never push gh-pages
  by hand. This repo publishes package trees only — no images, no host tools.
- `.github/ci/publish-pages.sh` exists in both feeds. A change that is not
  site-specific belongs in both; say so in the commit message.
- Before pushing CI changes: `docker run --rm -v "$PWD:/repo:ro" -w /repo rhysd/actionlint`
  and `docker run --rm -v "$PWD:/mnt:ro" -w /mnt koalaman/shellcheck:stable -x <scripts>`.
- Anything big: test locally first,
  `RELEASES=snapshot ARCHS=x86_64 PACKAGES="<pkg>" scripts/local-build.sh`.
- Expect long legs: `snapcast-mptcp` drags boost, both wpad variants build
  hostapd, `apman` pulls collectd. The SDK tree is thrown away per run, so every
  leg builds those from scratch (timeout 420 min).

## What is live

- `curl -s https://ddimension.github.io/openwrt-addon-feed/<channel>/<release>/<arch>/.published`
  → UTC time, channel, source commit, run id of that tree.
- `gh run list -R ddimension/openwrt-addon-feed -w build -L 5`.
- On a device: `apk list -I | grep -E 'apman|snap|wpad'`,
  `cat /etc/apk/repositories.d/*.list`.
