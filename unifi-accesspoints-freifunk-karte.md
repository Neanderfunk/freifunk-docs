# UniFi-Accesspoints hinter Freifunk-Routern: auf die Karte bringen und richtig einstellen

**Kurz:** Wer Ubiquiti-UniFi-Accesspoints hinter einem Freifunk-Router betreibt,
kann sie auf der Freifunk-Karte sichtbar machen, an ihrem Standort und
verbunden mit dem Freifunk-Router, über den sie ins Netz gehen. Dafür braucht
es zwei Dinge im UniFi-Controller: eine Koordinate je Accesspoint und eine Site
je Freifunk-Router. Dazu kommen ein paar Einstellungen, ohne die UniFi-Geräte
hinter einem Freifunk-Router Ärger machen.

**Gilt für** UniFi-Accesspoints mit UniFi-Network-Controller, die ihr Netz
von einem Freifunk-Router bekommen. Die Karte zeigt sie nur, wenn Eure
Freifunk-Community den Controller an ihre Karte anbindet; bei Freifunk im
Neanderland bereiten wir das gerade vor. Die Einstellungen in Teil 3 lohnen sich aber
auch ohne Karte. Stand 24.09.2026.

Entstanden ist die Anleitung für die Installation des LVR mit über 600
Accesspoints, verteilt auf mehrere Freifunk-Router. Sie gilt aber genauso für
eine Handvoll Geräte in einem Vereinsheim.

---

## Worum es geht

UniFi-Accesspoints sind keine Freifunk-Knoten. Sie funken das offene
Freifunk-WLAN, aber sie melden sich nicht selbst bei der Karte. Was die Karte
über sie weiß, liest sie aus Eurem UniFi-Controller: welche Geräte es gibt,
wo sie stehen und zu welchem Freifunk-Router sie gehören.

