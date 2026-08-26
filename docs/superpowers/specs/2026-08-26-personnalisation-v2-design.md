# Relance d'avis Google — Personnalisation V2 — Design

**Date** : 2026-08-26
**Statut** : approuvé, prêt pour plan d'implémentation

## Contexte

Suite au MVP V1 (voir `2026-08-08-relance-avis-google-design.md`), qui excluait volontairement le QR code, l'historique des envois et la personnalisation visuelle pour rester au plus simple. Frank souhaite maintenant ajouter ces fonctionnalités, en une itération, sur la base de l'app existante (`index.html`).

Quatre fonctionnalités demandées pour cette itération :
1. QR code du lien d'avis Google, affichable au comptoir
2. Historique des envois (pour éviter de relancer deux fois le même client)
3. Plusieurs modèles de message, sélectionnables à l'envoi
4. Couleur de marque et logo personnalisables

Hors périmètre de cette itération, prévu pour une prochaine mise à jour : bouton WhatsApp en complément du SMS, PWA installable.

## Décisions de design

- **Historique** : conserve prénom, numéro de téléphone (nettoyé) et date pour chaque envoi. Reste en local (`localStorage`), jamais transmis à un serveur — cohérent avec le principe "aucune donnée ne quitte l'appareil" du projet.
- **Détection de doublon** : avant l'envoi, si le numéro saisi existe déjà dans l'historique, un avertissement non bloquant s'affiche (date du dernier contact) ; le commerçant confirme un second clic pour envoyer quand même.
- **Modèles de message** : remplace le modèle unique de la V1 par une liste de modèles nommés, gérés depuis les réglages (ajouter / modifier / supprimer / marquer par défaut). Sélection au moment de l'envoi via un menu déroulant sur l'écran principal (affiché seulement s'il y a plus d'un modèle), qui mémorise le dernier choix.
- **QR code** : écran dédié, accessible depuis l'écran principal, affichant un grand QR code généré **localement** (aucun appel réseau, aucune donnée envoyée à un tiers) pointant vers le lien d'avis Google configuré. Généré via une petite bibliothèque de génération de QR code du domaine public, vendorisée directement dans `index.html` (pas de dépendance CDN, cohérent avec le principe "fonctionne hors-ligne").
- **Logo** : uploadé depuis les réglages, redimensionné côté client (canvas, ~240px de large max) avant d'être stocké en `localStorage` sous forme de data URL, pour éviter qu'une photo non compressée sature le stockage local du navigateur.
- **Couleur de marque** : sélecteur de couleur natif (`<input type="color">`) dans les réglages, remplace la couleur bleue actuelle (`--brand`, avec une teinte plus foncée dérivée pour `--brand-dark`) sur l'ensemble de l'app.

## Architecture

Toujours un seul fichier statique `index.html` (HTML + CSS + JS inline), aucun backend, aucun build. Trois clés `localStorage` :

- `avisRelance.config` — étendue :
  ```
  {
    businessName: string,
    reviewLink: string,
    templates: [{ name: string, text: string }],
    activeTemplateName: string,
    brandColor: string,       // ex: "#1d6fd6"
    logoDataUrl: string | null
  }
  ```
- `avisRelance.counter` — inchangée (entier).
- `avisRelance.history` — nouvelle :
  ```
  [{ prenom: string, telephone: string, date: string (ISO), modele: string }]
  ```

**Compatibilité avec les installations existantes** : au chargement, si une config V1 est détectée (`messageTemplate` en string plutôt que `templates` en liste), elle est migrée à la volée en un seul modèle nommé "Par défaut" dans `templates`. Pas de système de version formel — projet à faible enjeu, une seule migration simple suffit.

## Écrans

### Écran principal (étendu)

- En-tête : logo (si configuré) + nom du commerce, plus 3 icônes : ⚙️ réglages, 🕘 historique, ▦ QR code
- Compteur "X demandes envoyées" (inchangé)
- Sélecteur de modèle de message (menu déroulant), visible uniquement si plus d'un modèle existe ; mémorise le dernier choix utilisé
- Prénom du client (optionnel), Numéro de téléphone (obligatoire) — inchangés
- Bouton "Envoyer la demande d'avis" :
  - Numéro invalide → erreur inline (inchangé)
  - Numéro déjà présent dans l'historique → bandeau d'avertissement "⚠️ Déjà contacté le JJ/MM/AAAA — Envoyer quand même ?" ; le bouton devient une confirmation, un second clic déclenche l'envoi
  - Sinon → envoi direct (comme en V1)
  - À l'envoi confirmé : construction et ouverture du lien `sms:` (inchangé), incrément du compteur (inchangé), **ajout d'une entrée dans l'historique**

### Écran réglages (étendu)

- Nom du commerce, lien d'avis Google (inchangés)
- Upload logo (`<input type="file" accept="image/*">`), redimensionné automatiquement puis prévisualisé
- Sélecteur de couleur de marque
- Gestion des modèles de message : liste avec nom + texte de chaque modèle, actions ajouter / modifier / supprimer, un modèle marqué "par défaut" (pré-sélectionné à l'envoi suivant si aucun autre choix mémorisé). Au moins un modèle doit toujours exister — suppression du dernier modèle restant bloquée.
- Bouton "Enregistrer" (inchangé dans son rôle)

### Écran historique (nouveau)

- Liste des envois, plus récent en premier : prénom, numéro, date, nom du modèle utilisé
- Lecture seule pour cette itération (pas de suppression/édition d'entrées individuelles)
- Bouton retour vers l'écran principal
- Liste vide → message "Aucun envoi pour l'instant"

### Écran QR code (nouveau)

- Nom du commerce affiché en haut
- Grand QR code généré localement (bibliothèque vendorisée), encodant le lien d'avis Google configuré
- Bouton retour vers l'écran principal
- Si aucun lien d'avis n'est configuré, cet écran n'est pas accessible (config obligatoire avant toute utilisation, comme en V1)

## Détails techniques

- **Génération QR code** : bibliothèque `qrcode-generator` de Kazuhiko Arase (licence MIT, fichier JS unique, aucune dépendance), copiée intégralement dans `index.html`, aucun appel réseau à l'exécution.
- **Redimensionnement logo** : lecture du fichier via `FileReader`, dessin sur un `<canvas>` à une largeur max (~240px, hauteur proportionnelle), export en data URL (`canvas.toDataURL('image/jpeg', 0.8)` ou équivalent) avant stockage.
- **Couleur de marque** : la teinte foncée dérivée (`--brand-dark`, utilisée pour l'état actif des boutons) est calculée à partir de la couleur choisie (assombrissement programmatique, ex. -15% de luminosité), pas de second sélecteur.
- **Détection de doublon** : comparaison sur le numéro nettoyé (`cleanPhone`), recherche dans `avisRelance.history`.
- **Pas de tests automatisés prévus** — cohérent avec la V1 : vérification manuelle en navigateur, puis test réel sur téléphone.

## Hors périmètre (cette itération)

Reporté à une prochaine mise à jour, comme convenu :

- Bouton WhatsApp en complément du SMS
- Installation en app mobile (PWA / manifest)
- Édition ou suppression d'entrées individuelles dans l'historique
- Export de l'historique (CSV, etc.)

## Notes pour la suite

- Le dossier projet est maintenant `Desktop\relance avis google nouvelle version\` (renommé depuis `Desktop\avis-relance\`, fusion effectuée le 2026-08-26).
- Un dépôt git existe déjà pour ce projet (contrairement à ce qu'indiquait la note de la V1) — les commits habituels s'appliquent normalement pour ce plan.
