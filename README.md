# Automatisation de rapports Zabbix par email

Script Python qui se connecte à l'API Zabbix, génère un rapport Excel formaté et l'envoie par email automatiquement chaque semaine.

## Fonctionnalités

- **Filtrage intelligent** : exclut le bruit (alertes "Information", changements de vitesse Ethernet, mises à jour OS, services Google Updater)
- **Catégorisation** : problèmes regroupés par type (Serveurs, Équipements Réseau, Postes de travail, Périphériques)
- **Résumé exécutif** : métriques clés en haut du rapport (hôtes disponibles/down, alertes critiques, alertes filtrées)
- **Points d'attention** : mise en évidence des alertes Haut/Désastre
- **Design moderne** : police Calibri, lignes alternées, badges de sévérité colorés, onglets colorés
- **3 onglets** : Rapport Hebdo, Inventaire Hôtes, Alertes Filtrées
- **Envoi automatique** par email via SMTP avec le rapport Excel en pièce jointe
- **Planification cron** : envoi chaque lundi à 7h00

## Architecture

```
┌──────────────┐    API JSON-RPC     ┌──────────────┐
│  Script      │ ──────────────────> │  Zabbix      │
│  Python      │ <────────────────── │  Server      │
└──────┬───────┘                     └──────────────┘
       │
       │  SMTP (port 587/TLS)
       v
┌──────────────┐
│  Serveur     │ ──────> 📧 Rapport Excel en PJ
│  Mail        │
└──────────────┘
```

## Prérequis

- Serveur Zabbix fonctionnel (testé sur Zabbix 7.x)
- Python 3.10+
- Bibliothèque `openpyxl`
- Accès SMTP pour l'envoi d'emails
- Un compte utilisateur Zabbix dédié (recommandé)

## Installation

### 1. Installer les dépendances

```bash
pip3 install openpyxl --break-system-packages
```

### 2. Créer un utilisateur API dédié dans Zabbix

Il est recommandé de créer un compte séparé plutôt que d'utiliser le compte Admin :

1. Dans Zabbix : **Utilisateurs** → **Créer un utilisateur**
2. Définir un nom d'utilisateur (ex: `rapport-auto`)
3. Assigner au groupe **Zabbix administrators**
4. Attribuer le rôle **Admin role** ou **Super admin role**

### 3. Créer le dossier et le script

```bash
mkdir -p /chemin/vers/rapports_zabbix
```

Placer le script `zabbix_rapport_auto.py` dans ce dossier.

### 4. Configurer le script

Modifier les variables de configuration en haut du script :

```python
ZABBIX_URL = "https://votre-zabbix.example.com/api_jsonrpc.php"
ZABBIX_USER = "rapport-auto"
ZABBIX_PASS = "VotreMotDePasse"

SMTP_SERVER = "smtp.votre-provider.com"
SMTP_PORT = 587
SMTP_USER = "votre_compte_smtp"
SMTP_PASS = "votre_mot_de_passe_smtp"
SMTP_FROM = "Zabbix Alertes <alertes@votre-domaine.com>"

EMAIL_TO = ["admin@votre-domaine.com", "equipe@votre-domaine.com"]
```

### 5. Personnaliser le filtrage

Le script exclut automatiquement les alertes peu pertinentes. Vous pouvez ajuster les patterns dans la section `EXCLUDED_PATTERNS` :

```python
EXCLUDED_PATTERNS = [
    r"Ethernet has changed to lower speed",
    r"Operating system description has changed",
    r"GoogleUpdater",
    r"Number of installed packages has been changed",
]

# Sévérités exclues (0=Non classé, 1=Information)
EXCLUDED_SEVERITIES = ["0", "1"]
```

Les alertes filtrées restent visibles dans le 3ème onglet "Alertes filtrées" du rapport.

### 6. Personnaliser la catégorisation

Les hôtes sont automatiquement classés par type. Ajustez les mots-clés si nécessaire :

```python
NETWORK_KEYWORDS = ["aruba", "hp-2530", "switch", "cisco"]
NETWORK_AGENTS = ["2"]  # SNMP = équipement réseau
```

### 7. Tester

