# Paris en Selle — Photo Vote App · Contexte Claude Code

## Objectif du projet

Application web **Tinder-style** de vote sur des photos d'événement pour l'association Paris en Selle (association vélo parisienne). Permet à plusieurs personnes lors d'un séminaire de voter (smash/pass/coup de cœur) sur ~2447 photos prises par un photographe (Pierre M), organisées par lieu géographique.

---

## Artifact publié

- **URL artifact Claude** : https://claude.ai/artifact/M2C4r7W19s5Wx5aNCb26Mm
- **Repo GitHub** : git@github-pes:Waxpes/pes-photomatch.git
- **Déploiement cible** : GitHub Pages → `https://waxpes.github.io/pes-photomatch`
- **Fichier** : `index.html` (single-file app, ~70KB, zéro dépendance npm)

---

## Stack technique

- **Pure HTML/CSS/JS** — aucun framework, aucun build, aucun npm
- **Stockage** : `localStorage` (votes persistés par nom de fichier)
- **Multi-device optionnel** : Firebase Realtime Database via REST API (fetch, pas de SDK)
- **QR Code** : `qrcode@1.5.3` depuis jsDelivr CDN

---

## Structure des photos

Le photographe a livré un dossier zip organisé ainsi :

```
PES_Photos_PierreM/
  ALMA/
  BONUS-CARREFOUR_JAURES/
  BONUS-CARREFOUR_ORNANOxORDERNER/
  BONUS-COLONELFABIEN/
  BONUS-MOULINROUGE/
  BONUS-PLACE_CLICHY/
  BONUS-PLACE-BASTILLE/
  BONUS-PONTAUCHANGE/
  BONUS-PONTDELATOURNELLE/
  BONUS-RUEDEMEAUX/
  BONUS-SAINTMICHEL-NOTREDAME/
  BONUS-SOUS_METRO_AERIEN_LIGNE_2_JAURES_STALINGRAD/
  BONUS-STALINGRAD/
  BONUS-VELORUE_CHARENTON/
  BONUS-VELORUEAMSTERDAM/
  BONUS-VRAC/
  BVD_HAUSMANN/
  BVD_SAINT_GERMAIN/
  BVD_STMICHEL/
  RIVOLI-SEBASTOPOL/
  RUE_LA_FAYETTE/
  VELORUE_CANAL_ST_MARTIN/
  VELORUE_RUE_ST_JACQUES/
```

Chaque dossier contient des sous-dossiers (ex: `WEB_POSTPROS/`) avec les images JPG.

**Point clé** : la détection du lieu se fait avec `webkitRelativePath` → `parts[1]` (le 2ème segment, pas le dernier).

---

## Fonctionnalités implémentées

### Vote (écran principal)

- **Swipe gauche** = Pass ❌ · **Swipe droite** = Smash 💚 · **Bouton ⭐** = Coup de cœur
- Keyboard : `←` `→` pour voter, `↑` ou `Espace` coup de cœur, `R` reshuffle, `T` terminer session, `A` admin
- **Pile de 3 cartes** (depth effect), carte active draggable (mouse + touch)
- **Image** : `object-fit: contain`, fond transparent, card adaptative selon orientation (landscape → 500×340px, portrait → 320×480px)
- **Barre de contrôles** au-dessus des cartes :
    - Bouton lieu actif `📍 Tout aléatoire (2447) ▾` → ouvre bottom sheet de sélection
    - Chips queue : `🎲 Toutes` / `🆕 Pas encore vues` / `🔄 Déjà vues`
- Compteur session : smashes, passes, coups de cœur séparément
- Bouton `⏹ Fin` pour terminer sa session proprement (stats affichées)

### Chargement des photos (écran Setup)

```javascript
// Chargement depuis dossier local (recommandé — détecte les lieux)
<input type="file" webkitdirectory accept="image/*">
// → file.webkitRelativePath = "PES_Photos_PierreM/ALMA/WEB_POSTPROS/IMG_001.jpg"
// → location = parts[1] = "ALMA"

// Chargement fichiers individuels (sans lieux)
<input type="file" multiple accept="image/*">

// Chargement URLs (paste ou JSON array)
["https://url1.jpg", "https://url2.jpg"]
```

**⚠️ Problème blob URLs** : `URL.createObjectURL()` ne survit pas au rechargement de page. L'app le détecte et propose de re-sélectionner le dossier (les votes sont préservés car ID = `webkitRelativePath`).

### État (State)

