# Freifunk-Docs

Anleitungen und Hintergrundwissen rund um Freifunk-Knoten mit
[Gluon](https://gluon.readthedocs.io/), zusammengetragen bei Freifunk
Neanderland (Neanderfunk). Alles hier ist öffentlich und soll auch anderen
Communities und Knotenbetreibern helfen.

## Inhalt

| Dokument | Thema |
| --- | --- |
| [lan-ports-trennen-2023.2.md](lan-ports-trennen-2023.2.md) | LAN-Ports unter Gluon 2023.2 einzeln als Mesh- oder Client-Port einrichten (Geräte mit DSA); Sonderfall: Uplink vom WAN auf den LAN-Port legen, z. B. beim Cudy TR3000 |

## Was hier hineingehört

* Anleitungen, die man ohne Insiderwissen nachvollziehen kann: Befehle zum
  Kopieren, erklärt, mit Prüfschritt und Rückweg.
* Erfahrungen mit Geräten und Treibern, die anderen Zeit sparen.
* Jede Anleitung sagt, für welche Gluon-Version und welche Geräte sie gilt und
  wann sie zuletzt geprüft wurde.

## Was hier nicht hineingehört

Interna: Adressen einzelner Knoten oder Server, Zugangsdaten, Namen von
Betreibern, Details zu unserer Infrastruktur. So etwas bleibt intern.

## Befehle aus unserer Firmware

Manche Anleitungen nennen Hilfsbefehle wie `lanrole`, `wanrole`, `reconf` oder
`nodestatus`. Sie stammen aus dem Paket `neanderfunk-banner` in unserem
Paket-Feed ([Neanderfunk/packages](https://github.com/Neanderfunk/packages)).
Auf Firmware ohne dieses Paket steht jeweils der Weg mit `uci` daneben.

## Lizenz

Die Texte stehen unter
[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/deed.de)
(Namensnennung, Weitergabe unter gleichen Bedingungen), siehe `LICENSE`.
Namensnennung: „Freifunk Neanderland (Neanderfunk)“ mit Link auf dieses Repo.