```bash
# Tester l'envoi d'email (sans rapport)
python3 zabbix_rapport_auto.py --test-email

# Générer et envoyer un rapport complet
python3 zabbix_rapport_auto.py

# Générer sans envoyer par email
python3 zabbix_rapport_auto.py --no-email
```

### 8. Automatiser avec cron

```bash
# Ajouter au cron (chaque lundi à 7h00)
(crontab -l 2>/dev/null; echo "0 7 * * 1 /usr/bin/python3 /chemin/vers/rapports_zabbix/zabbix_rapport_auto.py >> /chemin/vers/rapports_zabbix/cron.log 2>&1") | crontab -

# Vérifier
crontab -l
```

## Contenu du rapport

### Onglet "Rapport Hebdo"

- **Métriques** : hôtes total, disponibles, non disponibles, alertes pertinentes, alertes critiques, alertes filtrées (affichage style dashboard avec gros chiffres)
- **Points d'attention** : alertes Haut/Désastre en évidence
- **Serveurs** : problèmes sur les serveurs (disques pleins, agents down, services arrêtés)
- **Équipements réseau** : problèmes sur switches et routeurs (ping down, interfaces down)
- **Postes de travail** : problèmes sur les postes Windows/Linux
- **Périphériques** : imprimantes, etc.

Chaque section a sa propre couleur et les problèmes sont triés par sévérité (critiques en premier).

### Onglet "Inventaire Hôtes"

Liste complète des hôtes supervisés avec adresse IP, type d'agent (ZBX/SNMP), groupes, état, disponibilité et catégorie. Les hôtes non disponibles sont surlignés en rouge.

### Onglet "Alertes filtrées"

Toutes les alertes exclues du rapport principal, avec la raison du filtrage. Permet de vérifier qu'aucune alerte importante n'a été masquée.

## Particularités Zabbix 7.x

L'API Zabbix 7.x utilise l'authentification par header HTTP au lieu du paramètre `auth` dans le body :

```python
headers["Authorization"] = f"Bearer {auth}"
```

L'endpoint `problem.get` n'accepte plus `sortfield: ["severity", "clock"]`, il faut utiliser `sortfield: "eventid"`.

## Accéder à Zabbix derrière un reverse proxy

Si Zabbix est configuré avec des virtual hosts Apache, l'accès par IP ne fonctionnera pas. Il faut ajouter une entrée dans le fichier hosts :

**Windows** (`C:\Windows\System32\drivers\etc\hosts`) :
```
192.168.x.x    zabbix.votre-domaine.local
```

**Linux/Mac** (`/etc/hosts`) :
```
192.168.x.x    zabbix.votre-domaine.local
```

## Résoudre un Zabbix Server bloqué

Si le service est en état `deactivating (stop-sigterm)` :

```bash
sudo systemctl kill -s SIGKILL zabbix-server
sudo systemctl start zabbix-server
```

## Récupérer la configuration SMTP existante de Zabbix

Si Zabbix envoie déjà des alertes, la config SMTP est dans la base :

```bash
mysql -u zabbix -p zabbix -e \
  "SELECT name, smtp_server, smtp_port, smtp_email, username FROM media_type WHERE type=0;"
```

## Structure

```
rapports_zabbix/
├── zabbix_rapport_auto.py          # Script principal
├── cron.log                        # Logs d'exécution
└── rapport_zabbix_YYYY-MM-DD.xlsx  # Rapports générés
```

## Dépannage

| Problème | Solution |
|----------|----------|
| `ModuleNotFoundError: openpyxl` | `pip3 install openpyxl --break-system-packages` |
| `email.mime.base64` not found | Corriger : `from email.mime.base import MIMEBase` |
| `Invalid parameter "auth"` | Utiliser le header `Authorization: Bearer` (Zabbix 7.x) |
| `Invalid parameter "/sortfield/1"` | Utiliser `"sortfield": "eventid"` |
| Rapport non envoyé par cron | `cat /chemin/rapports_zabbix/cron.log` |
| Accès web tombe sur Grafana | Utiliser le ServerName au lieu de l'IP |
| Trop d'alertes inutiles | Ajouter des patterns dans `EXCLUDED_PATTERNS` |

## Licence

MIT
