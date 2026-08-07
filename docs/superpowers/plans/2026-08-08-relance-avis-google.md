# Relance d'avis Google — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construire `Desktop\avis-relance\index.html`, une page HTML statique unique permettant à un commerçant de configurer son commerce une fois, puis d'envoyer en un geste un SMS pré-rempli demandant un avis Google après chaque client.

**Architecture:** Un seul fichier `index.html` (HTML + CSS + JS inline), sur le modèle de `Desktop\tontine\index.html`. Deux écrans (configuration / usage quotidien) affichés/masqués via des classes CSS, aucun framework, aucune dépendance externe, toutes les données en `localStorage` du navigateur du commerçant.

**Tech Stack:** HTML5, CSS3, JavaScript vanilla (ES5-compatible, IIFE unique). Aucun backend, aucune API payante, aucun build step.

## Global Constraints

- Fichier unique `index.html`, pas de dépendances externes, pas de build (spec : "Architecture").
- Toutes les données (config + compteur) en `localStorage` uniquement — aucune donnée envoyée à un serveur (spec : "Architecture", "Détails techniques").
- Interface entièrement en français (spec : "Architecture").
- Pas de framework de test automatisé — vérification manuelle en navigateur, puis test réel sur téléphone (spec : "Détails techniques").
- Pas de dépôt git pour ce projet, cohérent avec Tontino (spec : "Notes pour la suite"). Les étapes "commit" habituelles sont donc remplacées par une simple confirmation de sauvegarde du fichier.
- Lien `sms:` : séparateur `&body=` sur iOS (détecté via `/iPhone|iPad|iPod/i.test(navigator.userAgent)`), `?body=` sinon (spec : "Détails techniques").

---

## Environnement de test

Ce projet n'a pas de serveur de développement configuré. Avant la Task 1, créer une configuration de preview locale pour pouvoir vérifier le fichier dans le navigateur pendant l'implémentation.

- [ ] **Étape : créer `.claude/launch.json`**

Créer le fichier `C:\Users\frank\Desktop\avis-relance\.claude\launch.json` :

```json
{
  "version": "0.0.1",
  "configurations": [
    {
      "name": "avis-relance",
      "runtimeExecutable": "python",
      "runtimeArgs": ["-m", "http.server", "8787"],
      "port": 8787
    }
  ]
}
```

Ce serveur sert simplement les fichiers statiques du dossier `avis-relance` sur `http://localhost:8787`. Il sera démarré via l'outil de preview (`preview_start` avec `{name: "avis-relance"}`) dans les tâches suivantes.

---

### Task 1: Squelette de la page + écran de configuration

**Files:**
- Create: `C:\Users\frank\Desktop\avis-relance\index.html`

**Interfaces:**
- Produces (exposées sur `window.AvisRelance` pour vérification en console) :
  - `buildDefaultTemplate(businessName: string) -> string`
  - `isValidUrl(value: string) -> boolean`
- Produces (fonctions internes utilisées par les tâches suivantes, non exposées) :
  - `loadConfig() -> {businessName, reviewLink, messageTemplate} | null`
  - `saveConfig(cfg: object) -> void`
  - `getCounter() -> number`
  - `setCounter(n: number) -> void`
  - `showSetup(existingCfg: object|null) -> void`
  - `showMain() -> void`
- Produces (éléments DOM avec ces `id` exacts, réutilisés par la Task 2) :
  - `#clientPrenom`, `#clientPhone`, `#phoneError`, `#sendBtn`, `#confirmMsg`, `#counterValue`

- [ ] **Step 1 : écrire le fichier `index.html` complet**

