# Personnalisation V2 (QR code, historique, modèles multiples, logo/couleur) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Étendre `index.html` (app "Relance Avis") avec quatre fonctionnalités : modèles de message multiples, logo et couleur de marque personnalisables, historique des envois avec détection de doublon, et un écran QR code pointant vers le lien d'avis Google.

**Architecture:** Toujours un seul fichier `index.html` livré (HTML + CSS + JS inline), aucun backend, aucun build. Une bibliothèque tierce (génération de QR code) est vendorisée — copiée intégralement dans `index.html` au moment de l'implémentation — pas de `<script src>` externe dans le fichier final. Toutes les données (config, compteur, historique) restent en `localStorage`.

**Tech Stack:** HTML5, CSS3, JavaScript vanilla (ES5-compatible, IIFE unique, cohérent avec la V1). Bibliothèque vendorisée : `qrcode-generator` (Kazuhiko Arase, licence MIT).

## Global Constraints

- Fichier unique livré `index.html` — la bibliothèque QR est copiée dedans, pas chargée séparément (spec : "Architecture").
- Toutes les données restent en `localStorage`, aucune transmission serveur (spec : "Architecture").
- Interface entièrement en français (cohérent avec la V1).
- Pas de framework de test automatisé — vérification manuelle via l'outil de preview navigateur (spec : "Détails techniques").
- Migration automatique et silencieuse d'une config V1 (`messageTemplate` string) vers le format V2 (`templates` array) à la lecture, sans action utilisateur (spec : "Architecture").
- Un dépôt git existe pour ce projet — commit normal à la fin de chaque tâche.
- JavaScript ES5 (pas de `let`/`const`/arrow functions/template literals/`padStart`), cohérent avec le style existant du fichier.

---

## Environnement de test

Le serveur de preview existe déjà (`.claude/launch.json`, port 8787). Avant de commencer, s'assurer qu'il tourne :

```bash
cd "C:\Users\frank\Desktop\relance avis google nouvelle version"
python -m http.server 8787 &
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8787/index.html
```

Expected : `200`. Naviguer ensuite vers `http://localhost:8787/index.html` avec l'outil navigateur pour les vérifications de chaque tâche. Avant chaque tâche, effacer l'état précédent si besoin (`javascript_tool` → `localStorage.clear(); location.reload();`).

---

