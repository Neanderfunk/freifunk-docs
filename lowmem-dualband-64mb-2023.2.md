# Dualband-Router mit 64 MB RAM unter Gluon 2023.2

**Kurz:** Router mit nur 64 MB RAM und **zwei** Funkteilen (2,4 und 5 GHz)
kommen unter Gluon 2023.2 (OpenWrt 23.05) an die Speichergrenze. Die Folgen
reichen von zähem Betrieb über Ethernet, das still nichts mehr empfängt, bis
zu Neustarts in Schleife. Gluon führt viele dieser Geräte deshalb als
`broken`. Mit einigen Entlastungen laufen sie stabil; wie wir das machen und
was man selbst messen kann, steht hier. Wer neu kauft, nimmt ein Gerät mit
**128 MB oder mehr**.

**Gilt für** Gluon 2023.2 (OpenWrt 23.05, Kernel 5.15). Stand 15.09.2026.

---

## Welche Geräte

Betroffen ist die Kombination aus 64 MB RAM und zwei Funkteilen. Geräte mit
64 MB und nur einem Funkteil (z. B. TL-WR1043ND v2) laufen unauffällig.

| Plattform | Geräte (Auswahl) | Funkteile |
| --- | --- | --- |
| ath79 | TP-Link Archer C2 v3, C25 v1, C58 v1, C60 v1, D50 v1, TL-WR902AC v1; AVM FRITZ!WLAN Repeater 1750E | ath9k + ath10k |
| ath79 | D-Link DIR-825 B1, Netgear WNDR3700 (v1/v2) | 2 x ath9k |
| ramips-mt76x8 | Netgear R6120, TP-Link Archer C50 v3, Cudy WR1000 | mt7603 + mt76x2 |
| ramips-mt7620 | TP-Link Archer C20i | rt2800 + mt76x0 |

Am knappsten sind die ath79-Geräte mit ath10k; der Funktreiber belegt dort
selbst viel. Die ramips-Geräte mit mt76 haben im Betrieb spürbar mehr Luft.

## Fehlerbilder

Der Kernel belegt fast alles, für den Datei-Cache bleiben wenige MB. Daraus
entstehen drei Stufen:

1. **Dauerndes Nachladen vom Flash (Thrashing).** Programme und Bibliotheken
   fliegen laufend aus dem Cache und werden vom Flash neu gelesen. Die CPU ist
   dabei weitgehend frei, die Load liegt trotzdem bei 1 bis 2. SSH und
   Statusseite reagieren zäh.
2. **Ethernet empfängt still nichts mehr** (ath79, Treiber ag71xx): kein
   einziges RX-Paket, keine Fehlerzähler, keine Logzeile. Scheitert das
   Nachfüllen des RX-Rings an einer Speicheranforderung, hat der Treiber keinen
   Rückweg; `ip link set down/up` hilft nicht.
3. **Kernel-Panik in Schleife**, typisch mit Meldungen wie
   `skbuff alloc … failed`, `failed to post pci rx buf: -12`,
   `eth0: out of memory` und einem `BUG()` in ag71xx.

Ein Sonderfall: Ein **manueller `sysupgrade` bei laufendem WLAN** kann
scheitern. Die Funktreiber halten ihren Speicher, der Upgrade-Schritt findet
nicht genug, und ein Watchdog startet das Gerät neu. Geflasht wird dabei
nichts, das alte Image bleibt intakt. Der Autoupdater macht es richtig: Er
schaltet vor dem Flashen WLAN und Netz ab.

## Messen, ohne es schlimmer zu machen

* **Kein `echo 3 > /proc/sys/vm/drop_caches`.** Auf einem knappen Gerät treibt
  das alle Dienste gleichzeitig ins Nachladen; wir haben so ein Gerät
  minutenlang unerreichbar gemacht.
* **Die aussagekräftigste Größe ist `workingset_refault_file`** aus
  `/proc/vmstat`: Seiten, die kurz nach dem Verdrängen wieder gebraucht wurden.
  Als Differenz pro Stunde oder Minute betrachten. Dazu `MemAvailable` aus
  `/proc/meminfo` und die Load.
* **Die Speicheranzeige der Karte (`memory_usage`) taugt dafür nicht**: Sie
  zählt den Datei-Cache als belegt, und ein Gerät mit 80 % kann gesund sein
  oder thrashen. Die Load trennt besser.
