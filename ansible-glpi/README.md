# Déploiement automatisé de GLPI via Ansible

Projet de déploiement automatisé de [GLPI](https://glpi-project.org/) sur une architecture LAMP via Ansible, avec hardening via Fail2ban.

## Contexte

Déploiement d'un outil de gestion de parc (GLPI) pour l'ADRAR sur une infrastructure PaaS composée de deux serveurs :

| Serveur | Rôle |
|---------|------|
| `srv-webl1` | Serveur Web (Apache + GLPI) |
| `srv-dbl1` | Serveur de Base de données (MariaDB) |

## Architecture

```
┌─────────────────┐         ┌─────────────────┐
│   srv-webl1     │         │   srv-dbl1      │
│                 │────────▶│                 │
│  Apache2 + PHP  │  SQL    │  MariaDB        │
│  GLPI           │         │                 │
│  Fail2ban       │         │  Fail2ban       │
└─────────────────┘         └─────────────────┘
         ▲
         │ SSH
   ┌─────┴──────┐
   │  Ansible   │
   │  Control   │
   │   Node     │
   └────────────┘
```

## Prérequis

- Deux serveurs Debian 12 avec SSH et Python installés
- Serveur Ansible (Control Node) avec :
  - Python 3 + venv
  - Ansible installé via pip
  - Clé SSH déployée sur les nodes

## Installation

### 1. Créer le virtual environment

```bash
python3 -m venv ansible
source ansible/bin/activate
pip install ansible
```

### 2.Configurer l'inventaire

Éditer `inventory.ini` avec les IPs de vos serveurs.

### 3. Chiffrer le vault

```bash
ansible-vault encrypt vault/password.yml
```

### 4. Lancer le déploiement

```bash
# Installation d'Apache + PHP
ansible-playbook -i inventory.ini --user user-ansible --become --ask-become-pass playbooks/install-apache.yml

# Installation de MariaDB
ansible-playbook -i inventory.ini --user user-ansible --become --ask-become-pass playbooks/install-mariadb.yml

# Déploiement de GLPI
ansible-playbook -i inventory.ini --user user-ansible --become --ask-become-pass playbooks/install-glpi.yml --ask-vault-pass

# Installation de Fail2ban (hardening)
ansible-playbook -i inventory.ini --user user-ansible --become --ask-become-pass playbooks/install-fail2ban.yml
```

## Structure du projet

```
ansible-glpi/
├── inventory.ini
├── vault/
│   └── password.yml              # Variables sensibles (chiffré via ansible-vault)
├── playbooks/
│   ├── install-apache.yml
│   ├── install-mariadb.yml
│   ├── install-glpi.yml
│   └── install-fail2ban.yml
└── roles/
    ├── apache/                   # Installation Apache2 + PHP
    ├── mariadb/                  # Installation MariaDB
    ├── glpi/                     # Configuration GLPI (confapache + confdb)
    └── fail2ban/                 # Hardening Fail2ban (SSH + GLPI)
```

## Rôles

| Rôle | Description |
|------|-------------|
| `apache` | Installation et activation d'Apache2 + PHP |
| `mariadb` | Installation et configuration de MariaDB |
| `glpi/confapache` | Configuration Apache pour GLPI (VirtualHost, SSL) |
| `glpi/confdb` | Création de la base de données et de l'utilisateur GLPI |
| `fail2ban` | Installation et configuration de Fail2ban (SSH + GLPI) |

## Hardening Fail2ban

- **SSH** : ban après 3 tentatives échouées, IP bannie définitivement
- **GLPI** : ban après 3 tentatives de connexion échouées
- **IgnoreIP** : localhost + IP du serveur Ansible
