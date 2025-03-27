# Procédure d'Installation et Configuration - GLPI

Ce document détaille les étapes d'installation et de configuration de GLPI (Gestionnaire Libre de Parc Informatique), une solution open source de gestion des actifs informatiques, ainsi que son intégration avec Prometheus pour le monitoring.

## Table des matières

1. [Prérequis](#prérequis)
2. [Installation du serveur LAMP](#installation-du-serveur-lamp)
3. [Installation de GLPI](#installation-de-glpi)
4. [Configuration de base](#configuration-de-base)
5. [Installation des plugins](#installation-des-plugins)
6. [Configuration pour le monitoring avec Prometheus](#configuration-du-monitoring-avec-prometheus)
7. [Dépannage](#dépannage)
8. [Références](#références)

## Prérequis

- Une machine virtuelle Linux (Ubuntu Server 22.04 LTS recommandé)
- Minimum 4 Go de RAM, 2 vCPU
- 20 Go d'espace disque
- Accès réseau pour les utilisateurs
- Nom de domaine ou adresse IP statique

## Installation du serveur LAMP

### 1.1. Préparation du système

```bash
# Mise à jour du système
sudo apt update
sudo apt upgrade -y

# Installation des paquets essentiels
sudo apt install -y software-properties-common curl wget net-tools
```

### 1.2. Installation d'Apache

```bash
# Installation du serveur web Apache
sudo apt install -y apache2

# Activation du service
sudo systemctl enable apache2
sudo systemctl start apache2

# Vérification du statut
sudo systemctl status apache2
```

### 1.3. Installation de MariaDB

```bash
# Installation du serveur de base de données MariaDB
sudo apt install -y mariadb-server

# Sécurisation de l'installation
sudo mysql_secure_installation
```

Lors de l'exécution de `mysql_secure_installation`, suivez ces étapes :
- Définissez un mot de passe root (si demandé)
- Supprimez les utilisateurs anonymes
- Désactivez la connexion root à distance
- Supprimez la base de données de test
- Rechargez les privilèges

### 1.4. Installation de PHP

```bash
# Installation de PHP et des extensions nécessaires pour GLPI
sudo apt install -y php php-cli php-common php-mysql php-zip php-gd php-mbstring php-curl php-xml php-pear php-bcmath php-json php-imap php-intl php-ldap php-apcu php-cas php-xmlrpc php-intl php-zip php-bz2
```

### 1.5. Configuration de PHP pour GLPI

Modifiez le fichier php.ini pour optimiser les paramètres pour GLPI :

```bash
sudo nano /etc/php/8.1/apache2/php.ini
```

Modifiez les paramètres suivants :
```ini
memory_limit = 256M
file_uploads = On
max_execution_time = 300
session.auto_start = 0
session.use_trans_sid = 0
```

Redémarrez Apache pour appliquer les modifications :

```bash
sudo systemctl restart apache2
```

## Installation de GLPI

### 2.1. Téléchargement de GLPI

```bash
# Création du répertoire de téléchargement
sudo mkdir -p /var/www/html/glpi
cd /tmp

# Téléchargement de la dernière version de GLPI
wget https://github.com/glpi-project/glpi/releases/download/10.0.7/glpi-10.0.7.tgz

# Extraction de l'archive
tar -xzf glpi-10.0.7.tgz -C /var/www/html/
```

### 2.2. Configuration des permissions

```bash
# Attribution des permissions
sudo chown -R www-data:www-data /var/www/html/glpi/
sudo chmod -R 755 /var/www/html/glpi/
```

### 2.3. Création de la base de données

```bash
# Connexion à MariaDB
sudo mysql -u root -p

# Dans la console MySQL/MariaDB
CREATE DATABASE glpidb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'glpiuser'@'localhost' IDENTIFIED BY 'votre_mot_de_passe_sécurisé';
GRANT ALL PRIVILEGES ON glpidb.* TO 'glpiuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 2.4. Configuration du VirtualHost Apache

```bash
sudo nano /etc/apache2/sites-available/glpi.conf
```

Ajoutez la configuration suivante :

```apache
<VirtualHost *:80>
    ServerName glpi.votredomaine.com
    DocumentRoot /var/www/html/glpi

    <Directory /var/www/html/glpi>
        Options FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/glpi_error.log
    CustomLog ${APACHE_LOG_DIR}/glpi_access.log combined
</VirtualHost>
```

Activez le site et les modules nécessaires :

```bash
sudo a2ensite glpi.conf
sudo a2enmod rewrite
sudo systemctl restart apache2
```

## Configuration de base

### 3.1. Accès à l'interface web d'installation

Accédez à l'interface d'installation via votre navigateur :
```
http://glpi.votredomaine.com/
```
ou
```
http://adresse_ip_serveur/glpi/
```

### 3.2. Assistant d'installation

1. Choisissez votre langue et cliquez sur "OK"
2. Acceptez les termes de la licence (GNU GPL)
3. Cliquez sur "Installer"
4. Vérifiez que toutes les prérequis sont satisfaits (vert), corrigez les éventuels problèmes, puis cliquez sur "Continuer"
5. Configurez la connexion à la base de données :
   - Serveur : localhost
   - Utilisateur : glpiuser
   - Mot de passe : votre_mot_de_passe_sécurisé
   - Base de données : glpidb
6. Cliquez sur "Continuer" et attendez la fin de l'initialisation
7. Utilisez la base de données sélectionnée et cliquez sur "Continuer"
8. Une fois l'installation terminée, cliquez sur "Continuer"
9. Par défaut, les identifiants sont :
   - **glpi/glpi** pour le compte administrateur
   - **tech/tech** pour le compte technicien
   - **normal/normal** pour le compte normal
   - **post-only/post-only** pour le compte post-only

### 3.3. Première connexion et sécurisation

1. Connectez-vous avec le compte administrateur (glpi/glpi)
2. Allez dans Administration > Utilisateurs
3. Modifiez les mots de passe par défaut pour chaque compte
4. Créez des utilisateurs spécifiques selon vos besoins

### 3.4. Configuration de l'envoi d'e-mails

1. Allez dans Configuration > Notifications
2. Configurez les paramètres de notification par e-mail :
   - Serveur SMTP
   - Port
   - Authentification si nécessaire
   - Adresse d'expéditeur

```php
// Configuration dans config/mail.cfg.php
<?php
$phpmailer_config = [
   'host'       => 'smtp.votredomaine.com',
   'port'       => 587,
   'auth'       => true,
   'username'   => 'glpi@votredomaine.com',
   'password'   => 'mot_de_passe_mail',
   'secure'     => 'tls',
   'from'       => 'glpi@votredomaine.com',
   'fromname'   => 'GLPI Notifications',
   'replyto'    => 'support@votredomaine.com',
   'replytoname' => 'Support Technique',
];
```

## Installation des plugins

### 4.1. Plugin Fusion Inventory

Fusion Inventory permet d'inventorier automatiquement les équipements du réseau.

```bash
# Téléchargement du plugin
cd /tmp
wget https://github.com/fusioninventory/fusioninventory-for-glpi/releases/download/glpi10.0.7+1.0/fusioninventory-10.0.7+1.0.tar.bz2

# Extraction dans le dossier des plugins
sudo mkdir -p /var/www/html/glpi/plugins/
sudo tar -xjf fusioninventory-10.0.7+1.0.tar.bz2 -C /var/www/html/glpi/plugins/

# Correction des permissions
sudo chown -R www-data:www-data /var/www/html/glpi/plugins/
```

1. Dans l'interface GLPI, allez dans Administration > Plugins
2. Trouvez FusionInventory et cliquez sur "Installer"
3. Une fois installé, cliquez sur "Activer"

### 4.2. Plugin Dashboard

Le plugin Dashboard offre des tableaux de bord visuels pour GLPI.

```bash
# Téléchargement du plugin
cd /tmp
wget https://github.com/stdonato/glpi-dashboard/releases/download/10.0.5/glpi-dashboard.10.0.5.tar.gz

# Extraction dans le dossier des plugins
sudo tar -xzf glpi-dashboard.10.0.5.tar.gz -C /var/www/html/glpi/plugins/

# Correction des permissions
sudo chown -R www-data:www-data /var/www/html/glpi/plugins/
```

1. Dans l'interface GLPI, allez dans Administration > Plugins
2. Trouvez Dashboard et cliquez sur "Installer"
3. Une fois installé, cliquez sur "Activer"

### 4.3. Plugin Prometheus GLPI Exporter

Ce plugin permet d'exposer des métriques GLPI pour Prometheus.

```bash
# Clonage du dépôt
cd /tmp
git clone https://github.com/pluginsGLPI/telegrambot.git

# Copie dans le dossier des plugins
sudo cp -r telegrambot /var/www/html/glpi/plugins/prometheusexporter

# Correction des permissions
sudo chown -R www-data:www-data /var/www/html/glpi/plugins/
```

1. Dans l'interface GLPI, allez dans Administration > Plugins
2. Trouvez Prometheus Exporter et cliquez sur "Installer"
3. Une fois installé, cliquez sur "Activer"

## Configuration du monitoring avec Prometheus

### 5.1. Installation de l'Exporteur GLPI

Si le plugin Prometheus n'est pas disponible, nous pouvons créer un script personnalisé pour exposer les métriques GLPI :

```bash
# Création du répertoire pour l'exporteur
sudo mkdir -p /opt/glpi-exporter
sudo nano /opt/glpi-exporter/glpi_exporter.php
```

Contenu du fichier `glpi_exporter.php` :

```php
<?php
// Configuration de la base de données
$db_host = 'localhost';
$db_user = 'glpiuser';
$db_pass = 'votre_mot_de_passe_sécurisé';
$db_name = 'glpidb';

// Connexion à la base de données
$mysqli = new mysqli($db_host, $db_user, $db_pass, $db_name);
if ($mysqli->connect_error) {
    die('Connection failed: ' . $mysqli->connect_error);
}

// En-têtes pour Prometheus
header('Content-Type: text/plain');

// Nombre de tickets par statut
$query = "SELECT status, COUNT(*) as count FROM glpi_tickets GROUP BY status";
$result = $mysqli->query($query);

while ($row = $result->fetch_assoc()) {
    echo "glpi_tickets_by_status{status=\"" . getStatusName($row['status']) . "\"} " . $row['count'] . "\n";
}

// Nombre d'équipements par type
$query = "SELECT itemtype, COUNT(*) as count FROM glpi_computers GROUP BY itemtype";
$result = $mysqli->query($query);

while ($row = $result->fetch_assoc()) {
    echo "glpi_computers_by_type{type=\"" . $row['itemtype'] . "\"} " . $row['count'] . "\n";
}

// Fonction pour convertir les codes de statut en noms
function getStatusName($status) {
    switch ($status) {
        case 1: return "new";
        case 2: return "assigned";
        case 3: return "planned";
        case 4: return "waiting";
        case 5: return "solved";
        case 6: return "closed";
        default: return "unknown";
    }
}

$mysqli->close();
?>
```

Créez un serveur web léger pour exposer ces métriques :

```bash
sudo nano /opt/glpi-exporter/index.php
```

```php
<?php
include('glpi_exporter.php');
?>
```

Configuration d'Apache pour exposer les métriques :

```bash
sudo nano /etc/apache2/sites-available/glpi-metrics.conf
```

```apache
<VirtualHost *:9118>
    ServerName glpi-metrics.votredomaine.com
    DocumentRoot /opt/glpi-exporter

    <Directory /opt/glpi-exporter>
        Options FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/glpi_metrics_error.log
    CustomLog ${APACHE_LOG_DIR}/glpi_metrics_access.log combined
</VirtualHost>
```

```bash
sudo a2ensite glpi-metrics.conf
sudo systemctl restart apache2
```

### 5.2. Métriques Node Exporter pour le serveur GLPI

Installez Node Exporter pour surveiller le serveur lui-même :

```bash
# Téléchargement de Node Exporter
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz

# Extraction
tar -xzf node_exporter-1.5.0.linux-amd64.tar.gz

# Déplacement des fichiers
sudo mv node_exporter-1.5.0.linux-amd64/node_exporter /usr/local/bin/

# Création d'un utilisateur spécifique
sudo useradd -rs /bin/false node_exporter

# Création du service systemd
sudo nano /etc/systemd/system/node_exporter.service
```

Contenu du fichier `node_exporter.service` :

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

Démarrage du service :

```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
```

### 5.3. Configuration du pare-feu

```bash
sudo ufw allow 9118/tcp
sudo ufw allow 9100/tcp
```

### 5.4. Configuration de Prometheus

Ajoutez les cibles dans la configuration de Prometheus :

```yaml
scrape_configs:
  - job_name: 'glpi_metrics'
    scrape_interval: 1m
    static_configs:
      - targets: ['glpi-server-ip:9118']

  - job_name: 'glpi_node'
    scrape_interval: 15s
    static_configs:
      - targets: ['glpi-server-ip:9100']
```

## Métriques Surveillées

### Métriques GLPI

- **glpi_tickets_by_status** : Nombre de tickets par statut (nouveau, assigné, en attente, résolu, fermé)
- **glpi_tickets_by_category** : Nombre de tickets par catégorie
- **glpi_tickets_by_priority** : Nombre de tickets par priorité
- **glpi_computers_by_type** : Nombre d'ordinateurs par type
- **glpi_tickets_processing_time** : Temps de traitement des tickets

### Métriques du serveur

- **node_cpu_seconds_total** : Utilisation du CPU
- **node_memory_MemAvailable_bytes** : Mémoire disponible
- **node_filesystem_avail_bytes** : Espace disque disponible
- **node_network_transmit_bytes_total** : Trafic réseau sortant
- **node_network_receive_bytes_total** : Trafic réseau entrant

## Configuration des alertes

Configurez des alertes dans Prometheus pour surveiller l'état du service GLPI :

```yaml
groups:
- name: glpi_alerts
  rules:
  - alert: GLPIHighTicketCount
    expr: sum(glpi_tickets_by_status{status="new"}) > 50
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Nombre élevé de tickets en attente"
      description: "Il y a plus de 50 tickets non assignés depuis plus de 15 minutes."

  - alert: GLPIServerHighLoad
    expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Charge CPU élevée sur le serveur GLPI"
      description: "La charge CPU sur le serveur GLPI est supérieure à 80% depuis 10 minutes."

  - alert: GLPILowDiskSpace
    expr: node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100 < 10
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Espace disque faible sur le serveur GLPI"
      description: "Le serveur GLPI dispose de moins de 10% d'espace disque libre."
```

## Configuration des agents sur les postes de travail

### 6.1. Installation de l'agent FusionInventory sur Windows

1. Téléchargez l'agent FusionInventory pour Windows depuis [le site officiel](https://github.com/fusioninventory/fusioninventory-agent/releases)
2. Exécutez l'installateur avec les options suivantes :
   - Serveur : http://glpi.votredomaine.com/plugins/fusioninventory/
   - Exécuter en tant que service Windows
   - Planifier l'exécution automatique : Oui (toutes les 24 heures)

Pour le déploiement massif, créez un script de déploiement :

```powershell
# Script PowerShell de déploiement de l'agent FusionInventory
$installDir = "C:\Program Files\FusionInventory-Agent"
$serverUrl = "http://glpi.votredomaine.com/plugins/fusioninventory/"
$installerUrl = "https://github.com/fusioninventory/fusioninventory-agent/releases/download/2.6/fusioninventory-agent_windows-x64_2.6.exe"
$installerPath = "$env:TEMP\fusioninventory-agent_windows-x64_2.6.exe"

# Téléchargement de l'installateur
Invoke-WebRequest -Uri $installerUrl -OutFile $installerPath

# Installation silencieuse
Start-Process -FilePath $installerPath -ArgumentList "/S /server=$serverUrl /runnow /execmode=service /acceptlicense" -Wait

# Vérification de l'installation
if (Test-Path $installDir) {
    Write-Host "FusionInventory Agent installé avec succès"
} else {
    Write-Host "Échec de l'installation de FusionInventory Agent"
}
```

### 6.2. Installation de l'agent FusionInventory sur Linux

```bash
# Pour Ubuntu/Debian
sudo apt update
sudo apt install -y fusioninventory-agent

# Configuration
sudo nano /etc/fusioninventory/agent.cfg
```

Modifiez la configuration :
```
server = http://glpi.votredomaine.com/plugins/fusioninventory/
```

```bash
# Activation et démarrage du service
sudo systemctl enable fusioninventory-agent
sudo systemctl start fusioninventory-agent

# Exécution immédiate d'un inventaire
sudo fusioninventory-agent --server http://glpi.votredomaine.com/plugins/fusioninventory/
```

### 6.3. Déploiement par stratégie de groupe (GPO)

Pour Windows dans un environnement AD, créez une GPO pour déployer FusionInventory :

1. Créez un partage réseau contenant l'installateur et le script
2. Dans la console de gestion des stratégies de groupe, créez une nouvelle GPO
3. Sous Configuration ordinateur > Paramètres Windows > Scripts > Démarrage, ajoutez le script PowerShell de déploiement
4. Liez la GPO à l'UO contenant les ordinateurs cibles
5. Exécutez `gpupdate /force` sur les postes clients ou attendez le prochain cycle de mise à jour des GPO

## Dépannage

### Problèmes courants avec GLPI

1. **Erreurs 500 ou pages blanches** :
   - Vérifiez les logs Apache : `sudo tail -f /var/log/apache2/glpi_error.log`
   - Vérifiez les permissions : `sudo chown -R www-data:www-data /var/www/html/glpi/`

2. **Problèmes de base de données** :
   - Vérifiez la connexion à la base de données : `sudo mysql -u glpiuser -p glpidb`
   - Vérifiez l'intégrité de la base : Administration > Maintenance > Vérifier la base de données

3. **Problèmes d'inventaire** :
   - Vérifiez l'état de l'agent : `sudo systemctl status fusioninventory-agent`
   - Examinez les logs du serveur : `sudo tail -f /var/log/fusioninventory-agent/fusioninventory.log`

4. **Réinitialisation du mot de passe administrateur** :
   ```bash
   sudo mysql -u root -p
   USE glpidb;
   UPDATE glpi_users SET password=SHA2('nouveau_mot_de_passe_admin', 256) WHERE name='glpi';
   EXIT;
   ```

### Problèmes avec les plugins

1. **Plugin non visible dans l'interface** :
   - Vérifiez les permissions : `sudo chmod -R 755 /var/www/html/glpi/plugins/`
   - Vérifiez la compatibilité avec votre version de GLPI

2. **Erreurs lors de l'installation des plugins** :
   - Vérifiez les logs Apache
   - Vérifiez les dépendances PHP : `php -m`

## Références

- [Documentation officielle GLPI](https://glpi-project.org/fr/documentation-2/)
- [Documentation FusionInventory](https://fusioninventory.org/documentation/)
- [Plugin Dashboard pour GLPI](https://github.com/stdonato/glpi-dashboard)
- [Node Exporter pour Prometheus](https://github.com/prometheus/node_exporter)
- [Documentation Prometheus](https://prometheus.io/docs/introduction/overview/)
- [Documentation Grafana](https://grafana.com/docs/) 