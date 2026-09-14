# LAN-Ports trennen (Gluon 2023.2, Geräte mit DSA)

Anleitung zum Abarbeiten per SSH. Alle Befehle sind zum Kopieren gedacht.
Ersetze `NODE` durch die Adresse deines Knotens.

**Gilt für** Gluon 2023.2 auf Geräten mit DSA und mehreren LAN-Ports, zum
Beispiel:

| Gerät | LAN-Ports | WAN-Port |
| --- | --- | --- |
| GL.iNet GL-B1300 | `lan1 lan2` | `wan` |
| Xiaomi Mi Router 4A Gigabit | `lan1 lan2` | `wan` |
| AVM FRITZ!Box 4040 | `lan1 lan2 lan3 lan4` | `wan` |
| TOTOLINK X5000R, TP-Link Archer AX23, TL-WDR4900 | `lan1 lan2 lan3 lan4` | `wan` |
| ASUS TUF-AX4200 | `lan1 lan2 lan3 lan4` | `eth1` |
| Cudy WR3000, ZyXEL WSM20, ASUS RT-AX53U, Redmi AX6S | `lan1 lan2 lan3` | `wan` |
| MERCUSYS MR90X | `lan0 lan1 lan2` | `eth1` |
| Ubiquiti EdgeRouter X | `eth1 eth2 eth3 eth4` | `eth0` |
| Ubiquiti EdgeRouter X SFP | `eth1 eth2 eth3 eth4 eth5` | `eth0` |

Die Portnamen unterscheiden sich also von Gerät zu Gerät. Maßgeblich ist, was
Schritt 1 auf deinem Knoten anzeigt. In den Beispielen unten steht `lan1` bis
`lan4` — setze deine Namen ein.

**Gilt nicht für** Geräte mit altem Switch-Treiber (swconfig): Archer C7, C6,
C5, TL-WR1043, TL-WDR3600/4300, Netgear R6120 und ähnliche. Dort lassen sich
die LAN-Ports so nicht trennen. Ob dein Gerät passt, sagt dir der erste Befehl
in Schritt 1.

**Den WAN-Port nie anfassen.** Über ihn hängt der Knoten meist am Netz; wer ihn
umstellt, sperrt sich aus. Einzige Ausnahme: ein WAN-Port, der nicht zuverlässig
läuft (Cudy TR3000), siehe Abschnitt 7.

Die Rollen:

* **Mesh**: an diesem Port hängt ein weiterer Freifunk-Knoten
* **Client**: an diesem Port hängen normale Geräte (Laptop, Drucker, Kamera)
* **Uplink**: Internetzugang des Knotens (normalerweise nur WAN)

---

## 1. Einloggen und nachsehen

```sh
ssh root@NODE
```

Auf dem Knoten zuerst prüfen, ob die Anleitung zu deinem Gerät passt (eine
Zeile, komplett kopieren):

```sh
if ! ls -d /sys/class/net/*/dsa >/dev/null 2>&1; then echo "Leider kein DSA-Geraet (alter Switch-Treiber swconfig oder gar kein Switch). Mit dieser Anleitung geht es derzeit nicht."; elif [ "$(cat /lib/gluon/core/sysconfig/lan_ifname 2>/dev/null | wc -w)" -lt 2 ]; then echo "DSA-Geraet, aber weniger als zwei LAN-Ports. Hier gibt es nichts zu trennen."; else echo "OK! DSA-Geraet, LAN-Ports: $(cat /lib/gluon/core/sysconfig/lan_ifname). Weiter in der Anleitung."; fi
```

| Ausgabe | Bedeutung |
| --- | --- |
| `OK! DSA-Geraet, LAN-Ports: …` | passt, weiter unten |
| `Leider kein DSA-Geraet …` | **hier aufhören**: alter Switch-Treiber (swconfig) oder gar kein Switch. Ausnahme Cudy TR3000/GL-MT3000 mit Problemen am WAN: Abschnitt 7 |
| `DSA-Geraet, aber weniger als zwei LAN-Ports …` | **hier aufhören**: nur ein Port, nichts zu trennen |

