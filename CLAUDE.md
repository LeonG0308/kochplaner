# Kochplaner – Projektkontext für Claude Code

Vegetarischer Koch- und Einkaufsplaner als PWA für einen 2-Personen-Haushalt (München).
Privates Projekt von Leon (`leongorg03@gmail.com`) und seiner Freundin. Der Nutzer ist Laie –
Änderungen müssen ohne Nacharbeit funktionieren, Antworten gehören auf Deutsch und ohne Fachjargon.

## Kurzfassung

| | |
|---|---|
| **Live** | https://leong0308.github.io/kochplaner/ |
| **Repo** | `LeonG0308/kochplaner`, Branch `main`, **öffentlich** |
| **Hosting** | GitHub Pages (Deploy = Push auf `main`) |
| **Datenbank** | Firebase Firestore, Projekt `kochplaner-2499a`, Region `eur3` |
| **KI** | Claude API (Anthropic), Modell `claude-sonnet-4-6`, direkt aus dem Browser |
| **Lokaler Klon** | `repo/` in diesem Ordner |

## Dateien

Die App ist eine **Single-Page-App: alles steckt in `index.html`** (HTML + CSS + JS, ca. 1600 Zeilen,
kein Framework, kein Build-Schritt). Bearbeitet wird immer direkt diese Datei.

| Datei | Zweck |
|---|---|
| `index.html` | Komplette App |
| `sw.js` | Service Worker (PWA-Cache, network-first für HTML) |
| `manifest.json` | PWA-Manifest |
| `icon.png` | App-Icon 512×512 |
| `CLAUDE.md` | Diese Datei |

## Datenmodell (Firestore)

Ein einziges Dokument `app/shared` hält den kompletten gemeinsamen Zustand beider Nutzer:

```
app/shared
  recipes         Array<Recipe>
  weekPlans       Map "YYYY-MM-DD" -> { meals: [{recipeId, portions}] }
  floatingMeals   Array<{recipeId, portions}>
  cart            Array<{id, key, name, amount, unit, checked}>
  savedLists      Array<{id, name, date, items[]}>
  defaultPortions Int (Default 2)
  updatedAt       Timestamp

backups/JJJJ-MM-TT  (taegliche, unveraenderliche Kopie des kompletten Zustands)
  recipes, weekPlans, floatingMeals, savedLists, defaultPortions, recipeCount, savedAt

app/config        (getrennt, damit App-Daten und Zugangsdaten nie kollidieren)
  k1              Claude-API-Key, verschleiert (siehe "API-Keys")
  k2              OpenAI-API-Key, verschleiert
  updatedAt       Timestamp
```

`Recipe`: `{ id, name, emoji, category, notes, ingredients:[{name, amount, unit}], steps?:[], tags:{time, weight, cuisine, extra_tags[]} }`

**Zutatenmengen sind IMMER pro 1 Portion gespeichert.** Portionen werden erst beim Planen/Einkaufen multipliziert.
IDs: `genId()` = `Date.now().toString(36) + 5 Zufallszeichen` – die ersten 8 Zeichen sind also ein Zeitstempel
(nützlich für Forensik: `new Date(parseInt(id.slice(0,8),36))`).

## Absolute Regeln beim Arbeiten an der Sync-Schicht

Dieses Projekt hat **zweimal fast alle Rezepte verloren** (März 2026, August 2026). Ursache beide Male:
Die App hat bei einem Lesefehler mit leerem Zustand gestartet und diesen leeren Zustand nach Firestore
zurückgeschrieben. Deshalb gelten diese Regeln – sie dürfen nie wieder aufgeweicht werden:

1. **Nie schreiben, bevor einmal erfolgreich gelesen wurde.** `save()` bricht ab, solange `remoteLoaded === false`.
2. **Ein Lesefehler darf niemals Seed-Daten erzeugen.** Seeding ist ausschließlich erlaubt, wenn Firestore
   bestätigt, dass das Dokument nicht existiert (`snap.exists === false`) und kein lokales Backup vorliegt.
3. **Kein Schreibvorgang darf Daten vernichten**, der nicht vom Nutzer ausgelöst wurde: `save()` blockiert
   Schreibvorgänge, die Rezepte/Listen massiv reduzieren, außer der Aufrufer setzt explizit `{destructive:true}`
   (nur `deleteRecipe`, `deleteSavedList`, `confirmResetAll`).
4. **Jeder erfolgreiche Ladevorgang schreibt ein lokales Backup** (`kp_lastgood` + rotierende Historie
   `kp_history`, max. 5 Stände) in localStorage. Das ist die letzte Rettungsleine.
5. Offline/Fehler ⇒ App läuft im **Nur-Lese-Modus** mit Banner, statt zu raten.
6. **Tägliche Cloud-Sicherung**: Beim ersten erfolgreichen Laden pro Tag schreibt die App eine
   Kopie nach `backups/JJJJ-MM-TT`. Bestehende Tages-Sicherungen werden nie überschrieben.
   Firestore selbst hat auf dem Spark-Tarif **keine** Backups (weder PITR noch geplante Sicherungen),
   deshalb ist das die einzige Historie, die existiert.
7. **Geräte-Rettung**: Findet die App auf einem Gerät lokale Sicherungen mit Rezepten, die in der
   Datenbank fehlen, bietet sie diese per Banner zum Zurückholen an. Wiederherstellungen
   **ergänzen** immer nur (`mergeRecipes`) und löschen nie.
