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
Vor dem Aktivieren von `enabled` und `writesEnabled`: Abruf mit Admin-Anmeldung,
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

`node test-announcements.cjs` prüft den tatsächlichen Admin-Code mit einem CloudKit-Modell,
das globale Datensatz-IDs und Versionskonflikte erzwingt: Entwurf, Veröffentlichung,
Bearbeitung, Archivierung, Löschung, Rechtefehler und fehlerhafte Daten.
Dies ist kein Test gegen Apples Server. Zusätzliche XCTest-Fälle im App-Projekt prüfen
Datensatz-Decodierung, Zeitstempel, Öffentlichkeit und ungültige Konfigurationen.
Xcode-/iPhone-Tests und ein echter Admin→CloudKit→iPhone-Test bleiben separat nachzuweisen.

Für den echten Versandtest zuerst eine zeitlich begrenzte Testnachricht in Development
veröffentlichen, mit der aktualisierten Debug-App abrufen und wieder archivieren.
Nicht ohne abgestimmten Inhalt einen Test an alle Production-Nutzer senden.