### Task 1: Modèles de message multiples

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes : `buildDefaultTemplate`, `isValidUrl`, `loadConfig`/`saveConfig` existants, éléments DOM `#cfgBusinessName`, `#cfgReviewLink`, `#cfgLinkError`, `#cfgSaveBtn`, `#cfgCancelBtn`, `#clientPrenom`, `#clientPhone`, `#sendBtn`, `#counterValue`.
- Produces (exposées sur `window.AvisRelance`, en plus de l'existant) : `migrateConfig(raw: object|null) -> object|null`.
- Produces (schéma de config, utilisé par les tâches suivantes) : `loadConfig()` renvoie désormais `{businessName, reviewLink, templates: [{name, text}], defaultTemplateName, brandColor, logoDataUrl}` (au lieu de `messageTemplate: string`).
- Produces (éléments DOM avec ces `id`, réutilisés par les tâches suivantes) : `#templateSelect`, `#tplSelectWrap`.

- [ ] **Step 1 : ajouter la migration de config et le nouveau schéma**

Localiser dans `index.html` :
```js
  var CONFIG_KEY = 'avisRelance.config';
  var COUNTER_KEY = 'avisRelance.counter';
```
Remplacer par :
```js
  var CONFIG_KEY = 'avisRelance.config';
  var COUNTER_KEY = 'avisRelance.counter';
  var LAST_TEMPLATE_KEY = 'avisRelance.lastTemplate';
  var DEFAULT_TEMPLATE_NAME = 'Par défaut';
```

Localiser :
```js
  function loadConfig() {
    try {
      return JSON.parse(localStorage.getItem(CONFIG_KEY) || 'null');
    } catch (e) {
      return null;
    }
  }
```
Remplacer par :
```js
  function migrateConfig(raw) {
    if (!raw) return raw;
    if (raw.templates) return raw;
    var text = typeof raw.messageTemplate === 'string' ? raw.messageTemplate : buildDefaultTemplate(raw.businessName);
    return {
      businessName: raw.businessName || '',
      reviewLink: raw.reviewLink || '',
      templates: [{ name: DEFAULT_TEMPLATE_NAME, text: text }],
      defaultTemplateName: DEFAULT_TEMPLATE_NAME,
      brandColor: raw.brandColor || '#1d6fd6',
      logoDataUrl: raw.logoDataUrl || null
    };
  }

  function loadConfig() {
    try {
      var raw = JSON.parse(localStorage.getItem(CONFIG_KEY) || 'null');
      return migrateConfig(raw);
    } catch (e) {
      return null;
    }
  }
```

Localiser :
```js
  window.AvisRelance = {
    buildDefaultTemplate: buildDefaultTemplate,
    isValidUrl: isValidUrl,
    fillTemplate: fillTemplate,
    cleanPhone: cleanPhone,
    isValidPhone: isValidPhone,
    buildSmsHref: buildSmsHref
  };
```
Remplacer par :
```js
  window.AvisRelance = {
    buildDefaultTemplate: buildDefaultTemplate,
    isValidUrl: isValidUrl,
    fillTemplate: fillTemplate,
    cleanPhone: cleanPhone,
    isValidPhone: isValidPhone,
    buildSmsHref: buildSmsHref,
    migrateConfig: migrateConfig
  };
```

- [ ] **Step 2 : vérifier la migration en console**

`javascript_tool` → recharger si besoin (`location.reload()`), puis exécuter (doit renvoyer `true`) :
```js
JSON.stringify(AvisRelance.migrateConfig({businessName:'Test', reviewLink:'https://x.co', messageTemplate:'Merci{prenom} : {lien}'})) === JSON.stringify({businessName:'Test', reviewLink:'https://x.co', templates:[{name:'Par défaut', text:'Merci{prenom} : {lien}'}], defaultTemplateName:'Par défaut', brandColor:'#1d6fd6', logoDataUrl:null})
```
```js
AvisRelance.migrateConfig(null) === null
```
Expected : les deux renvoient `true`.

- [ ] **Step 3 : ajouter le style des lignes de modèle et du `<select>`**

Localiser :
```css
input,textarea{width:100%;padding:12px;border:1px solid var(--line);border-radius:10px;font-size:16px;background:#fff;font-family:inherit}
```
Remplacer par :
```css
input,textarea,select{width:100%;padding:12px;border:1px solid var(--line);border-radius:10px;font-size:16px;background:#fff;font-family:inherit}
.tpl-row{border:1px solid var(--line);border-radius:12px;padding:10px;margin-bottom:10px}
.tpl-badge{font-size:11px;font-weight:700;color:var(--brand);text-transform:uppercase;letter-spacing:.03em;margin-bottom:6px}
.tpl-row .tpl-name{font-weight:700;margin-bottom:6px}
.tpl-row textarea{min-height:70px}
.tpl-del{background:none;border:none;color:var(--err);font-size:12px;font-weight:600;cursor:pointer;padding:6px 0;margin-top:4px}
.tpl-del:disabled{opacity:.4;cursor:not-allowed}
```

- [ ] **Step 4 : remplacer le champ modèle unique par la gestion de plusieurs modèles**

Localiser :
```html
      <label for="cfgTemplate">Modèle de message</label>
      <textarea id="cfgTemplate"></textarea>
      <p class="hint">Utilisez {prenom} pour le prénom du client et {lien} pour le lien d'avis.</p>
```
Remplacer par :
```html
      <label>Modèles de message</label>
      <div id="tplList"></div>
      <button type="button" id="tplAddBtn" class="btn ghost">+ Ajouter un modèle</button>
      <p class="hint">Utilisez {prenom} pour le prénom du client et {lien} pour le lien d'avis. Le premier modèle de la liste est proposé par défaut à l'envoi.</p>
```

- [ ] **Step 5 : ajouter le sélecteur de modèle sur l'écran principal**

Localiser :
```html
    <div class="card">
      <label for="clientPrenom">Prénom du client (optionnel)</label>
```
Remplacer par :
```html
    <div class="card">
      <div id="tplSelectWrap" class="hidden">
        <label for="templateSelect">Modèle de message</label>
        <select id="templateSelect"></select>
      </div>

      <label for="clientPrenom">Prénom du client (optionnel)</label>
```

- [ ] **Step 6 : câbler la gestion des modèles dans le JS**

Localiser :
```js
  var cfgBusinessName = document.getElementById('cfgBusinessName');
  var cfgReviewLink = document.getElementById('cfgReviewLink');
  var cfgTemplate = document.getElementById('cfgTemplate');
  var cfgLinkError = document.getElementById('cfgLinkError');
```
Remplacer par :
```js
  var cfgBusinessName = document.getElementById('cfgBusinessName');
  var cfgReviewLink = document.getElementById('cfgReviewLink');
  var tplList = document.getElementById('tplList');
  var tplAddBtn = document.getElementById('tplAddBtn');
  var cfgLinkError = document.getElementById('cfgLinkError');
```

Localiser :
```js
  var gearBtn = document.getElementById('gearBtn');
  var counterValue = document.getElementById('counterValue');
```
Remplacer par :
```js
  var gearBtn = document.getElementById('gearBtn');
  var templateSelect = document.getElementById('templateSelect');
  var tplSelectWrap = document.getElementById('tplSelectWrap');
  var counterValue = document.getElementById('counterValue');
```

Localiser :
```js
  var lastAutoTemplate = '';
```
Remplacer par :
```js
  var workingTemplates = [];
```

- [ ] **Step 7 : ajouter les fonctions de rendu et de gestion des modèles**

Localiser :
```js
  // ---- screen switching ----
```
Remplacer par :
```js
  // ---- template list management ----

  function renderTemplateList() {
    tplList.innerHTML = '';
    workingTemplates.forEach(function (tpl, i) {
      var row = document.createElement('div');
      row.className = 'tpl-row';

      if (i === 0) {
        var badge = document.createElement('div');
        badge.className = 'tpl-badge';
        badge.textContent = 'Modèle par défaut';
        row.appendChild(badge);
      }

      var nameInput = document.createElement('input');
      nameInput.type = 'text';
      nameInput.value = tpl.name;
      nameInput.className = 'tpl-name';
      nameInput.addEventListener('input', function () {
        workingTemplates[i].name = nameInput.value;
      });

      var textArea = document.createElement('textarea');
      textArea.value = tpl.text;
      textArea.addEventListener('input', function () {
        workingTemplates[i].text = textArea.value;
      });

      var delBtn = document.createElement('button');
      delBtn.type = 'button';
      delBtn.className = 'tpl-del';
      delBtn.textContent = 'Supprimer ce modèle';
      delBtn.disabled = workingTemplates.length <= 1;
      delBtn.addEventListener('click', function () {
        if (workingTemplates.length <= 1) return;
        workingTemplates.splice(i, 1);
        renderTemplateList();
      });

      row.appendChild(nameInput);
      row.appendChild(textArea);
      row.appendChild(delBtn);
      tplList.appendChild(row);
    });
  }

  tplAddBtn.addEventListener('click', function () {
    workingTemplates.push({
      name: 'Modèle ' + (workingTemplates.length + 1),
      text: buildDefaultTemplate(cfgBusinessName.value)
    });
    renderTemplateList();
  });

  function populateTemplateSelect(cfg) {
    templateSelect.innerHTML = '';
    cfg.templates.forEach(function (tpl) {
      var opt = document.createElement('option');
      opt.value = tpl.name;
      opt.textContent = tpl.name;
      templateSelect.appendChild(opt);
    });

    tplSelectWrap.classList.toggle('hidden', cfg.templates.length <= 1);

    var lastUsed = localStorage.getItem(LAST_TEMPLATE_KEY);
    var names = cfg.templates.map(function (t) { return t.name; });
    templateSelect.value = (lastUsed && names.indexOf(lastUsed) !== -1) ? lastUsed : cfg.defaultTemplateName;
  }

  // ---- screen switching ----
```

- [ ] **Step 8 : mettre à jour `showSetup`, `showMain` et le bouton Enregistrer**

Localiser :
```js
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
```
Remplacer par :
```js
  function showSetup(existingCfg) {
    mainScreen.classList.add('hidden');
    setupScreen.classList.remove('hidden');

    var biz = existingCfg ? existingCfg.businessName : '';
    cfgBusinessName.value = biz || '';
    cfgReviewLink.value = existingCfg ? existingCfg.reviewLink : '';

    workingTemplates = existingCfg
      ? existingCfg.templates.map(function (t) { return { name: t.name, text: t.text }; })
      : [{ name: DEFAULT_TEMPLATE_NAME, text: buildDefaultTemplate(biz) }];
    renderTemplateList();

    cfgCancelBtn.classList.toggle('hidden', !existingCfg);
    cfgLinkError.classList.remove('show');
  }
```

Localiser :
```js
  function showMain() {
    setupScreen.classList.add('hidden');
    mainScreen.classList.remove('hidden');
    counterValue.textContent = String(getCounter());
    clientPrenom.value = '';
    clientPhone.value = '';
    phoneError.classList.remove('show');
    confirmMsg.classList.remove('show');
  }
```
Remplacer par :
```js
  function showMain() {
    setupScreen.classList.add('hidden');
    mainScreen.classList.remove('hidden');
    counterValue.textContent = String(getCounter());
    clientPrenom.value = '';
    clientPhone.value = '';
    phoneError.classList.remove('show');
    confirmMsg.classList.remove('show');

    var cfg = loadConfig();
    populateTemplateSelect(cfg);
  }
```

Localiser :
```js
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
```
Remplacer par :
```js
  cfgSaveBtn.addEventListener('click', function () {
    if (!isValidUrl(cfgReviewLink.value)) {
      cfgLinkError.classList.add('show');
      return;
    }
    cfgLinkError.classList.remove('show');

    var existingCfg = loadConfig();

    saveConfig({
      businessName: cfgBusinessName.value.trim(),
      reviewLink: cfgReviewLink.value.trim(),
      templates: workingTemplates.map(function (t) {
        return { name: t.name.trim() || 'Sans nom', text: t.text };
      }),
      defaultTemplateName: (workingTemplates[0].name.trim() || 'Sans nom'),
      brandColor: existingCfg ? existingCfg.brandColor : '#1d6fd6',
      logoDataUrl: existingCfg ? existingCfg.logoDataUrl : null
    });

    showMain();
  });
```

(Notez que le `cfgBusinessName.addEventListener('input', ...)` d'origine, qui synchronisait automatiquement le modèle unique, est supprimé — avec plusieurs modèles cette synchronisation n'a plus de sens ; le nom du commerce sert uniquement de base au texte pré-rempli d'un nouveau modèle.)

- [ ] **Step 9 : mettre à jour l'envoi pour utiliser le modèle sélectionné**

Localiser :
```js
  sendBtn.addEventListener('click', function () {
    var cleaned = cleanPhone(clientPhone.value);

    if (!isValidPhone(cleaned)) {
      phoneError.classList.add('show');
      confirmMsg.classList.remove('show');
      return;
    }
    phoneError.classList.remove('show');

    var cfg = loadConfig();
    if (!cfg) {
      showSetup(null);
      return;
    }
    var message = fillTemplate(cfg.messageTemplate, {
      prenom: clientPrenom.value,
      lien: cfg.reviewLink
    });

    var href = buildSmsHref(cleaned, message);

    var n = getCounter() + 1;
    setCounter(n);
    counterValue.textContent = String(n);

    confirmMsg.classList.add('show');
    setTimeout(function () {
      confirmMsg.classList.remove('show');
    }, 3000);
    clientPrenom.value = '';
    clientPhone.value = '';

    window.location.href = href;
  });
```
Remplacer par :
```js
  sendBtn.addEventListener('click', function () {
    var cleaned = cleanPhone(clientPhone.value);

    if (!isValidPhone(cleaned)) {
      phoneError.classList.add('show');
      confirmMsg.classList.remove('show');
      return;
    }
    phoneError.classList.remove('show');

    var cfg = loadConfig();
    if (!cfg) {
      showSetup(null);
      return;
    }

    var chosenName = templateSelect.value || cfg.defaultTemplateName;
    var chosen = cfg.templates.filter(function (t) { return t.name === chosenName; })[0] || cfg.templates[0];

    var message = fillTemplate(chosen.text, {
      prenom: clientPrenom.value,
      lien: cfg.reviewLink
    });

    var href = buildSmsHref(cleaned, message);

    localStorage.setItem(LAST_TEMPLATE_KEY, chosen.name);

    var n = getCounter() + 1;
    setCounter(n);
    counterValue.textContent = String(n);

    confirmMsg.classList.add('show');
    setTimeout(function () {
      confirmMsg.classList.remove('show');
    }, 3000);
    clientPrenom.value = '';
    clientPhone.value = '';

    window.location.href = href;
  });
```

- [ ] **Step 10 : vérifier le flux complet dans le navigateur**

`javascript_tool` → `localStorage.clear(); location.reload();`

1. `read_page` → `#tplList` contient une ligne avec le badge "Modèle par défaut", nom "Par défaut".
2. `computer` → taper `Le Petit Bistrot` dans `#cfgBusinessName`, cliquer sur `#tplAddBtn`.
3. `read_page` → `#tplList` contient maintenant 2 lignes, la seconde nommée "Modèle 2" avec un texte contenant "Le Petit Bistrot".
4. `computer` → taper `https://g.page/r/test123` dans `#cfgReviewLink`, cliquer "Enregistrer".
5. `read_page` → `#mainScreen` visible ; `#tplSelectWrap` **visible** (pas la classe `hidden`, car 2 modèles) ; `#templateSelect` a 2 `<option>`.
6. `javascript_tool` → `JSON.parse(localStorage.getItem('avisRelance.config')).templates.length === 2` doit renvoyer `true`.
7. `computer` → sélectionner "Modèle 2" dans `#templateSelect`, taper `0612345678` dans `#clientPhone`, cliquer "Envoyer la demande d'avis".
8. `javascript_tool` → `localStorage.getItem('avisRelance.lastTemplate') === 'Modèle 2'` doit renvoyer `true`.
9. `javascript_tool` → `location.reload()`, puis `read_page` → `#templateSelect` doit avoir la valeur `Modèle 2` sélectionnée (mémorisation du dernier choix).

Expected : toutes les vérifications passent.

- [ ] **Step 11 : commit**

```bash
cd "C:\Users\frank\Desktop\relance avis google nouvelle version"
git add index.html
git commit -m "feat: support multiple named message templates with per-send selection"
```

---

### Task 2: Logo et couleur de marque personnalisables

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes : `loadConfig`, `saveConfig`, `showSetup`, `showMain`, `cfgSaveBtn` handler (de la Task 1).
- Produces (exposées sur `window.AvisRelance`, en plus de l'existant) : `darkenColor(hex: string, percent: number) -> string`.
- Produces (éléments DOM avec ces `id`, réutilisés par les tâches suivantes) : `.icon-row` (conteneur des boutons icône du header principal, renommage de `.gear` en `.icon-btn`).

- [ ] **Step 1 : renommer `.gear` en `.icon-btn` et ajouter les styles logo/couleur**

Localiser :
```css
.gear{background:none;border:none;font-size:20px;cursor:pointer;color:var(--muted)}
```
Remplacer par :
```css
.icon-btn{background:none;border:none;font-size:20px;cursor:pointer;color:var(--muted)}
.icon-row{display:flex;gap:6px;align-items:center}
.logo-row{display:flex;align-items:center;gap:10px}
.logo-preview{width:40px;height:40px;border-radius:10px;object-fit:cover;background:var(--line);display:none}
.logo-preview.show{display:block}
.color-input{width:56px;height:40px;padding:2px;border:1px solid var(--line);border-radius:8px;background:#fff;cursor:pointer}
.header-logo{height:28px;border-radius:6px;margin-right:8px;vertical-align:middle}
```

- [ ] **Step 2 : mettre à jour le header de l'écran principal**

Localiser :
```html
  <div id="mainScreen" class="hidden">
    <header class="top">
      <div class="logo">Relance Avis</div>
      <button id="gearBtn" class="gear" aria-label="Réglages" title="Réglages">⚙️</button>
    </header>
```
Remplacer par :
```html
  <div id="mainScreen" class="hidden">
    <header class="top">
      <div class="logo" id="mainLogo">
        <img id="mainLogoImg" class="header-logo hidden" alt="">
        <span id="mainLogoText">Relance Avis</span>
      </div>
      <div class="icon-row">
        <button id="gearBtn" class="icon-btn" aria-label="Réglages" title="Réglages">⚙️</button>
      </div>
    </header>
```

- [ ] **Step 3 : ajouter les champs logo et couleur à l'écran de réglages**

Localiser :
```html
      <label for="cfgReviewLink">Lien "laisser un avis" Google</label>
      <input id="cfgReviewLink" type="url" placeholder="https://g.page/r/....">
      <p class="hint">Trouvable dans Google Business Profile ou sur votre fiche Google Maps.</p>
      <p id="cfgLinkError" class="error" role="alert">Merci de coller un lien valide (commençant par http:// ou https://).</p>

      <label>Modèles de message</label>
```
Remplacer par :
```html
      <label for="cfgReviewLink">Lien "laisser un avis" Google</label>
      <input id="cfgReviewLink" type="url" placeholder="https://g.page/r/....">
      <p class="hint">Trouvable dans Google Business Profile ou sur votre fiche Google Maps.</p>
      <p id="cfgLinkError" class="error" role="alert">Merci de coller un lien valide (commençant par http:// ou https://).</p>

      <label for="cfgLogoInput">Logo (optionnel)</label>
      <div class="logo-row">
        <img id="cfgLogoPreview" class="logo-preview" alt="Aperçu du logo">
        <input id="cfgLogoInput" type="file" accept="image/*">
      </div>
      <button type="button" id="cfgLogoRemoveBtn" class="btn ghost hidden">Retirer le logo</button>

      <label for="cfgColorInput">Couleur de marque</label>
      <input id="cfgColorInput" type="color" class="color-input" value="#1d6fd6">

      <label>Modèles de message</label>
```

- [ ] **Step 4 : câbler les variables DOM et l'aide au redimensionnement d'image**

Localiser :
```js
  var tplList = document.getElementById('tplList');
  var tplAddBtn = document.getElementById('tplAddBtn');
  var cfgLinkError = document.getElementById('cfgLinkError');
```
Remplacer par :
```js
  var tplList = document.getElementById('tplList');
  var tplAddBtn = document.getElementById('tplAddBtn');
  var cfgLogoInput = document.getElementById('cfgLogoInput');
  var cfgLogoPreview = document.getElementById('cfgLogoPreview');
  var cfgLogoRemoveBtn = document.getElementById('cfgLogoRemoveBtn');
  var cfgColorInput = document.getElementById('cfgColorInput');
  var cfgLinkError = document.getElementById('cfgLinkError');
```

Localiser :
```js
  var workingTemplates = [];
```
Remplacer par :
```js
  var workingTemplates = [];
  var workingLogoDataUrl = null;
  var mainLogoImg = document.getElementById('mainLogoImg');
  var mainLogoText = document.getElementById('mainLogoText');
```

- [ ] **Step 5 : ajouter les fonctions logo et couleur**

Localiser :
```js
  // ---- template list management ----
```
Remplacer par :
```js
  // ---- logo & brand color ----

  function resizeImageFile(file, maxWidth, callback) {
    var reader = new FileReader();
    reader.onload = function (e) {
      var img = new Image();
      img.onload = function () {
        var scale = Math.min(1, maxWidth / img.width);
        var canvas = document.createElement('canvas');
        canvas.width = Math.round(img.width * scale);
        canvas.height = Math.round(img.height * scale);
        var ctx = canvas.getContext('2d');
        ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
        callback(canvas.toDataURL('image/jpeg', 0.8));
      };
      img.src = e.target.result;
    };
    reader.readAsDataURL(file);
  }

  cfgLogoInput.addEventListener('change', function () {
    var file = cfgLogoInput.files && cfgLogoInput.files[0];
    if (!file) return;
    resizeImageFile(file, 240, function (dataUrl) {
      workingLogoDataUrl = dataUrl;
      cfgLogoPreview.src = dataUrl;
      cfgLogoPreview.classList.add('show');
      cfgLogoRemoveBtn.classList.remove('hidden');
    });
  });

  cfgLogoRemoveBtn.addEventListener('click', function () {
    workingLogoDataUrl = null;
    cfgLogoInput.value = '';
    cfgLogoPreview.classList.remove('show');
    cfgLogoPreview.src = '';
    cfgLogoRemoveBtn.classList.add('hidden');
  });

  function darkenColor(hex, percent) {
    var m = /^#?([0-9a-f]{6})$/i.exec(hex || '');
    if (!m) return hex;
    var num = parseInt(m[1], 16);
    var r = Math.max(0, Math.round(((num >> 16) & 255) * (1 - percent)));
    var g = Math.max(0, Math.round(((num >> 8) & 255) * (1 - percent)));
    var b = Math.max(0, Math.round((num & 255) * (1 - percent)));
    return '#' + ((1 << 24) + (r << 16) + (g << 8) + b).toString(16).slice(1);
  }

  function applyBrandColor(hex) {
    var color = hex || '#1d6fd6';
    document.documentElement.style.setProperty('--brand', color);
    document.documentElement.style.setProperty('--brand-dark', darkenColor(color, 0.15));
  }

  // ---- template list management ----
```

Puis, dans le bloc `window.AvisRelance = { ... };` (Task 1), ajouter `darkenColor` à la liste exposée :

Localiser :
```js
    buildSmsHref: buildSmsHref,
    migrateConfig: migrateConfig
  };
```
Remplacer par :
```js
    buildSmsHref: buildSmsHref,
    migrateConfig: migrateConfig,
    darkenColor: darkenColor
  };
```

- [ ] **Step 6 : appliquer logo/couleur dans `showSetup`, `showMain` et l'enregistrement**

Localiser (bloc `showSetup` de la Task 1) :
```js
    workingTemplates = existingCfg
      ? existingCfg.templates.map(function (t) { return { name: t.name, text: t.text }; })
      : [{ name: DEFAULT_TEMPLATE_NAME, text: buildDefaultTemplate(biz) }];
    renderTemplateList();

    cfgCancelBtn.classList.toggle('hidden', !existingCfg);
    cfgLinkError.classList.remove('show');
  }
```
Remplacer par :
```js
    workingTemplates = existingCfg
      ? existingCfg.templates.map(function (t) { return { name: t.name, text: t.text }; })
      : [{ name: DEFAULT_TEMPLATE_NAME, text: buildDefaultTemplate(biz) }];
    renderTemplateList();

    cfgColorInput.value = (existingCfg && existingCfg.brandColor) || '#1d6fd6';
    workingLogoDataUrl = existingCfg ? existingCfg.logoDataUrl : null;
    if (workingLogoDataUrl) {
      cfgLogoPreview.src = workingLogoDataUrl;
      cfgLogoPreview.classList.add('show');
      cfgLogoRemoveBtn.classList.remove('hidden');
    } else {
      cfgLogoPreview.classList.remove('show');
      cfgLogoPreview.src = '';
      cfgLogoRemoveBtn.classList.add('hidden');
    }

    cfgCancelBtn.classList.toggle('hidden', !existingCfg);
    cfgLinkError.classList.remove('show');
  }
```

Localiser (bloc `showMain` de la Task 1) :
```js
    var cfg = loadConfig();
    populateTemplateSelect(cfg);
  }
```
Remplacer par :
```js
    var cfg = loadConfig();
    populateTemplateSelect(cfg);

    applyBrandColor(cfg.brandColor);
    if (cfg.logoDataUrl) {
      mainLogoImg.src = cfg.logoDataUrl;
      mainLogoImg.classList.remove('hidden');
    } else {
      mainLogoImg.classList.add('hidden');
      mainLogoImg.src = '';
    }
    mainLogoText.textContent = cfg.businessName || 'Relance Avis';
  }
```

Localiser (bloc `cfgSaveBtn` de la Task 1) :
```js
      defaultTemplateName: (workingTemplates[0].name.trim() || 'Sans nom'),
      brandColor: existingCfg ? existingCfg.brandColor : '#1d6fd6',
      logoDataUrl: existingCfg ? existingCfg.logoDataUrl : null
    });
```
Remplacer par :
```js
      defaultTemplateName: (workingTemplates[0].name.trim() || 'Sans nom'),
      brandColor: cfgColorInput.value,
      logoDataUrl: workingLogoDataUrl
    });
```

Localiser (bloc d'init en toute fin de fichier) :
```js
  var cfg = loadConfig();
  if (cfg) {
    showMain();
  } else {
    showSetup(null);
  }
```
Remplacer par :
```js
  var cfg = loadConfig();
  applyBrandColor(cfg ? cfg.brandColor : '#1d6fd6');
  if (cfg) {
    showMain();
  } else {
    showSetup(null);
  }
```

- [ ] **Step 7 : vérifier couleur et logo dans le navigateur**

`javascript_tool` → `localStorage.clear(); location.reload();`

1. Console : `AvisRelance.darkenColor('#ffffff', 0.5) === '#808080'` doit renvoyer `true`.
2. `computer` → remplir "Le Petit Bistrot" + `https://g.page/r/test123`, cliquer "Enregistrer" (sans toucher logo/couleur).
3. `read_page` → `#mainLogoText` contient "Le Petit Bistrot" ; `#mainLogoImg` a la classe `hidden`.
4. `javascript_tool` → `getComputedStyle(document.documentElement).getPropertyValue('--brand').trim() === '#1d6fd6'` doit renvoyer `true`.
5. `computer` → cliquer sur `#gearBtn`, changer `#cfgColorInput` à `#ff0000` (via `form_input`), cliquer "Enregistrer".
6. `javascript_tool` → `getComputedStyle(document.documentElement).getPropertyValue('--brand').trim() === '#ff0000'` doit renvoyer `true`.
7. `computer` → rouvrir les réglages (`#gearBtn`). `javascript_tool` → simuler l'upload d'un logo :
```js
(function () {
  var pngDataUrl = 'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNk+A8AAQUBAScY42YAAAAASUVORK5CYII=';
  fetch(pngDataUrl).then(function (res) { return res.blob(); }).then(function (blob) {
    var file = new File([blob], 'logo.png', { type: 'image/png' });
    var dt = new DataTransfer();
    dt.items.add(file);
    document.getElementById('cfgLogoInput').files = dt.files;
    document.getElementById('cfgLogoInput').dispatchEvent(new Event('change'));
  });
})();
```
8. `computer` → `{action: "wait", duration: 1}` (laisser le temps au `FileReader`/`Image` de traiter).
9. `javascript_tool` → `document.getElementById('cfgLogoPreview').classList.contains('show') === true && document.getElementById('cfgLogoPreview').src.indexOf('data:image') === 0` doit renvoyer `true`.
10. `computer` → cliquer "Enregistrer".
11. `read_page` → `#mainLogoImg` n'a plus la classe `hidden`.

Expected : toutes les vérifications passent.

- [ ] **Step 8 : commit**

```bash
cd "C:\Users\frank\Desktop\relance avis google nouvelle version"
git add index.html
git commit -m "feat: customizable brand color and logo in header"
```

---

### Task 3: Historique des envois et détection de doublon

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes : `loadConfig`, `cleanPhone`, `templateSelect`, `sendBtn` handler (des Tasks 1-2), `.icon-row` (Task 2).
- Produces (exposées sur `window.AvisRelance`, en plus de l'existant) : `formatDateFr(iso: string) -> string`.
- Produces (localStorage) : clé `avisRelance.history`, liste de `{prenom, telephone, date, modele}`.

- [ ] **Step 1 : styles de l'écran historique et du bandeau d'avertissement**

Localiser :
```css
.confirm.show{display:block}
</style>
```
Remplacer par :
```css
.confirm.show{display:block}
.history-item{padding:10px 0;border-bottom:1px solid var(--line)}
.history-item:last-child{border-bottom:none}
.history-item .h-name{font-weight:700}
.history-item .h-meta{font-size:12px;color:var(--muted);margin-top:2px}
.history-empty{color:var(--muted);font-size:14px;text-align:center;padding:24px 0}
.back-link{display:inline-block;margin-bottom:14px;color:var(--brand);font-size:14px;font-weight:600;text-decoration:none;cursor:pointer;background:none;border:none;padding:0}
.warn{color:#a15c00;background:#fff3e0;border:1px solid #ffd9a0;border-radius:10px;padding:10px 12px;font-size:13px;margin-top:10px;display:none}
.warn.show{display:block}
</style>
```

- [ ] **Step 2 : ajouter le bouton historique et le bandeau de doublon**

Localiser (résultat de la Task 2) :
```html
      <div class="icon-row">
        <button id="gearBtn" class="icon-btn" aria-label="Réglages" title="Réglages">⚙️</button>
      </div>
```
Remplacer par :
```html
      <div class="icon-row">
        <button id="historyBtn" class="icon-btn" aria-label="Historique" title="Historique">🕘</button>
        <button id="gearBtn" class="icon-btn" aria-label="Réglages" title="Réglages">⚙️</button>
      </div>
```

Localiser :
```html
      <label for="clientPhone">Numéro de téléphone</label>
      <input id="clientPhone" type="tel" inputmode="tel" autocomplete="tel" placeholder="06 12 34 56 78">
      <p id="phoneError" class="error" role="alert">Merci de saisir un numéro de téléphone valide.</p>

      <button id="sendBtn" class="btn">Envoyer la demande d'avis</button>
```
Remplacer par :
```html
      <label for="clientPhone">Numéro de téléphone</label>
      <input id="clientPhone" type="tel" inputmode="tel" autocomplete="tel" placeholder="06 12 34 56 78">
      <p id="phoneError" class="error" role="alert">Merci de saisir un numéro de téléphone valide.</p>
      <p id="duplicateWarn" class="warn" role="alert"></p>

      <button id="sendBtn" class="btn">Envoyer la demande d'avis</button>
```

- [ ] **Step 3 : ajouter l'écran historique**

Localiser :
```html
  </div>

</div>

<script>
```
Remplacer par :
```html
  </div>

  <div id="historyScreen" class="hidden">
    <button type="button" id="historyBackBtn" class="back-link">← Retour</button>
    <h1>Historique</h1>
    <p class="sub">Derniers envois de demandes d'avis.</p>
    <div class="card">
      <div id="historyList"></div>
      <p id="historyEmpty" class="history-empty hidden">Aucun envoi pour l'instant.</p>
    </div>
  </div>

</div>

<script>
```

- [ ] **Step 4 : ajouter les fonctions de stockage et d'affichage de l'historique**

Localiser :
```js
  var LAST_TEMPLATE_KEY = 'avisRelance.lastTemplate';
  var DEFAULT_TEMPLATE_NAME = 'Par défaut';
```
Remplacer par :
```js
  var LAST_TEMPLATE_KEY = 'avisRelance.lastTemplate';
  var DEFAULT_TEMPLATE_NAME = 'Par défaut';
  var HISTORY_KEY = 'avisRelance.history';
```

Localiser :
```js
  var gearBtn = document.getElementById('gearBtn');
```
Remplacer par :
```js
  var gearBtn = document.getElementById('gearBtn');
  var historyBtn = document.getElementById('historyBtn');
  var historyScreen = document.getElementById('historyScreen');
  var historyBackBtn = document.getElementById('historyBackBtn');
  var historyList = document.getElementById('historyList');
  var historyEmpty = document.getElementById('historyEmpty');
  var duplicateWarn = document.getElementById('duplicateWarn');
  var pendingDuplicateConfirm = false;
```

Localiser :
```js
  // ---- logo & brand color ----
```
Remplacer par :
```js
  // ---- history ----

  function pad2(n) {
    return (n < 10 ? '0' : '') + n;
  }

  function formatDateFr(iso) {
    var d = new Date(iso);
    return pad2(d.getDate()) + '/' + pad2(d.getMonth() + 1) + '/' + d.getFullYear();
  }

  function getHistory() {
    try {
      var list = JSON.parse(localStorage.getItem(HISTORY_KEY) || '[]');
      return Array.isArray(list) ? list : [];
    } catch (e) {
      return [];
    }
  }

  function addHistoryEntry(entry) {
    var list = getHistory();
    list.unshift(entry);
    localStorage.setItem(HISTORY_KEY, JSON.stringify(list));
  }

  function findHistoryByPhone(cleanedPhone) {
    var list = getHistory();
    for (var i = 0; i < list.length; i++) {
      if (list[i].telephone === cleanedPhone) return list[i];
    }
    return null;
  }

  function renderHistory() {
    var list = getHistory();
    historyList.innerHTML = '';
    historyEmpty.classList.toggle('hidden', list.length > 0);
    list.forEach(function (entry) {
      var row = document.createElement('div');
      row.className = 'history-item';

      var name = document.createElement('div');
      name.className = 'h-name';
      name.textContent = (entry.prenom ? entry.prenom + ' — ' : '') + entry.telephone;

      var meta = document.createElement('div');
      meta.className = 'h-meta';
      meta.textContent = formatDateFr(entry.date) + ' · ' + entry.modele;

      row.appendChild(name);
      row.appendChild(meta);
      historyList.appendChild(row);
    });
  }

  function showHistory() {
    mainScreen.classList.add('hidden');
    setupScreen.classList.add('hidden');
    historyScreen.classList.remove('hidden');
    renderHistory();
  }

  historyBtn.addEventListener('click', showHistory);
  historyBackBtn.addEventListener('click', showMain);

  clientPhone.addEventListener('input', function () {
    pendingDuplicateConfirm = false;
    duplicateWarn.classList.remove('show');
  });

  // ---- logo & brand color ----
```

Puis ajouter `formatDateFr` à `window.AvisRelance` :

Localiser :
```js
    migrateConfig: migrateConfig,
    darkenColor: darkenColor
  };
```
Remplacer par :
```js
    migrateConfig: migrateConfig,
    darkenColor: darkenColor,
    formatDateFr: formatDateFr
  };
```

- [ ] **Step 5 : réinitialiser l'avertissement de doublon au changement d'écran**

Localiser (bloc `showMain`, résultat de la Task 2) :
```js
    applyBrandColor(cfg.brandColor);
    if (cfg.logoDataUrl) {
```
Remplacer par :
```js
    pendingDuplicateConfirm = false;
    duplicateWarn.classList.remove('show');

    applyBrandColor(cfg.brandColor);
    if (cfg.logoDataUrl) {
```

- [ ] **Step 6 : ajouter la détection de doublon et l'écriture d'historique à l'envoi**

Localiser (bloc `sendBtn`, résultat de la Task 1) :
```js
    var cfg = loadConfig();
    if (!cfg) {
      showSetup(null);
      return;
    }

    var chosenName = templateSelect.value || cfg.defaultTemplateName;
```
Remplacer par :
```js
    if (!pendingDuplicateConfirm) {
      var existing = findHistoryByPhone(cleaned);
      if (existing) {
        duplicateWarn.textContent = '⚠️ Déjà contacté le ' + formatDateFr(existing.date) + ' — cliquez à nouveau pour envoyer quand même.';
        duplicateWarn.classList.add('show');
        pendingDuplicateConfirm = true;
        return;
      }
    }
    pendingDuplicateConfirm = false;
    duplicateWarn.classList.remove('show');

    var cfg = loadConfig();
    if (!cfg) {
      showSetup(null);
      return;
    }

    var chosenName = templateSelect.value || cfg.defaultTemplateName;
```

Localiser :
```js
    localStorage.setItem(LAST_TEMPLATE_KEY, chosen.name);

    var n = getCounter() + 1;
```
Remplacer par :
```js
    localStorage.setItem(LAST_TEMPLATE_KEY, chosen.name);
    addHistoryEntry({
      prenom: clientPrenom.value.trim(),
      telephone: cleaned,
      date: new Date().toISOString(),
      modele: chosen.name
    });

    var n = getCounter() + 1;
```

- [ ] **Step 7 : vérifier l'historique et la détection de doublon dans le navigateur**

`javascript_tool` → `localStorage.clear(); location.reload();`

1. `computer` → compléter la config (nom + lien valide), "Enregistrer".
2. `computer` → cliquer `#historyBtn`. `read_page` → `#historyEmpty` visible, `#historyList` vide.
3. `computer` → cliquer `#historyBackBtn` (retour à l'écran principal).
4. `computer` → taper "Marie" dans `#clientPrenom`, `0612345678` dans `#clientPhone`, cliquer "Envoyer".
5. `computer` → cliquer `#historyBtn`. `read_page` → `#historyList` contient 1 `.history-item` dont le texte contient "Marie" et "0612345678".
6. `computer` → retour, taper `06 12 34 56 78` (même numéro, formatage différent) dans `#clientPhone`, cliquer "Envoyer" une première fois.
7. `read_page` → `#duplicateWarn` a la classe `show` ; `javascript_tool` → `localStorage.getItem('avisRelance.counter') === '1'` doit renvoyer `true` (pas d'incrément).
8. `computer` → cliquer "Envoyer" une seconde fois (confirmation).
9. `javascript_tool` → `localStorage.getItem('avisRelance.counter') === '2'` et `JSON.parse(localStorage.getItem('avisRelance.history')).length === 2` doivent renvoyer `true`.
10. `computer` → taper un nouveau numéro dans `#clientPhone`. `read_page` → `#duplicateWarn` n'a plus la classe `show`.
11. Console : `AvisRelance.formatDateFr(new Date().toISOString())` doit renvoyer la date du jour au format `JJ/MM/AAAA`.

Expected : toutes les vérifications passent.

- [ ] **Step 8 : commit**

```bash
cd "C:\Users\frank\Desktop\relance avis google nouvelle version"
git add index.html
git commit -m "feat: send history with duplicate-contact warning"
```

---

### Task 4: Écran QR code

**Files:**
- Modify: `index.html`
- Temporaire (supprimé en fin de tâche) : `vendor-qrcode.tmp.js`

**Interfaces:**
- Consumes : `loadConfig`, `showMain`, `.icon-row` (Tasks 1-3).
- Produces : fonction globale `qrcode(typeNumber, errorCorrectionLevel)` (fournie par la bibliothèque vendorisée `qrcode-generator`), utilisée uniquement en interne par l'écran QR.

- [ ] **Step 1 : télécharger la bibliothèque QR code réelle**

```bash
cd "C:\Users\frank\Desktop\relance avis google nouvelle version"
curl -sL -o vendor-qrcode.tmp.js "https://unpkg.com/qrcode-generator@1.4.4/qrcode.js"
head -c 300 vendor-qrcode.tmp.js
```
Expected : le début du fichier affiche un commentaire de licence mentionnant "QRCode for JavaScript" et Kazuhiko Arase. Si la commande échoue (pas de réseau) ou si le contenu ne correspond pas, s'arrêter et signaler le problème plutôt que d'inventer un contenu de remplacement.

- [ ] **Step 2 : vendoriser la bibliothèque dans `index.html`**

```bash
cd "C:\Users\frank\Desktop\relance avis google nouvelle version"
node -e "
var fs = require('fs');
var html = fs.readFileSync('index.html', 'utf8');
var lib = fs.readFileSync('vendor-qrcode.tmp.js', 'utf8');
var marker = '<script>\n(function () {';
if (html.indexOf(marker) === -1) { throw new Error('marker not found'); }
var injected = '<script>\n/* qrcode-generator by Kazuhiko Arase, MIT license, vendored for offline use */\n' + lib + '\n</script>\n' + marker;
html = html.replace(marker, injected);
fs.writeFileSync('index.html', html);
"
rm vendor-qrcode.tmp.js
grep -c "qrcode-generator by Kazuhiko Arase" index.html
```
Expected : la dernière commande affiche `1`.

- [ ] **Step 3 : vérifier que la bibliothèque se charge**

`javascript_tool` → recharger la page (`location.reload()`), puis :
```js
typeof qrcode === 'function'
```
Expected : `true`.

- [ ] **Step 4 : styles de l'écran QR code**

Localiser :
```css
.warn.show{display:block}
</style>
```
Remplacer par :
```css
.warn.show{display:block}
.qr-wrap{display:flex;flex-direction:column;align-items:center;padding:24px 0}
.qr-wrap img{border:1px solid var(--line);border-radius:12px;width:220px;height:220px}
.qr-business{font-weight:700;font-size:16px;margin-bottom:16px;text-align:center}
</style>
```

- [ ] **Step 5 : ajouter le bouton QR code et l'écran**

Localiser (résultat de la Task 3) :
```html
      <div class="icon-row">
        <button id="historyBtn" class="icon-btn" aria-label="Historique" title="Historique">🕘</button>
        <button id="gearBtn" class="icon-btn" aria-label="Réglages" title="Réglages">⚙️</button>
      </div>
```
Remplacer par :
```html
      <div class="icon-row">
        <button id="qrBtn" class="icon-btn" aria-label="QR code" title="QR code">▦</button>
        <button id="historyBtn" class="icon-btn" aria-label="Historique" title="Historique">🕘</button>
        <button id="gearBtn" class="icon-btn" aria-label="Réglages" title="Réglages">⚙️</button>
      </div>
```

Localiser :
```html
  </div>

</div>

<script>
```
Remplacer par :
```html
  </div>

  <div id="qrScreen" class="hidden">
    <button type="button" id="qrBackBtn" class="back-link">← Retour</button>
    <h1>QR code avis Google</h1>
    <p class="sub">À afficher au comptoir pour que le client scanne lui-même.</p>
    <div class="qr-wrap">
      <div class="qr-business" id="qrBusinessName"></div>
      <img id="qrImage" alt="QR code du lien d'avis Google">
    </div>
  </div>

</div>

<script>
```

(Ce motif reste unique dans le fichier à ce stade : la balise `<script>` n'apparaît qu'une seule fois dans `index.html`, donc `</div>\n\n</div>\n\n<script>` ne peut correspondre qu'à la fermeture de `historyScreen` suivie de la fermeture de `.app`, juste avant le script.)

- [ ] **Step 6 : câbler l'écran QR code**

Localiser :
```js
  var duplicateWarn = document.getElementById('duplicateWarn');
  var pendingDuplicateConfirm = false;
```
Remplacer par :
```js
  var duplicateWarn = document.getElementById('duplicateWarn');
  var pendingDuplicateConfirm = false;
  var qrBtn = document.getElementById('qrBtn');
  var qrScreen = document.getElementById('qrScreen');
  var qrBackBtn = document.getElementById('qrBackBtn');
  var qrImage = document.getElementById('qrImage');
  var qrBusinessName = document.getElementById('qrBusinessName');
```

Localiser la fin du bloc historique (juste après `historyBackBtn.addEventListener('click', showMain);`) :
```js
  historyBtn.addEventListener('click', showHistory);
  historyBackBtn.addEventListener('click', showMain);
```
Remplacer par :
```js
  historyBtn.addEventListener('click', showHistory);
  historyBackBtn.addEventListener('click', showMain);

  function renderQr(cfg) {
    qrBusinessName.textContent = cfg.businessName || '';
    var qr = qrcode(0, 'M');
    qr.addData(cfg.reviewLink);
    qr.make();
    qrImage.src = qr.createDataURL(8, 8);
  }

  function showQr() {
    var cfg = loadConfig();
    if (!cfg || !cfg.reviewLink) return;
    mainScreen.classList.add('hidden');
    qrScreen.classList.remove('hidden');
    renderQr(cfg);
  }

  qrBtn.addEventListener('click', showQr);
  qrBackBtn.addEventListener('click', showMain);
```

- [ ] **Step 7 : vérifier l'écran QR code dans le navigateur**

`javascript_tool` → `localStorage.clear(); location.reload();`

1. `computer` → compléter la config avec le lien `https://g.page/r/testQR`, "Enregistrer".
2. `computer` → cliquer `#qrBtn`.
3. `read_page` → `#qrScreen` visible, `#mainScreen` a la classe `hidden`.
4. `javascript_tool` → `document.getElementById('qrImage').src.indexOf('data:image') === 0 && document.getElementById('qrImage').src.length > 200` doit renvoyer `true`.
5. `read_page` → `#qrBusinessName` contient le nom du commerce configuré.
6. `computer` → cliquer `#qrBackBtn`. `read_page` → `#mainScreen` visible, `#qrScreen` a la classe `hidden`.

Expected : toutes les vérifications passent.

- [ ] **Step 8 : commit**

```bash
cd "C:\Users\frank\Desktop\relance avis google nouvelle version"
git add index.html
git commit -m "feat: dedicated QR code screen for the Google review link"
```

---

## Checklist manuelle post-implémentation (à faire par vous, non automatisable)

1. Sur un vrai téléphone, refaire le test SMS de la V1 (ouverture de l'appli SMS avec le bon modèle sélectionné).
2. Scanner le QR code affiché (écran QR) avec un second téléphone pour confirmer qu'il ouvre bien le lien d'avis Google.
3. Uploader un vrai logo (photo prise avec le téléphone) et vérifier visuellement le rendu dans l'en-tête.
4. Vérifier que la taille de `localStorage` reste raisonnable après upload d'un logo réel (`javascript_tool` → `JSON.stringify(localStorage).length`).