```javascript
let S = {
  photos:         [],     // [{id: webkitRelativePath, url: blob:..., location: 'ALMA'}]
  votes:          {},     // {[id]: {smash: number, pass: number, fave: number}}
  queue:          [],     // indices shufflés (Fisher-Yates)
  queuePos:       0,
  activeLocation: 'ALL', // filtre lieu actif
  queueFilter:    'all', // 'all' | 'unseen' | 'seen'
  sessionSmashes: 0,
  sessionPasses:  0,
  sessionFaves:   0,
  prevScreen:     'setup',
  adminView:      'grid',    // 'grid' | 'list' | 'listphoto' | 'table'
  adminSort:      'smash',   // 'smash' | 'fave' | 'seen' | 'unseen' | 'alpha'
  adminDir:       'desc',    // 'asc' | 'desc'
  adminChip:      'all',     // 'all' | 'smashed' | 'faved' | 'unseen'
  adminLoc:       null,      // null = vue lieux, string = détail d'un lieu
  locView:        'grid',
  locSort:        'smash',
  locDir:         'desc',
};
```

### Admin (onglets)

#### 🏆 Résultats
- Stats : smashes, coups de cœur, passes, photos vues, total, taux %
- 4 vues : ⊞ Grille / ☰ Liste / 🖼 Liste+photo / ▤ Tableau
- Tableau avec colonnes : `# | Photo | Nom | Lieu | 💚 | ⭐ | ❌ | Vus | %`
- Tri cliquable sur chaque colonne + bouton ↑/↓ asc/desc
- Filtres : Toutes / 💚 Smashées / ⭐ Coups de cœur / 🆕 Pas vues
- **Lightbox** : clic sur une image → plein écran + bouton télécharger + fermeture Échap
- Bouton reset discret en bas (demande de taper `RESET` pour confirmer)

#### 📍 Par lieu
- Grille des lieux triée par smashes avec miniature top photo, barre de progression
- Clic → vue détail du lieu avec les mêmes contrôles (vue/tri/direction)
- Clic sur une carte de lieu → lance directement le vote sur ce lieu

#### 🔥 Multi-device (Firebase)
- Firebase Realtime Database via REST API (fetch)
- **Validation URL** : accepte `firebaseio.com` ET `firebasedatabase.app` (région EU)
- Polling toutes les 5 secondes
- **⚠️ Limitation** : Firebase REST bloqué par la CSP de claude.ai → ne fonctionne que depuis GitHub Pages ou localhost
- Lien partageable avec Firebase encodé dans le hash : `#fb-{base64url(firebaseURL)}`
- QR code généré automatiquement (lib qrcode.js)

---

## Hash routing

```
#admin          → ouvre le panneau admin
#fb-{base64}    → auto-connecte Firebase (lien partageable multi-device)
```

---

## localStorage

```javascript
const SK  = 'pes-vote-v4';   // {photos, votes, savedAt}
const SFK = 'pes-fb-url';    // URL Firebase
```

---

## Design / Branding

- **Couleurs Paris en Selle** :
    - `--g: #59A829` (vert PES primaire)
    - `--gd: #2B6010` (vert foncé, texte/admin)
    - `--gs: #EEF8E6` (surface verte très claire)
    - `--gl: #B8DFA0` (bordures vertes)
    - `--y: #FFB800` (coup de cœur/étoile)
    - `--r: #D63031` (pass/rouge)
    - `--bg: #F5F8F2` (fond page)
- Logo : SVG vélo maison (cycliste + bicyclette)
- Font : system-ui

---

## Ce qui reste à faire / TODOs

1. **Firebase multi-device** : fonctionne depuis GitHub Pages (pas depuis l'artifact claude.ai)
2. **Chargement photos** : blob URLs mortes après refresh → toujours re-sélectionner le dossier
3. Éventuel : héberger les photos sur un CDN pour éviter le re-select

---

## Setup GitHub (SSH multi-comptes)

```
# ~/.ssh/config
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519        # compte pro

Host github-pes
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_rsa_waxpes     # compte PES (Waxpes)
```

```bash
# Cloner avec le bon compte
git clone git@github-pes:Waxpes/pes-photomatch.git

# Uploader le HTML
cp index.html pes-photomatch/
cd pes-photomatch
git add index.html
git commit -m "feat: Paris en Selle photo vote app"
git push

# Activer GitHub Pages : Settings → Pages → Branch: main / root
# URL résultante : https://waxpes.github.io/pes-photomatch
```

---

## Firebase Realtime Database Rules (permissif pour l'événement)

```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```