Woran der Befehl es erkennt: Bei DSA hat die Schnittstelle zwischen SoC und
Switch im Kernel ein Verzeichnis `dsa` (`/sys/class/net/eth0/dsa`, beim
EdgeRouter X `/sys/class/net/dsa/dsa`). swconfig-Geräte haben das nicht, dort
zeigt `swconfig list` einen `switch0`, und die LAN-Ports heißen `eth0.1` o. ä.

Dann nachsehen, wie die Ports heißen und welche Rolle sie haben:

```sh
uci show gluon | grep iface_
echo "LAN: $(cat /lib/gluon/core/sysconfig/lan_ifname)"
echo "WAN: $(cat /lib/gluon/core/sysconfig/wan_ifname)"
```

Das sieht etwa so aus:

```
gluon.iface_wan=interface
gluon.iface_wan.name='/wan'
gluon.iface_wan.role='uplink'
gluon.iface_lan=interface
gluon.iface_lan.name='/lan'
gluon.iface_lan.role='mesh'
LAN: lan1 lan2 lan3 lan4
WAN: wan
```

Steht dort statt `iface_lan` und `iface_wan` nur `iface_single` (so bei den
FRITZ!Boxen 7530, 7360 und 7362): **hier aufhören** und bei deiner
Community nachfragen.

Welcher Name zu welcher Buchse gehört, steht nicht immer so auf dem Gehäuse.
Steck ein Kabel in eine Buchse und sieh nach, wo eine `1` erscheint:

```sh
for p in $(cat /lib/gluon/core/sysconfig/lan_ifname); do echo "$p $(cat /sys/class/net/$p/carrier)"; done
```

## 2. Hilfsfunktion laden

Einmal pro SSH-Sitzung kopieren. Sie richtet einen Port als eigenen Mesh-Port
ein, getrennt von allen anderen Ports.

```sh
mesh_port() {
	uci set network.mesh_$1=interface
	uci set network.mesh_$1.ifname="$1"
	uci set network.mesh_$1.proto='gluon_wired'
	uci set network.mesh_$1.vxlan='0'
	uci set network.mesh_$1.macaddr="$(echo "$(cat /lib/gluon/core/sysconfig/primary_mac) $1" | md5sum | sed -E 's/^(..)(..)(..)(..)(..).*/02:\1:\2:\3:\4:\5/')"
	uci set network.mesh_$1.gluon_preserve='1'
}
```

## 3. Alte Rollen löschen

Löscht die Rollen der LAN-Ports und alles, was diese Anleitung früher schon
einmal angelegt hat. Vor jedem Szenario ausführen.

```sh
uci -q delete gluon.iface_lan.role
uci -q delete gluon.iface_mesh
uci -q delete gluon.iface_client
for s in $(uci show network | sed -n "s/^network\.\(mesh_[^.]*\)\.gluon_preserve='1'$/\1/p"); do uci delete network.$s; done
```

Die Sektion `iface_lan` selbst **bleibt stehen**, nur ohne Rollen. Wer sie
löscht, bekommt beim nächsten Neustart alle LAN-Ports wieder in einer Gruppe.

## 4. Neue Rollen anlegen: ein Szenario wählen

### A: Jeder LAN-Port einzeln im Mesh

```sh
mesh_port lan1
mesh_port lan2
mesh_port lan3
mesh_port lan4
```

### B: Zwei Ports Mesh (jeder für sich), zwei Ports Client

```sh
mesh_port lan1
mesh_port lan2
uci set gluon.iface_client=interface
uci set gluon.iface_client.name='lan3 lan4'
uci add_list gluon.iface_client.role='client'
```

### C: Ein Port Mesh, alle anderen Client

```sh
uci set gluon.iface_mesh=interface
uci set gluon.iface_mesh.name='lan1'
uci add_list gluon.iface_mesh.role='mesh'
uci set gluon.iface_client=interface
uci set gluon.iface_client.name='lan2 lan3 lan4'
uci add_list gluon.iface_client.role='client'
```

### D: Zurück zum Auslieferungszustand

Schritt 3 ausführen, dann:

```sh
uci delete gluon.iface_lan
```

Gluon legt die LAN-Gruppe beim Neustart mit den Standardrollen neu an (bei
Neanderfunk: alle LAN-Ports gemeinsam als Mesh; bei anderen Communities
steht das in deren `site.conf`).

