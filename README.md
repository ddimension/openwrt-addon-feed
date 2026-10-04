# ddimension OpenWrt add-on feed

Everything in our OpenWrt package world that is **not** the modem stack: AP
management, multiroom audio, the wpad variants, monitoring and the Lua bindings
they need. Built for every OpenWrt release × package architecture and published
as a signed apk repository on GitHub Pages.

The modem side — `wwand` with its LuCI apps, `qlog`/`qfirehose`/`qflash` and the
`ddimension-feed` bootstrap package — lives in
[ddimension/openwrt-repo](https://github.com/ddimension/openwrt-repo). The two
feeds are independent: nothing here depends on anything there, and both are
signed with the **same key**, so a device that trusts one trusts the other.

This repo was split off on 2026-10-04. Reason: a one-line change in `wwand` used
to rebuild `boost`, `hostapd` and `collectd` too, because the feed build throws
its SDK tree away on every run. Now a push here only builds what is here, and a
push there only what is there.

## Packages

| Package | What it is | Binary packages | Source | CI |
|---|---|---|---|---|
| `apman` | AP manager: ubus↔MQTT bridge, collectd plugin, on-AP RADIUS server ([apman/README.md](apman/README.md)) | same | [ddimension/apman-agent](https://github.com/ddimension/apman-agent) | ✓ |
| `libubus-lua-async` | the stock ubus Lua binding plus `conn:call_async()` | same | upstream [ubus](https://git.openwrt.org/project/ubus.git) | via `apman` |
| `homesync` | synchronised multiroom speaker: UCI front end for `snapclient-mptcp` | same | local (`files/`) | ✓ |
| `snapcast-mptcp` | Snapcast with Multipath-TCP | `snapserver-mptcp`, `snapclient-mptcp` | [ddimension/snapcast](https://github.com/ddimension/snapcast) | ✓ |
| `luacurl` | Lua binding for libcurl | same | upstream [Lua-cURL/Lua-cURLv3](https://github.com/Lua-cURL/Lua-cURLv3) | ✓ |
| `lua-mosquitto` | Lua binding for libmosquitto | same | upstream [flukso/lua-mosquitto](https://github.com/flukso/lua-mosquitto) | ✓ |
| `nsca-ng` | NSCA-ng client (`send_nsca`): passive check results to Nagios/Icinga over TLS-PSK; what `wwand-apntest` (other feed) reports through | same | upstream [weiss/nsca-ng](https://github.com/weiss/nsca-ng) | ✓ |
| `usb-relay-hid` | control for cheap USB HID relay boards | same | upstream [OzFalcon/usb-relay-hid](https://github.com/OzFalcon/usb-relay-hid) | ✓ |
| `wpad-ieee8021x` | `ieee8021x` netifd protocol: wired 802.1X through wpa_supplicant's ubus interface | same | local (`files/`) | ✓ |
| `wpad-saeradh2e` | OpenWrt's full/OpenSSL wpad plus our SAE-over-RADIUS patches (also in [ddimension/hostapd](https://github.com/ddimension/hostapd) `sae-radius-h2e`) | same | OpenWrt `hostapd` + patches | ✓ |
| `heatingrod` | PV-surplus heating rod controller, Rust ([heatingrod/README.md](heatingrod/README.md)) | same | bundled snapshot of [heatingrod-controller](https://github.com/ddimension/heatingrod-controller) | — |

`heatingrod` is deliberately not built by CI: it compiles a Rust host toolchain
from source (~35–45 GB of build directory). Build it on demand with
`scripts/local-build.sh`.

## Branches are channels

| Branch | Publishes | Moves by |
|---|---|---|
| `main` | `…/main/<release>/<arch>/` — development | every push; this is where you work |
| `stable` | `…/stable/<release>/<arch>/` — releases | `scripts/stable-take.sh` / cherry-picks; **a release is a tag** (`scripts/release-stable.sh`) |

A push to `stable` only builds. Only a release tag `YYYY.MM.DD[.N]` publishes the
stable channel, so stable can be prepared over several pushes. Same model as the
modem feed, and the same scripts.

## Binary package repositories

```
https://ddimension.github.io/openwrt-addon-feed/main/<release>/<arch>/
https://ddimension.github.io/openwrt-addon-feed/stable/<release>/<arch>/
```

`<release>` is `snapshot` or `openwrt-25.12`, `<arch>` the package architecture
(`apk --print-arch`, e.g. `aarch64_cortex-a53`). Each tree carries a
`.published` stamp — `<UTC time> <channel> <source commit> <run id>` — so you
can see what is live:

```sh
curl -s https://ddimension.github.io/openwrt-addon-feed/stable/openwrt-25.12/mipsel_24kc/.published
```

There is no pre-channel mirror `…/<release>/<arch>/` here: this repo was created
after the channel split, so no device ever followed one.

## On the device

Install `ddimension-feed` from the modem feed once — it writes one `.list` per
feed and brings the signing key:

```sh
apk --allow-untrusted \
  -X https://ddimension.github.io/openwrt-repo/stable/<release>/<arch>/packages.adb \
  add ddimension-feed
apk update
apk add apman            # or snapclient-mptcp, wpad-saeradh2e, …
```

By hand, for a single device:

```sh
cat >/etc/apk/repositories.d/ddimension-addon.list <<EOF
https://ddimension.github.io/openwrt-addon-feed/stable/<release>/<arch>/packages.adb
EOF
apk update
```

The key is the same one the modem feed uses
(`https://ddimension.github.io/openwrt-repo/keys/ddimension.pem`, also in
`keys/` here); a device that has it already needs nothing new.

## Usage as a feed in a build tree

```
src-git ddaddon https://github.com/ddimension/openwrt-addon-feed.git;stable
```

The feed name must **not** be `wwand` — that is the modem feed's name, and a
tree carrying both would have two `src-git` entries of the same name.

## Local test builds

```sh
RELEASES=snapshot ARCHS=x86_64 PACKAGES=apman scripts/local-build.sh
```

Same SDK containers and the same checks as CI. The mandatory
`--ulimit nofile` is inside the script. `scripts/update-hashes.sh <pkg>` fills
`PKG_MIRROR_HASH` after a source bump — always from the SDK, never by hand.

## Updating a package

1. Bump `PKG_VERSION`/`PKG_SOURCE_VERSION` and `PKG_RELEASE` in the package's
   `Makefile`; packaging-only changes bump `PKG_RELEASE` alone.
2. `scripts/update-hashes.sh <pkg>` and commit the Makefile change together with
   it — the old Makefile does not build the new tarball.
3. Push to `main` (one push, then wait: `cancel-in-progress` means a burst
   cancels everything but the last run).
4. For a release: take it onto `stable` (`scripts/stable-take.sh <pkg>` or
   `git cherry-pick -x`), let stable build, then `scripts/release-stable.sh`.

New package: add it to `.github/ci/packages`, or say in its README why not.

## CI

Runners, workflows, the gh-pages publisher and the release flow:
[.github/ci/README.md](.github/ci/README.md).
