# Quartz 4 – Anpassungen und Erweiterungen

Wiederherstellungsreferenz für alle Änderungen am Quartz-Standard.
Letzte Aktualisierung: 03.08.2026

---

## Übersicht: Update-Sicherheit

| Datei | Art | Sicher bei `npx quartz update`? |
|---|---|---|
| `quartz.layout.ts` | User-Config | ✅ Ja |
| `quartz/styles/custom.scss` | User-Customization | ✅ Ja |
| `quartz/static/lightbox.js` | Eigene Datei | ✅ Ja |
| `quartz/components/scripts/graph.inline.ts` | Framework-Kern | ⚠️ **Nein – manuell wiederherstellen** |

---

## ✅ 1. `quartz.layout.ts` — Graph-Konfiguration

**Was:** Zwei bedingte Graph-Komponenten (Startseite vs. Unterseiten). `#todo`-Tags werden im Graph ausgeblendet. Auf der Startseite wird der Graph mit voller Tiefe (`depth: -1`) angezeigt, auf allen anderen Seiten nur die direkten Nachbarn (`depth: 1`).

**Vollständige Graph-Konfiguration:**
```typescript
// Startseite (index): voller Graph
Component.ConditionalRender({
  component: Component.Graph({
    localGraph:  { depth: -1, repelForce: 1.2, centerForce: 0.5, linkDistance: 20,  removeTags: ["todo"] },
    globalGraph: { depth: -1, repelForce: 8,   centerForce: 0.7, linkDistance: 170, opacityScale: 3, removeTags: ["todo"] },
  }),
  condition: (page) => page.fileData.slug === "index",
}),

// Alle anderen Seiten: nur direkte Nachbarn
Component.ConditionalRender({
  component: Component.Graph({
    localGraph:  { depth: 1, repelForce: 1.2, centerForce: 0.5, linkDistance: 20,  removeTags: ["todo"] },
    globalGraph: { depth: 1, repelForce: 8,   centerForce: 0.7, linkDistance: 170, opacityScale: 3, removeTags: ["todo"] },
  }),
  condition: (page) => page.fileData.slug !== "index",
}),
```

> **Hinweis:** `repelForce` wird intern mit `-100` multipliziert → `repelForce: 8` entspricht D3-Stärke `-800`. Werte über `14` sprengen den Graph auseinander.

---

## ✅ 2. `quartz/styles/custom.scss` — Stile

**Was:** Seitenbreite responsiv (`clamp()`), breitere Desktop-Sidebar, Bilder zentriert, Callout-Icons ausgeblendet, drei Callout-Typen mit eigenem Design, Transclude-Platzhalter der Blockübersicht-Grafik ausgeblendet.

**Verwendung in Markdown:**
- `> [!example|titel] HS1:` → Handlungssituation (blauer Balken links)
- `> [!info]` → Kenntnisse (grauer Balken links, Titel versteckt)
- `> [!note]` → Hinweise (gelber Balken links, Titel versteckt)
- `> [!info|titel]` oder `> [!note|titel]` → Titel einblenden

**Vollständiger Inhalt:**
```scss
@use "./base.scss";

/* ── Seitenbreite an Bildschirm anpassen ──────────────────── */
.page {
  max-width: clamp(800px, 95vw, 2200px);
}

.page > #quartz-body {
  @media all and (min-width: 1200px) {
    grid-template-columns: 380px auto 380px;
  }
}

/* ── Bilder mit Abstand ──────────────────── */
article img {
  display: block;
  margin: 1.5rem auto;
}

/* ── Obsidian-Plugin-Embed (![[...html]]) auf der Webseite ausblenden ──── */
/* Quartz kennt .html nicht als Bild-Embed und rendert stattdessen einen
   Transclude-Blockquote-Platzhalter. Die echte Grafik kommt hier aus dem
   separaten <iframe>, das direkt darunter in derselben Datei steht. */
blockquote.transclude[data-url*="blockuebersicht"] {
  display: none;
}

/* ── Callout-Icons ausblenden ─────────────── */
.callout-icon {
    display: none;
}

/* Titel bei Note und Info standardmässig verstecken */
.callout[data-callout="note"] .callout-title,
.callout[data-callout="info"] .callout-title {
    display: none;
}

/* Titel einblenden mit: > [!info|titel] oder > [!note|titel] */
.callout[data-callout="note"][data-callout-metadata~="titel"] .callout-title,
.callout[data-callout="info"][data-callout-metadata~="titel"] .callout-title {
    display: flex;
}

/* Handlungssituationen [!example] – dezentes Blau */
.callout[data-callout="example"] {
    --color: rgb(26, 127, 168);
    --bg: rgba(26, 127, 168, 0.05);
    border: none;
    border-left: 3px solid rgba(26, 127, 168, 0.6);
    box-shadow: none;
}

/* Kenntnisse [!info] – dezentes Grau */
.callout[data-callout="info"] {
    --color: rgb(100, 100, 100);
    --bg: rgba(0, 0, 0, 0.03);
    border: none;
    border-left: 3px solid rgba(0, 0, 0, 0.2);
    box-shadow: none;
}

/* Hinweise [!note] – dezentes Gelb */
.callout[data-callout="note"] {
    --color: rgb(180, 140, 0);
    --bg: rgba(180, 140, 0, 0.05);
    border: none;
    border-left: 3px solid rgba(180, 140, 0, 0.4);
    box-shadow: none;
}
```

