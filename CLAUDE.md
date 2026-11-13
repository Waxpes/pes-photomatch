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
- **Multi-device optionnel** : Firebase Realtime Database via REST API (fetch + EventSource pour le temps réel, pas de SDK)
- **QR Code** : `qrcode-generator@1.4.4` depuis jsDelivr CDN (global `qrcode()`)
- **Font** : Lato (Google Fonts), logo PES embarqué en base64

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
  photos:         [],     // [{id: webkitRelativePath, url: 'blob:…' | 'https://…' | 'fb:<clé>', location, w?, h?}]
  votes:          {},     // {[id]: {smash, pass, fave}}
  queue:          [],     // indices shufflés (Fisher-Yates)
  queuePos:       0,
  activeLocation: 'ALL',  // filtre lieu actif (vote)
  queueFilter:    'all',  // 'all' | 'unseen' | 'seen'
  sessionSmashes: 0, sessionPasses: 0, sessionFaves: 0,
  prevScreen:     'setup',
  adminSort:      'smash', // clé de colonne : name | loc | smash | fave | pass | total | rate
  adminDir:       'desc',
  adminChip:      'all',   // 'all' | 'smashed' | 'faved' | 'seen' | 'unseen'
  adminLocFilter: 'ALL',   // filtre lieu du tableau Résultats
  adminLoc:       null,    // null = grille des lieux, string = détail d'un lieu
};
```

### Admin (onglets)

#### 🏆 Résultats
- Stats : smashes, coups de cœur, passes, photos vues, total, taux %
- **Vue tableau uniquement** : `# | Photo | Nom | Lieu | 💚 | ⭐ | ❌ | Vus | %`
- **Filtrer** = chips (Toutes / Smashées / Coups de cœur / Vues / Pas vues, avec compteurs) + select de lieu
- **Trier** = clic sur un en-tête de colonne (2e clic inverse l'ordre). Pas de select de tri.
- **Lightbox** : clic sur une miniature → plein écran + télécharger + Échap
- Bouton reset discret en bas (taper `RESET`)

#### 📍 Par lieu
- Grille des lieux triée par smashes (miniature top photo, barre de progression)
- Clic → détail du lieu : même tableau et mêmes filtres que Résultats + bouton "▶ Voter sur ce lieu"

#### ⚙️ Multi-device (roue crantée à droite des onglets)
- Connexion Firebase (URL `firebaseio.com` ou `firebasedatabase.app`)
- URL EU du projet : `https://pes---vote-photos-pro-default-rtdb.europe-west1.firebasedatabase.app`
- **Partage des photos** : le poste qui a chargé le dossier envoie des copies JPEG (1080 px q0.72 + miniature 320 px q0.6) en data URL. Reprise possible (les photos déjà présentes sont ignorées). Les originaux restent sur le poste, rien n'est publié dans le repo.
- Les appareils sans photos locales utilisables chargent automatiquement le catalogue partagé
- Votes : écriture atomique (`{".sv":{"increment":1}}` en PATCH), lecture en temps réel via le streaming REST (EventSource), Firebase fait foi
- Lien partageable : `#fb-{base64url(firebaseURL)}` + QR code
- **⚠️ Limitation** : Firebase REST bloqué par la CSP de claude.ai → ne fonctionne que depuis GitHub Pages ou localhost

### Arborescence Firebase

```
/pes/votes/{clé}          {smash, pass, fave}
/pes/photos/meta/{clé}    {id, loc, w, h}   (écrit en dernier : une photo listée est complète)
/pes/photos/full/{clé}    data URL 1080 px
/pes/photos/thumb/{clé}   data URL 320 px
```

La clé est l'`id` (chemin relatif) avec `. # $ [ ] / %` encodés en `%XX` (`fbKey()`).

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

- **Couleurs Paris en Selle** (reprises de parisenselle.fr) :
    - `--g: #269C91` (turquoise PES primaire)
    - `--gd: #1B6E66` (turquoise foncé, texte/admin)
    - `--gs: #E8F5F3` (surface très claire)
    - `--gl: #A9D9D4` (bordures)
    - `--y: #FFB800` (coup de cœur/étoile)
    - `--r: #D63031` (pass/rouge)
    - `--bg: #F3F8F7` (fond page)
- Logo : logo officiel PES (badge rond turquoise), embarqué en base64 dans la classe `.pes-logo`
- Font : Lato

---

## Ce qui reste à faire / TODOs

1. **Firebase multi-device** : fonctionne depuis GitHub Pages (pas depuis l'artifact claude.ai)
2. **Chargement local** : blob URLs mortes après refresh → re-sélectionner le dossier (ou utiliser les photos partagées)
3. **Quotas Firebase Spark** : 1 Go stocké, 10 Go/mois téléchargés. Vérifier la taille réelle après le partage des ~2447 photos.
4. **Règles Firebase** : ouvertes pendant l'événement, à refermer après

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
    "pes": {
      ".read": true,
      ".write": true
    }
  }
}
```