Das übernimmt ein Werkzeug von Freifunk München,
[unifi_respondd](https://github.com/freifunkMUC/unifi_respondd). Es fragt
den Controller regelmäßig ab und reicht die Accesspoints an die Karte weiter.
Standort und Zuordnung stehen im Controller aber erst drin, wenn Ihr sie
eintragt. Das ist einmalige Arbeit, danach pflegt sich die Karte von selbst.

**Eine Voraussetzung vorweg:** Das Werkzeug meldet nur Accesspoints, deren
WLAN-Name zu einem Muster passt, das Eure Community einstellt. Üblich ist
"enthält freifunk", egal ob groß oder klein geschrieben. Heißt Euer
Freifunk-WLAN anders, sagt es Eurer Community, sonst tauchen die Geräte nicht
auf.

## Teil 1: Koordinaten je Accesspoint

Muss für jeden Accesspoint einzeln gemacht werden.

- **Devices** öffnen
- Accesspoint anklicken
- rechts **Settings** öffnen
- Abschnitt **SNMP** aufklappen
- Feld **Location** ausfüllen
- **Apply Changes**

Die Menüpunkte können je nach Version des Controllers leicht anders heißen.

### Format

```
51.2506, 6.9746
```

- Dezimalgrad, **Punkt** als Dezimaltrennzeichen
- Breite zuerst, dann Länge, getrennt durch **Komma**
- vier Nachkommastellen genügen, das sind etwa zehn Meter
- keine Anführungszeichen, keine Himmelsrichtungen, kein Grad-Zeichen

### Woher die Koordinaten kommen

- **OpenStreetMap:** Rechtsklick auf den Punkt, "Adresse anzeigen", die
  Koordinaten stehen dann in der Adresszeile des Browsers.
- **Google Maps:** Rechtsklick auf den Punkt, die erste Zeile im Menü ist das
  Koordinatenpaar, ein Klick darauf kopiert es in der richtigen Form.
- Genauigkeit auf Gebäudeebene reicht. Stehen mehrere Accesspoints im selben
  Gebäude, darf dieselbe Koordinate mehrfach vorkommen.

### Was nicht hineingehört

- **Koordinaten, keine Adresse.** Eine Adresse übersetzt das Werkzeug über
  den OpenStreetMap-Dienst Nominatim. Das ist ungenauer, und die Adresse geht
  dafür an einen fremden Dienst. Findet der Dienst nichts, steht der
  Accesspoint auf der Karte bei 0/0, im Golf von Guinea.
- Das Feld **Contact** darf leer bleiben.
- Die Karte liest das Feld über den Controller, nicht per SNMP. SNMP selbst
  muss dafür **nicht** eingeschaltet sein.

**Prüfen:** Nach dem Speichern steht die Koordinate in der Geräteübersicht des
Accesspoints unter SNMP. Auf der Karte erscheint er, sobald Eure Community den
Controller angebunden hat.

## Teil 2: Eine Site je Freifunk-Router

Das Werkzeug, das die Accesspoints auf die Karte bringt, kennt je Site
**genau einen** Freifunk-Router. Hängen Eure Accesspoints hinter mehreren Freifunk-Routern, zum
Beispiel einer je Gebäude oder je Stockwerk, dann braucht jeder dieser Router
seine eigene Site. Sonst hängen auf der Karte alle Accesspoints am falschen
Router.

Hängen alle Accesspoints hinter einem einzigen Freifunk-Router, könnt Ihr
diesen Teil überspringen.

### Sites anlegen

- oben links auf den Namen der Site klicken, **Add New Site**
- je Freifunk-Router eine Site, etwa so:

| Site | hängt hinter |
| --- | --- |
| `ff-haus-a` | Freifunk-Router im Haus A |
| `ff-haus-b` | Freifunk-Router im Haus B |
| `ff-haus-c` | Freifunk-Router im Haus C |

- Die Namen stimmt Ihr vorher mit Eurer Freifunk-Community ab. Maßgeblich ist
  der Name der Site **so, wie er im Controller angezeigt wird**. Er wird
  genau so in die Konfiguration übernommen; ein Tippfehler heißt, dass die
  Site nicht gefunden wird.
- Die Namen später nicht mehr ändern.

### Zuerst die neue Site einrichten, dann umziehen

Die WLAN- und Netzwerkeinstellungen wandern beim Umzug **nicht** mit. Richtet
deshalb jede neue Site zuerst nach Teil 3 ein. Ein Accesspoint, der in eine
leere Site umzieht, funkt danach nichts.

### Accesspoints verschieben

- **Devices** öffnen, in der bisherigen Site
- Geräte auswählen, Mehrfachauswahl geht
- **Manage Device**, dann **Move to Site**, Ziel-Site wählen
- bestätigen

Dabei gut zu wissen:

- Der Accesspoint wird neu eingebunden. Er ist **ein bis zwei Minuten
  offline**, das WLAN fällt in dieser Zeit aus.
- Deshalb **Gruppe für Gruppe** vorgehen, nicht alle auf einmal.

**Prüfen:** In der Ziel-Site muss der Accesspoint den Status **Connected**
erreichen. Bleibt er bei **Adopting** oder **Pending** hängen, hilft meist,
ihn kurz vom Strom zu nehmen.

**Zurück:** Genauso, nur mit der alten Site als Ziel. Die Koordinaten aus
Teil 1 bleiben beim Umzug in beide Richtungen erhalten.

## Teil 3: Einstellungen je Site

Bevor Accesspoints in eine neue Site umziehen, und am besten auch in einer
bestehenden Site hinter einem Freifunk-Router.

### WLAN

- **Settings, WiFi:** das Freifunk-WLAN anlegen, bei einer neuen Site genau
  wie in der bisherigen
- die **SSID muss genau stimmen**, auch in Groß- und Kleinschreibung
- Sicherheit **Open**, ohne Passwort
- **Guest Policies und Captive Portal: aus.** Freifunk braucht keine
  Anmeldeseite.

### Netzwerk

- **Settings, Networks:** ein einziges Netz vom Typ **Corporate**, DHCP
  **None** beziehungsweise "Use existing DHCP". Die Adressen vergibt das
  Freifunk-Netz, nicht der Controller.
- kein VLAN, außer Ihr habt bisher schon eines verwendet

### Geräteverhalten

- **Settings, System, Uplink Connectivity Monitor: aus.** Sonst startet sich
  ein Accesspoint selbst neu, wenn er ein bestimmtes Ziel nicht anpingen kann,
  und hinter einem Freifunk-Router passiert genau das regelmäßig.
- **Auto-Optimize Network: aus**
- **Wireless Meshing: aus**, sofern alle Accesspoints per Kabel angebunden sind
- **Auto-Upgrade** nach eigenem Ermessen, für Freifunk spielt es keine Rolle

### Allgemein

- **Settings, System, Country und Timezone** richtig setzen, bei einer neuen
  Site wie in der bisherigen

## Die Reihenfolge

1. Koordinaten in der bestehenden Site eintragen. Das geht sofort und bleibt
   beim späteren Umzug erhalten.
2. Falls nötig, neue Sites anlegen und jede nach Teil 3 einrichten.
3. Accesspoints Gruppe für Gruppe verschieben, nach jeder Gruppe kurz prüfen.
4. Bei Eurer Freifunk-Community melden, mit den Namen der Sites.

Läuft der Controller bei Euch selbst, braucht die Community für die Anbindung
zweierlei: ein **Konto mit reinen Leserechten**, mehr ist nicht nötig, denn das
Werkzeug liest nur und schreibt nichts, und eine **Netzverbindung** zur
Oberfläche Eures Controllers, auf dem Port, unter dem Ihr sie erreicht, meist
8443 oder 443. Von wo sie zugreift, sagt sie Euch.

Liegt Eure Site auf einem Controller, den die Community selbst betreibt,
entfällt das.

## Kontakt

Bei Freifunk im Neanderland erreicht Ihr uns unter
[projekt@neanderfunk.de](mailto:projekt@neanderfunk.de).
