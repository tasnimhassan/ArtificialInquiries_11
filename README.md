# ArtificialInquiries_11 — Tracking Shifts

Outil numérique issu de l'**exercice 11 « Tracking Shifts »** du vademecum
*Artificial Inquiries – A Vademecum for Workers in the Age of AI* (Gabriel Alcaras, sept. 2025),
bloc n°03 « Prompting » (pp. 81-83 du livre).

Page de l'outil : <https://tasnimhassan.github.io/ArtificialInquiries_11/response.html>

## 1. Description de l'exercice

> « Reflect on your experience with LLMs since you began the exercises. Browse through your
> vademecum and your conversation history in your LLM of choice. »

Consignes (Brief 02 – *How to Describe Your Practices*) :

- **Going wide** : couvrir tous les aspects de la pratique (technique, professionnel, émotionnel, éthique…).
- **Documenting your practice** : appuyer chaque réponse sur des **exemples précis**.

Deux questions ouvertes (texte original du vademecum, puis version proposée par l'outil) :

1. *How has your practice evolved?* → « Comment votre pratique a-t-elle évolué ? »
2. *What aspects of your practice have remained consistent?* → « Quels aspects de votre pratique sont restés constants ? »

## 2. Transformation en outil numérique

Le formulaire papier est une page blanche : l'outil numérique doit apporter de la **structure**
et de la **traçabilité** (exemples documentés, liens vers les conversations LLM, dimensions de la pratique).

### Fonctionnalités

**Disponibles**

- Saisie d'une **réflexion** (titre, type évolution / constance, question, réponse avec exemples précis, date, version).
- **Créer, lire, modifier et supprimer** des réflexions dans l'Omeka S commun.
- Quatre modes de connexion : **CAS Paris 8**, **Google**, **GitHub**, ou **clés API Omeka S** personnelles.
- Mode **lecture seule** sans connexion (bouton « Voir sans clés »).
- Interface et questions en français.

**Prévues**

- Découpage de chaque réponse en **observations** rattachées à une ou plusieurs **dimensions**
  (technique, professionnelle, émotionnelle, éthique, autre).
- Pour chaque observation : au moins un **exemple documenté** (extrait de conversation, capture, date, lien
  vers un exercice du vademecum).
- **Frise chronologique** des évolutions (avant / après le début des exercices).
- Export JSON-LD / CSV.
- Lien entre une réflexion et son auteur (`Participant`).

### Solution technique

| Brique | Choix | Rôle |
|---|---|---|
| Stockage / publication | **Omeka S** (Apache + PHP + MySQL) | Items, vocabulaires, médias |
| Modèle de données | Resource templates Omeka S (voir `diagram_class.md`) | `Participant`, `Reflexion`, `Observation`, `Exemple`… |
| Saisie | Page HTML/JS `response.html` + API REST Omeka S (`/api/items`) | Formulaire sur mesure, hébergé sur GitHub Pages |
| Relais | **Cloudflare Worker** `worker.js` | Autorisation CORS, vérification des connexions, écriture dans Omeka S |
| Connexions | **CAS Paris 8**, **Firebase Authentication** (Google, GitHub), clés API Omeka S | Identifier la personne qui écrit |
| Références | **Zotero** (groupe `THYP_26-27`) | Références & annotations (vademecum, protocole) |
| Visualisation | Mermaid / timeline JS | Frise des évolutions (prévue) |
| Versionnement | GitHub | Dépôt `ArtificialInquiries_11` |

Alternative écartée pour la saisie : modules Omeka S *Contribute* ou *Webform*.

### Fonctionnement des connexions

La page ne peut pas appeler directement l'Omeka de l'université (pas de CORS). Elle passe par le Worker,
qui relaie les requêtes.

| Connexion | Création / modification / suppression | Clés Omeka utilisées |
|---|---|---|
| CAS Paris 8 | Oui | Clés du projet (secrets du Worker) |
| Google | Oui | Clés du projet (secrets du Worker) |
| GitHub | Oui | Clés du projet (secrets du Worker) |
| Clés API Omeka S | Oui | Clés personnelles de la personne |
| Sans connexion | Non (lecture seule) | — |

Pour le CAS, la page redirige vers `cas.univ-paris8.fr` avec `renew=true` : l'identifiant et le mot de passe
sont toujours redemandés. Le CAS renvoie un ticket que le Worker vérifie, puis le Worker remet à la page un jeton
de session valable **1 heure**.

Le Worker n'autorise l'écriture que sur les items du modèle **« Artificial Inquiries – Réflexion »**.
Les autres ressources restent en lecture seule.

Le détail est dans `diagram_class.md` (architecture et séquence).

## 3. Déploiement

### 3.1 Installation locale d'Omeka S

1. Installer Apache, PHP ≥ 8.1 (extensions `pdo_mysql`, `xml`, `intl`, `gd`/`imagick`), MySQL/MariaDB.
2. Créer la base : `CREATE DATABASE omekas CHARACTER SET utf8mb4;`
3. Télécharger Omeka S : <https://omeka.org/s/download/>, décompresser dans le dossier web.
4. Renseigner `config/database.ini` (user, password, dbname, host).
5. `chmod -R 775 files/` et activer `mod_rewrite`.
6. Ouvrir `http://localhost/omeka-s/install` puis créer le compte administrateur.
7. Tester l'API : `http://localhost/omeka-s/api/items`.

### 3.2 Page web (GitHub Pages)

1. Activer GitHub Pages sur le dépôt.
2. Dans `response.html`, vérifier `WORKER` (adresse du Worker) et `FIREBASE_CONFIG` (identifiants publics du projet Firebase).

### 3.3 Worker Cloudflare

1. Créer un Worker et y coller `worker.js`.
2. Vérifier en haut du fichier : `OMEKA`, `CAS`, `ORIGIN` (adresse GitHub Pages) et `FIREBASE_PROJECT`.
3. Dans **Settings → Variables and Secrets**, ajouter deux **secrets** :
   - `OMEKA_KEY_IDENTITY` : key identity d'un compte Omeka S autorisé à écrire ;
   - `OMEKA_KEY_CREDENTIAL` : key credential associée.
   (Clés créées dans Omeka S : compte → onglet **Clés API**.)
4. Cliquer sur **Deploy**.

Sans ces deux secrets, l'écriture avec le CAS, Google ou GitHub échoue avec « Le serveur n'est pas configuré pour l'écriture ».

### 3.4 Firebase (Google et GitHub)

Dans Firebase Authentication : activer les fournisseurs Google et GitHub, et ajouter `tasnimhassan.github.io`
dans **Settings → Authorized domains**.

### 3.5 CAS Paris 8

L'adresse de la page doit être déclarée comme service autorisé sur le CAS de l'université
(demande au service informatique de Paris 8) :
`https://tasnimhassan.github.io/ArtificialInquiries_11/response.html`

## 4. Sécurité

**Ne jamais écrire les clés API dans un fichier du dépôt.** Elles se placent uniquement dans les secrets Cloudflare.

## 5. Structure du dépôt

```
ArtificialInquiries_11/
├── README.md
├── diagram_class.md              # diagramme de classes, architecture, séquence
├── response.html                 # page de saisie (GitHub Pages)
├── worker.js                     # relais Cloudflare (CORS, connexions, écriture)
├── artificial_inquiries.ttl      # vocabulaire (préfixe artinq)
├── resource-template/            # modèles de ressources Omeka S (JSON)
│   ├── artinq_Participant.json
│   ├── artinq_Reflexion.json
│   ├── artinq_Observation.json
│   ├── artinq_Dimension.json
│   ├── artinq_Exemple.json
│   ├── artinq_Exercice.json
│   └── artinq_Media.json
└── omk.sql                       # export de la base
```
