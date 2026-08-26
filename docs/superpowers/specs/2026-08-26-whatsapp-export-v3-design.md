# Relance d'avis Google — Bouton WhatsApp & export d'historique (V3) — Design

**Date** : 2026-08-26
**Statut** : approuvé, prêt pour plan d'implémentation

## Contexte

Suite à la V2 (voir `2026-08-26-personnalisation-v2-design.md`) qui ajoutait modèles multiples, logo/couleur, historique et QR code. Deux fonctionnalités avaient été explicitement reportées à une prochaine mise à jour : le bouton WhatsApp en complément du SMS, et l'export de l'historique. C'est l'objet de cette itération.

## Décisions de design

- **WhatsApp** : second bouton d'envoi sur l'écran principal, en complément du bouton SMS existant (renommé "Envoyer par SMS" pour la clarté) — pas de remplacement, pas de choix de canal par défaut dans les réglages.
- **Conversion du numéro pour WhatsApp** : les liens `wa.me` exigent un format international sans le `0` initial (ex. `33612345678`). Un numéro commençant par `0` est automatiquement converti en `33...` pour la construction du lien WhatsApp uniquement — le numéro utilisé pour le SMS n'est pas affecté. Hypothèse volontairement limitée aux numéros français standards (cohérent avec le public visé : commerçants en France).
- **Détection de doublon** : s'applique de la même façon quel que soit le bouton cliqué (basée sur le numéro de téléphone, indépendante du canal).
- **Compteur** : reste unique et cumule les deux canaux — pas de compteur séparé SMS/WhatsApp.
- **Historique** : chaque entrée gagne un champ `canal` (`SMS` ou `WhatsApp`), affiché sur l'écran historique et inclus dans l'export.
- **Export CSV** : bouton "Exporter en CSV" sur l'écran historique, désactivé si l'historique est vide. Colonnes : prénom, téléphone, date, modèle, canal. Génération et téléchargement 100% locaux (`Blob` + lien temporaire), aucune dépendance externe, cohérent avec l'architecture existante.

## Architecture

Toujours un seul fichier statique `index.html`, aucun backend, aucun build. Extension du schéma `avisRelance.history` existant :

```
[{ prenom: string, telephone: string, date: string (ISO), modele: string, canal: "SMS" | "WhatsApp" }]
```

**Compatibilité avec l'historique existant** : les entrées déjà enregistrées par la V2 n'ont pas de champ `canal`. À la lecture (`getHistory`), une entrée sans `canal` est traitée comme `"SMS"` par défaut (c'était le seul canal disponible avant cette itération) — pas de migration en écriture nécessaire, cohérent avec l'approche de lecture défensive déjà utilisée pour la config.

## Écrans

### Écran principal (étendu)

- Les champs prénom/téléphone/modèle restent inchangés et partagés entre les deux canaux.
- Bouton "Envoyer par SMS" (existant, juste renommé).
- Nouveau bouton "Envoyer par WhatsApp" juste en dessous, même style visuel (`.btn`), couleur de marque appliquée aux deux.
- Le clic sur l'un ou l'autre déclenche : la même vérification de doublon déjà en place, l'incrément du même compteur, l'ajout d'une entrée d'historique avec le `canal` correspondant, puis l'ouverture du lien (`sms:` ou `https://wa.me/...`).

### Écran historique (étendu)

- Chaque ligne affiche désormais aussi le canal (ex. "Marie — 0612345678 · SMS" ou "· WhatsApp").
- Bouton "Exporter en CSV" en haut de l'écran, à côté du titre ou juste en dessous — désactivé (grisé, non cliquable) si la liste est vide.

## Détails techniques

- **Construction du numéro WhatsApp** : `toWhatsAppNumber(cleanedNumber)` — si le numéro nettoyé commence par `0`, remplace ce `0` par `33` ; sinon (déjà au format `+33...` ou `33...` ou autre), le laisse tel quel après avoir retiré un éventuel `+` initial.
- **Lien WhatsApp** : `https://wa.me/<numéro international>?text=<message encodé via encodeURIComponent>`.
- **Lecture rétrocompatible du canal** : `entry.canal || 'SMS'` partout où une entrée d'historique est affichée ou exportée.
- **Export CSV** : construit une chaîne CSV (en-tête + une ligne par entrée, champs échappés selon les règles CSV standard — guillemets doublés si le champ contient une virgule, un guillemet ou un retour à la ligne), préfixée d'un BOM UTF-8 (`﻿`) pour un rendu correct des accents à l'ouverture dans Excel. Téléchargement déclenché via `Blob` + `URL.createObjectURL` + un élément `<a download="historique.csv">` temporaire, cohérent avec le principe "aucune donnée ne quitte l'appareil" (le fichier est généré et téléchargé localement, jamais envoyé à un serveur).
- **Pas de tests automatisés prévus** — cohérent avec la V1/V2 : vérification manuelle en navigateur.

## Hors périmètre (cette itération)

- Édition ou suppression d'entrées individuelles de l'historique (déjà hors périmètre en V2).
- Choix du canal par défaut dans les réglages.
- Support de numéros internationaux autres que français pour la conversion WhatsApp.
- Historique borné/purgé automatiquement (pas de cap sur le nombre d'entrées).

## Notes pour la suite

- Dossier projet inchangé : `Desktop\relance avis google nouvelle version\`.
- Dépôt git existant, commits normaux à chaque tâche du plan.
