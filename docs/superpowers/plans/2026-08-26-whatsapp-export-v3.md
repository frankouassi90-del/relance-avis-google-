# WhatsApp & export d'historique (V3) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ajouter un second bouton d'envoi par WhatsApp (en complément du SMS existant) et un export CSV de l'historique des envois, sur l'app "Relance Avis" (`index.html`).

**Architecture:** Toujours un seul fichier `index.html` (HTML + CSS + JS inline), aucun backend, aucun build. Extension du schéma `avisRelance.history` avec un champ `canal` ; export CSV généré et téléchargé 100% localement (`Blob` + lien temporaire).

**Tech Stack:** HTML5, CSS3, JavaScript vanilla (ES5-compatible, IIFE unique, cohérent avec le reste du fichier).

## Global Constraints

- Fichier unique livré `index.html` — aucune dépendance externe ajoutée (spec : "Architecture").
- Toutes les données restent en `localStorage` / téléchargement local — aucune transmission serveur (spec : "Détails techniques").
- Interface entièrement en français.
- Pas de framework de test automatisé — vérification manuelle via navigateur.
- Une entrée d'historique sans `canal` (écrite par la V2, avant cette itération) doit être traitée comme `"SMS"` par défaut à la lecture — pas de migration en écriture (spec : "Architecture").
- JavaScript ES5 (pas de `let`/`const`/arrow functions/template literals), cohérent avec le style existant du fichier (`var`, `function() {}`, concaténation par `+`).
- Un dépôt git existe pour ce projet — commit normal à la fin de chaque tâche.

---

## Environnement de test

Le serveur de preview existe déjà (`.claude/launch.json`, port 8787) :

```bash
cd "C:\Users\frank\Desktop\relance avis google nouvelle version"
python -m http.server 8787 &
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8787/index.html
```

