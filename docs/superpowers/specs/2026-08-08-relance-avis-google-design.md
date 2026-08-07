# Relance d'avis Google — Design

**Date** : 2026-08-08
**Statut** : approuvé, prêt pour plan d'implémentation

## Contexte

Recoupe l'idée #2 de l'initiative outils-pme (`Desktop\outils-pme\02-avis-google.md`), mais recentré : cette fiche couvrait surveillance des avis existants + réponse pré-rédigée. Ce projet couvre l'autre moitié — la sollicitation proactive d'avis juste après le passage d'un client, par SMS pré-rempli envoyé manuellement par le commerçant.

Contraintes du portefeuille d'outils PME (voir `Desktop\outils-pme\INDEX.md`) : zéro budget, hébergement gratuit, MVP le plus pauvre possible, pas de manipulation de données sensibles tant que non nécessaire. Frank est en reprise d'activité professionnelle, budget serré à court terme.

## Objectif

Un outil mono-commerce (une instance configurée par commerçant, vendable en porte-à-porte comme Tontino) qui laisse le commerçant, juste après chaque client, déclencher l'envoi d'un SMS pré-rempli demandant un avis Google — en un geste, sans coût récurrent, sans backend.

## Décisions de design

Ces choix ont été validés en session de brainstorming (voir échange précédent) :

- **Mécanisme d'envoi** : lien `sms:` pré-rempli ouvrant l'appli SMS native du téléphone du commerçant, envoi manuel (pas de passerelle SMS payante type Twilio).
- **Portée** : mono-commerce. Chaque commerçant configure sa propre instance (nom du commerce, lien Google Avis). Pas d'authentification, pas de multi-tenant.
- **Flux d'usage** : saisie manuelle, un client à la fois, au comptoir. Pas d'import de liste, pas de QR code en V1.
- **Configuration du lien Google Avis** : le commerçant le colle lui-même au premier lancement (trouvé via Google Business Profile ou Google Maps). Pas d'appel API Google Business.
- **Suivi** : un compteur simple ("X demandes envoyées"), stocké localement. Aucune donnée personnelle de client conservée.

## Architecture

Une seule page HTML statique (`index.html`), sur le modèle de Tontino (`Desktop\tontine\index.html`) : mobile-first, français, CSS inline, aucune dépendance externe, aucun build. Toutes les données (config + compteur) vivent dans le `localStorage` du navigateur du commerçant — rien ne transite par un serveur.

Déploiement : fichier statique, hébergeable gratuitement (Vercel/Netlify/page statique) ou simplement ouvert depuis le téléphone du commerçant et ajouté à l'écran d'accueil.

Dossier projet : `Desktop\avis-relance\`.

## Écrans

### 1. Configuration (premier lancement uniquement)

Affiché automatiquement si aucune config n'existe en `localStorage`. Champs :

- **Nom du commerce** (texte libre)
- **Lien Google Avis** (obligatoire — URL vers la page "laisser un avis" de la fiche Google du commerce)
- **Modèle de message** (texte pré-rempli avec un exemple par défaut, modifiable) :
  > Merci pour votre visite chez [Commerce] ! Un petit avis Google nous aiderait beaucoup : [lien]

  Le modèle supporte deux emplacements réservés : `{prenom}` (remplacé par le prénom saisi, ou supprimé proprement si vide) et `{lien}` (remplacé par le lien Google Avis configuré). Le nom du commerce est injecté directement dans le modèle par défaut, pas via un placeholder séparé.

Un bouton "Enregistrer" sauvegarde la config en `localStorage` et bascule sur l'écran principal. Cette config reste modifiable ensuite via une icône réglages en haut de l'écran principal.

### 2. Écran principal (usage quotidien)

- Compteur en haut : "X demandes envoyées" (0 au départ)
- Champ **Prénom du client** (optionnel)
- Champ **Numéro de téléphone** (obligatoire, clavier numérique sur mobile — `inputmode="tel"`)
- Bouton **"Envoyer la demande d'avis"**

Au clic, avec un numéro valide renseigné :
1. Construction du message final à partir du modèle configuré (remplacement des placeholders)
2. Construction du lien `sms:` avec le numéro et le message encodé (voir section Détails techniques pour la compatibilité iOS/Android)
3. Navigation vers ce lien — ouvre l'appli SMS native avec le message pré-rempli, le commerçant relit et appuie sur Envoyer lui-même
4. Le compteur local est incrémenté d'un cran (c'est une approximation : on compte le geste d'ouverture du SMS, pas la confirmation d'envoi réelle — impossible à vérifier depuis une page web, et suffisant pour l'usage visé)
5. Le formulaire se vide, prêt pour le client suivant

Validation minimale avant envoi : le numéro doit contenir au moins 10 chiffres (formats français acceptés : `06...`, `07...`, `+336...`, espaces/points tolérés et nettoyés avant construction du lien). Si invalide, message d'erreur inline, pas d'ouverture du lien `sms:`.

## Détails techniques

- **Lien `sms:`** : la syntaxe du paramètre de corps diffère entre iOS (`sms:NUMERO&body=...`) et Android (`sms:NUMERO?body=...`). Détection via un test sur `navigator.userAgent` (`/iPhone|iPad|iPod/`) : séparateur `&` si iOS détecté, `?` sinon (Android et fallback desktop).
- **Encodage** : le message (avec accents, apostrophes, emoji éventuels) doit être encodé avec `encodeURIComponent`.
- **Stockage `localStorage`** : deux clés — une pour la config (`{ businessName, reviewLink, messageTemplate }`), une pour le compteur (entier). Pas d'autre donnée persistée.
- **Pas de tests automatisés prévus** — page statique simple, vérification manuelle en navigateur (desktop pour la mise en page, puis test réel sur téléphone pour confirmer l'ouverture de l'appli SMS).

## Hors périmètre (V1)

Explicitement écarté pour rester au MVP le plus pauvre possible, sans fermer la porte à une V2 :

- Bouton WhatsApp en complément du SMS (même mécanique `wa.me`, ajout facile plus tard si demandé par un prospect)
- QR code à afficher en caisse pour auto-relance par le client
- Historique détaillé par client (impliquerait de conserver des données personnelles — question RGPD à traiter si ce besoin apparaît)
- Génération/récupération automatique du lien Google Avis via l'API Google Business Profile
- Envoi automatique via passerelle SMS payante

## Notes pour la suite

- À la fin de l'implémentation, mettre à jour `Desktop\outils-pme\02-avis-google.md` pour distinguer clairement les deux volets (surveillance/réponse aux avis existants vs relance proactive post-client) et pointer vers `Desktop\avis-relance\`.
- Pas de dépôt git initialisé pour ce projet, cohérent avec le précédent Tontino (outil statique simple, pas de suivi de version jugé nécessaire à ce stade).
