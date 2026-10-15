# Protokoll: Incident Morpheus + Agenda Wochenmeeting

Oct 13, 2026 · @Mustafa Rafati

## Incident: Morpheus nicht erreichbar

Morpheus war am 13.10.2026 für Teile der Nutzerschaft nicht erreichbar; der Zugang war nicht vollständig weg, sondern selektiv gestört.

Beobachtungen aus dem Troubleshooting:

- Einzelne Komponenten waren nicht erreichbar, nicht die gesamte Plattform.
- Kurz zuvor wurden Zertifikate eingespielt. Dies wurde als mögliche Ursache benannt.
- Ein Datenbankcluster war nicht erreichbar.
- Entscheidendes Muster: Bereits angemeldete Nutzer konnten weiterarbeiten. Neue Anmeldungen schlugen fehl.
- Nach der Rotation eines Nodes verbesserte sich das Verhalten.

Noch nicht festgehalten und im Meeting zu ergänzen: Startzeit und Ende der Störung, wer sie gemeldet hat, welcher Node rotiert wurde und ob die Zertifikatsänderung dokumentiert war.

## Technische Analyse (Hypothesen, nicht belegt)

Das Muster "alte Sessions laufen, neue Logins scheitern" passt zu einer Zertifikatsänderung, die nur neue Verbindungen betrifft. Belegt ist das bisher nicht.