```html
<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
<title>Relance Avis — Demandez un avis Google en un geste</title>
<meta name="description" content="Envoyez une demande d'avis Google par SMS en un geste après chaque client.">
<style>
:root{
  --bg:#f7f8fa; --card:#ffffff; --ink:#1c2430; --muted:#6b7684;
  --brand:#1d6fd6; --brand-dark:#154f9c;
  --ok:#1d8f5e; --err:#c0392b; --line:#e4e8ee;
  --radius:16px; --shadow:0 2px 10px rgba(28,36,48,.07);
}
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:system-ui,-apple-system,"Segoe UI",sans-serif;background:var(--bg);color:var(--ink);min-height:100vh}
.app{max-width:480px;margin:0 auto;padding:16px 16px 40px}
header.top{display:flex;align-items:center;justify-content:space-between;padding:10px 0 18px}
.logo{font-size:20px;font-weight:800;color:var(--brand)}
h1{font-size:20px;margin-bottom:6px}
.sub{color:var(--muted);font-size:14px;margin-bottom:18px}
.card{background:var(--card);border:1px solid var(--line);border-radius:var(--radius);box-shadow:var(--shadow);padding:16px;margin-bottom:12px}
label{display:block;font-size:14px;font-weight:600;margin:14px 0 6px}
label:first-child{margin-top:0}
input,textarea{width:100%;padding:12px;border:1px solid var(--line);border-radius:10px;font-size:16px;background:#fff;font-family:inherit}
textarea{resize:vertical;min-height:90px}
.hint{font-size:12px;color:var(--muted);margin-top:4px}
.btn{display:block;width:100%;padding:14px;border:none;border-radius:12px;background:var(--brand);color:#fff;font-size:16px;font-weight:700;cursor:pointer;text-align:center;margin-top:16px}
.btn:active{background:var(--brand-dark)}
.btn.ghost{background:none;color:var(--muted);font-weight:600;padding:8px;margin-top:8px;width:auto}
.error{color:var(--err);font-size:13px;margin-top:6px;display:none}
.error.show{display:block}
.counter{background:linear-gradient(135deg,var(--brand),var(--brand-dark));color:#fff;border-radius:var(--radius);padding:18px;margin-bottom:14px;text-align:center}
.counter .n{font-size:32px;font-weight:800}
.counter .l{font-size:13px;opacity:.9}
.gear{background:none;border:none;font-size:20px;cursor:pointer;color:var(--muted)}
.hidden{display:none}
.confirm{color:var(--ok);font-size:14px;margin-top:10px;display:none;text-align:center;font-weight:600}
.confirm.show{display:block}
</style>
</head>
<body>
<div class="app">

  <div id="setupScreen" class="hidden">
    <header class="top"><div class="logo">Relance Avis</div></header>
    <h1>Configuration</h1>
    <p class="sub">À faire une seule fois. Ces informations restent sur cet appareil.</p>
    <div class="card">
      <label for="cfgBusinessName">Nom du commerce</label>
      <input id="cfgBusinessName" type="text" placeholder="Ex : Le Petit Bistrot">

      <label for="cfgReviewLink">Lien "laisser un avis" Google</label>
      <input id="cfgReviewLink" type="url" placeholder="https://g.page/r/....">
      <p class="hint">Trouvable dans Google Business Profile ou sur votre fiche Google Maps.</p>
      <p id="cfgLinkError" class="error">Merci de coller un lien valide (commençant par http:// ou https://).</p>

      <label for="cfgTemplate">Modèle de message</label>
      <textarea id="cfgTemplate"></textarea>
      <p class="hint">Utilisez {prenom} pour le prénom du client et {lien} pour le lien d'avis.</p>

      <button id="cfgSaveBtn" class="btn">Enregistrer</button>
      <button id="cfgCancelBtn" class="btn ghost hidden">Annuler</button>
    </div>
  </div>

  <div id="mainScreen" class="hidden">
    <header class="top">
      <div class="logo">Relance Avis</div>
      <button id="gearBtn" class="gear" aria-label="Réglages" title="Réglages">⚙️</button>
    </header>
    <div class="counter">
      <div class="n" id="counterValue">0</div>
      <div class="l">demandes envoyées</div>
    </div>
    <div class="card">
      <label for="clientPrenom">Prénom du client (optionnel)</label>
      <input id="clientPrenom" type="text" placeholder="Ex : Marie">

      <label for="clientPhone">Numéro de téléphone</label>
      <input id="clientPhone" type="tel" inputmode="tel" placeholder="06 12 34 56 78">
      <p id="phoneError" class="error">Merci de saisir un numéro de téléphone valide.</p>

      <button id="sendBtn" class="btn">Envoyer la demande d'avis</button>
      <p id="confirmMsg" class="confirm">SMS ouvert ✓</p>
    </div>
  </div>

</div>

<script>
(function () {
  'use strict';

  var CONFIG_KEY = 'avisRelance.config';
  var COUNTER_KEY = 'avisRelance.counter';

  var setupScreen = document.getElementById('setupScreen');
  var mainScreen = document.getElementById('mainScreen');

  var cfgBusinessName = document.getElementById('cfgBusinessName');
  var cfgReviewLink = document.getElementById('cfgReviewLink');
  var cfgTemplate = document.getElementById('cfgTemplate');
  var cfgLinkError = document.getElementById('cfgLinkError');
  var cfgSaveBtn = document.getElementById('cfgSaveBtn');
  var cfgCancelBtn = document.getElementById('cfgCancelBtn');

  var gearBtn = document.getElementById('gearBtn');
  var counterValue = document.getElementById('counterValue');
  var clientPrenom = document.getElementById('clientPrenom');
  var clientPhone = document.getElementById('clientPhone');
  var phoneError = document.getElementById('phoneError');
  var sendBtn = document.getElementById('sendBtn');
  var confirmMsg = document.getElementById('confirmMsg');

  var lastAutoTemplate = '';

  // ---- storage helpers ----

  function loadConfig() {
    try {
      return JSON.parse(localStorage.getItem(CONFIG_KEY) || 'null');
    } catch (e) {
      return null;
    }
  }

  function saveConfig(cfg) {
    localStorage.setItem(CONFIG_KEY, JSON.stringify(cfg));
  }

  function getCounter() {
    var n = parseInt(localStorage.getItem(COUNTER_KEY) || '0', 10);
    return isNaN(n) ? 0 : n;
  }

  function setCounter(n) {
    localStorage.setItem(COUNTER_KEY, String(n));
  }

  // ---- pure helpers (exposed for manual testing via console) ----

  function buildDefaultTemplate(businessName) {
    var biz = businessName && businessName.trim() ? businessName.trim() : 'nous';
    return 'Merci{prenom} pour votre visite chez ' + biz + ' ! Un petit avis Google nous aiderait beaucoup : {lien}';
  }

  function isValidUrl(value) {
    return /^https?:\/\/.+/i.test((value || '').trim());
  }

  window.AvisRelance = {
    buildDefaultTemplate: buildDefaultTemplate,
    isValidUrl: isValidUrl
  };

  // ---- screen switching ----

  function showSetup(existingCfg) {
    mainScreen.classList.add('hidden');
    setupScreen.classList.remove('hidden');

    var biz = existingCfg ? existingCfg.businessName : '';
    cfgBusinessName.value = biz || '';
    cfgReviewLink.value = existingCfg ? existingCfg.reviewLink : '';

    var tpl = existingCfg ? existingCfg.messageTemplate : buildDefaultTemplate(biz);
    cfgTemplate.value = tpl;
    lastAutoTemplate = existingCfg ? '' : tpl;

    cfgCancelBtn.classList.toggle('hidden', !existingCfg);
    cfgLinkError.classList.remove('show');
  }

  function showMain() {
    setupScreen.classList.add('hidden');
    mainScreen.classList.remove('hidden');
    counterValue.textContent = String(getCounter());
    clientPrenom.value = '';
    clientPhone.value = '';
    phoneError.classList.remove('show');
    confirmMsg.classList.remove('show');
  }

  // ---- config screen wiring ----

  cfgBusinessName.addEventListener('input', function () {
    if (cfgTemplate.value === lastAutoTemplate) {
      var next = buildDefaultTemplate(cfgBusinessName.value);
      cfgTemplate.value = next;
      lastAutoTemplate = next;
    }
  });

  cfgSaveBtn.addEventListener('click', function () {
    if (!isValidUrl(cfgReviewLink.value)) {
      cfgLinkError.classList.add('show');
      return;
    }
    cfgLinkError.classList.remove('show');

    saveConfig({
      businessName: cfgBusinessName.value.trim(),
      reviewLink: cfgReviewLink.value.trim(),
      messageTemplate: cfgTemplate.value
    });

    showMain();
  });

  cfgCancelBtn.addEventListener('click', function () {
    showMain();
  });

  gearBtn.addEventListener('click', function () {
    showSetup(loadConfig());
  });

  // ---- init ----

  var cfg = loadConfig();
  if (cfg) {
    showMain();
  } else {
    showSetup(null);
  }
})();
</script>
</body>
</html>
```

