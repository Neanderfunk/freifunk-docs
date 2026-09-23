# Freifunk-Docs

Anleitungen und Hintergrundwissen rund um Freifunk-Knoten mit
[Gluon](https://gluon.readthedocs.io/), zusammengetragen bei Freifunk
Neanderland (Neanderfunk). Alles hier ist öffentlich und soll auch anderen
Communities und Knotenbetreibern helfen.

## Inhalt

| Dokument | Thema |
| --- | --- |
| [lan-ports-trennen-2023.2.md](lan-ports-trennen-2023.2.md) | LAN-Ports unter Gluon 2023.2 einzeln als Mesh- oder Client-Port einrichten (Geräte mit DSA); Sonderfall: Uplink vom WAN auf den LAN-Port legen, z. B. beim Cudy TR3000 |
| [rtl8221b-2g5-wan-2023.2.md](rtl8221b-2g5-wan-2023.2.md) | 2,5G-WAN mit Realtek RTL8221B auf MT7981 (Cudy TR3000, WR3000H, M3000): Symptome, Hintergrund, Belege sammeln, Abhilfe |
| [lowmem-dualband-64mb-2023.2.md](lowmem-dualband-64mb-2023.2.md) | Dualband-Router mit 64 MB RAM (Archer C25, R6120, WR1000 u. a.): Fehlerbilder, Messen, Entlastungen, Update |
| [mips-tlb-cold-start-5.15.md](mips-tlb-cold-start-5.15.md) | **Englisch.** MIPS-Router (ath79, lantiq, ramips) bleiben auf Kernel 5.15.190 bis 5.15.208 beim Kaltstart stehen, Warmstart geht: Erkennen, betroffene Geräte, Ursache, die Korrektur aus 5.15.209 und wie man selbst nachmisst |

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
