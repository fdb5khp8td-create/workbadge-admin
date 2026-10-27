# WorkBadge: Nachrichten von der Website zur iPhone-App

## Tatsächlicher Prüfstand

Die öffentliche Seite war erreichbar, aber ihre CloudKit-Konfiguration war deaktiviert,
der Web-API-Token leer und Schreiben gesperrt. Im lokalen App-Projekt waren sowohl
Firestore-Projekt als auch HTTPS-Feed leer. Eine Nachricht konnte deshalb nicht ankommen.

Der Admin-Code wurde korrigiert: CloudKit-Datensatz-IDs müssen über alle Datensatztypen
einer Zone eindeutig sein. Entwürfe heißen jetzt `draft-<id>`, öffentliche Mitteilungen
`<id>`. Die iPhone-Anbindung liest nur `WorkBadgeAnnouncement` mit
`status == published` und `audience == all`. Der Entwurfstyp wird niemals abgefragt.

## Gemeinsame Datenquelle

- Container: `iCloud.com.example.MoodBadge` (bereits in den App-Entitlements vorhanden).
- Public Database, Record Types `WorkBadgeAnnouncement` und `WorkBadgeAnnouncementDraft`.
- `id`, `title`, `text`, `kind`, `audience`, `status`, `placement`, `dismissMode`,
  `ctaLabel`, `ctaUrl`: String.
- `priority`: Int64; news=0, update=1, maintenance=2, important=3.
- `createdAt`, `updatedAt`, `startAt`, `endAt`: Date/Time; die letzten beiden optional.
- QUERYABLE-Indizes für `recordName`, `status`, `audience` bereitstellen.

## Browser-Verbindung ohne veröffentlichten Token

Unter Einstellungen wird der vorhandene, auf `https://admin.workbadge.de` beschränkte
Web-API-Token eingegeben und die Umgebung gewählt (Development = aus Xcode installierte
App, Production = App Store/TestFlight). Token und Umgebung liegen nur im
`sessionStorage` dieses Tabs: Sie überstehen die Apple-Weiterleitung im selben Tab und
werden beim Schließen des Tabs oder mit „Sitzung trennen“ gelöscht. Nicht im Repository,
nicht in Cookies oder localStorage.
Die Apple-Anmeldung erfolgt im selben Tab. Apple liefert die Sitzung als `ckWebAuthToken`
und `ckSession`; beide werden vor dem Laden externer Ressourcen aus der Adresszeile entfernt
und nur im Arbeitsspeicher an CloudKit JS übergeben. Nach einem Neuladen ist daher eine neue
Apple-Anmeldung nötig. In der CloudKit-Konsole ist URL Redirect auf
`https://admin.workbadge.de/` eingestellt.

Fehlt dem angemeldeten Konto die Rolle WorkBadgeAdmin, zeigt die Seite die eigene
CloudKit-Nutzer-ID und die Umgebung an, in der die Rolle zuzuweisen ist; sie prüft danach
alle 30 Sekunden erneut.

Speichern: Neue Datensätze werden angelegt, bestehende mit Änderungskennung vollständig
ersetzt (`replace`). Leere Start-/Endzeiten werden weggelassen statt als `null` gesendet;
eine entfernte Endzeit verschwindet dadurch auch in CloudKit.

Vor dem Anmelden prüft die Seite den anonymen öffentlichen Abruf und die Verweigerung
anonymer Entwurfsabfragen. Nach der Anmeldung werden zusätzlich beide Datensatztypen
mit den Rechten des angemeldeten Kontos abgefragt. Schreibaktionen bleiben bis zu
bestandenen Prüfungen gesperrt; die tatsächlichen Rechte erzwingt weiterhin CloudKit.

In der Console bestätigt: Nachrichtentypen und WorkBadgeAdmin-Rolle in Development
vorhanden, World nur Leserecht für WorkBadgeAnnouncement, keine allgemeinen
Schreibrechte auf Nachrichtentypen. Der fehlende audience-Index wurde ergänzt.
In Production ist bislang nur der Users-Typ vorhanden. Eine vollständige
Schema-Übernahme würde auch die vorher vorhandenen BingoGame/BingoPlayer-Typen
und deren Rechte übernehmen; diese Freigabe wurde noch nicht durchgeführt.

