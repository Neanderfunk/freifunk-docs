# UniFi-Accesspoints auf die Freifunk-Karte bringen

**Kurz:** Wer Ubiquiti-UniFi-Accesspoints hinter einem Freifunk-Router betreibt,
kann sie auf der Freifunk-Karte an ihrem Standort sichtbar machen. Dafür
braucht es im UniFi-Controller eine Koordinate je Accesspoint.

**Gilt für** UniFi-Accesspoints mit UniFi-Network-Controller, die ihr Netz
von einem Freifunk-Router bekommen. Die Karte zeigt sie nur, wenn Eure
Freifunk-Community den Controller an ihre Karte anbindet; bei Freifunk im
Neanderland bereiten wir das gerade vor. Stand 25.09.2026.

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

**Eine Voraussetzung vorweg:** Auf der Karte erscheinen nur Accesspoints, die
ein WLAN mit "Freifunk" im Namen aussenden. Ein Accesspoint ohne ein solches
WLAN ist kein Freifunk-Accesspoint, und auf einer Freifunk-Karte hätte er
nichts verloren.

## Koordinaten je Accesspoint

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

## Die Reihenfolge

1. Koordinaten je Accesspoint eintragen. Das geht sofort.
2. Bei Eurer Freifunk-Community melden.

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
