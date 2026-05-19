# Awesome Copilot Instructions

Dieses Repository ist **keine Projekt-Template-Struktur**, sondern eine **Bibliothek** mit wiederverwendbaren Copilot Instructions.

Du waehlst daraus die passenden Dateien fuer dein konkretes Projekt aus und legst sie im Zielprojekt unter `.github/instructions/` ab.

## Wie du es richtig verwendest

1. Oeffne diese Bibliothek und waehle nur die fuer dein Projekt relevanten Instructions.
2. Kopiere diese Dateien in dein Zielprojekt nach `.github/instructions/`.
3. Copilot in VS Code liest diese Dateien dort automatisch ein und wendet Regeln ueber `applyTo` auf passende Dateien an.

## Beispielstruktur im Zielprojekt

```text
mein-typescript-webprojekt/
└── .github/
    └── instructions/
        ├── typescript.instructions.md
        ├── security-and-owasp.instructions.md
        └── performance-optimization.instructions.md
```

## Merksatz

> [!IMPORTANT]
> **!!! MERKSATZ !!!**
>
> **NICHT das komplette Template kopieren.**
>
> **Nur passende Instructions aus dieser Bibliothek auswaehlen und in `.github/instructions/` des jeweiligen Projekts ablegen.**
>
> ***** GLITZER-ALARM: DIESE ZEILE BITTE NIE IGNORIEREN. *****