- [ ] **Step 2 : vérifier l'écran de configuration au premier lancement**

Démarrer la preview : appeler l'outil de preview avec `{name: "avis-relance"}`, puis naviguer vers `http://localhost:8787`.

Avant de vérifier, effacer tout état précédent : `javascript_tool` → `localStorage.clear(); location.reload();`

Ensuite :
1. `read_page` → confirmer que `#setupScreen` est visible et `#mainScreen` a la classe `hidden`.
2. `javascript_tool` → `document.getElementById('cfgTemplate').value` doit être exactement `"Merci{prenom} pour votre visite chez nous ! Un petit avis Google nous aiderait beaucoup : {lien}"` (business name vide → "nous").
3. `computer` → taper `Le Petit Bistrot` dans le champ Nom du commerce (`#cfgBusinessName`).
4. `javascript_tool` → `document.getElementById('cfgTemplate').value` doit maintenant contenir `"chez Le Petit Bistrot"`.
5. `computer` → cliquer sur "Enregistrer" sans avoir rempli le lien Google Avis.
6. `read_page` → `#cfgLinkError` doit avoir la classe `show`, et l'écran de configuration doit rester affiché (pas de bascule vers l'écran principal).
7. `computer` → taper `https://g.page/r/test123` dans `#cfgReviewLink`, cliquer à nouveau sur "Enregistrer".
8. `read_page` → `#mainScreen` doit être visible, `#setupScreen` doit avoir la classe `hidden`, `#counterValue` doit afficher `0`.
9. `javascript_tool` → `JSON.parse(localStorage.getItem('avisRelance.config'))` doit renvoyer `{businessName: "Le Petit Bistrot", reviewLink: "https://g.page/r/test123", messageTemplate: "..."}` avec le template contenant `Le Petit Bistrot`.

