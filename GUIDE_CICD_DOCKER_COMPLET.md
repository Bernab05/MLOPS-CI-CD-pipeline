# Guide Complet : CI/CD, Docker & Pipelines de Deploiement
## Du Code Source au Cloud - De GitHub a Docker Hub et AWS

---

## Table des Matieres

1. [Introduction aux concepts fondamentaux](#1-introduction-aux-concepts-fondamentaux)
2. [Comprendre Docker en profondeur](#2-comprendre-docker-en-profondeur)
3. [Comprendre CI/CD](#3-comprendre-cicd)
4. [GitHub Actions : Le moteur du pipeline](#4-github-actions--le-moteur-du-pipeline)
5. [Analyse detaillee des fichiers du projet](#5-analyse-detaillee-des-fichiers-du-projet)
6. [Le Pipeline CI/CD complet : GitHub → Docker Hub](#6-le-pipeline-cicd-complet--github--docker-hub)
7. [Deploiement sur AWS (ECS/ECR)](#7-deploiement-sur-aws-ecsecr)
8. [Comparaison Docker Hub vs AWS ECR](#8-comparaison-docker-hub-vs-aws-ecr)
9. [Guide pratique pas a pas](#9-guide-pratique-pas-a-pas)
10. [Troubleshooting et bonnes pratiques](#10-troubleshooting-et-bonnes-pratiques)

---

## 1. Introduction aux concepts fondamentaux

### Qu'est-ce qu'un pipeline de deploiement ?

Un pipeline de deploiement est une **chaine automatisee** qui prend votre code source
et le transforme en application deployee, accessible aux utilisateurs.

```
 VISION GLOBALE DU PIPELINE
 ===========================

 +-------------+      +----------------+      +----------------+      +------------------+
 |             |      |                |      |                |      |                  |
 |  DEVELOPER  |----->|    GITHUB      |----->|  GITHUB        |----->|   DOCKER HUB     |
 |  (code)     |      |  (repository)  |      |  ACTIONS       |      |   ou AWS ECR     |
 |             |      |                |      |  (CI/CD)       |      |   (registre)     |
 +-------------+      +----------------+      +----------------+      +------------------+
                                                                             |
       +----------+      +----------------+                                  |
       |          |      |                |                                  |
       |  USERS   |<-----|   SERVEUR      |<---------------------------------+
       |          |      |  (AWS ECS,     |         deploiement
       +----------+      |   VM, etc.)    |
                         +----------------+

 Legende:
 --------
 DEVELOPER   = Vous ecrivez du code et le poussez sur GitHub
 GITHUB      = Heberge votre code source (repository distant)
 ACTIONS     = Execute automatiquement tests + build + deploiement
 DOCKER HUB  = Stocke vos images Docker (comme un "app store" d'images)
 AWS ECR     = Equivalent Docker Hub chez Amazon (privee, integree AWS)
 SERVEUR     = La ou votre application tourne pour les utilisateurs
```

### Les 3 piliers de ce guide

```
          +------------------+
          |   1. DOCKER      |
          |   Containeriser  |
          |   l'application  |
          +--------+---------+
                   |
          +--------v---------+
          |   2. CI/CD       |
          |   Automatiser    |
          |   le pipeline    |
          +--------+---------+
                   |
          +--------v---------+
          |   3. CLOUD       |
          |   Deployer       |
          |   (DockerHub/AWS)|
          +------------------+
```

---

## 2. Comprendre Docker en profondeur

### 2.1 Pourquoi Docker ?

**Le probleme classique** : "Ca marche sur ma machine !"

Sans Docker :
```
 Developpeur A             Developpeur B              Serveur Production
 +-----------------+       +-----------------+        +-----------------+
 | Python 3.10     |       | Python 3.9      |        | Python 3.8      |
 | Flask 3.1       |       | Flask 2.0       |        | Flask 3.0       |
 | Ubuntu 22.04    |       | macOS 13        |        | Amazon Linux 2  |
 +-----------------+       +-----------------+        +-----------------+
       MARCHE !                BUG ETRANGE             CRASH AU LANCEMENT

 --> Chaque environnement est different = problemes garantis
```

Avec Docker :
```
 Developpeur A             Developpeur B              Serveur Production
 +-----------------+       +-----------------+        +-----------------+
 | +-------------+ |       | +-------------+ |        | +-------------+ |
 | | CONTAINER   | |       | | CONTAINER   | |        | | CONTAINER   | |
 | | Python 3.10 | |       | | Python 3.10 | |        | | Python 3.10 | |
 | | Flask 3.1   | |       | | Flask 3.1   | |        | | Flask 3.1   | |
 | | Debian slim | |       | | Debian slim | |        | | Debian slim | |
 | +-------------+ |       | +-------------+ |        | +-------------+ |
 | Ubuntu 22.04    |       | macOS 13        |        | Amazon Linux 2  |
 +-----------------+       +-----------------+        +-----------------+
       MARCHE !                 MARCHE !                   MARCHE !

 --> Le container est IDENTIQUE partout = comportement garanti
```

### 2.2 Vocabulaire essentiel Docker

```
 HIERARCHIE DES CONCEPTS DOCKER
 ================================

 +------------------+
 |   DOCKERFILE     |  <-- Recette de cuisine (fichier texte)
 |   (instructions) |      "Comment construire l'image"
 +--------+---------+
          |
          | docker build
          v
 +------------------+
 |   IMAGE          |  <-- Le plat prepare, congele (immutable)
 |   (template)     |      "Modele pret a l'emploi"
 +--------+---------+
          |
          | docker run
          v
 +------------------+
 |   CONTAINER      |  <-- Le plat servi, en cours d'utilisation
 |   (instance)     |      "Application qui tourne"
 +------------------+

 +------------------+
 |   REGISTRY       |  <-- Le supermarche de plats congeles
 |   (Docker Hub,   |      "Stockage et distribution d'images"
 |    AWS ECR)      |
 +------------------+
```

| Concept       | Analogie                    | Description                                           |
|---------------|----------------------------|-------------------------------------------------------|
| **Dockerfile** | Recette de cuisine         | Fichier texte avec les instructions pour creer l'image |
| **Image**      | Plat congele               | Template immutable, prete a etre executee             |
| **Container**  | Plat servi au client       | Instance en cours d'execution d'une image             |
| **Registry**   | Supermarche / entrepot     | Plateforme de stockage et distribution d'images       |
| **Tag**        | Etiquette sur le produit   | Version/identifiant d'une image (ex: `v1.0`, `latest`) |

### 2.3 Le Dockerfile de notre projet - Ligne par ligne

Voici le Dockerfile du dossier `pipeline/` avec l'explication de chaque instruction :

```dockerfile
# ----- LIGNE 1 : IMAGE DE BASE -----
FROM python:3.10-slim
#
# FROM = "A partir de quelle image de base construire"
# python:3.10-slim = image officielle Python 3.10, version allegee (slim)
#
# Pourquoi "slim" ? L'image complete fait ~900MB, la slim fait ~150MB.
# Elle contient Python + les outils essentiels, sans les extras inutiles.
#
# Analogie : choisir le modele de base de sa voiture avant de la customiser


# ----- LIGNE 2 : REPERTOIRE DE TRAVAIL -----
WORKDIR /app
#
# WORKDIR = "Dans quel dossier travailler a l'interieur du container"
# /app = cree le dossier /app et s'y positionne
#
# Toutes les commandes suivantes s'executeront dans /app
# Analogie : cd /app  mais en creant le dossier s'il n'existe pas


# ----- LIGNE 3 : COPIER LES FICHIERS -----
COPY . /app/
#
# COPY [source locale] [destination dans le container]
# .     = tout le repertoire courant (la ou se trouve le Dockerfile)
# /app/ = destination dans le container
#
# Cette commande copie TOUS vos fichiers (app.py, model, templates, etc.)
# SAUF ceux listes dans .dockerignore
# Analogie : copier tous vos documents dans une valise


# ----- LIGNE 4 : INSTALLER LES DEPENDANCES -----
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir --trusted-host pypi.python.org -r requirements.txt
#
# RUN = executer une commande pendant la construction de l'image
# --no-cache-dir   = ne pas garder le cache pip (image plus legere)
# --upgrade pip    = mettre a jour pip d'abord
# -r requirements.txt = installer toutes les libs listees dans ce fichier
#
# Le "&&" enchaine les 2 commandes en une seule couche (layer) Docker
# Analogie : installer tous les logiciels necessaires sur un nouvel ordinateur


# ----- LIGNE 5 : COMMANDE DE LANCEMENT -----
CMD ["python", "app.py"]
#
# CMD = commande executee quand le container demarre
# ["python", "app.py"] = lance l'application Flask
#
# ATTENTION : CMD vs RUN
#   RUN  = s'execute pendant le BUILD (construction de l'image)
#   CMD  = s'execute pendant le RUN (demarrage du container)
#
# Analogie : RUN = installer l'appli, CMD = lancer l'appli
```

### 2.4 Le fichier .dockerignore

```
# Contenu du fichier .dockerignore :
test.py          # Les tests ne sont pas necessaires en production
readme.md        # La documentation n'a pas besoin d'etre dans l'image
.git*            # Le dossier .git est inutile et TRES lourd
/venv            # L'environnement virtuel local (on installe via requirements.txt)
```

```
 COMMENT .dockerignore FONCTIONNE
 ==================================

 Dossier projet (source)              Image Docker (destination)
 +---------------------------+        +---------------------------+
 |  app.py               [x] |  COPY  |  app.py                   |
 |  app_monitoring.py    [x] | -----> |  app_monitoring.py        |
 |  requirements.txt     [x] |        |  requirements.txt         |
 |  catboost_model-2.pkl [x] |        |  catboost_model-2.pkl     |
 |  templates/           [x] |        |  templates/               |
 |  Dockerfile           [x] |        |  Dockerfile               |
 |  test.py              [ ] |  EXCLU |                           |
 |  readme.md            [ ] |  EXCLU |                           |
 |  .git/                [ ] |  EXCLU |                           |
 |  venv/                [ ] |  EXCLU |                           |
 +---------------------------+        +---------------------------+

 [x] = copie     [ ] = ignore (grace a .dockerignore)
```

**Pourquoi c'est important ?**
- Reduit la taille de l'image (`.git/` peut faire des centaines de MB)
- Accelere le build (moins de fichiers a copier)
- Securite (ne pas embarquer des fichiers sensibles)

### 2.5 Cycle de vie Docker : Les commandes essentielles

```
 CYCLE DE VIE COMPLET D'UNE IMAGE DOCKER
 ==========================================

                        docker build -t monimage:v1 .
                  +----------------------------------------+
                  |                                        |
  Dockerfile -----+----> IMAGE locale                     |
                         (monimage:v1)                     |
                              |                            |
                  +-----------+-----------+                |
                  |                       |                |
            docker run               docker push           |
                  |                       |                |
                  v                       v                |
            CONTAINER              REGISTRY (Docker Hub)   |
            (instance               (monimage:v1           |
             en cours)               stockee en ligne)     |
                                          |                |
                                    docker pull            |
                                          |                |
                                          v                |
                                    IMAGE locale           |
                                    (sur un autre PC)      |
                                          |                |
                                    docker run             |
                                          |                |
                                          v                |
                                    CONTAINER              |
                                    (sur l'autre PC)       |
                  +----------------------------------------+
```

**Commandes en detail :**

```bash
# 1. CONSTRUIRE une image a partir du Dockerfile
docker build -t mon_user/mon_app:v1 .
#   -t           = tagger l'image avec un nom
#   mon_user/    = votre username Docker Hub
#   mon_app      = nom de l'image
#   :v1          = tag de version
#   .            = contexte de build (dossier courant)

# 2. LISTER les images locales
docker images

# 3. LANCER un container a partir de l'image
docker run -p 5000:5000 mon_user/mon_app:v1
#   -p 5000:5000 = mapper le port 5000 du container au port 5000 de la machine
#   -d           = (optionnel) mode detache (en arriere-plan)

# 4. VOIR les containers en cours
docker ps

# 5. ARRETER un container
docker stop <container_id>

# 6. SE CONNECTER a Docker Hub
docker login -u mon_user -p mon_password

# 7. POUSSER l'image sur Docker Hub
docker push mon_user/mon_app:v1

# 8. TELECHARGER une image depuis Docker Hub
docker pull mon_user/mon_app:v1
```

### 2.6 Comprendre les layers (couches) Docker

```
 STRUCTURE EN COUCHES D'UNE IMAGE DOCKER
 ==========================================

 Chaque instruction du Dockerfile cree une COUCHE (layer).
 Les couches sont mises en cache et reutilisees si inchangees.

 +------------------------------------------+
 | CMD ["python", "app.py"]                 |  Layer 4 (lecture seule)
 +------------------------------------------+
 | RUN pip install -r requirements.txt      |  Layer 3 (lecture seule)
 +------------------------------------------+
 | COPY . /app/                             |  Layer 2 (lecture seule)
 +------------------------------------------+
 | WORKDIR /app                             |  Layer 1 (lecture seule)
 +------------------------------------------+
 | FROM python:3.10-slim                    |  Layer 0 (image de base)
 +------------------------------------------+

 Avantage du cache :
 -------------------
 Si seul app.py change, Docker reutilise les layers 0 et 1
 et ne reconstruit qu'a partir de la layer 2.
 --> Build beaucoup plus rapide !
```

---

## 3. Comprendre CI/CD

### 3.1 Definitions

```
 CI/CD = Continuous Integration / Continuous Delivery (ou Deployment)
 =====================================================================

 +------------------------------------------------------------------+
 |                                                                    |
 |  CI = CONTINUOUS INTEGRATION (Integration Continue)                |
 |  -----------------------------------------------                   |
 |  A chaque push de code :                                          |
 |    1. Le code est recupere                                        |
 |    2. Les dependances sont installees                              |
 |    3. Le code est formate (black)                                 |
 |    4. Le code est analyse/linte (pylint)                          |
 |    5. Les tests sont executes (pytest)                            |
 |                                                                    |
 |  OBJECTIF : Detecter les erreurs LE PLUS TOT POSSIBLE             |
 |                                                                    |
 +------------------------------------------------------------------+
 |                                                                    |
 |  CD = CONTINUOUS DELIVERY / DEPLOYMENT (Livraison Continue)        |
 |  ---------------------------------------------------------        |
 |  Si la CI reussit :                                               |
 |    1. L'image Docker est construite                                |
 |    2. L'image est poussee sur le registre (Docker Hub / ECR)      |
 |    3. (Optionnel) L'app est deployee sur le serveur (AWS ECS)     |
 |                                                                    |
 |  OBJECTIF : Livrer automatiquement une version deployable          |
 |                                                                    |
 +------------------------------------------------------------------+
```

### 3.2 Le flux CI/CD de notre projet

```
 PIPELINE CI/CD DETAILLE DU PROJET
 ====================================

 Developpeur
     |
     |  git push / Pull Request sur 'main'
     v
 +------------------------------------------------------------------------+
 |  GITHUB ACTIONS  (declenchement automatique)                           |
 |                                                                        |
 |  JOB 1 : ci_pipeline (Integration Continue)                           |
 |  +------------------------------------------------------------------+  |
 |  |                                                                  |  |
 |  |   [1] Checkout        Recupere le code source du repository      |  |
 |  |          |                                                       |  |
 |  |          v                                                       |  |
 |  |   [2] Setup Python    Installe Python 3.10                       |  |
 |  |          |                                                       |  |
 |  |          v                                                       |  |
 |  |   [3] Install deps    pip install -r requirements.txt            |  |
 |  |          |                                                       |  |
 |  |          v                                                       |  |
 |  |   [4] Format          black app.py (formate le code)             |  |
 |  |          |                                                       |  |
 |  |          v                                                       |  |
 |  |   [5] Lint            pylint --disable=R,C app.py                |  |
 |  |          |                                                       |  |
 |  |          v                                                       |  |
 |  |   [6] Test            pytest -vv test.py                         |  |
 |  |                                                                  |  |
 |  |   Si ECHEC a n'importe quelle etape --> PIPELINE STOPPE          |  |
 |  |   Si SUCCES partout --> passe au job suivant                     |  |
 |  +------------------------------------------------------------------+  |
 |         |                                                              |
 |         | needs: [ci_pipeline]  (attend que CI soit OK)                |
 |         v                                                              |
 |  JOB 2 : cd_pipeline (Livraison Continue)                              |
 |  +------------------------------------------------------------------+  |
 |  |                                                                  |  |
 |  |   [1] Checkout        Recupere le code source                    |  |
 |  |          |                                                       |  |
 |  |          v                                                       |  |
 |  |   [2] Docker Login    Se connecte a Docker Hub                   |  |
 |  |          |             (avec les secrets GitHub)                  |  |
 |  |          v                                                       |  |
 |  |   [3] Get Date        Genere un tag avec la date actuelle        |  |
 |  |          |             (ex: 2026-02-27--30-15)                   |  |
 |  |          v                                                       |  |
 |  |   [4] Build Image     docker build --tag user/repo:date          |  |
 |  |          |                                                       |  |
 |  |          v                                                       |  |
 |  |   [5] Push Image      docker push user/repo:date                 |  |
 |  |                       --> Image disponible sur Docker Hub         |  |
 |  +------------------------------------------------------------------+  |
 +------------------------------------------------------------------------+
```

### 3.3 Pourquoi chaque etape CI est importante

```
 ETAPES CI ET LEUR ROLE
 ========================

 FORMAT (black)          LINT (pylint)           TEST (pytest)
 +------------------+    +------------------+    +------------------+
 | Uniformise le    |    | Detecte les      |    | Verifie que le   |
 | style du code    |    | erreurs de       |    | code fonctionne  |
 |                  |    | logique, les     |    | correctement     |
 | Exemples :       |    | variables        |    |                  |
 | - indentation    |    | inutilisees,     |    | Exemples :       |
 | - espaces        |    | imports manquants|    | - prediction     |
 | - longueur ligne |    |                  |    |   correcte       |
 |                  |    | --disable=R,C    |    | - API repond     |
 | black app.py     |    | desactive les    |    | - modele charge  |
 |                  |    | conventions (C)  |    |                  |
 |                  |    | et refactoring(R)|    | pytest test.py   |
 +------------------+    +------------------+    +------------------+
        |                        |                       |
        v                        v                       v
   Code lisible            Code sans bugs          Code fonctionnel
   et coherent             potentiels              et valide
```

---

## 4. GitHub Actions : Le moteur du pipeline

### 4.1 Qu'est-ce que GitHub Actions ?

GitHub Actions est un service d'**automatisation** integre a GitHub. Il permet
d'executer du code en reponse a des evenements (push, pull request, etc.)
sur des machines virtuelles fournies par GitHub.

```
 ARCHITECTURE GITHUB ACTIONS
 ==============================

 REPOSITORY GITHUB
 +--------------------------------------------------+
 |                                                    |
 |  .github/                                         |
 |    workflows/                                     |
 |      github-docker-cicd.yaml  <-- Votre pipeline  |
 |      aws.yml                  <-- Pipeline AWS     |
 |                                                    |
 |  pipeline/                                        |
 |    app.py                                         |
 |    Dockerfile                                     |
 |    requirements.txt                               |
 |    test.py                                        |
 |    ...                                            |
 +--------------------------------------------------+
         |
         |  Evenement detecte (push sur main)
         v
 +--------------------------------------------------+
 |  GITHUB ACTIONS RUNNER (machine virtuelle)        |
 |  OS: ubuntu-latest                                |
 |                                                    |
 |  Execute les etapes definies dans le YAML         |
 |  Acces aux SECRETS du repository                   |
 |  Resultat visible dans l'onglet "Actions"          |
 +--------------------------------------------------+
```

### 4.2 Structure d'un fichier workflow YAML

```yaml
# Structure generique expliquee :

name: Nom du workflow          # Nom affiche dans l'interface GitHub

env:                           # Variables d'environnement globales
  MA_VAR: valeur

on:                            # QUAND declencher le pipeline
  push:
    branches: [main]           #   --> a chaque push sur main
  pull_request:
    branches: [main]           #   --> a chaque PR vers main

jobs:                          # LES TRAVAUX A EXECUTER

  mon_job_1:                   # Nom du premier job
    runs-on: ubuntu-latest     # Sur quelle machine
    steps:                     # Liste des etapes
      - uses: action@version   # Utiliser une action pre-faite
      - name: Ma commande      # Etape personnalisee
        run: echo "Hello"      # Commande a executer

  mon_job_2:                   # Deuxieme job
    needs: [mon_job_1]         # S'execute APRES mon_job_1
    runs-on: ubuntu-latest
    steps:
      - name: ...
```

### 4.3 Les Secrets GitHub

Les secrets sont des variables **chiffrees** stockees dans les parametres de votre
repository. Ils ne sont jamais affiches dans les logs.

```
 CONFIGURATION DES SECRETS
 ============================

 GitHub Repository --> Settings --> Secrets and variables --> Actions

 +----------------------------------+
 | Repository Secrets               |
 +----------------------------------+
 | DOCKER_USER     = votre_username |  <-- Votre identifiant Docker Hub
 | DOCKER_PASSWORD = votre_token    |  <-- Token d'acces Docker Hub
 | REPO_NAME       = nom_du_repo   |  <-- Nom du repo sur Docker Hub
 +----------------------------------+

 Pour AWS (si utilise) :
 +----------------------------------+
 | AWS_ACCESS_KEY_ID     = AKIA...  |
 | AWS_SECRET_ACCESS_KEY = wJal...  |
 +----------------------------------+

 Dans le YAML, on y accede via : ${{ secrets.NOM_DU_SECRET }}
```

**Comment creer un secret :**
1. Allez sur votre repository GitHub
2. `Settings` → `Secrets and variables` → `Actions`
3. Cliquez sur `New repository secret`
4. Entrez le nom (ex: `DOCKER_USER`) et la valeur
5. Cliquez sur `Add secret`

---

## 5. Analyse detaillee des fichiers du projet

### 5.1 Arborescence du dossier `pipeline/`

```
 pipeline/
 |
 |-- .github/
 |   |-- workflows/
 |       |-- github-docker-cicd.yaml    # Pipeline CI/CD principal (ACTIF)
 |       |-- aws.yml                    # Pipeline AWS ECS (COMMENTE)
 |
 |-- templates/
 |   |-- index.html                     # Interface web du formulaire
 |
 |-- app.py                             # Application Flask (version simple)
 |-- app_monitoring.py                  # Application Flask + monitoring Arize
 |-- catboost_model-2.pkl               # Modele ML pre-entraine (CatBoost)
 |-- Dockerfile                         # Instructions pour creer l'image Docker
 |-- .dockerignore                      # Fichiers a exclure de l'image Docker
 |-- .gitignore                         # Fichiers a exclure du depot Git
 |-- requirements.txt                   # Dependances Python
 |-- test.py                            # Tests unitaires
 |-- readme.md                          # Documentation du projet
```

### 5.2 app.py - L'application Flask

C'est le **coeur** du projet : une application web de prediction de maladies cardiaques.

```python
from flask import Flask, render_template, request
import pickle
import pandas as pd

# Initialise l'application Flask
app = Flask(__name__)

# Charge le modele CatBoost pre-entraine depuis un fichier binaire
model = pickle.load(open("catboost_model-2.pkl", "rb"))


def model_pred(features):
    """Fonction de prediction utilisee aussi par les tests."""
    test_data = pd.DataFrame([features])
    prediction = model.predict(test_data)
    return int(prediction[0])


@app.route("/", methods=["GET"])
def Home():
    """Page d'accueil : affiche le formulaire HTML."""
    return render_template("index.html")


@app.route("/predict", methods=["POST"])
def predict():
    """Recoit les donnees du formulaire et retourne la prediction."""
    if request.method == "POST":
        # Recupere les 6 parametres medicaux du formulaire
        Age = int(request.form["Age"])
        RestingBP = int(request.form["RestingBP"])
        Cholesterol = int(request.form["Cholesterol"])
        Oldpeak = float(request.form["Oldpeak"])
        FastingBS = int(request.form["FastingBS"])
        MaxHR = int(request.form["MaxHR"])

        # Lance la prediction avec le modele CatBoost
        prediction = model.predict(
            [[Age, RestingBP, Cholesterol, FastingBS, MaxHR, Oldpeak]]
        )

        # Retourne le resultat a l'utilisateur
        if prediction[0] == 1:
            return render_template(
                "index.html",
                prediction_text="Kindly make an appointment with the doctor!",
            )
        else:
            return render_template(
                "index.html", prediction_text="You are well. No worries :)"
            )
    else:
        return render_template("index.html")


if __name__ == "__main__":
    # Lance le serveur sur toutes les interfaces, port 5000
    # host="0.0.0.0" est ESSENTIEL pour Docker (pas 127.0.0.1)
    app.run(host="0.0.0.0", port=5000, debug=True)
```

```
 FLUX DE L'APPLICATION
 =======================

 Navigateur                     Serveur Flask (container Docker)
 +------------------+           +----------------------------------+
 |                  |  GET /    |                                  |
 |  Utilisateur     |---------->|  Home() -> index.html            |
 |  ouvre la page   |           |  (affiche le formulaire)         |
 |                  |<----------|                                  |
 |                  |   HTML    |                                  |
 |                  |           |                                  |
 |  Remplit le      | POST      |                                  |
 |  formulaire      | /predict  |                                  |
 |  et clique       |---------->|  predict()                       |
 |  "Result"        |           |    -> recupere les donnees       |
 |                  |           |    -> model.predict(donnees)     |
 |                  |           |    -> renvoie le resultat        |
 |                  |<----------|                                  |
 |  "Vous etes      |   HTML    |                                  |
 |   en bonne       |           |                                  |
 |   sante !"       |           |                                  |
 +------------------+           +----------------------------------+
```

### 5.3 test.py - Les tests unitaires

```python
from app import model_pred

# Donnees de test : un patient avec ces parametres medicaux
new_data = {
    'Age': 68,
    'RestingBP': 150,
    'Cholesterol': 195,
    'Oldpeak': 0.0,
    'FastingBS': 1,
    'MaxHR': 132,
}


def test_predict():
    """
    Verifie que le modele predit correctement pour ce patient.
    On s'attend a prediction == 1 (risque cardiaque detecte).
    Si la prediction est differente, le test echoue avec le message.
    """
    prediction = model_pred(new_data)
    assert prediction == 1, "incorrect prediction"
```

```
 POURQUOI CE TEST EST CRUCIAL DANS LE PIPELINE
 ================================================

 Sans test :
 git push --> build --> deploy --> BUG EN PRODUCTION !

 Avec test :
 git push --> test --> ECHEC detecte --> pipeline STOPPE --> pas de deploy
                                         --> developpeur alerte
                                         --> corrige avant de redeployer
```

### 5.4 requirements.txt - Les dependances

```
catboost==1.2.8       # Librairie ML pour le modele de prediction
Flask==3.1.2          # Framework web pour l'application
pytest==8.4.2         # Framework de tests unitaires
pylint==3.3.8         # Outil d'analyse statique du code
black==25.9.0         # Formateur automatique de code Python
pandas==2.3.2         # Manipulation de donnees (DataFrames)
Werkzeug==3.1.3       # Serveur WSGI utilise par Flask
numpy==2.0.0          # Calcul numerique (dependance de CatBoost/pandas)
arize==7.18.1         # Monitoring ML (pour app_monitoring.py)
```

### 5.5 app_monitoring.py - Version avec monitoring

Cette version enrichie ajoute le **monitoring ML** via Arize :

```
 COMPARAISON : app.py vs app_monitoring.py
 ============================================

 app.py (simple)                    app_monitoring.py (avec monitoring)
 +---------------------------+      +------------------------------------------+
 | Charge le modele          |      | Charge le modele                         |
 | Recoit la requete         |      | Recoit la requete                        |
 | Fait la prediction        |      | Fait la prediction                       |
 | Retourne le resultat      |      | ENVOIE les donnees a Arize (monitoring)  |
 |                           |      | Retourne le resultat                     |
 +---------------------------+      +------------------------------------------+
                                          |
                                          v
                                    +------------------+
                                    | ARIZE PLATFORM   |
                                    | - data drift     |
                                    | - performance    |
                                    | - alertes        |
                                    +------------------+
```

Le monitoring est essentiel en MLOps pour detecter :
- **Data drift** : les donnees en production different de celles d'entrainement
- **Model drift** : la performance du modele se degrade avec le temps
- **Anomalies** : des predictions aberrantes

### 5.6 templates/index.html - L'interface utilisateur

Le formulaire web qui collecte les parametres medicaux du patient :

```
 +------------------------------------------+
 |        Predictive Analysis                |
 |                                           |
 |  Age:               [___________]         |
 |  RestingBP(100-200):[___________]         |
 |  Cholesterol:       [___________]         |
 |  FastingBS (0 ou 1):[___________]         |
 |  MaxHR (60-205):    [___________]         |
 |  Oldpeak (0.0-6.2): [___________]         |
 |                                           |
 |         [ Result ]                        |
 |                                           |
 |  >> You are well. No worries :)           |
 +------------------------------------------+
```

Le formulaire envoie les donnees en POST a `/predict`, Flask les traite
avec le modele CatBoost et retourne le resultat sur la meme page.

---

## 6. Le Pipeline CI/CD complet : GitHub → Docker Hub

### 6.1 Fichier github-docker-cicd.yaml - Analyse complete

```yaml
# ===== NOM DU WORKFLOW =====
name: Github-Docker Hub MLOps pipeline - Kamila
# Ce nom apparait dans l'onglet "Actions" de GitHub


# ===== VARIABLES D'ENVIRONNEMENT =====
env:
  DOCKER_USER: ${{secrets.DOCKER_USER}}          # Username Docker Hub
  DOCKER_PASSWORD: ${{secrets.DOCKER_PASSWORD}}    # Password/Token Docker Hub
  REPO_NAME: ${{secrets.REPO_NAME}}                # Nom du repo Docker Hub
# Ces variables sont accessibles par TOUS les jobs du workflow


# ===== DECLENCHEURS =====
on:
  push:
    branches:
    - main                # Declenchement sur push vers main
  pull_request:
    branches:
    - main                # Declenchement sur PR vers main


# ===== DEFINITION DES JOBS =====
jobs:

  # ---------- JOB 1 : INTEGRATION CONTINUE ----------
  ci_pipeline:
    runs-on: ubuntu-latest     # Machine virtuelle Ubuntu fournie par GitHub

    steps:
      # Etape 1 : Clone le repository
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0        # Historique complet (utile pour certaines analyses)

      # Etape 2 : Installe Python 3.10
      - name: Set up Python 3.10
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      # Etape 3 : Installe les dependances
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi

      # Etape 4 : Formate le code avec Black
      - name: Format
        run: |
          black app.py

      # Etape 5 : Analyse statique avec Pylint
      - name: Lint
        run: |
          pylint --disable=R,C app.py

      # Etape 6 : Execute les tests
      - name: Test
        run: |
          python -m pytest -vv test.py


  # ---------- JOB 2 : LIVRAISON CONTINUE ----------
  cd_pipeline:
    runs-on: ubuntu-latest
    needs: [ci_pipeline]       # ATTEND que ci_pipeline soit termine avec succes

    steps:
      # Etape 1 : Clone le repository
      - uses: actions/checkout@v4

      # Etape 2 : Connexion a Docker Hub
      - name: docker login
        run: |
          docker login -u $DOCKER_USER -p $DOCKER_PASSWORD

      # Etape 3 : Genere un tag avec la date
      - name: Get current date
        id: date
        run: echo "::set-output name=date::$(date +'%Y-%m-%d--%M-%S')"
        # Cree une sortie nommee "date" avec format : 2026-02-27--30-15

      # Etape 4 : Construit l'image Docker
      - name: Build the Docker image
        run: docker build . --file Dockerfile --tag $DOCKER_USER/$REPO_NAME:${{ steps.date.outputs.date }}
        # Tag final : username/reponame:2026-02-27--30-15

      # Etape 5 : Pousse l'image sur Docker Hub
      - name: Docker Push
        run: docker push $DOCKER_USER/$REPO_NAME:${{ steps.date.outputs.date }}
```

### 6.2 Diagramme de sequence temporelle

```
 CHRONOLOGIE D'UNE EXECUTION DU PIPELINE
 ==========================================

 Temps -->

 DEVELOPPEUR      GITHUB           GITHUB ACTIONS        DOCKER HUB
     |               |                   |                    |
     |  git push     |                   |                    |
     |-------------->|                   |                    |
     |               |  declenche        |                    |
     |               |  workflow         |                    |
     |               |------------------>|                    |
     |               |                   |                    |
     |               |            [CI PIPELINE]               |
     |               |                   |                    |
     |               |             checkout code              |
     |               |             setup python               |
     |               |             install deps               |
     |               |             format (black)             |
     |               |             lint (pylint)              |
     |               |             test (pytest)              |
     |               |                   |                    |
     |               |            [CI OK ? oui]               |
     |               |                   |                    |
     |               |            [CD PIPELINE]               |
     |               |                   |                    |
     |               |             checkout code              |
     |               |             docker login ------------>| authentification
     |               |             docker build               |
     |               |             docker push  ------------>| image stockee
     |               |                   |                    |
     |               |  notification     |                    |
     |<--------------|  (succes/echec)   |                    |
     |               |                   |                    |

 Duree typique : 3-8 minutes pour l'ensemble du pipeline
```

---

## 7. Deploiement sur AWS (ECS/ECR)

### 7.1 AWS ECR vs Docker Hub

```
 COMPARAISON : DOCKER HUB vs AWS ECR
 ======================================

 +---------------------------+     +---------------------------+
 |      DOCKER HUB           |     |      AWS ECR              |
 |      (hub.docker.com)     |     |  (Elastic Container       |
 |                           |     |   Registry)               |
 +---------------------------+     +---------------------------+
 | Type : Public/Prive       |     | Type : Prive par defaut   |
 | Prix : Gratuit (limites)  |     | Prix : Pay-as-you-go      |
 | Acces : Username/Password |     | Acces : AWS IAM           |
 | Integration : Universelle |     | Integration : Ecosystem   |
 |                           |     |              AWS           |
 +---------------------------+     +---------------------------+
       |                                  |
       |  docker push                     |  docker push
       v                                  v
 +---------------------------+     +---------------------------+
 |  Image disponible pour    |     |  Image disponible pour    |
 |  TOUT LE MONDE            |     |  les services AWS :       |
 |  (si repo public)         |     |  - ECS (containers)       |
 |                           |     |  - EKS (Kubernetes)       |
 |  Deploiement manuel ou    |     |  - Lambda (serverless)    |
 |  via docker pull          |     |  - App Runner             |
 +---------------------------+     +---------------------------+
```

### 7.2 L'ecosysteme AWS pour les containers

```
 ARCHITECTURE AWS POUR LE DEPLOIEMENT
 =======================================

 +-------------------------------------------------------+
 |  AWS CLOUD                                             |
 |                                                        |
 |  +------------------+    +------------------------+    |
 |  |   AWS ECR        |    |   AWS ECS              |    |
 |  |   (Registry)     |--->|   (Elastic Container   |    |
 |  |                  |    |    Service)             |    |
 |  |  Stocke les      |    |                        |    |
 |  |  images Docker   |    |  Execute les           |    |
 |  +------------------+    |  containers            |    |
 |                          |                        |    |
 |                          |  +------------------+  |    |
 |                          |  | CLUSTER          |  |    |
 |                          |  | (MlopsCluster)   |  |    |
 |                          |  |                  |  |    |
 |                          |  | +----+ +----+    |  |    |
 |                          |  | |TASK| |TASK|    |  |    |
 |                          |  | | 1  | | 2  |    |  |    |
 |                          |  | +----+ +----+    |  |    |
 |                          |  +------------------+  |    |
 |                          +------------------------+    |
 |                                                        |
 +-------------------------------------------------------+
          |
          | port 5000
          v
     Utilisateurs
```

**Vocabulaire AWS ECS :**

| Concept AWS        | Equivalent Docker local        | Description                          |
|-------------------|-------------------------------|--------------------------------------|
| **ECR**           | Docker Hub                     | Registre d'images Docker prive       |
| **ECS**           | docker run                     | Service qui execute les containers   |
| **Cluster**       | Machine avec Docker installe   | Groupe de ressources pour les tasks  |
| **Task Definition** | docker-compose.yml           | Description du container a lancer    |
| **Task**          | Container en cours             | Instance d'un container              |
| **Service**       | docker run --restart always    | Maintient N tasks en execution       |

### 7.3 Le fichier aws.yml explique

Ce fichier est actuellement **commente** dans le projet, mais voici comment il fonctionne :

```yaml
# Structure du pipeline AWS (decommente et explique) :

name: Deploy to Amazon ECS

on:
  push:
    branches: ["main"]

env:
  AWS_REGION: eu-west-3                    # Region AWS (Paris)
  ECR_REPOSITORY: kamilakare/mlops         # Nom du repo ECR
  ECS_SERVICE: mlops_service               # Nom du service ECS
  ECS_CLUSTER: MlopsCluster               # Nom du cluster ECS
  ECS_TASK_DEFINITION: mlops_catboost      # Nom de la Task Definition
  CONTAINER_NAME: mlops_catboost           # Nom du container dans la task

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      # 1. Recuperer le code
      - name: Checkout
        uses: actions/checkout@v4

      # 2. Configurer les credentials AWS
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-west-3

      # 3. Se connecter a ECR (le "Docker Hub" d'AWS)
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      # 4. Construire et pousser l'image vers ECR
      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}    # Utilise le hash du commit comme tag
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      # 5. Recuperer la Task Definition actuelle
      - name: Download task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition ${{ env.ECS_TASK_DEFINITION }} \
            --query "taskDefinition | {containerDefinitions, family, ...}" \
            > task-definition.json

      # 6. Mettre a jour l'image dans la Task Definition
      - name: Fill in the new image ID
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ steps.build-image.outputs.image }}

      # 7. Deployer la nouvelle Task Definition sur ECS
      - name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true    # Attend que le deploiement soit stable
```

### 7.4 Flux complet du deploiement AWS

```
 PIPELINE COMPLET GITHUB --> AWS
 ==================================

 git push (main)
     |
     v
 GitHub Actions
     |
     |  1. Configure AWS credentials (IAM)
     |  2. Login to ECR
     v
 +------------------+
 |  AWS ECR         |  <-- docker build + docker push
 |  (Image stockee) |
 +--------+---------+
          |
          |  3. Mise a jour Task Definition
          v
 +------------------+
 |  AWS ECS         |
 |  +------------+  |
 |  | Task Def   |  |  <-- "lance ce container avec cette image"
 |  | (nouvelle  |  |
 |  |  version)  |  |
 |  +-----+------+  |
 |        |         |
 |        v         |
 |  +------------+  |
 |  | Service    |  |  <-- "maintiens ce container actif"
 |  | (mlops_    |  |
 |  |  service)  |  |
 |  +-----+------+  |
 |        |         |
 |        v         |
 |  +------------+  |
 |  | Cluster    |  |  <-- "ressources de calcul"
 |  | (Mlops     |  |
 |  |  Cluster)  |  |
 |  +------------+  |
 +------------------+
          |
          v
     Application accessible
     (via Load Balancer ou IP publique)
```

---

## 8. Comparaison Docker Hub vs AWS ECR

```
 +------------------------+-------------------------+-------------------------+
 |  Critere               |  DOCKER HUB             |  AWS ECR                |
 +========================+=========================+=========================+
 |  Prix                  |  Gratuit (1 repo prive) |  ~0.10$/GB/mois         |
 |                        |  Pro: 5$/mois           |  + 0.09$/GB transfert   |
 +------------------------+-------------------------+-------------------------+
 |  Repos prives          |  1 (gratuit)            |  Illimites              |
 +------------------------+-------------------------+-------------------------+
 |  Securite              |  Basique                |  IAM, VPC, chiffrement  |
 +------------------------+-------------------------+-------------------------+
 |  Integration CI/CD     |  Universelle            |  Optimisee pour AWS     |
 +------------------------+-------------------------+-------------------------+
 |  Scan de vulnerabilite |  Oui (payant)           |  Oui (integre)          |
 +------------------------+-------------------------+-------------------------+
 |  Utilisation ideale    |  Projets open source    |  Production AWS         |
 |                        |  Prototypage            |  Entreprise             |
 +------------------------+-------------------------+-------------------------+
 |  Commande push         |  docker push user/repo  |  docker push            |
 |                        |                         |  account.dkr.ecr.       |
 |                        |                         |  region.amazonaws.com/  |
 |                        |                         |  repo                   |
 +------------------------+-------------------------+-------------------------+
```

```
 QUAND UTILISER QUOI ?
 =======================

 Projet perso / Open Source / Apprentissage
     --> DOCKER HUB (simple, gratuit)

 Production / Entreprise / Deploiement AWS
     --> AWS ECR (securise, integre)

 Les deux en meme temps ? Oui !
     --> Docker Hub pour partager publiquement
     --> AWS ECR pour deployer en production
```

---

## 9. Guide pratique pas a pas

### 9.1 Pre-requis

```
 CHECKLIST DES PRE-REQUIS
 ==========================

 [ ] Git installe            (git --version)
 [ ] Docker installe         (docker --version)
 [ ] Compte GitHub           (github.com)
 [ ] Compte Docker Hub       (hub.docker.com)
 [ ] (Optionnel) Compte AWS  (aws.amazon.com)
 [ ] Python 3.10+            (python --version)
```

### 9.2 Etape 1 : Creer le repository GitHub

```bash
# 1. Initialisez votre projet
mkdir mon-projet-mlops
cd mon-projet-mlops
git init

# 2. Creez la structure
mkdir -p .github/workflows templates

# 3. Ajoutez vos fichiers (app.py, Dockerfile, etc.)
# ... (copiez les fichiers du dossier pipeline/)

# 4. Premier commit
git add .
git commit -m "Initial commit: Flask ML app with Docker"

# 5. Connectez a GitHub
git remote add origin https://github.com/VOTRE_USER/VOTRE_REPO.git
git branch -M main
git push -u origin main
```

### 9.3 Etape 2 : Creer un token Docker Hub

```
 Docker Hub (hub.docker.com)
 |
 |--> Account Settings
 |    |
 |    |--> Security
 |         |
 |         |--> New Access Token
 |              |
 |              |--> Nom: "github-actions"
 |              |--> Permissions: Read & Write
 |              |--> Generate
 |              |
 |              |--> COPIEZ LE TOKEN (il ne sera plus visible ensuite !)
```

### 9.4 Etape 3 : Configurer les secrets GitHub

```
 GitHub Repository
 |
 |--> Settings
 |    |
 |    |--> Secrets and variables
 |         |
 |         |--> Actions
 |              |
 |              |--> New repository secret:
 |              |    Nom: DOCKER_USER
 |              |    Valeur: votre_username_dockerhub
 |              |
 |              |--> New repository secret:
 |              |    Nom: DOCKER_PASSWORD
 |              |    Valeur: le_token_copie_precedemment
 |              |
 |              |--> New repository secret:
 |                   Nom: REPO_NAME
 |                   Valeur: nom_de_votre_image
```

### 9.5 Etape 4 : Tester localement avec Docker

```bash
# 1. Construire l'image
docker build -t mon-app-ml:test .

# 2. Lancer le container
docker run -p 5000:5000 mon-app-ml:test

# 3. Ouvrir dans le navigateur
#    http://localhost:5000

# 4. Verifier que la prediction fonctionne
#    Remplissez le formulaire et cliquez "Result"

# 5. Arreter le container
docker ps                      # noter le CONTAINER ID
docker stop <CONTAINER_ID>
```

### 9.6 Etape 5 : Pousser et verifier le pipeline

```bash
# Faites une modification (ex: un commentaire dans app.py)
git add app.py
git commit -m "test: trigger CI/CD pipeline"
git push origin main
```

Puis allez sur GitHub → Onglet **Actions** → Observez l'execution :

```
 INTERFACE GITHUB ACTIONS
 ==========================

 +---------------------------------------------------------------+
 |  Actions   (onglet)                                           |
 |                                                                |
 |  Github-Docker Hub MLOps pipeline - Kamila                    |
 |                                                                |
 |  +----------------------------------------------------------+ |
 |  | Run #1  - test: trigger CI/CD pipeline                    | |
 |  |                                                          | |
 |  |  ci_pipeline  [==========] SUCCES  (2min 30s)            | |
 |  |      |                                                   | |
 |  |      v                                                   | |
 |  |  cd_pipeline  [==========] SUCCES  (1min 45s)            | |
 |  |                                                          | |
 |  |  Status: SUCCES                                          | |
 |  +----------------------------------------------------------+ |
 +---------------------------------------------------------------+
```

### 9.7 Etape 6 (Optionnel) : Configurer AWS

```bash
# Pre-requis : AWS CLI installe et configure

# 1. Creer un repository ECR
aws ecr create-repository --repository-name mlops-app --region eu-west-3

# 2. Creer un cluster ECS
aws ecs create-cluster --cluster-name MlopsCluster --region eu-west-3

# 3. Creer une Task Definition (via la console AWS ou un fichier JSON)

# 4. Creer un Service ECS
aws ecs create-service \
  --cluster MlopsCluster \
  --service-name mlops_service \
  --task-definition mlops_catboost \
  --desired-count 1

# 5. Ajouter les secrets dans GitHub :
#    AWS_ACCESS_KEY_ID
#    AWS_SECRET_ACCESS_KEY

# 6. Decommenter le fichier aws.yml et pousser
```

---

## 10. Troubleshooting et bonnes pratiques

### 10.1 Erreurs courantes

```
 PROBLEME                          SOLUTION
 =========                         ========

 "docker login failed"             Verifiez DOCKER_USER et DOCKER_PASSWORD
                                   dans les secrets GitHub. Utilisez un
                                   ACCESS TOKEN, pas votre mot de passe.

 "pip install failed"              Verifiez que requirements.txt existe
                                   et que les versions sont compatibles.

 "pytest failed"                   Le modele .pkl n'est pas dans le repo.
                                   Verifiez que catboost_model-2.pkl est
                                   bien commite (pas dans .gitignore).

 "pylint failed"                   Corrigez les erreurs de code.
                                   --disable=R,C ignore les avertissements
                                   de convention et refactoring.

 "docker build failed"             Verifiez que le Dockerfile est correct
                                   et que tous les fichiers necessaires
                                   sont presents.

 "port already in use"             Un autre processus utilise le port 5000.
 (test local)                      Utilisez: docker run -p 5001:5000 ...

 "cannot connect to Docker"        Docker Desktop n'est pas lance.
 (test local)                      Demarrez Docker Desktop.
```

### 10.2 Bonnes pratiques

```
 +----------------------------------------------------------------+
 |  SECURITE                                                       |
 |  - Ne JAMAIS mettre de mots de passe dans le code               |
 |  - Toujours utiliser les secrets GitHub                         |
 |  - Utiliser des tokens Docker Hub (pas le mot de passe)         |
 |  - Ajouter .env dans .gitignore                                 |
 +----------------------------------------------------------------+
 |  DOCKER                                                         |
 |  - Utiliser des images slim ou alpine (plus legeres)            |
 |  - Toujours specifier les versions dans requirements.txt        |
 |  - Utiliser .dockerignore pour exclure les fichiers inutiles    |
 |  - Ne pas utiliser debug=True en production                     |
 +----------------------------------------------------------------+
 |  CI/CD                                                          |
 |  - Toujours avoir des tests avant le deploiement                |
 |  - Tagger les images avec la date ou le hash du commit          |
 |  - Ne pas deployer directement sur main sans review             |
 |  - Utiliser des branches + Pull Requests                        |
 +----------------------------------------------------------------+
 |  MLOps                                                          |
 |  - Monitorer les predictions en production (Arize, MLflow...)   |
 |  - Versionner les modeles (pas seulement le code)               |
 |  - Tester le modele, pas seulement le code                      |
 |  - Documenter les metriques du modele                           |
 +----------------------------------------------------------------+
```

### 10.3 Schema recapitulatif global

```
 PIPELINE MLOPS COMPLET : DU CODE AU CLOUD
 ============================================

  DEVELOPPEMENT                CI (Integration Continue)
 +----------------+     +------------------------------------------+
 |                |     |                                          |
 | Code Python    |     |  1. Checkout code                        |
 | + Modele ML    |---->|  2. Setup Python                         |
 | + Dockerfile   |     |  3. Install dependencies                 |
 | + Tests        |     |  4. Format (black)                       |
 |                |     |  5. Lint (pylint)                        |
 | git push       |     |  6. Test (pytest)                        |
 +----------------+     |                                          |
                        |  RESULTAT: Code valide et teste           |
                        +--------------------+---------------------+
                                             |
                                             | si succes
                                             v
                        +------------------------------------------+
                        |  CD (Livraison Continue)                  |
                        |                                          |
                        |  1. Login Docker Hub (ou ECR)             |
                        |  2. Build image Docker                    |
                        |  3. Tag image (date/commit)               |
                        |  4. Push image sur registre               |
                        |                                          |
                        |  RESULTAT: Image prete a deployer         |
                        +-----+------------------+-----------------+
                              |                  |
                              v                  v
                     +----------------+  +------------------+
                     |  DOCKER HUB   |  |  AWS ECR         |
                     |  (public)     |  |  (prive)         |
                     +-------+-------+  +--------+---------+
                             |                   |
                    docker pull           ECS Task Update
                             |                   |
                             v                   v
                     +----------------+  +------------------+
                     |  Serveur /     |  |  AWS ECS         |
                     |  VM / local    |  |  (Cluster)       |
                     +-------+--------+  +--------+---------+
                             |                    |
                             v                    v
                     +-------+--------------------+---------+
                     |                                      |
                     |     APPLICATION EN PRODUCTION         |
                     |     http://mon-app:5000               |
                     |                                      |
                     |  +-----------------------------+     |
                     |  |  Monitoring (Arize)         |     |
                     |  |  - Performance du modele    |     |
                     |  |  - Data drift               |     |
                     |  |  - Alertes                  |     |
                     |  +-----------------------------+     |
                     |                                      |
                     +--------------------------------------+
```

---

## Glossaire

| Terme | Definition |
|-------|-----------|
| **CI** | Continuous Integration - automatisation des tests a chaque changement de code |
| **CD** | Continuous Delivery/Deployment - automatisation du deploiement |
| **Docker** | Plateforme de containerisation d'applications |
| **Image Docker** | Template immutable contenant l'application et ses dependances |
| **Container** | Instance en cours d'execution d'une image Docker |
| **Dockerfile** | Fichier d'instructions pour construire une image Docker |
| **Docker Hub** | Registre public d'images Docker |
| **AWS ECR** | Elastic Container Registry - registre prive AWS |
| **AWS ECS** | Elastic Container Service - orchestrateur de containers AWS |
| **GitHub Actions** | Service CI/CD integre a GitHub |
| **Workflow** | Fichier YAML definissant le pipeline CI/CD |
| **Secret** | Variable chiffree stockee dans GitHub |
| **Tag** | Identifiant de version d'une image Docker |
| **Layer** | Couche dans une image Docker (une par instruction) |
| **Registry** | Service de stockage et distribution d'images Docker |
| **MLOps** | Ensemble de pratiques pour le deploiement et la maintenance de modeles ML |
| **CatBoost** | Algorithme de gradient boosting pour la classification/regression |
| **Flask** | Micro-framework web Python |
| **Arize** | Plateforme de monitoring pour modeles ML en production |
| **pylint** | Outil d'analyse statique pour Python |
| **black** | Formateur de code Python |
| **pytest** | Framework de tests unitaires Python |

---

*Document genere pour le cours MLOps - DU Data Analytics*
*Derniere mise a jour : Fevrier 2026*