## 5. Übernehmen und neu starten

```sh
uci commit gluon
uci commit network
start-stop-daemon -S -b -m -p /tmp/gluon-reconfigure.pid -x /bin/sh -- -c 'gluon-reconfigure >/tmp/gluon-reconfigure.log 2>&1; reboot'
```

Die letzte Zeile läuft auch dann weiter, wenn die SSH-Verbindung abreißt (ein
einfaches `gluon-reconfigure &` tut das nicht zuverlässig). Der Knoten startet
danach neu und ist nach 2–3 Minuten wieder erreichbar.

Mit dem Paket `neanderfunk-banner` gibt es dafür den Befehl `reconf`. Dann genügt statt der
letzten Zeile:

```sh
reconf
```

## 6. Prüfen

```sh
ssh root@NODE
uci show gluon | grep iface_
uci show network | grep gluon_preserve
batctl if
```

`batctl if` zeigt jeden Mesh-Port, **an dem ein Kabel steckt**, als eigene
Zeile mit `active`. Ein Mesh-Port ohne Kabel fehlt dort und erscheint von
selbst, sobald eines steckt. Hängt am Port ein anderer Knoten, taucht er in
`batctl n` mit dem Port auf, an dem er steckt.

## Gut zu wissen

* Die Einstellungen **überleben Firmware-Updates**, auch über den Autoupdater.
* Im Config-Mode unter **Erweiterte Einstellungen → Netzwerk** stehen danach:
  „LAN-Schnittstellen" **ohne Haken — so lassen**, und die Zeilen aus Szenario B
  und C (`lan3 lan4` usw.), die man dort auch umstellen kann. Die einzelnen
  Mesh-Ports aus `mesh_port` erscheinen dort nicht.
* Ein Port, der in keinem Szenario vorkommt, ist abgeschaltet.
* Mit `neanderfunk-banner` zeigen `lanrole` (ohne weitere Angabe) und `nodestatus`
  die Belegung aller Ports, auch der einzelnen Mesh-Ports. `lanrole mesh` o.ä.
  verweigert, solange Ports schon anderweitig belegt sind — erst Schritt 3.

---

## 7. Sonderfall: Uplink vom WAN-Port auf den LAN-Port legen