8. `confirmResetAll` fasst die Rezepte nicht mehr an – ein Ein-Klick-Totalverlust ist unmöglich.

## API-Keys

**Niemals einen API-Key in den Code oder ins Repo schreiben.** Das Repo ist öffentlich; Anthropic hat
im Februar 2026 mehrfach automatisch Keys widerrufen, die per Push im Repo landeten (siehe Mails
"Anthropic credentials detected on GitHub").

Keys liegen in Firestore unter `app/config`, XOR-verschleiert (`obf()`/`deobf()`), plus als lokale Kopie
in `localStorage` für den Offline-Fall. Dadurch muss der Key nur **einmal auf einem Gerät** eingetragen
werden und steht danach auf allen Geräten zur Verfügung. Ein Gerät, das noch einen lokalen Key hat,
lädt ihn beim Start automatisch hoch (Migration), falls in Firestore noch keiner liegt.

Trade-off, der bewusst so gewählt wurde: Der Lese-/Schreibzugriff auf `app/config` ist offen, der Key
ist damit technisch nicht geheim. Verschleierung schützt nur gegen zufälliges Mitlesen. Bei Verdacht:
Key im Anthropic-Console rotieren, einmal neu eintragen – er verteilt sich dann von selbst.

## Datenverlust-Historie (Recherche vom 31.08.2026)

Es wurde erschöpfend nach den Rezepten gesucht, die zwischen dem 27.02.2026 und dem 12.08.2026
hinzugefügt wurden. Sie existieren **nirgends** mehr. Geprüft und ausgeschlossen: Firestore
(nur `app/shared` + `app/config`, per Konsole mit Admin-Sicht verifiziert), PITR und geplante
Sicherungen (auf dem Spark-Tarif gar nicht verfügbar), lokale Browser-Speicher aller Profile
dieses Rechners, 700 Claude-Chats, die komplette GitHub-Historie (25 Commits, nie Daten),
Gmail und Google Drive. Nicht prüfbar waren die Handys – dafür existiert jetzt die Geräte-Rettung.

## Firestore-Regeln – der harte Schutz

Seit 31.08.2026 erzwingt die **Datenbank selbst**, dass Rezepte nicht verschwinden können.
Kein Client – auch kein fehlerhafter – kann die Sammlung mehr leeren:

```
rules_version = '2';
service cloud.firestore {
match /databases/{database}/documents {
match /app/shared {
allow read: if true;
allow create: if request.resource.data.get('recipes', []).size() > 0;
allow update: if request.resource.data.get('recipes', []).size() > 0 && request.resource.data.get('recipes', []).size() * 2 >= resource.data.get('recipes', []).size();
allow delete: if false;
}
match /app/config {
allow read, write: if true;
}
match /backups/{snapshot} {
allow read, create: if true;
allow update, delete: if false;
}
}
}
```

Konsequenzen, die man kennen muss:
- Ein Schreibvorgang, der **0 Rezepte** enthält, wird mit HTTP 403 abgelehnt.
- Ein Schreibvorgang, der **mehr als die Hälfte** der Rezepte auf einmal entfernt, wird abgelehnt.
  Einzelne Rezepte löschen funktioniert weiter, alle auf einen Schlag nicht mehr.
- `app/shared` kann nicht gelöscht werden.
- Sicherungen unter `backups/` sind **unveränderlich**: anlegen ja, ändern und löschen nein.
- Das alte Ablaufdatum (22.03.2027) ist entfernt – die Regeln laufen nicht mehr aus.

Verifiziert am 31.08.2026 mit 12 Tests gegen die Live-Datenbank (`_backup/ruletest.js`).

## Features (Kurzüberblick)

- **Tabs**: Rezepte · Planer (Wochenplan Mo–So + „Floating Meals") · Listen · Warenkorb
- **Rezept-Import per KI**: Text, Bild oder URL (URL nutzt Web Search + Extended Thinking)
- **Chat-Bot** (💬, unten rechts): kennt alle Rezepte/Pläne/Listen und kann über einen JSON-`actions`-Block
  16 Aktionen ausführen (Rezepte anlegen/ändern/löschen, Mahlzeiten planen, Warenkorb, Listen)
- **Auto-Tagging** importierter Rezepte, Filter nach Zeit/Schwere/Küche, KI-Filter per Freitext
- **Einkaufsliste** aus Wochenplan generieren, Aggregation über `makeKey(name, unit)`
- **Export** nach Apple Notes via `navigator.share`, Knuspr-Prompt-Generator

## Deploy

```bash
cd repo
git add -A && git commit -m "..." && git push
```
GitHub Pages veröffentlicht `main` automatisch (ca. 1 Minute). Danach `sw.js` Cache-Version
(`CACHE_NAME`) und die Registrierung in `index.html` (`sw.js?v=N`) hochzählen, sonst sehen
installierte PWAs die Änderung verzögert.

## Nützliche Kommandos

```bash
# Aktuellen Datenbankstand ansehen (Lesezugriff ist offen)
curl -s "https://firestore.googleapis.com/v1/projects/kochplaner-2499a/databases/(default)/documents/app/shared?key=AIzaSyDruR4dDDlEMJ2Jv7a0cAusd-RidLn8F90"
```

Sicherungen und Forensik-Skripte dieses Projekts liegen in `../_backup/`.