Live gegen Apple geprüft: Der anonyme Abruf veröffentlichter Nachrichten mit den
Filtern `status == published` und `audience == all` funktioniert (derzeit 0 Datensätze).
Die anonyme Abfrage von WorkBadgeAnnouncementDraft wird abgewiesen.
Ein authentifizierter Schreibtest wurde noch nicht durchgeführt: Die zusätzliche
Apple-Anmeldung für die Admin-Website ist offen. Der erste Anmeldeversuch scheiterte
bei Apples altem Post-Message-Rückgabehelfer; der frische Versuch mit gespeicherter
URL-Rückleitung wartet auf die Benutzeranmeldung.

## CloudKit-Berechtigungen (vor Freigabe prüfen)

| Datensatztyp | World | Authenticated/Creator | WorkBadgeAdmin |
| --- | --- | --- | --- |
| WorkBadgeAnnouncement | Lesen | Keine zusätzlichen Schreibrechte | Lesen, Erstellen, Ändern |
| WorkBadgeAnnouncementDraft | Kein Zugriff | Kein Zugriff | Lesen, Erstellen, Ändern |

Nur das ausdrücklich bestimmte Administrator-iCloud-Konto der Rolle WorkBadgeAdmin
zuweisen. Andere CloudKit-Datensatztypen (z.B. Bingo) nicht verändern.
Öffentlich sind nur freigegebene Texte, keine Entwürfe oder Zugangsdaten.
Ein clientseitiger `writesEnabled`-Schalter ersetzt diese Serverrechte nicht.

Den Web-API-Token auf den Origin `https://admin.workbadge.de` begrenzen.
Keine Server-Schlüssel oder privaten Schlüssel im öffentlichen Repository hinterlegen.
Vor dem Freischalten der Schreibaktionen: Abruf mit Admin-Anmeldung,
anonymer Abruf öffentlicher Inhalte, Verweigerung von Entwurfsabruf und Schreibversuchen
ohne Admin-Rolle prüfen. Neue Zugriffsrechte und Tokens erst nach ausdrücklicher Freigabe
einrichten. Schema und Rollen vor einer App-Store-/TestFlight-Nutzung in Production prüfen.

## App-Version und Umgebungen

Das aktualisierte Quellprojekt liegt separat im beigefügten App-Paket. In Info.plist sind
CloudKitEnabled und der bestehende Container eingetragen. Die verwendete Umgebung kommt
aus dem iCloud-Entitlement des signierten Builds: Xcode-Debug üblicherweise Development,
App Store/TestFlight Production. Die Admin-Umgebung muss dazu passen.
Die bereits installierte App wird durch eine Website-Änderung nicht aktualisiert.
Auf einem Mac mit Xcode bauen/signieren und auf dem iPhone installieren.

## Nachweis

`node test-announcements.cjs` und `node test-signin-flow.cjs` prüfen den tatsächlichen Admin-Code mit einem CloudKit-Modell,
das globale Datensatz-IDs und Versionskonflikte erzwingt: Entwurf, Veröffentlichung,
Bearbeitung, Archivierung, Löschung, Rechtefehler und fehlerhafte Daten.
Dies ist kein Test gegen Apples Server. Zusätzliche XCTest-Fälle im App-Projekt prüfen
Datensatz-Decodierung, Zeitstempel, Öffentlichkeit und ungültige Konfigurationen.
Xcode-/iPhone-Tests und ein echter Admin→CloudKit→iPhone-Test bleiben separat nachzuweisen.

Für den echten Versandtest zuerst eine zeitlich begrenzte Testnachricht in Development
veröffentlichen, mit der aktualisierten Debug-App abrufen und wieder archivieren.
Nicht ohne abgestimmten Inhalt einen Test an alle Production-Nutzer senden.