Morpheus ist tatsächlich eine Java-Anwendung. Der Application Tier besteht laut Herstellerdoku aus [Nginx und Tomcat](https://docs.morpheusdata.com/en/6.0.0/getting_started/installation/overview.html), also einem Java-Servlet-Container; die Appliance bringt ein eigenes [Java-Laufzeitpaket und Java-Bibliotheken wie den MySQL-Connector](https://docs.morpheusdata.com/en/6.2.11/) mit. Der transaktionale Datenbank-Tier ist eine MySQL-kompatible Datenbank.

Warum das Verhalten zu Java passt (plausibel, im Einzelfall zu prüfen):

- Die JVM lädt ihren Truststore beim Start. Ein neu eingespieltes Zertifikat wird von einem laufenden Prozess typischerweise erst nach einem Neustart berücksichtigt.
- Bestehende Verbindungen in einem Connection Pool bleiben offen und funktionieren weiter. Nur neu aufgebaute Verbindungen durchlaufen den TLS-Handshake erneut und scheitern dann.
- Ein Login erzeugt in der Regel neue Verbindungen zur Datenbank. Eine laufende Session nicht.

Wichtige Einordnung: Das ist kein Java-exklusives Problem. Jede Laufzeitumgebung mit eigenem Truststore und Verbindungspooling kann sich so verhalten. In Java-Umgebungen fällt es nur besonders häufig auf.

| Beobachtung | Mögliche Erklärung | Prüfschritt |
| --- | --- | --- |
| Neue Logins scheitern, alte Sessions laufen | Neue TLS-Verbindungen scheitern an einem Zertifikat, alte Verbindungen bestehen fort | Zeitstempel der Zertifikatseinspielung mit dem Beginn der Loginfehler vergleichen |
| Datenbankcluster nicht erreichbar | Zertifikat der DB-Verbindung nicht im Truststore der Anwendung | Truststore-Inhalt und Zertifikatskette gegen das neue Zertifikat prüfen |
| Nach Node-Rotation lief es besser | JVM-Neustart lud den Truststore neu und baute den Pool neu auf | Prüfen, ob der Fehler nur auf den nicht rotierten Nodes weiterbestand |

Ungeklärt bleibt, ob wirklich das Zertifikat die Ursache war oder nur zeitlich zusammenfiel. Ohne Logs vom Zeitpunkt der ersten Loginfehler ist das nicht zu entscheiden.

## Zuständigkeiten

Die Einrichtung in Morpheus erfolgt im Regelfall nicht durch die Endnutzer selbst, sondern durch die Server-Abteilungen.

| Bereich | Zuständig | Anmerkung |
| --- | --- | --- |
| Einrichtung Linux-Systeme | LINSRV | Nicht durch Endnutzer |
| Einrichtung Windows-Systeme | WINSRV | Nicht durch Endnutzer |
| Ansible Automation Platform (AAP) | Eigenes Team, nicht unser Cloud-Team | Wir sind Nutzer, nicht Betreiber |
| Morpheus, GitLab | Unser Cloud-Team | Abgrenzung im Meeting zu bestätigen |

Die Zeile zu Morpheus und GitLab ist meine Annahme aus dem Gesprächskontext. Bitte im Meeting prüfen und korrigieren.

## GitLab Geo

Geo repliziert eine selbstbetriebene GitLab-Instanz an weitere Standorte. Es dient zwei Zwecken: schnellerer lesender Zugriff für verteilte Teams und ein Warm-Standby für den Notfall. Es ist laut [GitLab-Dokumentation](https://docs.gitlab.com/administration/geo/) ausdrücklich keine Hochverfügbarkeitslösung von der Stange und nur in den Tiers Premium und Ultimate für GitLab Self-Managed verfügbar.

Aufbau:

- Die Originalinstanz heißt Primary Site, die Repliken heißen Secondary Sites.
- Schreibvorgänge in der Datenbank passieren ausschließlich auf der Primary. Die Secondary erhält Änderungen über PostgreSQL Streaming Replication.
- Für den Nutzer verhalten sich Secondary Sites dennoch wie vollwertige Lese- und Schreibinstanzen, weil sie Schreiboperationen transparent an die Primary weiterleiten. Lesende Git-Operationen bedient die Secondary selbst, wenn sie aktuell ist.
- Jede Secondary braucht zwei PostgreSQL-Datenbanken: die lesende Replik und eine eigene Tracking-Datenbank, die festhält, was bereits repliziert wurde.

Grenzen, die für eine Entscheidung wichtig sind:

- Geo ist Active-Passive und nur eventually consistent. Ein Failover erfordert menschliches Eingreifen und kann geringen Datenverlust bedeuten.
- Geo repliziert auch Korruption. Es ersetzt keine Backups.
- Alle Sites müssen exakt dieselbe GitLab-Version fahren.
- Für OAuth-Logins muss die Primary erreichbar sein. Bestehende Sessions und Git sind davon nicht betroffen. Dieses Muster ähnelt dem, was wir beim Morpheus-Incident gesehen haben.

Offen: Was genau unser Geo-Setup sein soll, geht aus meinen Notizen nicht hervor. Siehe die Fragen unter Offene Punkte.

## Zusammenspiel der Werkzeuge

Wir nutzen die Red Hat Ansible Automation Platform (AAP), um Aufgaben zu automatisieren. Die Playbooks liegen versioniert im GitLab, die Zielsysteme sind VMs in Morpheus, und AAP ist die ausführende Zwischenstelle zwischen beiden.

&#91;embedded content: Toolchain · GitLab, AAP, Morpheus\]

Der Code wird also nicht dort ausgeführt, wo er liegt, und nicht dort verwaltet, wo er wirkt. Das betrifft auch die Zuständigkeit: AAP wird von einem anderen Team betrieben.

## Lernaufgabe: eigene Teststrecke

Um die Toolchain praktisch zu durchlaufen, soll eine kleine RHEL-VM in Morpheus bereitgestellt und über die ganze Kette hinweg angefasst werden.

- [ ] Kleine RHEL-VM (Red Hat Enterprise Linux) in Morpheus bereitstellen lassen
- [ ] Projekt und einfaches Playbook in GitLab anlegen
- [ ] Playbook über AAP gegen die VM ausführen
- [ ] Lauf wiederholen und prüfen, ob das Playbook idempotent ist
- [ ] Erkenntnisse und Stolpersteine notieren

Vorab zu klären: Wer stellt die VM bereit (vermutlich LINSRV), welche Berechtigungen brauche ich in AAP, und wer gibt das AAP-seitig frei. Da AAP von einem anderen Team betrieben wird, ist das kein Selbstbedienungsvorgang.

## Offene Punkte für das Meeting

Hauptpunkt ist der Nutzerweg in Morpheus. Der Rest sind Nachfragen aus dem Incident und zu Geo.

**Nutzerweg Morpheus (Hauptpunkt)**

1. Eine Fachabteilung erhält den Auftrag, etwas bereitstellen zu lassen. Was ist der erste formale Schritt: Ticket, Serviceportal, Formular, direkter Kontakt?
2. Was sieht ein normaler Nutzer, wenn er sich an Morpheus anmeldet? Welche Rolle und welcher Umfang sind sichtbar?
3. An welcher Stelle kommt unser Team ins Spiel, an welcher LINSRV oder WINSRV?
4. Welche Freigaben oder Genehmigungen liegen dazwischen, und wer erteilt sie?
5. Was bekommt die Fachabteilung am Ende übergeben, und wer betreibt es danach?
6. Gibt es dazu bereits eine dokumentierte Beschreibung, die ich lesen kann, statt sie neu zu erheben?

**Incident**

7. War die Zertifikatseinspielung angekündigt und dokumentiert?
8. Welcher Node wurde rotiert, und war das eine geplante Maßnahme oder ein Versuch?
9. Gibt es Logs vom Zeitpunkt der ersten fehlgeschlagenen Logins?
10. Braucht es eine Nachbereitung, und wenn ja, wer schreibt sie?

**Geo**

11. Was ist das Ziel: Disaster Recovery, Beschleunigung für verteilte Teams oder Entlastung der CI-Last?
12. Wie viele Standorte sind geplant, und welche Lizenz-Tier haben wir?
13. Wer betreibt die Secondaries, und wie wird der Failover geübt?

## Glossar

| Begriff | Bedeutung |
| --- | --- |
| idempotent | Eine Operation führt zum selben Endzustand, egal wie oft sie läuft. Ein Ansible-Playbook soll idempotent sein: der zweite Lauf ändert nichts mehr, weil der Sollzustand schon erreicht ist. |
| InnoDB | Die Standard-Speicher-Engine von MySQL und MariaDB. Transaktionsfähig mit Zeilensperren, im Gegensatz zum älteren MyISAM. "Inno" kommt vom finnischen Hersteller Innobase Oy. |
| Node rotieren | Umgangssprachlich: einen Knoten eines Clusters neu starten oder durch einen frischen ersetzen, meist nacheinander statt alle gleichzeitig. Achtung: "Rotation" bezeichnet daneben auch den Austausch von Zertifikaten oder Schlüsseln. Welche Bedeutung hier gemeint war, ist zu klären. |
| Truststore | Der Speicher einer Anwendung mit den Zertifikaten, denen sie vertraut. Die JVM liest ihn beim Start. |
| Geo | GitLab-Funktion zur Replikation einer Instanz an weitere Standorte, mit einer schreibenden Primary und lesenden Secondaries. |
| AAP | Ansible Automation Platform von Red Hat. Führt Playbooks zentral aus, mit Zugangsdaten, Rechten und Protokollierung. |
| Playbook | Eine Ansible-Datei, die den gewünschten Zustand eines Systems beschreibt. Bei uns versioniert im GitLab. |