---

## ✅ 3. `quartz/static/lightbox.js` — Lightbox für normale Bilder

**Was:** Klick auf ein Bild (kein SVG) öffnet es in einem dunklen Overlay. SVG-Bilder sind im Selektor weiterhin ausgeschlossen (Relikt aus der Zeit mit `svg-lightbox.js`, das inzwischen entfernt wurde, da im ME-Vault aktuell keine SVGs mehr vorkommen) — harmlos, aber falls je wieder eine SVG-Grafik eingebunden wird, bekommt sie ohne eigenes Lightbox-Skript kein Klick-Verhalten.

```javascript
document.addEventListener("DOMContentLoaded", () => { setupLightbox() })
document.addEventListener("nav", () => { setupLightbox() })

function setupLightbox() {
  const overlay = document.getElementById("lightbox-overlay") ?? createOverlay()
  document.querySelectorAll("article img:not([src$='.svg'])").forEach((img) => {
    img.style.cursor = "zoom-in"
    img.addEventListener("click", () => {
      overlay.querySelector("img").src = img.src
      overlay.style.display = "flex"
    })
  })
}

function createOverlay() {
  const overlay = document.createElement("div")
  overlay.id = "lightbox-overlay"
  overlay.style.cssText = `
    display:none; position:fixed; inset:0; background:rgba(0,0,0,0.85);
    z-index:9999; justify-content:center; align-items:center; cursor:zoom-out;
  `
  const img = document.createElement("img")
  img.style.cssText = "max-width:90vw; max-height:90vh; border-radius:6px;"
  overlay.appendChild(img)
  overlay.addEventListener("click", () => (overlay.style.display = "none"))
  document.body.appendChild(overlay)
  return overlay
}
```

---

## ⚠️ 4. `quartz/components/scripts/graph.inline.ts` — Graph-Rendering

> **Diese Datei wird bei `npx quartz update` überschrieben.**
> Nach jedem Update die drei folgenden Stellen manuell wiederherstellen.

### Änderung A — Knotengrösse nach eingehenden Links (ca. Zeile 216)

**Wozu:** Leistungsziele (d2.3, a1.1 etc.) auf die viele Blöcke verweisen werden gross dargestellt. Blöcke selbst (ID4, KT3), die nur verlinken, bleiben klein.

```typescript
function nodeRadius(d: NodeData) {
  const numIncoming = graphData.links.filter(
    (l) => l.target.id === d.id,
  ).length
  return 2 + Math.sqrt(numIncoming) * 4
}
```

### Änderung B — Texte mit Zeilenumbruch (im `new Text({...})` Block, ca. Zeile 391)

**Wozu:** Lange Knotennamen werden umgebrochen statt überlappend dargestellt.

```typescript
style: {
  fontSize: fontSize * 15,
  fill: computedStyleMap["--dark"],
  fontFamily: computedStyleMap["--bodyFont"],
  wordWrap: true,        // ← NEU
  wordWrapWidth: 120,    // ← NEU
  align: "center",       // ← NEU
},
```

### Änderung C — Kollisionsradius (`.force("collide", ...)`, ca. Zeile 180)

**Wozu:** Knoten halten grösseren Abstand → Labels überlappen sich nicht.

```typescript
.force("collide", forceCollide<NodeData>((n) => nodeRadius(n) + 55).iterations(5))
```

---

## Blockübersicht 1. Lehrjahr — `_Bilder/blockuebersicht.html`

Ersetzt die alte SVG-Chronologie (dieser Abschnitt beschrieb bisher `Chronologie_1.Lehrjahr.svg`, EI-Erbe, wird von ME nicht mehr verwendet). Selbst gebaute, in sich geschlossene HTML/CSS/JS-Datei statt SVG: pro Semester eine Zeile mit proportional breiten Kacheln (Breite = `lektionen_vorgabe`), Hover zeigt Titel + Lektionen in einer festen Infozeile, Klick navigiert direkt zur Blockseite.

