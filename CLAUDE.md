# ⚠️ RÈGLE ABSOLUE — Maintenir README.md et CLAUDE.md à jour

Après **chaque modification fonctionnelle** (nouvelle feature, changement de comportement, modification d'architecture), mettre à jour **systématiquement** :

- **`CLAUDE.md`** : logique technique, fonctions, constantes, comportements, pièges
- **`README.md`** : fonctionnalités visibles par l'utilisateur, stack, structure du projet

**Ne pas attendre que l'utilisateur le demande.** Si un changement de code mérite d'être documenté, le faire dans le même commit ou dans un commit/PR dédié immédiatement après.

---

# ⚠️ ERREUR RÉCURRENTE DE CLAUDE — Worktree vs dépôt principal

Claude travaille souvent dans un **worktree temporaire** (`\.claude\worktrees\...`) au lieu du dépôt principal.
Les modifications dans le worktree ne sont **pas visibles dans GitHub Desktop** et ne peuvent pas être committées.

**Règle absolue : toujours éditer les fichiers dans `C:\Users\petil\Documents\GitHub\LocalTierList\`**
Jamais dans `.claude\worktrees\*`.

Si des modifications ont été faites dans le worktree par erreur, les copier vers le dépôt principal avant de conclure.

---

# Tier List Maker — Contexte projet

## Vue d'ensemble
Application web **monofichier** (`index.html`) pour créer des tier lists, pensée **mobile-first** mais fonctionnelle à la souris. **100% client-side, zéro réseau** : les images uploadées sont redimensionnées via canvas, converties en data URL et persistées dans `localStorage` avec le reste de l'état. Hébergée statiquement sur GitHub Pages.

L'utilisateur uploade des images (elles arrivent dans la "Réserve d'images" en bas), puis les glisse dans les rangs (S/A/B/C par défaut). Un mode édition permet de renommer/recolorer/supprimer/ajouter des rangs et de supprimer des images au tap. Chaque action déclenche une sauvegarde automatique.

---

## Fichiers clés

| Fichier | Rôle |
|---|---|
| `index.html` | **App complète** — HTML + CSS + JS vanilla (~360 lignes), aucune dépendance |
| `assets/images/github/` | Images du README (header.png, star.gif, UI.png) + `Icon.ico` (favicon, référencé dans le `<head>`) |

---

## Stack (`index.html`)

- **Vanilla JS strict** (`"use strict"`) — pas de framework, pas de build, pas de CDN, **aucune dépendance externe**
- **⚠️ CSP** (`<meta http-equiv="Content-Security-Policy">` dans le `<head>`) : `default-src 'none'; script-src 'unsafe-inline'; style-src 'unsafe-inline'; img-src 'self' data: blob:; base-uri 'none'; form-action 'none'` — **aucun appel réseau externe possible, par construction**. `'unsafe-inline'` est requis car tout le script/style est inline (monofichier) ; `'self'` dans `img-src` est requis par le **favicon** (`<link rel="icon">` → `assets/images/github/Icon.ico`). **Toute nouvelle ressource (script externe, fetch, image http) sera bloquée par le navigateur tant que la CSP n'est pas mise à jour.**
- **⚠️ Aucun `innerHTML` nulle part** — tout le DOM est construit par `createElement` / `textContent`. **Ne jamais introduire de `innerHTML`** : c'est la garantie anti-XSS du projet (le `localStorage` est restauré sans échappement possible sinon).
- **Pointer Events** pour le drag & drop unifié souris + tactile (pas de `mousedown`/`touchstart` séparés)
- **localStorage** clé unique `tierlist-v1` — tout l'état dans un seul JSON

### Constantes module-level
```js
STORAGE_KEY    = "tierlist-v1"
IMG_SIZE       = 150   // taille de stockage des images (2x les 75px affichés, pour le retina)
DRAG_THRESHOLD = 8     // px de mouvement avant qu'un pointerdown devienne un drag (sinon c'est un tap)
SCROLL_MARGIN  = 80    // zone en haut/bas de l'écran qui déclenche l'auto-scroll pendant un drag
DEFAULT_TIERS  = [S #7fff7f (vert), A #ffff7f (jaune), B #ffbf7f (orange), C #ff7f7f (rouge)]  // dégradé inversé : le meilleur rang est vert
```

### CSS structurant
- Le **mode édition** est entièrement piloté par la classe `edit-mode` sur `<body>` — les boutons ➕/♻️, les panneaux d'édition et le hint sont en `display: none` et révélés par `body.edit-mode ...`. Pas d'état JS pour le mode édition : `document.body.classList.contains("edit-mode")` est la source de vérité.
- **⚠️ `touch-action: none` est posé UNIQUEMENT sur `img`** — la page reste scrollable partout ailleurs. Ne pas l'étendre aux conteneurs (ça tuerait le scroll mobile).
- `img.dragging` : `position: fixed` + `z-index: 1000` + `pointer-events: none` (indispensable pour que `elementFromPoint` voie la zone sous l'image, pas l'image elle-même)
- La couleur d'un tier vit à DEUX endroits : `label.style.background` (affichage) et `tier.dataset.color` (sérialisation) — les deux sont mis à jour ensemble par l'input color

---

## Format de sauvegarde (localStorage `tierlist-v1`)

```js
{
  tiers: [
    { n: "S",            // nom du rang (textContent du .label)
      c: "#ff7f7f",      // couleur (dataset.color du .tier)
      imgs: [{ s: "data:image/webp;base64,...",  // src (data URL uniquement)
               a: "fichier.png" }] }              // alt (nom du fichier d'origine)
  ],
  pool: [{ s, a }]       // réserve d'images, même format
}
```

**⚠️ Le DOM EST l'état** : il n'y a pas de modèle JS séparé. `save()` sérialise directement le DOM (`querySelectorAll`), et l'init reconstruit le DOM depuis le JSON. Toute mutation du DOM des tiers/images doit être suivie d'un `save()` ou `saveSoon()`.

---

## Fonctions module-level (`index.html`)

```js
function serializeImgs(parent)        // [...parent.querySelectorAll("img")] → [{ s: src, a: alt }]
function save()                       // sérialise tout le DOM → localStorage (try/catch quota)
function saveSoon()                   // save() debouncé à 300ms (pour les inputs texte/couleur)
function validImgEntry(d)             // garde de restauration : d.s est une string ET commence par "data:image/"
function makeImg(src, alt)            // crée un <img> (draggable=false, alt par défaut)
function createTier(name, color, imgs=[])  // construit un .tier complet (label + .images + .edit-panel) et l'append au container
function dropTargetAt(x, y)           // elementFromPoint → .images ou .pool (label/panel d'un tier → ses .images)
function updateHover(x, y) / clearHover()  // gère la classe .drop-hover sur la zone survolée
function positionDragged(x, y)        // place l'image draguée (left/top = pointeur − 37px, centre de l'image)
function autoScrollStep()             // boucle rAF : scrolle si le doigt est à < SCROLL_MARGIN du bord, vitesse proportionnelle
function endDrag(e)                   // pointerup/pointercancel : drop OU tap-suppression (mode édition)
function fileToDataURL(file)          // Promise → data URL 150×150 WebP 0.8 (fallback JPEG 0.85), null si illisible
(function init())                     // IIFE : restaure localStorage (validé) ou crée DEFAULT_TIERS
```

### Sauvegarde — règles
- `save()` immédiat après toute action ponctuelle (drop, suppression, ajout/suppression de rang, upload, reset)
- `saveSoon()` (debounce 300ms) **uniquement** pour les inputs continus (nom du rang, sélecteur couleur)
- Quota plein (`localStorage.setItem` throw) → `alert` **une seule fois** (`storageFullWarned`, ré-armé au premier save réussi)
- Restauration défensive dans `init()` : `JSON.parse` sous try/catch, `Array.isArray` sur `tiers`/`pool`/`imgs`, couleur validée par `/^#[0-9a-fA-F]{6}$/` (fallback `#6b7280`), images filtrées par `validImgEntry` (data URL image obligatoire — un `src` http ou `javascript:` forgé est rejeté)

---

## Drag & drop — Pointer Events (listeners sur `document`)

### État module-level
```js
let drag = null;      // { img, x, y, lastX, lastY, id, active } — null si aucun drag
let hovered = null;   // zone .images/.pool actuellement surlignée (.drop-hover)
let scrollRAF = null; // id requestAnimationFrame de l'auto-scroll
```

### Flux
1. **`pointerdown`** : ignoré si un drag est déjà en cours, si `!e.isPrimary` (multi-touch), si la cible n'est pas un `IMG`, ou si clic souris non-gauche. Crée `drag` avec `active: false` + `setPointerCapture` (sous try/catch — le pointeur peut déjà être relâché).
2. **`pointermove`** (filtré par `pointerId`) : tant que le déplacement est `< DRAG_THRESHOLD` (8px, `Math.hypot`), rien ne bouge — c'est un tap potentiel. Au-delà : `active = true`, classe `.dragging`, démarrage de la boucle `autoScrollStep` (rAF). Ensuite : `positionDragged` + `updateHover` à chaque move.
3. **`pointerup` / `pointercancel`** → `endDrag(e)` :
   - `drag` est remis à `null` **immédiatement et dans tous les cas** (même drag annulé)
   - **Drag actif** : stop rAF, clearHover, cible calculée via `dropTargetAt` **AVANT de retirer `.dragging`** (l'image a encore `pointer-events: none`, donc `elementFromPoint` voit la zone dessous) — uniquement sur `pointerup` (un `pointercancel` annule sans drop), puis `appendChild` dans la cible + `save()`. Pas de cible → l'image reste où elle était dans le DOM.
   - **Tap (non-actif)** : si `pointerup` ET mode édition → `confirm()` puis suppression de l'image + `save()`. Hors mode édition, un tap ne fait rien.

### Auto-scroll pendant le drag
`autoScrollStep` tourne en `requestAnimationFrame` tant que le drag est actif : si `lastY` est à moins de `SCROLL_MARGIN` (80px) du bord haut/bas, `window.scrollBy` avec une vitesse proportionnelle à la profondeur dans la zone (`/4`). Après chaque scroll, `updateHover` est rappelé (la page a bougé sous le doigt). La boucle s'auto-termine quand `drag` est null.

### Neutralisations globales
- `dragstart` → `preventDefault()` (drag natif HTML5 désactivé partout) + `img.draggable = false` à la création
- `contextmenu` sur les `IMG` → `preventDefault()` (pas de menu "Enregistrer l'image" au long-press mobile)
- CSS : `user-select: none`, `-webkit-touch-callout: none` sur les images

---

## Upload (`fileToDataURL`)

- `URL.createObjectURL(file)` → `new Image()` → dessin sur un canvas `IMG_SIZE × IMG_SIZE` (150×150)
- **Recadrage "cover"** : `r = Math.max(IMG_SIZE/w, IMG_SIZE/h)`, image centrée — même rendu que le `object-fit: cover` de l'affichage
- Export `canvas.toDataURL("image/webp", 0.8)` ; **si le navigateur ne sait pas encoder WebP** (le data URL ne commence pas par `data:image/webp`, cas Safari ancien), fallback `image/jpeg` 0.85
- **`URL.revokeObjectURL` systématique** (onload ET onerror) — pas de fuite mémoire
- Image illisible → `console.warn` + `resolve(null)` (jamais de reject : la boucle d'upload continue avec les fichiers suivants)
- Handler du `<input type="file">` : boucle `for...of` séquentielle (await), append dans `pool`, puis **`e.target.value = ""`** (permet de ré-uploader le même fichier) + `save()`

---

## Mode édition

- Toggle pur CSS : `document.body.classList.toggle("edit-mode")` — aucun état JS
- Révèle par CSS : boutons ➕ Ajouter / ♻️ Réinitialiser, le hint jaune, les `.edit-panel` de chaque tier, la bordure des tiers
- **Tap sur une image = suppression** (avec `confirm`) — géré dans `endDrag`, branche non-active
- Panneau de chaque tier (construit dans `createTier`) :
  - input texte → `label.textContent` + `saveSoon()`
  - input color → `label.style.background` + `tier.dataset.color` + `saveSoon()`
  - bouton Supprimer → **les images du tier sont déplacées dans la réserve** (pas supprimées) puis `tier.remove()` + `save()`
- ♻️ Réinitialiser : `confirm()` → `localStorage.removeItem` + vidage DOM (`textContent = ""`) + recréation des `DEFAULT_TIERS`

---

## Bugs connus / règles à respecter

### elementFromPoint et l'image draguée
`dropTargetAt` repose sur le fait que l'image draguée a `pointer-events: none` (classe `.dragging`). Dans `endDrag`, **toujours calculer la cible AVANT de retirer la classe `.dragging`** — sinon `elementFromPoint` retourne l'image elle-même et le drop échoue.

### drag toujours réinitialisé
`drag = null` est posé **en tête de `endDrag`**, avant tout traitement — un `return` ou une exception au milieu ne doit jamais laisser un drag fantôme (sinon plus aucun `pointerdown` n'est accepté).

### setPointerCapture sous try/catch
Sur un tap très bref, le pointeur peut être déjà relâché quand `setPointerCapture` s'exécute → exception `InvalidStateError`. Le try/catch est obligatoire.

### touch-action: none — périmètre minimal
Uniquement sur `img`. L'étendre aux `.tier`/`.pool`/`body` casserait le scroll tactile de la page. C'est la combinaison `touch-action: none` (CSS) + Pointer Events qui permet le drag tactile sans `preventDefault` sur des listeners passifs.

### Pas d'innerHTML, jamais
Le contenu restauré depuis `localStorage` (noms de rangs, alt, src) passe par `textContent` / propriétés DOM. Introduire un `innerHTML` rouvrirait une XSS persistante via une sauvegarde forgée — et la CSP `script-src 'unsafe-inline'` ne bloquerait pas une injection inline.

### Validation à la restauration
Toute donnée lue depuis `localStorage` est non fiable : `validImgEntry` (préfixe `data:image/` obligatoire pour les `src`), regex couleur, `Array.isArray`, `String(t.n ?? "")`. **Conserver ces gardes** lors de toute évolution du format (et versionner la clé : `tierlist-v2`, etc. si le format devient incompatible).

### save() vs saveSoon()
Ne pas remplacer les `save()` immédiats par `saveSoon()` : sur mobile, l'OS peut tuer l'onglet à tout moment — une action ponctuelle (drop, suppression) doit être persistée immédiatement. Le debounce n'existe que pour les frappes clavier.

### Limites connues
- `localStorage` est limité à ~5 Mo selon les navigateurs : avec des images WebP 150×150 (~3-6 Ko chacune), ça laisse plusieurs centaines d'images — au-delà, l'alerte "Stockage plein" se déclenche
- Données sur un seul appareil/navigateur, pas de sync ni d'export (piste d'évolution : export/import JSON)
- Pas de PWA (pas de manifest ni service worker) — mais le fichier fonctionne hors-ligne une fois ouvert, puisqu'il ne charge aucune ressource externe
