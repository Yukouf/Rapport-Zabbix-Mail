# Automatisation de rapports Zabbix par email

Guide complet pour mettre en place un système de rapports hebdomadaires automatisés depuis Zabbix, avec génération Excel et envoi par email.

## Contexte

Zabbix est un outil de supervision réseau et système. Par défaut, il envoie des alertes unitaires par email, mais ne propose pas de rapport de synthèse hebdomadaire. Ce projet automatise la génération d'un rapport Excel complet (problèmes en cours, inventaire des hôtes, statistiques de disponibilité) et son envoi par email chaque semaine.

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

Placer le script `zabbix_rapport_auto.py` dans ce dossier (voir section Script ci-dessous).

### 4. Configurer le script

Modifier les variables de configuration en haut du script :

```python
# Connexion Zabbix
ZABBIX_URL = "https://votre-zabbix.example.com/api_jsonrpc.php"
ZABBIX_USER = "rapport-auto"
ZABBIX_PASS = "VotreMotDePasse"

# Configuration SMTP
SMTP_SERVER = "smtp.votre-provider.com"
SMTP_PORT = 587
SMTP_USER = "votre_compte_smtp"
SMTP_PASS = "votre_mot_de_passe_smtp"
SMTP_FROM = "Zabbix Alertes <alertes@votre-domaine.com>"

# Destinataires
EMAIL_TO = ["admin@votre-domaine.com"]

# Dossier de sauvegarde
REPORT_DIR = "/chemin/vers/rapports_zabbix"
```

### 5. Tester

```bash
# Tester l'envoi d'email (sans rapport)
python3 /chemin/vers/rapports_zabbix/zabbix_rapport_auto.py --test-email

# Générer et envoyer un rapport complet
python3 /chemin/vers/rapports_zabbix/zabbix_rapport_auto.py

# Générer sans envoyer par email
python3 /chemin/vers/rapports_zabbix/zabbix_rapport_auto.py --no-email
```

### 6. Automatiser avec cron

```bash
# Ouvrir l'éditeur cron
crontab -e

# Ajouter cette ligne pour un envoi chaque lundi à 7h00
0 7 * * 1 /usr/bin/python3 /chemin/vers/rapports_zabbix/zabbix_rapport_auto.py >> /chemin/vers/rapports_zabbix/cron.log 2>&1
```

Vérifier :

```bash
crontab -l
```

## Le script

### Fonctionnement

Le script effectue les opérations suivantes dans l'ordre :

1. **Authentification** via l'API JSON-RPC de Zabbix (Bearer token pour Zabbix 7.x)
2. **Récupération des hôtes** avec leurs interfaces, IP et groupes
3. **Calcul de la disponibilité** (disponible / non disponible / inconnu)
4. **Récupération des problèmes** en cours non supprimés
5. **Résolution des triggers** pour associer chaque problème à son hôte
6. **Génération du rapport Excel** avec deux onglets :
   - **Rapport Hebdo** : résumé du parc, compteurs par sévérité, tableau des problèmes
   - **Inventaire Hôtes** : liste complète avec IP, type d'agent, état et disponibilité
7. **Envoi par email** avec le fichier Excel en pièce jointe

### Particularités Zabbix 7.x

L'API de Zabbix 7.x a changé le mode d'authentification. L'ancien paramètre `auth` dans le body JSON n'est plus accepté. Il faut utiliser un header HTTP `Authorization: Bearer <token>` :

```python
def zabbix_api(method, params, auth=None):
    payload = {"jsonrpc": "2.0", "method": method, "params": params, "id": 1}
    headers = {"Content-Type": "application/json-rpc"}
    if auth:
        headers["Authorization"] = f"Bearer {auth}"
    req = urllib.request.Request(ZABBIX_URL, data=json.dumps(payload).encode(), headers=headers)
    # ...
```

De même, l'API `problem.get` n'accepte plus `sortfield: ["severity", "clock"]`. Il faut utiliser `sortfield: "eventid"`.

### Contenu du rapport Excel

**Onglet "Rapport Hebdo"** :
- Résumé du parc (hôtes total, disponibles, non disponibles, inconnus)
- Compteurs par sévérité (Désastre, Haut, Moyen, Avertissement, Information)
- Tableau détaillé des problèmes avec : date, hôte, sévérité, description, durée, statut d'acquittement, tags
- Code couleur par sévérité (rouge pour Désastre, orange pour Haut, etc.)
- Mise en évidence des problèmes Haut/Désastre non acquittés

**Onglet "Inventaire Hôtes"** :
- Liste de tous les hôtes supervisés
- Adresse IP, type d'agent (ZBX/SNMP), groupes, état, disponibilité
- Hôtes non disponibles surlignés en rouge

## Retrouver l'accès à Zabbix derrière un reverse proxy

Si Zabbix est configuré derrière un reverse proxy Apache avec des virtual hosts, l'accès par IP ne fonctionnera pas (il tombera sur le mauvais service). Il faut utiliser le nom de domaine configuré dans le `ServerName` du virtual host.

### Diagnostic

```bash
# Voir les virtual hosts actifs
ls -la /etc/apache2/sites-enabled/
cat /etc/apache2/sites-enabled/*

# Chercher les ServerName
grep -r "ServerName" /etc/apache2/sites-enabled/
```

### Solution

Ajouter une entrée dans le fichier hosts de la machine client :

**Windows** (`C:\Windows\System32\drivers\etc\hosts`) :
```
192.168.x.x    zabbix.votre-domaine.local
```

**Linux/Mac** (`/etc/hosts`) :
```
192.168.x.x    zabbix.votre-domaine.local
```

Puis accéder via `https://zabbix.votre-domaine.local`.

## Résoudre un Zabbix Server bloqué

Si le service Zabbix est en état `deactivating (stop-sigterm)` depuis longtemps :

```bash
# Vérifier l'état
systemctl status zabbix-server

# Forcer l'arrêt
sudo systemctl kill -s SIGKILL zabbix-server

# Redémarrer
sudo systemctl start zabbix-server

# Vérifier
systemctl status zabbix-server
```

## Récupérer la configuration SMTP de Zabbix

Si Zabbix envoie déjà des alertes par email, la configuration SMTP est stockée dans la base de données :

```bash
mysql -u zabbix -p<password> zabbix -e \
  "SELECT name, smtp_server, smtp_port, smtp_email, username FROM media_type WHERE type=0;"
```

## Structure du projet

```
rapports_zabbix/
├── zabbix_rapport_auto.py    # Script principal
├── cron.log                  # Logs d'exécution du cron
└── rapport_zabbix_YYYY-MM-DD.xlsx  # Rapports générés
```

## Dépannage

| Problème | Solution |
|----------|----------|
| `ModuleNotFoundError: openpyxl` | `pip3 install openpyxl --break-system-packages` |
| `email.mime.base64` not found | Corriger l'import : `from email.mime.base import MIMEBase` |
| `Invalid parameter "auth"` | Utiliser le header `Authorization: Bearer` (Zabbix 7.x) |
| `Invalid parameter "/sortfield/1"` | Utiliser `"sortfield": "eventid"` au lieu d'un tableau |
| Rapport non envoyé par cron | Vérifier `cat /chemin/rapports_zabbix/cron.log` |
| Accès web tombe sur Grafana | Utiliser le ServerName au lieu de l'IP (virtual host) |

## Licence

MIT
