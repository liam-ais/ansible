# Guide : Création et utilisation d'un Ansible Execution Environment

## Objectif

Cette procédure décrit la création et l'utilisation d'un **Ansible Execution Environment (EE)** afin de fournir un environnement d'exécution standardisé pour les playbooks Ansible.

L'Execution Environment est une image de conteneur qui embarque l'ensemble des composants nécessaires à l'exécution d'Ansible :

| Composant | Rôle |
|-----------|------|
| Ansible Core | Moteur d'automatisation |
| Collections Ansible | Modules et plugins |
| Bibliothèques Python | Dépendances Python |
| Paquets système | Outils système requis |

---

## Pourquoi utiliser un Execution Environment ?

### Standardisation

- Tous les utilisateurs exécutent les playbooks dans le **même environnement**.
- Les différences de versions entre les postes de travail ou les serveurs sont éliminées.
- L'utilisation d'un EE permet de garantir que les playbooks seront exécutés dans un environnement identique quel que soit le serveur ou le poste de travail utilisé.

### Portabilité

Le comportement des playbooks reste identique quel que soit l'environnement d'exécution. Le même EE peut être utilisé :

- Sur un poste d'administration
- Dans un pipeline CI/CD
- Dans AWX
- Dans Red Hat Ansible Automation Platform

### Gestion des dépendances

Aucune installation supplémentaire n'est nécessaire sur le serveur exécutant les playbooks. Toutes les dépendances sont intégrées dans l'image :

- Bibliothèques Python
- Collections Ansible
- Outils système

### Isolation

- Chaque projet peut disposer de son propre Execution Environment.
- Les dépendances de plusieurs projets ne se mélangent plus, ce qui évite les conflits de versions.

### Reproductibilité

- Chaque image est versionnée.
- Il est ainsi possible de réutiliser exactement le même environnement plusieurs mois après sa création afin de garantir des résultats identiques.

---

## Prérequis

| Composant | Rôle |
|-----------|------|
| Podman ou Docker | Construction de l'image |
| Python 3 | Exécution d'Ansible Builder |
| pip | Installation des dépendances Python |
| ansible-builder | Génération de l'image EE |
| ansible-navigator | Exécution des playbooks dans un EE |

---

## Installation des prérequis

### Python

```bash
dnf install python3.12
dnf install python-pip
```

### Podman

```bash
sudo dnf install podman
```

### Ansible Builder

Ansible Builder permet de construire l'Execution Environment et de générer l'image incluant Ansible Core, Python, les collections Ansible, les bibliothèques Python et les paquets système.

```bash
pip install ansible-builder
```

**Vérification :**

```bash
ansible-builder --version
```

### Ansible Navigator

Ansible Navigator sert à interagir avec l'image Ansible créée via Ansible Builder. Il permet d'exécuter les commandes à l'intérieur de l'Execution Environment.

```bash
pip install ansible-navigator
```

**Vérification :**

```bash
ansible-navigator --version
```

---

## Choix de l'image de base

Dans le cadre de cette procédure, l'image communautaire sera utilisée :

```
ghcr.io/ansible-community/community-ee-base:latest
```

> **Remarque :** Les images `ee-minimal-rhel` et `ee-supported-rhel` publiées sur `registry.redhat.io` sont réservées aux utilisateurs disposant d'une souscription Red Hat Ansible Automation Platform. La version communautaire est donc privilégiée dans cette procédure.

---

## Structure du projet

Créez un dossier avec la structure suivante :

```
ee-demo/
├── execution-environment.yml
├── requirements.yml
├── requirements.txt
├── bindep.txt
├── inventory.ini
└── test.yml
```

| Fichier | Rôle |
|---------|------|
| `execution-environment.yml` | Configuration de la construction de l'EE |
| `requirements.yml` | Déclarations des collections Ansible |
| `requirements.txt` | Dépendances Python |
| `bindep.txt` | Paquets système requis |
| `inventory.ini` | Inventaire Ansible |
| `test.yml` | Playbook de test |

> **Note :** Les fichiers d'exemple sont disponibles dans le dossier [`examples/`](examples/) de ce projet.

---

## Création du fichier execution-environment.yml

```yaml
version: 3

images:
  base_image:
    name: ghcr.io/ansible-community/community-ee-base:latest

dependencies:
  galaxy: requirements.yml
  python: requirements.txt
  system: bindep.txt
```

### Explication des sections

| Section | Description |
|---------|-------------|
| `version` | Version du format du fichier (ici 3) |
| `images.base_image` | Image de base utilisée pour construire l'EE |
| `dependencies.galaxy` | Fichier déclarant les collections Ansible à installer |
| `dependencies.python` | Fichier déclarant les bibliothèques Python à installer |
| `dependencies.system` | Fichier déclarant les paquets système à installer |

---

## Déclaration des dépendances

### Collections Ansible (`requirements.yml`)

```yaml
collections:
  - name: ansible.posix
  - name: community.general
```

### Dépendances Python (`requirements.txt`)

```
requests
jmespath
```

### Paquets système (`bindep.txt`)

```
git
openssh-clients
iputils
```

> Chaque type de dépendance est séparé car Ansible Builder les traite différemment lors de la construction de l'image.

---

## Construction de l'image

```bash
ansible-builder build \
  --container-runtime podman \
  -t ee-demo:1.0
```

### Vérification

```bash
podman images
# ou
docker images
```

L'image `ee-demo:1.0` devrait apparaître dans la liste.

---

## Inspection de l'image

### Lister les images

```bash
podman images
```

### Lancer un shell dans l'image

```bash
podman run -it ee-demo:1.0 bash
```

### Vérifications à l'intérieur du conteneur

```bash
ansible --version
python3 --version
ansible-galaxy collection list
pip list
```

Cela permet de vérifier que tout est bien embarqué dans l'image.

---

## Création d'un playbook de test

Créez le fichier `test.yml` (disponible dans [`playbooks/test.yml`](playbooks/test.yml)) :

```yaml
---
- name: Test Execution Environment
  hosts: localhost
  gather_facts: true

  tasks:
    - name: Ping localhost
      ansible.builtin.ping:

    - name: Afficher la version d'Ansible
      ansible.builtin.debug:
        var: ansible_version.full

    - name: Afficher la version de Python
      ansible.builtin.debug:
        var: ansible_python.version.full
```

Ce playbook permet de vérifier si :

- Ansible fonctionne
- Python fonctionne
- Les facts fonctionnent

---

## Inventaire

Dans le fichier `inventory.ini`, ajoutez :

```ini
localhost ansible_connection=local
```

---

## Exécution du playbook avec l'EE

```bash
ansible-navigator run test.yml \
  -i inventory.ini \
  --execution-environment true \
  --eei ee-demo:1.0
```

### Options

| Option | Description |
|--------|-------------|
| `--execution-environment true` | Indique que le playbook doit être exécuté dans un EE |
| `--eei ee-demo:1.0` | Spécifie l'image à utiliser pour lancer le playbook |
| `-i inventory.ini` | Indique le fichier d'inventaire |

---

## Résultat attendu

```
Play name: Test Execution Environment:1
Task name: Ping localhost
Ok: localhost
  ping: pong
```

---

## Remarques

> Les conteneurs utilisés par Ansible EE ne s'exécutent que lorsque vous lancez un playbook. Si vous faites :
>
> ```bash
> podman ps -a
> ```
>
> Vous ne verrez aucun conteneur qui tourne (hors lancement de playbook).
