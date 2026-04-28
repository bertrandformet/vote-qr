# Vote QR — Système de vote en temps réel

Outil de vote par QR code pour formateurs. Les participants scannent un QR code avec leur téléphone, leur vote s'enregistre instantanément dans Google Sheets, et les résultats s'affichent en temps réel sur le dashboard du formateur.

**Auteur :** Bertrand Formet — [Licence Creative Commons Attribution 4.0 (CC BY 4.0)](LICENSE)

---

## Utilisation

### Avant la session

1. Ouvrir **`dashboard.html`** dans un navigateur (fichier local, pas besoin de serveur).
2. Dans le champ **URL GitHub Pages**, saisir l'URL racine où `vote.html` est hébergé — par exemple `https://votre-nom.github.io/vote-qr` — puis cliquer **Générer QR codes**.  
   Les 4 QR codes apparaissent, chacun associé à une réponse.

### Pendant la session

3. Saisir l'**affirmation** à soumettre au groupe dans le champ texte.
4. Cliquer **▶ Lancer le vote**. Le compteur démarre, les résultats se rafraîchissent toutes les 3 secondes.
5. Projeter les QR codes :
   - **⛶ Plein écran** — ouvre une page sombre adaptée à la projection.
   - **🖨 Export A4** — ouvre une page claire prête à imprimer.
6. Les participants scannent le QR code correspondant à leur réponse. La page de confirmation (`vote.html`) s'affiche sur leur téléphone.
7. Cliquer **■ Arrêter** pour clore le vote. Les résultats restent affichés.

### Réponses disponibles

| QR | Réponse | Couleur |
|----|---------|---------|
| `?c=1` | Tout à fait d'accord | Vert |
| `?c=2` | Plutôt d'accord | Vert clair |
| `?c=3` | Plutôt pas d'accord | Orange |
| `?c=4` | Pas du tout d'accord | Rouge |

---

## Redéployer ailleurs

Le système repose sur trois éléments : **Google Sheets** (stockage), **Google Apps Script** (API), **GitHub Pages** (page de vote accessible aux téléphones).

### 1. Créer le Google Sheet

1. Nouveau Google Sheets, renommer la feuille **`Feuille 1`** (exact).
2. En ligne 2 : mettre `0` dans les cellules A2, B2, C2, D2. E2 reste vide.
3. Noter l'**ID du classeur** : dans l'URL du Sheet, c'est la longue chaîne entre `/d/` et `/edit` — `https://docs.google.com/spreadsheets/d/`**`1yynehP...BY3Lo`**`/edit`.

### 2. Créer et déployer l'Apps Script

Dans le Sheet : **Extensions → Apps Script**. Remplacer tout le contenu par :

```javascript
const SHEET_NAME = 'Feuille 1';

function doGet(e)  { return handleRequest(e); }
function doPost(e) { return handleRequest(e); }

function handleRequest(e) {
  const sheet    = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(SHEET_NAME);
  const action   = e.parameter.action;
  const lock     = LockService.getScriptLock();
  const acquired = lock.tryLock(5000);

  if (action === 'write') {
    const c   = parseInt(e.parameter.vote);  // paramètre 'vote', pas 'c'
    const col = ['A','B','C','D'][c - 1];
    const cell = sheet.getRange(col + '2');
    cell.setValue(cell.getValue() + 1);
    if (acquired) lock.releaseLock();
    return HtmlService.createHtmlOutput('ok');
  }

  let result = {};

  if (action === 'read') {
    result = {
      c1: sheet.getRange('A2').getValue(),
      c2: sheet.getRange('B2').getValue(),
      c3: sheet.getRange('C2').getValue(),
      c4: sheet.getRange('D2').getValue(),
      question: sheet.getRange('E2').getValue()
    };
  } else if (action === 'reset') {
    ['A2','B2','C2','D2'].forEach(r => sheet.getRange(r).setValue(0));
    sheet.getRange('E2').setValue(e.parameter.question);
    result = { ok: true };
  }

  if (acquired) lock.releaseLock();
  return ContentService
    .createTextOutput(JSON.stringify(result))
    .setMimeType(ContentService.MimeType.JSON);
}
```

**Déployer :**

1. **Déployer → Nouveau déploiement → Application Web**
2. Exécuter en tant que : **Moi**
3. Accès : **Tout le monde**
4. Copier l'URL de déploiement : c'est la longue chaîne entre `/s/` et `/exec` — `https://script.google.com/macros/s/`**`AKfycbz...`**`/exec`.
> À chaque modification du script, créer un **nouveau déploiement** (pas "Mettre à jour") pour que les changements prennent effet.

### 3. Héberger `vote.html` sur GitHub Pages

1. Créer un dépôt GitHub public (ex. `vote-qr`).
2. Y déposer `vote.html` à la racine.
3. **Settings → Pages → Branch: main → / (root) → Save**.
4. L'URL sera `https://votre-nom.github.io/vote-qr`.

> **Important :** les participants accèdent à cette URL depuis leur téléphone. Elle doit être publique et HTTPS.

### 4. Mettre à jour les URLs dans les deux fichiers

**Dans `vote.html`**, ligne `WEBHOOK_ECRITURE` :

```javascript
const WEBHOOK_ECRITURE = 'https://script.google.com/macros/s/VOTRE_ID/exec';
```

**Dans `dashboard.html`**, deux constantes à mettre à jour :

```javascript
// Ligne WEBHOOK — utilisée pour reset et pour la vérification de connexion
const WEBHOOK = 'https://script.google.com/macros/s/VOTRE_ID/exec';

// Ligne CSV_URL — lecture directe du Sheet (plus rapide que l'Apps Script)
const CSV_URL = 'https://docs.google.com/spreadsheets/d/VOTRE_SHEET_ID/gviz/tq?tqx=out:csv&range=A2:E2&sheet=Feuille%201';
```

Remplacer `VOTRE_ID` par l'URL Apps Script et `VOTRE_SHEET_ID` par l'ID du classeur Google Sheets.

---

## Architecture

```
Téléphone participant
  └─ scanne QR → charge vote.html?c=1..4 (GitHub Pages)
                  └─ fetch GET /exec?action=write&vote=N (Apps Script)
                              └─ incrémente cellule A2..D2 (Google Sheets)

Dashboard formateur (fichier local)
  └─ pollResults() toutes les 3s → lit CSV directement depuis Google Sheets (gviz/tq)
  └─ lancerVote()  → GET /exec?action=reset&question=... (Apps Script)
```

**Pourquoi le paramètre s'appelle `vote` et non `c` ?**  
L'infrastructure de Google filtrent les paramètres courts comme `c=2`, `c=3`, `c=4` (confondus avec des paramètres de tracking). Le renommage en `vote=` est indispensable pour que les choix 2, 3 et 4 fonctionnent.

**Pourquoi `fetch` avec `mode: 'no-cors'` dans `vote.html` ?**  
La requête vers Apps Script est cross-origin. Avec `no-cors`, le navigateur envoie la requête mais ne lit pas la réponse. C'est suffisant : on n'a besoin que de déclencher l'écriture, pas de lire le résultat.

**Pourquoi lire le Sheet via `gviz/tq` plutôt que via l'Apps Script ?**  
Le endpoint `gviz/tq` retourne un CSV public directement depuis Google Sheets, sans passer par Apps Script. C'est plus rapide et ne consomme pas de quota d'exécution.
