# UniFi-Accesspoints auf die Freifunk-Karte bringen

**Kurz:** Wer Ubiquiti-UniFi-Accesspoints hinter einem Freifunk-Router betreibt,
kann sie auf der Freifunk-Karte an ihrem Standort sichtbar machen. Dafür
braucht es im UniFi-Controller eine Koordinate je Accesspoint.

**Vorausgesetzt ist ein Controller, an den Eure Freifunk-Community
herankommt.** Dafür gibt es zwei Wege: Entweder Ihr bekommt ein Konto auf
ihrem Controller und verwaltet Eure Accesspoints dort, oder Ihr gebt ihr
lesenden Zugriff auf Euren eigenen. Bei Freifunk im Neanderland schreibt Ihr
dafür an [projekt@neanderfunk.de](mailto:projekt@neanderfunk.de). Ohne einen
dieser beiden Wege erscheinen die Geräte nicht auf der Karte.

**Gilt für** UniFi-Accesspoints mit UniFi-Network-Controller, die ihr Netz
von einem Freifunk-Router bekommen. Stand 25.09.2026.

Entstanden ist die Anleitung für die Installation des LVR mit mehreren
hundert Accesspoints hinter rund hundert Freifunk-Routern. Sie gilt aber genauso für
eine Handvoll Geräte in einem Vereinsheim.

---

## Worum es geht

UniFi-Accesspoints sind keine Gluon-Knoten, sie melden sich daher nicht mit
dem Respondd-Protokoll bei der Karte. Was die Karte über sie weiß, liest sie
aus dem UniFi-Controller: welche Geräte es gibt und wo sie stehen.

Das übernimmt ein Werkzeug von Freifunk München,
[unifi_respondd](https://github.com/freifunkMUC/unifi_respondd). Es fragt
den Controller regelmäßig ab und reicht die Accesspoints an die Karte weiter.
Den Standort kann niemand herleiten, der steht im Controller erst drin,
wenn Ihr ihn eintragt. Das ist einmalige Arbeit, danach pflegt sich die Karte
von selbst.

**Hinter welchem Freifunk-Router** ein Accesspoint hängt, müsst Ihr dagegen
nicht eintragen. Das ermittelt die Karte bei Freifunk im Neanderland selbst
aus dem Netz. Ihr legt dafür also **keine** zusätzlichen Sites an und
verschiebt keine Geräte.

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

### Was ins Feld kommt

Am besten das Koordinatenpaar, Breite zuerst:

```
51.2874, 6.3538
```

Vier Nachkommastellen genügen, das sind etwa zehn Meter. Genauigkeit auf
Gebäudeebene reicht, und stehen mehrere Accesspoints im selben Gebäude, darf
dieselbe Koordinate mehrfach vorkommen.

**Schreibt Ihr reine Zahlen, kommt es auf die Reihenfolge an:** erst die
Breite, dann die Länge. In Deutschland ist die Breite dabei immer die größere
Zahl, sie liegt zwischen 47 und 55, die Länge zwischen 6 und 15. Vertauscht
landet der Accesspoint bei Madagaskar im Wasser, und das kann niemand erraten.

Steht eine Himmelsrichtung dabei, ist auch das egal.

Ansonsten müsst Ihr auf die Schreibweise nicht achten. Was Ihr aus Google Maps
oder OpenStreetMap kopiert habt, könnt Ihr so einsetzen, auch die Schreibweise
mit Grad und Minuten oder einen Link aus der Adresszeile des Browsers.

### Woher die Koordinaten kommen

**Am bequemsten über die Karte selbst.** Auf
[neander.map.freifunk.space](https://neander.map.freifunk.space/) gibt es oben
rechts ein Stecknadel-Symbol, **Koordinaten wählen**. Schaltet es ein und
klickt dann die Stelle in der Karte an. Im Seitenfenster erscheinen
Breitengrad und Längengrad, dazu ein Feld **Breite, Länge**, jeweils mit einer
Schaltfläche zum Kopieren. Dieses Feld könnt Ihr unverändert in **Location**
einsetzen. Die Luftbildansicht hilft dabei, das richtige Gebäude zu treffen.

Sonst gehen auch:

- **Google Maps:** Rechtsklick auf den Punkt, die erste Zeile im Menü ist das
  Koordinatenpaar, ein Klick darauf kopiert es.
- **OpenStreetMap:** Rechtsklick auf den Punkt, "Adresse anzeigen", die
  Koordinaten stehen dann in der Adresszeile des Browsers.

### Was nicht funktioniert

- **Eine Straßenadresse als Text.** Es gibt bewusst keine Adresssuche, sonst
  ginge Eure Adresse dafür an einen fremden Dienst.
- **Kurzlinks aus der Teilen-Funktion** (`maps.app.goo.gl/...`). Darin stehen
  keine Koordinaten. Nehmt den Link aus der Adresszeile oder gleich die Zahlen.
- Widersprüchliches, etwa ein Minuszeichen zusammen mit einer
  Himmelsrichtung oder zweimal dieselbe Richtung.

Das Feld **Contact** darf leer bleiben. Die Karte liest das Standortfeld über
den Controller, nicht per SNMP; SNMP selbst muss dafür **nicht** eingeschaltet
sein.

**Prüfen:** Nach dem Speichern steht die Koordinate in der Geräteübersicht des
Accesspoints unter SNMP, und auf der Karte erscheint er wenige Minuten später
an seinem Standort.

Ohne Koordinate geht übrigens nichts verloren: Der Accesspoint ist trotzdem
da, mit seinem Router und seinen Clients, nur eben nicht auf der Landkarte.

## Was danach auf der Karte zu sehen ist

Jeder Accesspoint steht als eigener Punkt auf der Karte, verbunden mit dem
Freifunk-Router, hinter dem er hängt. Angezeigt werden sein Standort, die Zahl
der verbundenen Geräte, die Kanalauslastung je Band, ein paar Minuten
verzögert, sowie Systemlast und Speicher des Accesspoints. Die verbundenen
Geräte zählen dabei beim Accesspoint und nicht noch einmal beim Router.

Fällt ein Accesspoint aus, erscheint er nach etwa zwanzig Minuten als offline,
mit dem Zeitpunkt, an dem er zuletzt gesehen wurde. Meldet er sich neunzig
Tage lang nicht, verschwindet er von der Karte.

Die Angabe **zuerst gesehen** meint den Tag, an dem das Gerät in den
Controller aufgenommen wurde, nicht den Tag, an dem die Karte es zum ersten
Mal gezeigt hat.

## Die Reihenfolge

1. Koordinaten je Accesspoint eintragen. Das geht sofort.
2. Bei Eurer Freifunk-Community melden.

Läuft der Controller bei Euch selbst, braucht die Community dafür ein **Konto
mit reinen Leserechten**, mehr ist nicht nötig, denn das Werkzeug liest nur und
schreibt nichts, und eine **Netzverbindung** zur Oberfläche Eures Controllers,
auf dem Port, unter dem Ihr sie erreicht, meist 8443 oder 443. Von wo sie
zugreift, sagt sie Euch.

## Kontakt

Bei Freifunk im Neanderland erreicht Ihr uns unter
[projekt@neanderfunk.de](mailto:projekt@neanderfunk.de).
