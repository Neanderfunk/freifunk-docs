# Xiaomi Redmi AX6S / AX3200: Umstieg von Gluon 2023.2 auf 2025.1

**Kurz:** Mit Gluon 2025.1 ändert sich beim Redmi AX6S die Aufteilung des
Flash-Speichers. Ein normales Update kann das nicht, der Umstieg geht nur von
Hand per SSH. **Alle Einstellungen gehen dabei verloren**, eine Sicherung, die
Gluon 2025.1 übernehmen könnte, gibt es nicht.

**Gilt für** den Xiaomi Redmi Router AX6S und den baugleichen Xiaomi Router
AX3200 mit Gluon 2023.2 (OpenWrt 23.05), Umstieg auf Gluon 2025.1
(OpenWrt 24.10). Stand 03.10.2026.

---

## Warum das nicht von allein geht

OpenWrt 24.10 lädt das System bei diesem Gerät anders: Der Xiaomi-Bootloader
startet jetzt einen zweiten Bootloader, und der lädt das System aus einem
UBI-Bereich. Dafür werden die Partitionen `kernel` und `ubi` neu beschrieben.
Ein Firmware-Update aus dem laufenden System kann diesen Wechsel nicht
vornehmen, das steht so auch in den Release Notes von Gluon 2025.1. Bei
Neanderfunk bekommt das Gerät 2025.1 deshalb nicht über den Autoupdater, der
Knoten bleibt auf 2023.2, bis jemand den Umstieg von Hand macht.

Was erhalten bleibt: Die Partitionen `factory` (Funk-Kalibrierung) und `bdata`
(MAC-Adressen) werden nicht beschrieben. Der Knoten behält damit seine
Knoten-ID und erscheint auf der Karte als derselbe Knoten.

Rückweg, falls etwas schiefgeht: serielle Konsole oder TFTP, siehe
"Debricking" im OpenWrt-Wiki (Link unten).

## 1. Einstellungen aufschreiben

Per SSH auf den Knoten und ausgeben lassen:

```
uci export gluon-node-info
uci export gluon
```

Die Ausgabe auf dem eigenen Rechner speichern. Wichtig sind vor allem:

* Knotenname
* Kontakt
* Standort (Koordinaten) und ob er auf der Karte erscheinen soll
* Mesh-VPN an oder aus, Bandbreitenbegrenzung
* privates WLAN, falls genutzt
* Outdoor-Modus, falls genutzt

Alternativ Bildschirmfotos aller Seiten im Konfigurationsmodus machen.

## 2. Bootloader-Einstellungen prüfen

Damit der Xiaomi-Bootloader nicht auf die Original-Firmware zurückfällt,
müssen im Bootloader-Speicher diese Werte stehen:

```
fw_printenv boot_fw1 flag_try_sys1_failed flag_try_sys2_failed flag_boot_success flag_last_success
```

Erwartet:

```
boot_fw1=run boot_rd_img;bootm
flag_try_sys1_failed=8
flag_try_sys2_failed=8
flag_boot_success=1
flag_last_success=1
```

Die aktuelle 2023.2-Firmware von Neanderfunk setzt diese Werte bei jedem
Start selbst. Weicht etwas ab, vor dem Weitermachen setzen:

```
fw_setenv boot_fw1 'run boot_rd_img;bootm'
fw_setenv flag_try_sys1_failed 8
fw_setenv flag_try_sys2_failed 8
fw_setenv flag_boot_success 1
fw_setenv flag_last_success 1
```

## 3. Kalibrierdaten sichern (empfohlen)

```
cat /proc/mtd
```

Die Zeilen mit `"factory"` und `"bdata"` suchen und ihre Nummern einsetzen
(hier als Beispiel `mtd5` und `mtd4`):

```
cat /dev/mtd5 > /tmp/factory.img
cat /dev/mtd4 > /tmp/bdata.img
```

Beide Dateien auf den eigenen Rechner holen, zum Beispiel mit
`scp root@<knoten>:/tmp/*.img .` (klappt das nicht, mit `scp -O`).

## 4. Image besorgen

Gebraucht wird das **factory**-Image von Gluon 2025.1 für die eigene Domain,
der Dateiname endet auf `-xiaomi-redmi-router-ax6s-factory.bin`. Bei
Neanderfunk steht es im Firmware-Selector
([routersoftware.ffnef.de](https://routersoftware.ffnef.de/)), sobald Gluon
2025.1 dort erschienen ist.

Nicht das `sysupgrade`-Image nehmen und nicht das Image von openwrt.org: Das
hätte kein Freifunk.

Das Image nach `/tmp/factory.bin` auf den Knoten kopieren, zum Beispiel mit
`scp` (bzw. `scp -O`).

## 5. Flashen

Ab hier gibt es kein Zurück. Die Befehle stammen aus dem OpenWrt-Wiki für
dieses Gerät (Link unten) und funktionieren genauso aus Gluon heraus:

```
cd /tmp
mount -o remount,ro /
mount -o remount,ro /overlay
dd if=factory.bin bs=1M count=4 | mtd write - kernel
dd if=factory.bin bs=1M skip=4 | mtd -r write - ubi
```

Meldet die erste `mount`-Zeile `Resource busy`, einfach weitermachen, das
steht so auch im Wiki. Nach der letzten Zeile startet der Router von selbst
neu. Währenddessen nicht den Strom ziehen.

## 6. Neu einrichten und Updates wieder möglich machen

Der Knoten startet im Konfigurationsmodus, wie ein neu gekaufter
Freifunk-Router. Die Werte aus Schritt 1 wieder eingeben.

Danach per SSH die Kennung `compat_version` auf `2.0` setzen. Images ab 2025.1
tragen für dieses Gerät `2.0`; steht im System noch `1.0`, lehnt der Router
jedes weitere Update ab, auch über den Autoupdater. Neuere Neanderfunk-Images
setzen den Wert bei der Installation selbst, der Befehl schadet dann nicht:

```
uci set system.@system[0].compat_version='2.0'
uci commit system
```

Prüfen:

```
uci get system.@system[0].compat_version
```

Hier muss jetzt `2.0` stehen.

## 7. Bescheid sagen

Kurz an projekt@neanderfunk.de schreiben, wenn der Knoten umgestellt ist.
Danach nehmen wir das Gerät wieder in die automatischen Updates auf.

---

## Quellen

* OpenWrt-Wiki, Xiaomi AX3200 / Redmi AX6S, Abschnitt "Upgrading from 23.05
  and earlier to upcoming 24.10 or snapshot" (Flash-Befehle, Hinweis auf
  `compat_version`) und "Debricking": https://openwrt.org/toh/xiaomi/ax3200
* Gluon v2025.1 Release Notes, Liste der Geräte, die nicht automatisch
  aktualisiert werden können:
  https://gluon.readthedocs.io/en/latest/releases/v2025.1.html
* OpenWrt 24.10, `target/linux/mediatek/image/mt7622.mk`
  (`DEVICE_COMPAT_VERSION := 2.0` für dieses Gerät) und
  `package/base-files/files/lib/upgrade/fwtool.sh` (lehnt Images mit anderer
  Hauptversion ab)

Lizenz: CC BY-SA 4.0