* **Jede SSH-Sitzung ist selbst Last.** Besser von außen messen, zum Beispiel
  per respondd.

## Was hilft

Die Maßnahmen in der Firmware von Freifunk Neanderland. Die Patches liegen im Firmware-Repo, Branch `v2023.2.x`, unter
[`patches/lowmem/`](https://github.com/Adorfer/firmware/tree/v2023.2.x/patches/lowmem)
und [`patches/kernel/`](https://github.com/Adorfer/firmware/tree/v2023.2.x/patches/kernel),
die Paketlisten in
[`templates/common/image-customization.lua`](https://github.com/Adorfer/firmware/blob/v2023.2.x/templates/common/image-customization.lua).

| Maßnahme | wirkt auf |
| --- | --- |
| zram-swap (komprimierter Swap im RAM): selten genutzte Seiten der Dienste werden komprimiert, mehr Platz für den Cache | nur diese Geräte |
| Features weglassen: TLS, WPA3, SQM für das Mesh-VPN, usteer, sftp-Server | nur diese Geräte |
| kleinere WLAN-Puffer, nach RAM gestaffelt (Backport aus Gluon, dort ab 2025.1 enthalten) | alle, nach RAM |
| `vm.watermark_boost_factor=0`: Der Kernel hebt die Speichergrenzen sonst kurzzeitig um mehrere MB an, und der OOM-Killer schlägt zu früh zu | bis 64 MB |
| minütliche Jobs in Shell statt Lua (`gluon-state-check`, tunneldigger-Watchdog), weniger Nachladen | alle |
| ag71xx: leerer RX-Ring ohne `BUG()` | ath79 |
| kleinere Fragmentpuffer (`ipfrag`/`ip6frag`/`nf_conntrack_frag6`, Backport aus Gluon main) | bis 64 MB |
| `vm.min_free_kbytes` bleibt beim OpenWrt-Wert **8192**, siehe nächster Abschnitt | bis 64 MB |

**Ergebnisse:**

* **Archer C25 v1** (ath9k + ath10k, der knappste Fall): zwei Tage ohne
  Thrashing und ohne ungeplanten Neustart. MemAvailable im Stundenmittel
  2–4 MB, zram mit 1,5–2 MB von 27 MB belegt, Refaults 0–60 pro Stunde statt
  Tausender pro Minute.
* **Cudy WR1000** (mt76): nach dem Update 11 MB MemAvailable, Refaults 0,
  zram praktisch unbenutzt.

## `vm.min_free_kbytes` nicht senken, wenn ath10k im Gerät ist

Naheliegend wäre, auf 64-MB-Geräten weniger Speicher als Reserve
zurückzuhalten. Gluon main macht das: Commit `a505f767` („gluon-core: adapt
sysctl for 64M RAM“) mit dem Fix `c6ac8914` senkt `vm.min_free_kbytes` auf
Geräten mit weniger als 64 MB MemTotal von 8192 (OpenWrt 23.05) auf 2048. Das
bringt einige MB mehr für den Datei-Cache. **Auf Geräten mit ath10k kostet es
unter WLAN-Last das 5-GHz-Funkteil.**

**Symptom:** Unter Last auf 5 GHz steht einmal im Kernel-Log

```
ath10k_pci 0000:00:00.0: rx ring became corrupted: -5
ath10k_pci 0000:00:00.0: HTC Rx: invalid eid 102
```

Danach ist 5 GHz tot: keine Nachbarn, kein Empfang, Sendefehler. Es erholt
sich nicht von selbst, und ein `wifi`-Neustart hilft nicht (das
Mesh-Interface verschwand dabei ganz). Erst ein Neustart des Geräts bringt
5 GHz zurück. 2,4 GHz (ath9k) läuft die ganze Zeit weiter, das Gerät bleibt
erreichbar. Deshalb fällt es im Betrieb leicht nicht auf.

**Ursache:** ath10k füllt seine RX-DMA-Puffer im Interrupt nach, mit
`GFP_ATOMIC`. Solche Anforderungen dürfen nicht warten, bis Speicher
freigeräumt ist; sie leben von der Reserve, deren Größe `min_free_kbytes`
bestimmt. Mit 2048 ist die Reserve so klein, dass das Nachfüllen unter Last
scheitert, und der Ring geht kaputt.

**Gemessen** an einem TP-Link Archer C25 v1 (QCA9561 + QCA9887) mit Gluon
2023.2, einmal mit 2048 und einmal mit 8192, sonst gleich:

| `vm.min_free_kbytes` | Last | Ergebnis |
| --- | --- | --- |
| 2048 | Ping-Flood über die 5-GHz-Mesh-Strecke (direkt per Link-Local, nicht über batman-adv) | sofort `rx ring became corrupted`, 5 GHz tot bis zum Neustart; **3-mal reproduziert** |
| 8192 | dieselbe Last, 2,5 Minuten, ~1200 Pakete/s Empfang | fehlerfrei, 5 GHz stabil, obwohl MemAvailable dabei auf 1,3–2,4 MB sank |

Beim ersten Mal trat der Fehler bei einer Lastprobe mit viel Dateizugriff
auf. Die Speicheranzeige (MemAvailable 12 MB kurz vorher) hat davor nicht
gewarnt; sie zählt den Datei-Cache mit, den atomare Anforderungen nicht
nutzen können.

**Betroffen** sind die 64-MB-Geräte mit ath10k: Archer C2 v3, C25 v1, C58 v1,
C60 v1, D50 v1, TL-WR902AC v1, FRITZ!WLAN Repeater 1750E. Gemessen haben wir
nur den C25 (QCA9887); die anderen nutzen denselben Treiberweg. Geräte mit
128 MB sind nicht betroffen, weil die Einstellung dort nicht greift. Für ath9k
und mt76 mit 2048 haben wir keine Messung, weder für den Nutzen noch für das
Risiko.

**Was wir tun:** Wir übernehmen aus dem Backport nur die kleineren
Fragmentpuffer, `vm.min_free_kbytes` bleibt bei 8192. Zusätzlich erkennt ein
Check im Paket
[`neanderfunk-hotfix`](https://github.com/Neanderfunk/packages/tree/v2023.2.x/neanderfunk-hotfix)
(`ath10k_rxhang`) die Meldung im Kernel-Log und startet das Gerät neu, falls
der Zustand trotzdem auftritt.

**Für andere Communities:** Wer eine Gluon-Version mit `a505f767` **und**
`c6ac8914` baut und ath10k-Geräte mit 64 MB im Netz hat, sollte den Wert für
diese Geräte zurücksetzen, zum Beispiel mit einer Datei, die nach der von
Gluon gelesen wird (die Dateien in `/etc/sysctl.d/` werden alphabetisch
angewendet, die letzte gewinnt):

```
# /etc/sysctl.d/36-min-free-ath10k.conf
vm.min_free_kbytes=8192
```

Ohne `c6ac8914` wirkt `a505f767` nicht (falsche Dateiendung, Vergleich in
falscher Einheit); Gluon v2025.1.3 enthält nur `a505f767`.

## squashfs mit 64 statt 256 KiB Blockgröße?

Gluon baut das Root-Dateisystem mit 256 KiB Blöcken. Mit 64 KiB spart man auf
diesen Geräten etwa 0,7 MB feste Puffer, dafür wird das Image um etwa 7 %
größer. Einstellen lässt sich das nur je Target oder, mit einem Patch, je
Gerät. Wir bleiben bei 256 KiB, weil die Maßnahmen oben ausreichen. Ein
vorbereiteter Mechanismus für die Blockgröße je Gerät liegt in
[`patches/parked/`](https://github.com/Adorfer/firmware/tree/v2023.2.x/patches/parked).

## Firmware aktualisieren

* Am besten über den **Autoupdater**: Er schaltet WLAN und Netz vor dem
  Flashen ab.
* Von Hand: direkt nach einem Neustart, wenn noch am meisten RAM frei ist.
  Oder mit einem Helfer, der vorher Dienste anhält. In der Firmware von
  Freifunk Neanderland macht das der Befehl `flash <image-url>` bzw.
  `sysupgrade <verzeichnis-url>` in der Login-Shell, aus dem Paket
  `neanderfunk-banner`.

## Offen

* Die ramips-Geräte (R6120, C50 v3, WR1000, C20i) stehen erst seit
  September 2026 auf der Liste. Felderfahrung mit den Entlastungen gibt es
  bisher vom WR1000, der erste R6120 folgt.
* Ob ein Zwischenwert für `vm.min_free_kbytes` (etwa 4096) auf ath10k hält,
  ist nicht gemessen. Ebenso offen: ob ath9k- und mt76-Geräte von 2048
  profitieren oder darunter leiden.
* Rückmeldungen und Messwerte sind willkommen, gern als Issue in diesem Repo.