**Einbindung in `index.md` — zwei Zeilen, je eine pro Plattform:**
```markdown
![[blockuebersicht.html|-x-]]

<iframe src="_Bilder/blockuebersicht.html" style="width:100%; border:none;" onload="this.style.height = this.contentWindow.document.body.scrollHeight + 'px';"></iframe>
```
- Erste Zeile: nur für Obsidian, via Community-Plugin **„Embed HTML" (mnaoumov, Plugin-ID `obsidian-embed-html`)** — muss zusätzlich zum Installieren unter Community-Plugins aktiv geschaltet werden.
- Zweite Zeile: nur für Quartz, funktioniert weil `remarkRehype({ allowDangerousHtml: true })` in `quartz/processors/parse.ts` rohes HTML durchreicht. `onload` passt die Höhe automatisch an den echten Inhalt an.
- Die custom.scss-Regel aus Abschnitt 2 blendet den Transclude-Platzhalter aus, den die erste Zeile auf der Quartz-Seite sonst zusätzlich erzeugen würde (`.html` ist keine von Quartz erkannte Bild-Embed-Endung).

**Echte Verlinkung:** Ziel-URLs relativ zu `_Bilder/`, abgeleitet aus dem gebauten `public/`-Ordner (z.B. `../1.-Lehrjahr/1.-Semester/AS1_Baustelle-einrichten`). Jede Kachel ein echtes `<a href="..." target="_top">` — `target="_top"` zwingend, sonst navigiert nur der Iframe.

**Stolpersteine:**
- `content/` im Quartz-Repo ist kein Symlink, sondern eine manuell synchronisierte Kopie — vor jedem lokalen Build geänderte Vault-Dateien manuell nach `content/` kopieren.
- **CSS Container Queries (`container-type: inline-size`) sind mit dem Obsidian-Plugin inkompatibel** — die Containment-Regel verhindert, dass der Browser die Elementbreite aus dem Inhalt ableiten kann, was die Auto-Grössen-Messung des Plugins zerschiesst. Für responsive Schriftgrössen stattdessen `vw`-basiertes `clamp()`, z.B. `clamp(14px, calc(11.1px + 0.76vw), 19px)`.
- Quartz' Assets-Emitter entfernt bei `.html`-Dateien die Endung beim Kopieren (`slugifyFilePath` in `quartz/util/path.ts` behandelt `.html` wie `.md`) — funktioniert in der Praxis trotzdem (Dev-Server liefert die Datei auch ohne Endungs-Match korrekt aus). Falls das je bricht: Datei als `.htm` speichern (nicht auf der Strip-Liste).

**Quartz-Slugify-Regeln** (für korrekte URLs, gilt weiterhin):
- Leerzeichen → `-`
- Umlaute bleiben (ä, ö, ü)
- Klammern bleiben — z.B. `AS4_PSA-(Elektro)`
- `«»` bleiben erhalten

---

## Auswertungen — Dataview in Quartz

Quartz unterstützt Dataview-Abfragen nicht nativ (Dataview ist ein Obsidian-Plugin, Quartz ist statisch). Die Lösung: das Obsidian-Plugin **Dataview Serializer** rendert Abfragen vor dem Build zu statischem Markdown vor.

**Ordner:** `content/Auswertungen/`

**Funktionsweise:**
1. Im Obsidian-Editor läuft das Dataview Serializer Plugin im Hintergrund
2. Es liest den `<!-- QueryToSerialize: ... -->` Kommentar und führt die Dataview-Abfrage aus
3. Das Ergebnis wird als statische Markdown-Tabelle zwischen `<!-- SerializedQuery: ... -->` und `<!-- SerializedQuery END -->` gespeichert
4. Quartz rendert diese Tabelle als normales Markdown — ohne Dataview-Kenntnis

**Beispiel** (`content/Auswertungen/ToDo.md`):
```markdown
<!-- QueryToSerialize: TABLE WITHOUT ID item.section AS "Abschnitt", item.text AS "Aufgabe"
FROM "" FLATTEN file.lists AS item WHERE contains(item.tags, "todo") -->
<!-- SerializedQuery: ... -->
| Abschnitt | Aufgabe |
|---|---|
| ... | ... #todo |
<!-- SerializedQuery END -->
```

**Hinweis:** Die Tabelle wird nur aktualisiert, wenn die Datei in Obsidian geöffnet und das Plugin aktiv ist. Nach Änderungen im Vault die Datei kurz in Obsidian öffnen, dann committen.

**Verbindung zum Graph:** `#todo`-Tags sind im Graph ausgeblendet (`removeTags: ["todo"]` in `quartz.layout.ts`).

---

## Wiederherstellung nach `npx quartz update`

1. `quartz.layout.ts` — Graph-Konfiguration aus Abschnitt 1 prüfen (meist unverändert)
2. `quartz/styles/custom.scss` — Inhalt aus Abschnitt 2 prüfen (meist unverändert)
3. `quartz/static/lightbox.js` — Falls überschrieben: Inhalt aus Abschnitt 3 einfügen
4. `quartz/components/scripts/graph.inline.ts` — **Die drei Änderungen A, B, C aus Abschnitt 4 manuell einpflegen**
