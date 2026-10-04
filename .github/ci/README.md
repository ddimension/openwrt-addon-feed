# CI- und Build-Infrastruktur (Addon-Feed)

Alles läuft auf **eigenen self-hosted Runnern**. `ddimension` ist ein
User-Account, keine Org → Runner sind **repo-scoped**; ein Runner kann nicht
zwei Repos bedienen. Deshalb hat dieses Repo seine **eigenen** CTs (bei der
Aufteilung am 2026-10-04 drei der elf), angelegt wie im Modem-Feed beschrieben,
nur mit `--repo https://github.com/ddimension/openwrt-addon-feed`:

```bash
cd ~/projects/containers/openwrt-runner && ./mkimg
./pct-create-runner.sh --storage lvm-thin --ctid <id> --name <name> \
  --repo https://github.com/ddimension/openwrt-addon-feed \
  --token-file github-token --docker --data 80 --ip <CIDR> --gw <GW>
gh api repos/ddimension/openwrt-addon-feed/actions/runners   # online? Label openwrt?
```

Die harten Regeln für die CTs (nie privilegiert, `ulimit`-Fix im Image, niemals
`dockerd` von Hand, statische IP, `--data`-Volume, GitHub-PAT statt Docker-PAT)
stehen ausführlich in `.github/ci/README.md` des Modem-Feeds — sie gelten hier
unverändert.

Ein Workflow: **`build.yml`** baut die Pakete aus `.github/ci/packages` für
2 Releases × 8 Architekturen und publiziert nach
`https://ddimension.github.io/openwrt-addon-feed/<kanal>/<release>/<arch>/`.
Geschrieben wird gh-pages **ausschließlich** über `.github/ci/publish-pages.sh`.

## Kanäle

| Ereignis | baut | publiziert |
|---|---|---|
| Push auf `main` | ja | `main/<release>/<arch>/` |
| Push auf `stable` | ja | **nein** |
| Release-Tag `YYYY.MM.DD[.N]` | ja | `stable/<release>/<arch>/` |
| `workflow_dispatch` auf anderem Branch | ja | nein |

stable darf also über mehrere Pushes vorbereitet werden; erst der Tag
(`scripts/release-stable.sh`) macht ein Release. Keinen Alt-Pfad
`<release>/<arch>/` wie im Modem-Feed — dieses Repo entstand nach der
Kanal-Trennung.

## Warum dieses Repo existiert

Der Feed-Build wirft seinen SDK-Baum pro Lauf weg (frischer Container, nur
`dl`/`ccache` sind Volumes). Jede Änderung baute deshalb die komplette
Abhängigkeitskette **aller** Pakete neu — ein Einzeiler in `wwand` auch boost,
hostapd und collectd. Seit der Aufteilung baut ein Push nur noch, was im eigenen
Repo liegt. Gemessen am Modem-Feed vorher: 16 Legs, 51 h Rechenzeit, 5,4 h
Wall-Clock pro Push.

Die teuren Ketten liegen jetzt hier: `snapcast-mptcp` → boost, beide
`wpad`-Varianten → hostapd, `apman` → collectd (gnutls, libmicrohttpd),
`nsca-ng` → openssl. Dafür läuft dieses Repo selten (apman: 1 Commit in 30
Tagen). `timeout-minutes: 420` ist Kopffreiheit, kein Ziel.

## Fallstricke, die wir mitgenommen haben

- **Geteiltes Checkout-Verzeichnis:** alle Jobs eines Runners teilen dasselbe
  Workspace-Verzeichnis. Jeder Job beginnt deshalb mit
  `rm -rf "${GITHUB_WORKSPACE:?}"/*` + vollem Checkout.
- **`--ulimit nofile`** ist in `scripts/local-build.sh` Pflicht: Dockers
  quasi-unbegrenztes fd-Limit lässt fakeroot/`apk mkpkg` in der fd-close-Schleife
  CPU verbrennen.
- **hostapd-Variantenrennen:** `PKG_PARALLEL_VARIANTS` gegen den globalen
  `$(TMP_DIR)/$(1).list` in `include/package-pack.mk` → sporadisch
  `mv: cannot stat …hostapd-utils.list` bzw. „Package wpa-cli is missing
  dependencies". Das trifft genau dieses Repo (beide wpad-Varianten). Gegenmittel
  im SDK-Entrypoint: `-j` auf die cgroup-Quota des CTs begrenzen (Schritt
  „Parallelism") und bei Fehler ein serieller `-j1`-Versuch.
- **Kein docker.io:** die SDK-Action baut ihr Image mit einfachem `docker build`
  aus ghcr.io; die TLS-Timeouts von docker.io haben früher ganze Matrizen
  gerissen.
- **Nur der neueste Commit publiziert** (`publish-feed.sh`): ein Re-Run eines
  älteren Laufs baut, publiziert aber nicht — sonst rollte er den Kanal zurück.
  Bei stable gilt das Äquivalent für Tags: der Tag muss noch auf dem Commit des
  Laufs stehen und das neueste Release sein.
- **`publish-pages.sh` liegt in beiden Feeds.** Nicht-site-spezifische
  Änderungen gehören in beide Kopien.

## Versionshistorie

Ein Feed-Ziel wird nicht ersetzt, sondern zusammengeführt: vorhandene `.apk`
bleiben, gekürzt auf die **neuesten 10 je Paket**
(`publish-feed.sh --keep 'main/*=10' --keep 'stable/*=10'`, `KEEP_VERSIONS`).
Danach baut `.github/ci/apk-retention.sh` `packages.adb` neu und **signiert**
ihn (im Container `image-registry.ddimension.net/myadmin/apk-tools`, weil weder
Runner noch Host apk v3 haben), dazu `index.json` und `versions.json`. Der
Publish-Job braucht dafür `PRIVATE_KEY` und einen Registry-Login
(`REGISTRY_USERNAME`/`REGISTRY_TOKEN`). Auf dem Gerät: `apk add apman=68-r1`
geht zurück und pinnt, `apk add apman` löst den Pin.

Pakete, die der Build nicht mehr erzeugt, verschwinden samt Historie — nur was
der frische Baum enthält, wird mitgenommen.

## Kurzreferenz

```bash
gh run list -R ddimension/openwrt-addon-feed -w build -L 5
gh workflow run build.yml -R ddimension/openwrt-addon-feed --ref main
curl -s https://ddimension.github.io/openwrt-addon-feed/main/snapshot/x86_64/.published
.github/ci/publish-feed.sh <run-id>      # nur der Publish ist gescheitert
scripts/stable-take.sh apman             # nach stable holen
scripts/release-stable.sh                # Release = Tag auf origin/stable
```
