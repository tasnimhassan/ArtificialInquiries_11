# Diagrammes — Ex11 « Tracking Shifts »

Ce fichier contient trois diagrammes :

1. le **diagramme de classes** du modèle de données ;
2. le **schéma d’architecture** (page web, Worker Cloudflare, Omeka S, authentification) ;
3. le **diagramme de séquence** d’une écriture (créer, modifier, supprimer une réflexion).

## 1. Diagramme de classes

```mermaid
classDiagram
    class Participant {
        +int id
        +String login
        +String profession
        +String llmUtilise
    }

    class Reflexion {
        +int id
        +String titre
        +Date dateSaisie
        +TypeReflexion type
        +String question
        +String texte
        +int version
    }

    class Observation {
        +int id
        +String resume
        +Periode periode
        +Date dateObservation
    }

    class Dimension {
        +int id
        +String libelle
    }

    class Exemple {
        +int id
        +String description
        +Date date
        +String extraitConversation
        +String urlConversation
    }

    class Exercice {
        +String numero
        +String titre
        +int pageVademecum
    }

    class Media {
        +int id
        +String fichier
        +String typeMime
    }

    class TypeReflexion {
        <<enumeration>>
        EVOLUTION
        CONSTANCE
    }

    class Periode {
        <<enumeration>>
        AVANT
        PENDANT
        APRES
    }

    Participant "1" --> "0..*" Reflexion : rédige
    Reflexion "1" *-- "1..*" Observation : contient
    Observation "0..*" --> "1..*" Dimension : concerne
    Observation "1" *-- "1..*" Exemple : documentée par
    Exemple "0..*" --> "0..1" Exercice : renvoie à
    Exemple "1" o-- "0..*" Media : illustré par
    Reflexion --> TypeReflexion
    Observation --> Periode
```

### État d’avancement

| Classe | État | Remarque |
|---|---|---|
| `Reflexion` | **Implémentée** | Créer, lire, modifier, supprimer depuis `response.html` |
| `Participant` | Prévue | Pas encore de lien auteur ↔ réflexion |
| `Observation`, `Exemple` | Prévues | Pour l’instant, le texte de la réponse contient directement les exemples |
| `Dimension`, `Exercice`, `Media` | Prévues | — |

### Correspondance `Reflexion` ↔ propriétés Omeka S

| Attribut | Propriété Omeka S (essayée dans cet ordre) | Champ de la page |
|---|---|---|
| `titre` | `dcterms:title` | Titre |
| `dateSaisie` | `artinq:date`, sinon `artinq:dateSaisie` | Date de saisie |
| `type` | `artinq:type`, sinon `artinq:typeReflexion` | Type |
| `question` | `artinq:description`, sinon `artinq:question` | Question |
| `texte` | `artinq:texte` | Votre réponse |
| `version` | `artinq:version` | Version |

Classe de ressource : `artinq:Reflexion`. Modèle de ressource : « Artificial Inquiries – Réflexion ».

Questions proposées par défaut (valeurs de `question`) :

- `EVOLUTION` : « Comment votre pratique a-t-elle évolué ? »
- `CONSTANCE` : « Quels aspects de votre pratique sont restés constants ? »

## Dimensions prévues (valeurs initiales)

`technique`, `professionnelle`, `émotionnelle`, `éthique`, `sociale`, `autre`

## 2. Architecture

```mermaid
flowchart LR
    U[Personne connectée] --> P[Page response.html<br/>GitHub Pages]
    P -->|redirection| CAS[CAS Paris 8]
    P -->|fenêtre pop-up| FB[Firebase Auth<br/>Google / GitHub]
    P -->|requêtes API<br/>+ jeton ou clés personnelles| W[Worker Cloudflare<br/>artinq-relais]
    W -->|vérifie le ticket| CAS
    W -->|vérifie le jeton| GK[Clés publiques Google]
    W -->|API REST<br/>clés du projet ou clés personnelles| O[(Omeka S commun<br/>humanum-p8.fr)]
```

Le Worker ajoute l’autorisation CORS que le serveur de l’université ne donne pas, vérifie l’identité, puis relaie vers Omeka S.

| Connexion | Preuve envoyée au Worker | Clés Omeka utilisées pour écrire |
|---|---|---|
| CAS Paris 8 | Jeton signé par le Worker, valable 1 heure | Clés du projet (secrets du Worker) |
| Google / GitHub | Jeton d’identité Firebase, renouvelé automatiquement | Clés du projet (secrets du Worker) |
| Clés API Omeka S | En-têtes `X-Omeka-Key-Identity` et `X-Omeka-Key-Credential` | Clés personnelles de la personne |

## 3. Séquence d’une écriture (CAS Paris 8)

```mermaid
sequenceDiagram
    actor U as Personne
    participant P as response.html
    participant C as CAS Paris 8
    participant W as Worker Cloudflare
    participant O as Omeka S

    U->>P: Clique sur « Continuer vers Paris 8 »
    P->>C: Redirection (login?renew=true&service=…)
    U->>C: Saisit identifiant et mot de passe
    C->>P: Retour avec ?ticket=ST-…
    P->>W: GET /cas/validate?ticket=…&service=…
    W->>C: serviceValidate
    C-->>W: Identifiant validé
    W-->>P: Jeton de session signé (1 h)
    U->>P: Crée, modifie ou supprime une réflexion
    P->>W: POST / PUT / DELETE /items (Authorization: Bearer …)
    W->>W: Vérifie le jeton, le modèle « Réflexion » et la ressource
    W->>O: Même requête avec les clés du projet
    O-->>W: Réponse
    W-->>P: Réponse (avec CORS)
```

Les connexions Google et GitHub suivent le même schéma, sans la redirection CAS : la page envoie directement le jeton Firebase au Worker.
