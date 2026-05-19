# ANLEITUNG: Aus dieser Bibliothek ein Projekt-Repository aufsetzen

Diese Anleitung ist extra **langsam, klar und Schritt fuer Schritt** geschrieben.
Ziel: Du nimmst diesen Ordner **nicht** als fertiges Template, sondern als **Bibliothek** und baust daraus ein sauberes Projekt-Repo.

---

## 0) Kurz in einem Satz (damit der Kopf ruhig bleibt)

Du erstellst erst dein eigenes Projekt-Repository, dann kopierst du nur die **passenden** Instructions nach:

`dein-projekt\.github\instructions\`

---

## 1) Was ist was?

- `awesome-copilot-instructions` = Sammlung/Bibliothek (Quelle)
- `dein-projekt` = dein echtes Arbeits-Repository (Ziel)
- `.github\instructions\` = dort liegen im Zielprojekt die aktiven Instructions

**Wichtig:** Nicht alles blind kopieren. Nur das, was zu deinem Projekt passt.

---

## 2) Voraussetzungen

Du brauchst:

1. Git installiert
2. GitHub-Account
3. VS Code mit GitHub Copilot
4. Diese Bibliothek lokal auf deinem Rechner (hast du bereits)

Optional, aber hilfreich:

- PowerShell
- GitHub CLI (`gh`)

---

## 3) Beispiel: Neues TypeScript-Webprojekt aufsetzen

Wir machen es als klares Beispiel. Danach kannst du die gleichen Schritte fuer andere Projekte nutzen.

### Schritt 3.1: Projektordner erstellen

```powershell
cd C:\Users\Christian
mkdir mein-typescript-webprojekt
cd .\mein-typescript-webprojekt
git init
```

### Schritt 3.2: Basisdateien anlegen

```powershell
New-Item -ItemType Directory -Path .github\instructions -Force | Out-Null
New-Item -ItemType File -Path README.md -Force | Out-Null
```

In `README.md` kannst du erstmal nur reinschreiben:

```text
# mein-typescript-webprojekt
```

---

## 4) Die richtigen Instructions aus der Bibliothek auswaehlen

Quelle (deine Bibliothek):

`C:\Users\Christian\awesome-copilot-instructions`

Fuer ein TypeScript-Webprojekt sind oft sinnvoll:

1. TypeScript
2. Security/OWASP
3. Performance
4. Optional: Testing/Docs je nach Bedarf

### Schritt 4.1: Bibliothek im Explorer oder VS Code oeffnen

Schau in die thematischen Ordner (`01-...`, `02-...`, ...), bis du passende `*.instructions.md` Dateien findest.

### Schritt 4.2: Nur relevante Dateien kopieren

Beispiel (Dateinamen bitte an deine echten Dateien anpassen):

```powershell
Copy-Item "C:\Users\Christian\awesome-copilot-instructions\07-web-frontend-and-node\typescript.instructions.md" ".github\instructions\typescript.instructions.md"
Copy-Item "C:\Users\Christian\awesome-copilot-instructions\02-quality-security-and-docs\security-and-owasp.instructions.md" ".github\instructions\security-and-owasp.instructions.md"
Copy-Item "C:\Users\Christian\awesome-copilot-instructions\07-web-frontend-and-node\performance-optimization.instructions.md" ".github\instructions\performance-optimization.instructions.md"
```

Wenn eine Datei nicht existiert: im Bibliotheksordner den genauen Namen nachsehen und Befehl anpassen.

---

## 5) Pruefen, ob die Struktur stimmt

Im Zielprojekt:

```powershell
Get-ChildItem .github\instructions
```

Du solltest jetzt mehrere `*.instructions.md` Dateien sehen.

---

## 6) Erste Version committen und nach GitHub pushen

### Schritt 6.1: Dateien stagen und committen

```powershell
git add .
git commit -m "Initial project setup with Copilot instructions"
```

### Schritt 6.2: Repository auf GitHub anlegen und pushen

Mit `gh` (wenn installiert):

```powershell
gh repo create mein-typescript-webprojekt --private --source . --remote origin --push
```

Ohne `gh`:

1. Auf GitHub ein neues leeres Repo erstellen
2. Dann lokal:

```powershell
git remote add origin https://github.com/<dein-user>/mein-typescript-webprojekt.git
git branch -M main
git push -u origin main
```

---

## 7) Wie du spaeter sauber weiterarbeitest (sehr wichtig)

Wenn sich dein Projekt aendert:

1. Neue Tech dazugekommen? -> passende neue Instruction aus Bibliothek kopieren
2. Nicht mehr gebraucht? -> Datei in `.github\instructions\` entfernen
3. Danach committen:

```powershell
git add .github\instructions
git commit -m "Adjust Copilot instructions for current project scope"
git push
```

---

## 8) Typische Fehler (und direkte Loesung)

### Fehler A: "Ich habe die ganze Bibliothek kopiert"

Loesung:

1. In deinem Zielprojekt nur `.github\instructions\` behalten
2. Unpassende Dateien loeschen
3. Committen

### Fehler B: "Copilot reagiert komisch/zu breit"

Loesung:

1. Zu viele Instructions aktiv? Ausduennen
2. Ueberlappende Instructions reduzieren
3. Projekt-spezifisch statt universal denken

### Fehler C: "Nach Pause weiss ich nicht, wo ich war"

Loesung:

1. Immer nach jeder Aenderung committen
2. Kleine Commit-Nachrichten schreiben
3. Im README des Zielprojekts 3 Zeilen notieren:
   - Welche Instructions aktiv sind
   - Warum genau diese
   - Was als Naechstes geplant ist

---

## 9) Mini-Checkliste zum Abhaken

- [ ] Neues Zielprojekt erstellt
- [ ] `.github\instructions\` angelegt
- [ ] Nur passende `*.instructions.md` kopiert
- [ ] Struktur geprueft
- [ ] Ersten Commit gemacht
- [ ] Nach GitHub gepusht

Wenn alles abgehakt ist, bist du sauber startklar.
