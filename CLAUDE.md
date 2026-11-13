# Paris en Selle — Photo Vote App · Contexte Claude Code

## Objectif du projet

Application web **Tinder-style** de vote sur des photos d'événement pour l'association Paris en Selle (association vélo parisienne). Permet à plusieurs personnes lors d'un séminaire de voter (smash/pass/coup de cœur) sur ~2447 photos prises par un photographe (Pierre M), organisées par lieu géographique.

---

## Déploiement

- **Déploiement cible** : GitHub Pages (Settings > Pages > Branch: main / root) → `https://waxpes.github.io/pes-photomatch`
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

**Mode votant (vue par défaut)** : sur le lien simple comme sur le QR code, l'app se connecte à la base de l'événement (`DEFAULT_FB_URL`, sauf lien `#fb-` ou déconnexion volontaire mémorisée `pes-fb-url = off`). L'écran d'accueil affiche seulement les explications de vote, les consignes et un bouton "Voter" (`startVoterSession()`, qui démarre sur les photos pas encore notées s'il en reste). Le lien Administration et le chargement de fichiers restent accessibles via le pied de page discret "Charger d'autres photos sur cet appareil".

**Mode organisateur** (photos locales ou aucune photo) : écran de chargement ci-dessous.

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

#### ⚙️ Réglages (roue crantée à droite des onglets, protégée par mot de passe)
- **Verrou** : `const ADMIN_PASS={salt,hash,iter}` dans `index.html` (empreinte PBKDF2-SHA256, 210 000 itérations). Seule l'empreinte est commitée. Hash vide = mode configuration : l'écran de verrou génère la ligne à coller. Déverrouillage mémorisé en `sessionStorage` (`pes-admin-ok`). Protection côté navigateur uniquement (contre les curieux, pas contre un attaquant).
- **Sauvegarde** : téléchargement JSON (lu à frais dans Firebase, sinon copie locale), restauration (remplace les votes, confirmation `RESTAURER`), sauvegarde automatique toutes les minutes dans un fichier local (File System Access API, Chrome/Edge desktop) qui se suspend si le nombre de votes baisse.
- Format de sauvegarde : `{app:'pes-photomatch', format:1, savedAt, source, votes:{[id]:{smash,pass,fave}}, ranking:[…]}`
- **Zone sensible** : reset des votes (confirmation `RESET`). Export et reset ne sont plus accessibles ailleurs.
- Connexion Firebase (URL `firebaseio.com` ou `firebasedatabase.app`)
- URL de la base de l'événement : constante `DEFAULT_FB_URL` dans `index.html`
- **Partage des photos** : le poste qui a chargé le dossier envoie des copies JPEG (900 px q0.68 + miniature 240 px q0.6) en data URL, avec leur taille (`bytes`) dans la meta. Reprise possible (les photos déjà présentes sont ignorées). Les originaux restent sur le poste, rien n'est publié dans le repo.
- **Garde-fou stockage** : après 20 photos, l'envoi projette le total à partir des tailles réelles et s'arrête au-delà de 900 Mo (`FB_SAFE_BYTES`, plan Spark = 1 Go). Jauge d'occupation dans le panneau.
- **Cache appareil** : les images Firebase sont mises en Cache Storage (`pes-photos-v1`, clé versionnée par `bytes`), donc téléchargées une seule fois par appareil.
- **Reset des votes** : supprime uniquement `/pes/votes`. **Supprimer les photos partagées** (confirmation `PHOTOS`) supprime uniquement `/pes/photos`.
- Les appareils sans photos locales utilisables chargent automatiquement le catalogue partagé
- Votes : écriture atomique (`{".sv":{"increment":1}}` en PATCH). Lecture : flux temps réel (EventSource) **uniquement quand l'écran admin est visible** ; ailleurs, lecture ponctuelle (`refreshVotesOnce()`) au démarrage d'une session de vote et au retour à l'accueil. Limite Spark : 100 connexions simultanées, un votant n'en garde aucune. Firebase fait foi.
- Lien partageable + QR code : l'URL simple pour la base par défaut ; `#fb-{base64url(firebaseURL)}` seulement pour une autre base (les anciens liens `#fb-` restent lus)
- Écritures en `?print=silent` (réponse 204) : Firebase compte les réponses dans le quota de téléchargement (10 Go/mois en Spark)

### Arborescence Firebase

```
/pes/votes/{clé}          {smash, pass, fave}
/pes/photos/meta/{clé}    {id, loc, w, h, bytes}   (écrit en dernier : une photo listée est complète)
/pes/photos/full/{clé}    data URL 900 px
/pes/photos/thumb/{clé}   data URL 240 px
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
- Icônes : sprite SVG Lucide (ISC) en tête de `<body>`, helper `ic(name, cls)`. Pas d'emoji dans l'interface.

---

## Ce qui reste à faire / TODOs

1. **Chargement local** : blob URLs mortes après refresh → re-sélectionner le dossier (ou utiliser les photos partagées)
2. **Quotas Firebase Spark** : 1 Go stocké, 10 Go/mois téléchargés. Vérifier la taille réelle après le partage des ~2447 photos.
3. **Règles Firebase** : ouvertes pendant l'événement, à refermer après

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