Expected : toutes les vérifications ci-dessus passent. Si une échoue, corriger le code avant de continuer.

- [ ] **Step 3 : vérifier le rechargement et la réouverture via l'icône réglages**

1. `javascript_tool` → `location.reload()`.
2. `read_page` → `#mainScreen` doit s'afficher directement (pas l'écran de configuration), car une config existe déjà en `localStorage`.
3. `computer` → cliquer sur l'icône réglages (`#gearBtn`).
4. `read_page` → `#setupScreen` doit s'afficher avec `#cfgBusinessName` pré-rempli à `Le Petit Bistrot`, `#cfgReviewLink` pré-rempli à `https://g.page/r/test123`, et le bouton `#cfgCancelBtn` visible (pas de classe `hidden`).
5. `computer` → cliquer sur "Annuler" (`#cfgCancelBtn`).
6. `read_page` → doit revenir sur `#mainScreen` sans avoir modifié la config (`javascript_tool` → relire `localStorage.getItem('avisRelance.config')` et confirmer qu'il est inchangé).

Expected : toutes les vérifications ci-dessus passent.

- [ ] **Step 4 : sauvegarde**

Fichier créé et vérifié. Pas de dépôt git pour ce projet — aucune étape de commit nécessaire.

---

### Task 2: Écran principal — envoi de la demande d'avis

**Files:**
- Modify: `C:\Users\frank\Desktop\avis-relance\index.html`

**Interfaces:**
- Consumes (de la Task 1) : `loadConfig()`, `getCounter()`, `setCounter(n)`, éléments DOM `#clientPrenom`, `#clientPhone`, `#phoneError`, `#sendBtn`, `#confirmMsg`, `#counterValue`.
- Produces (exposées sur `window.AvisRelance`, en complément de celles de la Task 1) :
  - `fillTemplate(template: string, data: {prenom: string, lien: string}) -> string`
  - `cleanPhone(raw: string) -> string`
  - `isValidPhone(cleaned: string) -> boolean`
  - `buildSmsHref(number: string, message: string) -> string`

- [ ] **Step 1 : ajouter les fonctions pures d'envoi**

Dans `index.html`, insérer ces quatre fonctions juste après la fonction `isValidUrl` existante (avant le bloc `window.AvisRelance = {...}`) :

Old string à localiser :
```js
  function isValidUrl(value) {
    return /^https?:\/\/.+/i.test((value || '').trim());
  }

  window.AvisRelance = {
    buildDefaultTemplate: buildDefaultTemplate,
    isValidUrl: isValidUrl
  };
```

Remplacer par :
```js
  function isValidUrl(value) {
    return /^https?:\/\/.+/i.test((value || '').trim());
  }

  function fillTemplate(template, data) {
    var prenom = data && data.prenom ? String(data.prenom).trim() : '';
    var lien = (data && data.lien) || '';
    var prenomPart = prenom ? ' ' + prenom : '';
    return template.split('{prenom}').join(prenomPart).split('{lien}').join(lien);
  }

  function cleanPhone(raw) {
    raw = raw || '';
    return raw.replace(/[^\d+]/g, '');
  }

  function isValidPhone(cleaned) {
    return /^(\+33[1-9]\d{8}|0[1-9]\d{8})$/.test(cleaned);
  }

  function buildSmsHref(number, message) {
    var isIOS = /iPhone|iPad|iPod/i.test(navigator.userAgent);
    var sep = isIOS ? '&' : '?';
    return 'sms:' + number + sep + 'body=' + encodeURIComponent(message);
  }

  window.AvisRelance = {
    buildDefaultTemplate: buildDefaultTemplate,
    isValidUrl: isValidUrl,
    fillTemplate: fillTemplate,
    cleanPhone: cleanPhone,
    isValidPhone: isValidPhone,
    buildSmsHref: buildSmsHref
  };
```

- [ ] **Step 2 : vérifier les fonctions pures en console**

Recharger la page (`javascript_tool` → `location.reload()`), puis exécuter via `javascript_tool` (chaque ligne doit renvoyer `true`) :

```js
AvisRelance.fillTemplate('Merci{prenom} de venir : {lien}', {prenom: 'Marie', lien: 'http://x'}) === 'Merci Marie de venir : http://x'
```
```js
AvisRelance.fillTemplate('Merci{prenom} de venir : {lien}', {prenom: '', lien: 'http://x'}) === 'Merci de venir : http://x'
```
```js
AvisRelance.cleanPhone('06 12 34 56 78') === '0612345678'
```
```js
AvisRelance.isValidPhone('0612345678') === true
```
```js
AvisRelance.isValidPhone('123') === false
```
```js
AvisRelance.isValidPhone('+33612345678') === true
```
```js
AvisRelance.buildSmsHref('0612345678', 'salut').indexOf('sms:0612345678') === 0
```

Expected : les 7 expressions renvoient `true`. Si l'une échoue, corriger la fonction correspondante avant de continuer.

- [ ] **Step 3 : câbler le bouton d'envoi**

Insérer le gestionnaire du bouton d'envoi juste après le bloc `gearBtn.addEventListener(...)` existant, avant le commentaire `// ---- init ----`.

Old string à localiser :
```js
  gearBtn.addEventListener('click', function () {
    showSetup(loadConfig());
  });

  // ---- init ----
```

Remplacer par :
```js
  gearBtn.addEventListener('click', function () {
    showSetup(loadConfig());
  });

  // ---- main screen wiring ----

  sendBtn.addEventListener('click', function () {
    var cleaned = cleanPhone(clientPhone.value);

    if (!isValidPhone(cleaned)) {
      phoneError.classList.add('show');
      confirmMsg.classList.remove('show');
      return;
    }
    phoneError.classList.remove('show');

    var cfg = loadConfig();
    var message = fillTemplate(cfg.messageTemplate, {
      prenom: clientPrenom.value,
      lien: cfg.reviewLink
    });

    var href = buildSmsHref(cleaned, message);

    var n = getCounter() + 1;
    setCounter(n);
    counterValue.textContent = String(n);

    confirmMsg.classList.add('show');
    clientPrenom.value = '';
    clientPhone.value = '';

    window.location.href = href;
  });

  // ---- init ----
```

- [ ] **Step 4 : vérifier le flux d'envoi complet dans le navigateur**

Une config valide doit déjà exister depuis la Task 1 (sinon, la recréer via l'écran de configuration).

1. `read_page` → confirmer `#counterValue` affiche `0`.
2. `computer` → cliquer sur "Envoyer la demande d'avis" sans avoir rempli le téléphone.
3. `read_page` → `#phoneError` doit avoir la classe `show`, `#counterValue` doit toujours afficher `0` (aucun envoi comptabilisé).
4. `computer` → taper `Marie` dans `#clientPrenom`, taper `06 12 34 56 78` dans `#clientPhone`, cliquer sur "Envoyer la demande d'avis".
5. `read_page` → `#counterValue` doit maintenant afficher `1`, `#clientPrenom` et `#clientPhone` doivent être vides à nouveau.
6. `javascript_tool` → `localStorage.getItem('avisRelance.counter')` doit renvoyer `"1"`.
7. `read_console_messages` → un message lié à la navigation vers `sms:` (type "not allowed to navigate" ou absence de handler) est normal et attendu dans un navigateur desktop ; ce n'est pas une erreur à corriger. Vérifier qu'aucune autre erreur JavaScript n'apparaît.

Expected : toutes les vérifications ci-dessus passent, sans erreur JS inattendue dans la console.

- [ ] **Step 5 : vérifier l'affichage mobile**

`resize_window` → `{preset: "mobile"}`, puis `javascript_tool` → `location.reload()`.

`computer` → `{action: "screenshot"}` pour confirmer visuellement que la carte tient dans l'écran sans débordement horizontal, que les boutons sont assez grands pour être touchés, et que le compteur est lisible.

Expected : aucune barre de défilement horizontale, mise en page cohérente avec le design mobile-first prévu.

- [ ] **Step 6 : sauvegarde**

Fichier modifié et vérifié. Pas de dépôt git pour ce projet — aucune étape de commit nécessaire.

---

### Task 3: Finitions — documentation et checklist de test réel

**Files:**
- Modify: `C:\Users\frank\Desktop\outils-pme\02-avis-google.md`

**Interfaces:**
- Aucune (tâche documentaire, ne modifie pas `index.html`).

- [ ] **Step 1 : distinguer les deux volets dans la fiche existante**

Dans `Desktop\outils-pme\02-avis-google.md`, insérer une note après la ligne `**Statut** : idée brute — non validée terrain` (ligne 3) pour distinguer le volet "surveillance/réponse aux avis existants" (toujours à l'état idée) du volet "relance proactive post-client" (maintenant construit).

Old string à localiser :
```
**Statut** : idée brute — non validée terrain
```

Remplacer par :
```
**Statut** : idée brute — non validée terrain (volet surveillance/réponse aux avis existants, décrit ci-dessous)

**Note** : le volet "relance proactive post-client" (SMS pré-rempli demandant un avis juste après le passage d'un client) a été développé séparément — voir `Desktop\avis-relance\`. Ce document ne couvre donc plus que la partie surveillance + réponse aux avis déjà publiés.
```

- [ ] **Step 2 : vérifier la modification**

Relire le fichier modifié et confirmer que la note apparaît bien juste après la ligne de statut, sans avoir modifié le reste du contenu de la fiche (sections "Le problème", "Pourquoi ça fait mal", etc. inchangées).

- [ ] **Step 3 : sauvegarde**

Fichier modifié. Pas de dépôt git pour ce projet — aucune étape de commit nécessaire.

---

## Checklist manuelle post-implémentation (à faire par vous, non automatisable)

Ce point ne peut pas être vérifié par un agent — il nécessite un vrai téléphone :

1. Ouvrir `index.html` (ou l'URL hébergée) depuis un téléphone (iPhone et/ou Android).
2. Configurer avec le vrai lien "laisser un avis" Google du commerce visé.
3. Saisir un numéro de téléphone réel (le vôtre, pour test) et cliquer sur "Envoyer la demande d'avis".
4. Confirmer que l'appli SMS native s'ouvre avec le numéro et le message déjà remplis, sur iOS **et** sur Android si possible (la syntaxe du lien `sms:` diffère entre les deux).
5. Confirmer que le lien Google Avis contenu dans le message ouvre bien la bonne page une fois cliqué depuis l'app SMS.
