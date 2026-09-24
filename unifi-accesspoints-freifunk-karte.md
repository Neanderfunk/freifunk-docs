# UniFi-Accesspoints hinter Freifunk-Routern: auf die Karte bringen und richtig einstellen

**Kurz:** Wer Ubiquiti-UniFi-Accesspoints hinter einem Freifunk-Router betreibt,
kann sie auf der Freifunk-Karte an ihrem Standort sichtbar machen. Dafür
braucht es im UniFi-Controller vor allem eines: eine Koordinate je
Accesspoint. Dazu kommen ein paar Einstellungen, ohne die UniFi-Geräte hinter
einem Freifunk-Router Ärger machen.

**Gilt für** UniFi-Accesspoints mit UniFi-Network-Controller, die ihr Netz
von einem Freifunk-Router bekommen. Die Karte zeigt sie nur, wenn Eure
Freifunk-Community den Controller an ihre Karte anbindet; bei Freifunk im
Neanderland bereiten wir das gerade vor. Die Einstellungen in Teil 2 lohnen sich aber
auch ohne Karte. Stand 25.09.2026.

Entstanden ist die Anleitung für die Installation des LVR mit mehreren
hundert Accesspoints hinter rund hundert Freifunk-Routern. Sie gilt aber genauso für
eine Handvoll Geräte in einem Vereinsheim.

---

## Worum es geht

UniFi-Accesspoints sind keine Freifunk-Knoten. Sie funken das offene
Freifunk-WLAN, aber sie melden sich nicht selbst bei der Karte. Was die Karte
über sie weiß, liest sie aus Eurem UniFi-Controller: welche Geräte es gibt
und wo sie stehen.

Das übernimmt ein Werkzeug von Freifunk München,
[unifi_respondd](https://github.com/freifunkMUC/unifi_respondd). Es fragt
den Controller regelmäßig ab und reicht die Accesspoints an die Karte weiter.
Den Standort kann niemand herleiten, der steht im Controller erst drin,
wenn Ihr ihn eintragt. Das ist einmalige Arbeit, danach pflegt sich die Karte
von selbst.

**Hinter welchem Freifunk-Router** ein Accesspoint hängt, müsst Ihr dagegen
nicht eintragen. Das wollen wir bei Freifunk im Neanderland künftig
automatisch aus dem Freifunk-Netz ermitteln; das ist geplant, aber noch nicht
gebaut. Ihr legt dafür also **keine** zusätzlichen Sites an und verschiebt
keine Geräte.

In seiner ursprünglichen Form kennt das Werkzeug allerdings je Site genau
einen Freifunk-Router. Betreibt Eure Community es so und hängen Eure
Accesspoints hinter mehreren Routern, sprecht mit ihr ab, wie sie die
Zuordnung lösen will.

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

## Teil 2: Einstellungen im Controller

Für die Site, in der Eure Accesspoints hinter einem Freifunk-Router stehen.

### WLAN

- **Settings, WiFi:** das Freifunk-WLAN
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

- **Settings, System, Country und Timezone** richtig setzen

## Die Reihenfolge

1. Koordinaten je Accesspoint eintragen. Das geht sofort.
2. Die Einstellungen aus Teil 2 prüfen.
3. Bei Eurer Freifunk-Community melden.

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