Hier gilt ausnahmsweise nicht „den WAN-Port nie anfassen“. Gedacht ist der
Abschnitt für Geräte, deren WAN-Port nicht zuverlässig läuft, vor allem den
**Cudy TR3000**: Sein 2,5G-WAN hängt an einem Realtek RTL8221B und bleibt
unter Gluon 2023.2 (OpenWrt 23.05) mitunter tot, sogar über Neustarts hinweg
(openwrt/openwrt#17505). Der LAN-Port (1G, interner PHY des SoC) ist davon
nicht betroffen. Der Knoten holt sich den Uplink dann über LAN, und der
WAN-Port wird abgeschaltet.

Der DSA-Test aus Schritt 1 meldet bei diesen Geräten „kein DSA-Geraet“, weil
sie gar keinen Switch haben. Für diesen Abschnitt ist das richtig so.

| Gerät | LAN | WAN | Anmerkung |
| --- | --- | --- | --- |
| Cudy TR3000 v1 (auch 256MB) | `eth1`, 1G | `eth0`, 2,5G, RTL8221B | der Anlass für diesen Abschnitt |
| GL.iNet GL-MT3000 | `eth1`, 1G | `eth0`, 2,5G | gleicher Aufbau; in unserem Netz bisher unauffällig |

Bei einem 1G-Gegenüber (FritzBox, die meisten Switches) geht dabei nichts
verloren, der WAN lief dann ohnehin nur mit 1G.

### 7.1 Vorher: wie komme ich an den Knoten?

Nach dem Umstellen hängt der Uplink an der anderen Buchse. Deshalb **nicht
über den WAN-Weg einloggen, den man gerade abschaltet**. Besser eine dieser
Möglichkeiten:

* **WLAN des Knotens.** Mit der Freifunk-SSID (oder der Offline-SSID)
  verbinden, dann `ssh root@nextnode`. Das ist immer der Knoten, an dem man
  gerade hängt.
* **Config-Mode.** Ohne SSH: Reset-Taste drücken, bis der Knoten im
  Setup-Modus ist. Dann unter **Erweiterte Einstellungen → Netzwerk** beim LAN
  den Haken „Uplink“ setzen und beim WAN alle Haken entfernen, speichern. Das
  ist der Weg ohne Kommandozeile; das Ergebnis ist dasselbe.

### 7.2 Umstellen

Erst die Rollen ansehen:

```sh
lanrole
```

Dann in **genau dieser Reihenfolge**. So gibt es zu jedem Zeitpunkt einen
Uplink, und `lanrole` bzw. `wanrole` fragen nicht nach:

```sh
lanrole uplink
wanrole none
reconf
```

Ohne `lanrole` (Befehl nicht gefunden, Firmware ohne `neanderfunk-banner`)
dasselbe mit uci:

```sh
uci -q delete gluon.iface_lan.role
uci add_list gluon.iface_lan.role='uplink'
uci -q delete gluon.iface_wan.role
uci commit gluon
start-stop-daemon -S -b -m -p /tmp/gluon-reconfigure.pid -x /bin/sh -- -c 'gluon-reconfigure >/tmp/gluon-reconfigure.log 2>&1; reboot'
```

Der Knoten startet neu.

### 7.3 Kabel umstecken, und das WAN-Kabel ganz abziehen

* Das Kabel zum Heimrouter vom **WAN**-Port in den **LAN**-Port stecken.
* **Am WAN-Port bleibt nichts stecken.** Beim TR3000 hängen beide Ports an
  derselben Paket-Engine des SoC. Hängt der 2,5G-Port, kann er den LAN-Port
  mitreißen. Ohne Kabel kann er gar nicht erst hängen.

Falls der WAN-Port schon hing: **Neustart ohne gestecktes WAN-Kabel.** Laut
Betreiber half beim TR3000 nur das; ein Neustart mit gestecktem Kabel änderte
nichts.

### 7.4 Prüfen

```sh
lanrole
cat /sys/class/net/eth1/carrier
ls /sys/class/net/br-wan/brif/
batctl gwl | head -3
```

* `lanrole` zeigt LAN mit `uplink` und WAN ohne Rolle.
* `carrier` ist `1`, sobald das Kabel im LAN-Port steckt.
* In `br-wan` steht `eth1`, nicht mehr `eth0`. Die Uplink-Bridge heißt
  weiter `br-wan`, egal an welchem Port sie hängt.
* `batctl gwl` zeigt einen Gateway, sobald der Tunnel steht (1–2 Minuten).

### 7.5 Zurück

Zuerst wieder das Kabel in den WAN-Port stecken, dann:

```sh
wanrole uplink
lanrole mesh
reconf
```

`lanrole mesh` ist bei Neanderfunk die Vorgabe für LAN. Wer vorher etwas anderes
hatte, setzt das.

Die Rollen **überleben Firmware-Updates**. Wer den WAN-Port später wieder
nutzen will, stellt ihn also ausdrücklich zurück.

---

Geprüft am 2026-09-10 auf einem Xiaomi Mi Router 4A Gigabit mit Gluon 2023.2
(Szenarien A, B und D, jeweils mit Neustart). Die DSA-Prüfung am 2026-09-13
auf sechs Testknoten: OK beim Mi 4A Gigabit (`lan1 lan2`) und beim EdgeRouter X
(`eth1`–`eth4`), „weniger als zwei LAN-Ports“ beim ZyXEL NWA55AXE, „kein
DSA“ beim TL-WR1043ND v2 und TL-WDR3600 (swconfig) und beim FUTRO S550 (x86,
kein Switch).

Abschnitt 7 (14.09.2026) stützt sich auf den Cudy TR3000 eines Betreibers. Der
hat den LAN-Port als Uplink genommen, den WAN freigelassen und damit einen
stabilen Betrieb erreicht. Die Befehlsfolge `lanrole uplink`, `wanrole none`,
`reconf` ist noch an keinem TR3000 durchgespielt. Das Hilfsprogramm dahinter
(`portrole`) prüft die Reihenfolge (es bleibt immer ein Uplink). Wer es als
Erster macht: bitte Rückmeldung als Issue in diesem Repo.