Expected : `200`. Naviguer ensuite vers `http://localhost:8787/index.html` avec l'outil navigateur pour les vérifications de chaque tâche. Avant chaque tâche, effacer l'état précédent si besoin (`javascript_tool` → `localStorage.clear(); location.reload();`). **Si un onglet déjà ouvert donne des résultats de clic incohérents (aucun changement d'état après un `.click()` qui ne devrait pourtant pas échouer), fermez-le et ouvrez-en un nouveau avant de conclure à un bug — c'est un artefact d'outil déjà observé sur ce projet, pas un défaut de l'app.**

---

### Task 1: Bouton WhatsApp

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes : `cleanPhone`, `isValidPhone`, `loadConfig`, `templateSelect`, `fillTemplate`, `buildSmsHref`, `findHistoryByPhone`, `addHistoryEntry`, `formatDateFr`, `pendingDuplicateConfirm`, `duplicateWarn`, `confirmMsg`, `clientPrenom`, `clientPhone`, `counterValue`, `getCounter`, `setCounter`, `LAST_TEMPLATE_KEY` (existants).
- Produces (exposées sur `window.AvisRelance`, en plus de l'existant) : `toWhatsAppNumber(cleaned: string) -> string`, `buildWhatsAppHref(number: string, message: string) -> string`.
- Produces (schéma) : chaque entrée `avisRelance.history` gagne un champ `canal: "SMS" | "WhatsApp"`.
- Produces (éléments DOM, réutilisés par la Task 2 pour l'affichage) : `#whatsappBtn`.

- [ ] **Step 1 : ajouter les fonctions pures de conversion et de lien WhatsApp**

Localiser :
```js
  function buildSmsHref(number, message) {
    var isIOS = /iPhone|iPad|iPod/i.test(navigator.userAgent);
    var sep = isIOS ? '&' : '?';
    return 'sms:' + number + sep + 'body=' + encodeURIComponent(message);
  }

  // ---- history ----
```
Remplacer par :
```js
  function buildSmsHref(number, message) {
    var isIOS = /iPhone|iPad|iPod/i.test(navigator.userAgent);
    var sep = isIOS ? '&' : '?';
    return 'sms:' + number + sep + 'body=' + encodeURIComponent(message);
  }

  function toWhatsAppNumber(cleaned) {
    var n = (cleaned || '').replace(/^\+/, '');
    if (n.charAt(0) === '0') {
      return '33' + n.slice(1);
    }
    return n;
  }

  function buildWhatsAppHref(number, message) {
    return 'https://wa.me/' + number + '?text=' + encodeURIComponent(message);
  }

  // ---- history ----
```

- [ ] **Step 2 : exposer les nouvelles fonctions sur `window.AvisRelance`**

Localiser :
```js
    buildSmsHref: buildSmsHref,
    migrateConfig: migrateConfig,
    darkenColor: darkenColor,
    formatDateFr: formatDateFr
  };
```
Remplacer par :
```js
    buildSmsHref: buildSmsHref,
    migrateConfig: migrateConfig,
    darkenColor: darkenColor,
    formatDateFr: formatDateFr,
    toWhatsAppNumber: toWhatsAppNumber,
    buildWhatsAppHref: buildWhatsAppHref
  };
```

- [ ] **Step 3 : vérifier les fonctions pures en console**

`javascript_tool` (chaque ligne doit renvoyer `true`) :
```js
AvisRelance.toWhatsAppNumber('0612345678') === '33612345678'
```
```js
AvisRelance.toWhatsAppNumber('+33612345678') === '33612345678'
```
```js
AvisRelance.toWhatsAppNumber('33612345678') === '33612345678'
```
```js
AvisRelance.buildWhatsAppHref('33612345678', 'Salut').indexOf('https://wa.me/33612345678?text=') === 0
```

- [ ] **Step 4 : ajouter le bouton WhatsApp et renommer le bouton SMS**

Localiser :
```html
      <button id="sendBtn" class="btn">Envoyer la demande d'avis</button>
      <p id="confirmMsg" class="confirm">SMS ouvert ✓</p>
```
Remplacer par :
```html
      <button id="sendBtn" class="btn">Envoyer par SMS</button>
      <button id="whatsappBtn" class="btn">Envoyer par WhatsApp</button>
      <p id="confirmMsg" class="confirm"></p>
```

- [ ] **Step 5 : câbler la référence DOM du nouveau bouton**

Localiser :
```js
  var sendBtn = document.getElementById('sendBtn');
  var confirmMsg = document.getElementById('confirmMsg');
```
Remplacer par :
```js
  var sendBtn = document.getElementById('sendBtn');
  var whatsappBtn = document.getElementById('whatsappBtn');
  var confirmMsg = document.getElementById('confirmMsg');
```

- [ ] **Step 6 : factoriser l'envoi en une fonction partagée par canal**

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
    var chosen = cfg.templates.filter(function (t) { return t.name === chosenName; })[0] || cfg.templates[0];

    var message = fillTemplate(chosen.text, {
      prenom: clientPrenom.value,
      lien: cfg.reviewLink
    });

    var href = buildSmsHref(cleaned, message);

    localStorage.setItem(LAST_TEMPLATE_KEY, chosen.name);
    addHistoryEntry({
      prenom: clientPrenom.value.trim(),
      telephone: cleaned,
      date: new Date().toISOString(),
      modele: chosen.name
    });

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
  function sendViaChannel(canal) {
    var cleaned = cleanPhone(clientPhone.value);

    if (!isValidPhone(cleaned)) {
      phoneError.classList.add('show');
      confirmMsg.classList.remove('show');
      return;
    }
    phoneError.classList.remove('show');

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
    var chosen = cfg.templates.filter(function (t) { return t.name === chosenName; })[0] || cfg.templates[0];

    var message = fillTemplate(chosen.text, {
      prenom: clientPrenom.value,
      lien: cfg.reviewLink
    });

    var href = canal === 'WhatsApp'
      ? buildWhatsAppHref(toWhatsAppNumber(cleaned), message)
      : buildSmsHref(cleaned, message);

    localStorage.setItem(LAST_TEMPLATE_KEY, chosen.name);
    addHistoryEntry({
      prenom: clientPrenom.value.trim(),
      telephone: cleaned,
      date: new Date().toISOString(),
      modele: chosen.name,
      canal: canal
    });

    var n = getCounter() + 1;
    setCounter(n);
    counterValue.textContent = String(n);

    confirmMsg.textContent = canal === 'WhatsApp' ? 'WhatsApp ouvert ✓' : 'SMS ouvert ✓';
    confirmMsg.classList.add('show');
    setTimeout(function () {
      confirmMsg.classList.remove('show');
    }, 3000);
    clientPrenom.value = '';
    clientPhone.value = '';

    window.location.href = href;
  }

  sendBtn.addEventListener('click', function () {
    sendViaChannel('SMS');
  });

  whatsappBtn.addEventListener('click', function () {
    sendViaChannel('WhatsApp');
  });
```

- [ ] **Step 7 : vérifier le flux d'envoi WhatsApp dans le navigateur**

`javascript_tool` → `localStorage.clear(); location.reload();`

1. `computer` → compléter la config (nom + lien valide), "Enregistrer".
2. `read_page` → `#sendBtn` affiche "Envoyer par SMS", `#whatsappBtn` existe et affiche "Envoyer par WhatsApp".
3. `computer` → taper `Marie` dans `#clientPrenom`, `0612345678` dans `#clientPhone`, cliquer `#whatsappBtn`.
4. `javascript_tool` → `JSON.parse(localStorage.getItem('avisRelance.history'))[0]` doit avoir `canal === 'WhatsApp'` et `telephone === '0612345678'` (le numéro stocké reste au format saisi, non converti — seule la construction du lien `wa.me` convertit).
5. `javascript_tool` → `localStorage.getItem('avisRelance.counter') === '1'` doit renvoyer `true`.
6. `computer` → taper un nouveau numéro `0698765432`, cliquer `#sendBtn` (SMS cette fois).
7. `javascript_tool` → `JSON.parse(localStorage.getItem('avisRelance.history'))[0].canal === 'SMS'` doit renvoyer `true`, et le compteur doit maintenant valoir `2`.
8. `javascript_tool` → recharger, cliquer `#whatsappBtn` avec le numéro `0612345678` déjà contacté (doublon) → `#duplicateWarn` doit s'afficher, sans incrément du compteur avant la confirmation ; un second clic sur `#whatsappBtn` envoie et écrit une nouvelle entrée avec `canal: 'WhatsApp'`.

Expected : toutes les vérifications ci-dessus passent.

- [ ] **Step 8 : commit**

```bash
cd "C:\Users\frank\Desktop\relance avis google nouvelle version"
git add index.html
git commit -m "feat: add WhatsApp send button alongside SMS"
```

---

### Task 2: Export CSV de l'historique

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes : `getHistory`, `formatDateFr`, `historyList`, `historyEmpty`, `renderHistory` (existants), champ `entry.canal` introduit par la Task 1.
- Produces (exposées sur `window.AvisRelance`, en plus de l'existant) : `csvEscape(value: any) -> string`, `buildHistoryCsv(list: array) -> string`.

- [ ] **Step 1 : afficher le canal dans chaque ligne d'historique**

Localiser :
```js
      var meta = document.createElement('div');
      meta.className = 'h-meta';
      meta.textContent = formatDateFr(entry.date) + ' · ' + entry.modele;
```
Remplacer par :
```js
      var meta = document.createElement('div');
      meta.className = 'h-meta';
      meta.textContent = formatDateFr(entry.date) + ' · ' + entry.modele + ' · ' + (entry.canal || 'SMS');
```

- [ ] **Step 2 : ajouter le bouton d'export dans l'écran historique**

Localiser :
```html
    <h1>Historique</h1>
    <p class="sub">Derniers envois de demandes d'avis.</p>
    <div class="card">
      <div id="historyList"></div>
      <p id="historyEmpty" class="history-empty hidden">Aucun envoi pour l'instant.</p>
    </div>
```
Remplacer par :
```html
    <h1>Historique</h1>
    <p class="sub">Derniers envois de demandes d'avis.</p>
    <button type="button" id="historyExportBtn" class="btn ghost">Exporter en CSV</button>
    <div class="card">
      <div id="historyList"></div>
      <p id="historyEmpty" class="history-empty hidden">Aucun envoi pour l'instant.</p>
    </div>
```

- [ ] **Step 3 : câbler la référence DOM et désactiver le bouton si l'historique est vide**

Localiser :
```js
  var historyEmpty = document.getElementById('historyEmpty');
```
Remplacer par :
```js
  var historyEmpty = document.getElementById('historyEmpty');
  var historyExportBtn = document.getElementById('historyExportBtn');
```

Localiser :
```js
  function renderHistory() {
    var list = getHistory();
    historyList.innerHTML = '';
    historyEmpty.classList.toggle('hidden', list.length > 0);
```
Remplacer par :
```js
  function renderHistory() {
    var list = getHistory();
    historyList.innerHTML = '';
    historyEmpty.classList.toggle('hidden', list.length > 0);
    historyExportBtn.disabled = list.length === 0;
```

- [ ] **Step 4 : ajouter la génération et le téléchargement du CSV**

Localiser :
```js
  function showHistory() {
    showScreen(historyScreen);
    renderHistory();
  }

  historyBtn.addEventListener('click', showHistory);
  historyBackBtn.addEventListener('click', showMain);
```
Remplacer par :
```js
  function showHistory() {
    showScreen(historyScreen);
    renderHistory();
  }

  historyBtn.addEventListener('click', showHistory);
  historyBackBtn.addEventListener('click', showMain);

  function csvEscape(value) {
    var s = String(value === null || value === undefined ? '' : value);
    if (/["\n,]/.test(s)) {
      s = '"' + s.replace(/"/g, '""') + '"';
    }
    return s;
  }

  function buildHistoryCsv(list) {
    var rows = [['Prenom', 'Telephone', 'Date', 'Modele', 'Canal']];
    list.forEach(function (entry) {
      rows.push([
        entry.prenom || '',
        entry.telephone || '',
        formatDateFr(entry.date),
        entry.modele || '',
        entry.canal || 'SMS'
      ]);
    });
    return rows.map(function (row) {
      return row.map(csvEscape).join(',');
    }).join('\r\n');
  }

  historyExportBtn.addEventListener('click', function () {
    var list = getHistory();
    if (!list.length) return;
    var csv = buildHistoryCsv(list);
    var blob = new Blob(['\uFEFF' + csv], { type: 'text/csv;charset=utf-8;' });
    var url = URL.createObjectURL(blob);
    var a = document.createElement('a');
    a.href = url;
    a.download = 'historique.csv';
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  });
```

- [ ] **Step 5 : exposer les fonctions pures sur `window.AvisRelance`**

Localiser :
```js
    toWhatsAppNumber: toWhatsAppNumber,
    buildWhatsAppHref: buildWhatsAppHref
  };
```
Remplacer par :
```js
    toWhatsAppNumber: toWhatsAppNumber,
    buildWhatsAppHref: buildWhatsAppHref,
    csvEscape: csvEscape,
    buildHistoryCsv: buildHistoryCsv
  };
```

- [ ] **Step 6 : vérifier l'affichage du canal et l'export en console**

1. Console : vérifier `AvisRelance.buildHistoryCsv` sur des données de test :
```js
AvisRelance.buildHistoryCsv([{prenom:'Marie',telephone:'0612345678',date:'2026-08-26T10:00:00.000Z',modele:'Par défaut',canal:'WhatsApp'}])
```
Expected : renvoie une chaîne dont la première ligne est `Prenom,Telephone,Date,Modele,Canal` et la seconde `Marie,0612345678,26/08/2026,Par défaut,WhatsApp`.
2. Console : `AvisRelance.csvEscape('Le "Petit", Bistrot') === '"Le ""Petit"", Bistrot"'` doit renvoyer `true`.
3. Console : `AvisRelance.buildHistoryCsv([{prenom:'X',telephone:'0600000000',date:'2026-08-26T10:00:00.000Z'}]).indexOf(',SMS') !== -1` doit renvoyer `true` (entrée sans `canal` traitée comme `'SMS'`).

- [ ] **Step 7 : vérifier l'écran historique et le téléchargement dans le navigateur**

Une config valide et au moins un envoi (SMS et WhatsApp) doivent déjà exister depuis la Task 1.

1. `computer` → cliquer sur l'icône historique (`#historyBtn`).
2. `read_page` → chaque ligne de `#historyList` affiche désormais le canal (ex. "... · SMS" ou "... · WhatsApp") ; `#historyExportBtn` n'est pas désactivé.
3. `computer` → cliquer sur `#historyExportBtn`.
4. `javascript_tool` → confirmer qu'aucune erreur JS n'est apparue (`read_console_messages` avec `onlyErrors: true`) ; le téléchargement d'un fichier déclenché par un clic utilisateur ne peut pas être intercepté facilement par l'outil de test, donc cette étape valide surtout l'absence d'erreur au clic — le contenu exact du CSV est déjà validé via les vérifications console de l'étape précédente.
5. `javascript_tool` → `localStorage.clear(); location.reload();` puis compléter une config sans jamais envoyer de demande ; `computer` → aller sur l'historique ; `read_page` → `#historyExportBtn` doit avoir l'attribut `disabled`.

Expected : toutes les vérifications ci-dessus passent.

- [ ] **Step 8 : commit**

```bash
cd "C:\Users\frank\Desktop\relance avis google nouvelle version"
git add index.html
git commit -m "feat: CSV export for send history, display channel per entry"
```

---

## Checklist manuelle post-implémentation (à faire par vous, non automatisable)

1. Sur un vrai téléphone, tester le bouton WhatsApp : confirmer que l'app WhatsApp s'ouvre avec le bon contact/numéro et le message pré-rempli, pour un numéro français saisi en `06...`.
2. Ouvrir le fichier `historique.csv` téléchargé dans Excel ou Google Sheets et confirmer que les accents s'affichent correctement (test du BOM UTF-8) et que les colonnes sont bien séparées.
3. Tester l'export avec un historique contenant un prénom incluant une virgule ou des guillemets, pour confirmer l'échappement CSV.
