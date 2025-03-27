# Procédure de Gestion et Surveillance des Certificats SSL/TLS

Ce document détaille les étapes de mise en place d'un système de surveillance des certificats SSL/TLS pour prévenir les expirations inattendues et garantir la sécurité de l'infrastructure.

## Table des matières

1. [Prérequis](#prérequis)
2. [Installation du service de surveillance](#installation-du-service-de-surveillance)
3. [Configuration des exporteurs](#configuration-des-exporteurs)
4. [Intégration avec Prometheus](#intégration-avec-prometheus)
5. [Configuration des alertes](#configuration-des-alertes)
6. [Tableaux de bord Grafana](#tableaux-de-bord-grafana)
7. [Gestion des renouvellements](#gestion-des-renouvellements)
8. [Automatisation avec certbot](#automatisation-avec-certbot)
9. [Dépannage](#dépannage)
10. [Références](#références)

## Prérequis

- Une machine virtuelle Linux (Ubuntu Server 22.04 LTS recommandé)
- Accès administrateur à la machine
- Accès aux serveurs hébergeant les certificats à surveiller
- Prometheus déjà configuré
- Grafana déjà configuré
- Connexion Internet pour l'accès aux API externes (optionnel)

## Installation du service de surveillance

### 1.1. Installation de SSL Exporter

```bash
# Création du répertoire pour SSL Exporter
sudo mkdir -p /opt/ssl-exporter

# Téléchargement de SSL Exporter
cd /tmp
wget https://github.com/ribbybibby/ssl_exporter/releases/download/v2.4.1/ssl_exporter-2.4.1.linux-amd64.tar.gz

# Extraction
tar -xzf ssl_exporter-2.4.1.linux-amd64.tar.gz

# Déplacement des fichiers
sudo cp ssl_exporter-2.4.1.linux-amd64/ssl_exporter /opt/ssl-exporter/
```

### 1.2. Configuration du service

```bash
# Création du fichier de configuration
sudo nano /opt/ssl-exporter/config.yml
```

Contenu du fichier `config.yml` :

```yaml
modules:
  https:
    prober: https
    timeout: 5s
    http:
      preferred_ip_protocol: "ip4"
      tls_config:
        insecure_skip_verify: false
  tcp:
    prober: tcp
    timeout: 5s
    tcp:
      preferred_ip_protocol: "ip4"
      tls_config:
        insecure_skip_verify: false
  smtp:
    prober: tcp
    timeout: 5s
    tcp:
      preferred_ip_protocol: "ip4"
      tls_config:
        insecure_skip_verify: false
      query_response:
        - expect: "^220 ([^ ]+) ESMTP"
        - send: "EHLO ssl_exporter"
        - expect: "^250-"
        - send: "STARTTLS"
        - expect: "^220"
        - starttls: true
```

### 1.3. Création du service systemd

```bash
# Création du fichier de service systemd
sudo nano /etc/systemd/system/ssl-exporter.service
```

Contenu du fichier `ssl-exporter.service` :

```ini
[Unit]
Description=SSL Certificate Exporter
After=network.target

[Service]
Type=simple
User=nobody
Group=nogroup
ExecStart=/opt/ssl-exporter/ssl_exporter --config.file=/opt/ssl-exporter/config.yml
Restart=always
RestartSec=1
StartLimitInterval=0

[Install]
WantedBy=multi-user.target
```

### 1.4. Démarrage du service

```bash
# Activation des permissions
sudo chmod +x /opt/ssl-exporter/ssl_exporter

# Activation et démarrage du service
sudo systemctl daemon-reload
sudo systemctl enable ssl-exporter
sudo systemctl start ssl-exporter

# Vérification du statut
sudo systemctl status ssl-exporter
```

### 1.5. Configuration du pare-feu

```bash
# Ouverture du port pour SSL Exporter
sudo ufw allow 9219/tcp
```

## Configuration des exporteurs

### 2.1. Création du fichier de cibles

Créez un fichier pour répertorier tous les certificats à surveiller :

```bash
sudo nano /opt/ssl-exporter/targets.yml
```

Exemple de contenu pour `targets.yml` :

```yaml
- targets:
  - glpi.votredomaine.com:443
  - vpn.votredomaine.com:443
  - mail.votredomaine.com:443
  - ldaps.votredomaine.com:636
  - radius.votredomaine.com:443
  labels:
    env: production
- targets:
  - test.votredomaine.com:443
  - dev.votredomaine.com:443
  labels:
    env: test
```

### 2.2. Installation de l'exporteur Blackbox (optionnel pour vérifications supplémentaires)

```bash
# Téléchargement de Blackbox Exporter
cd /tmp
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.23.0/blackbox_exporter-0.23.0.linux-amd64.tar.gz

# Extraction
tar -xzf blackbox_exporter-0.23.0.linux-amd64.tar.gz

# Création du répertoire
sudo mkdir -p /opt/blackbox-exporter

# Déplacement des fichiers
sudo cp blackbox_exporter-0.23.0.linux-amd64/blackbox_exporter /opt/blackbox-exporter/
```

Configuration de Blackbox Exporter :

```bash
sudo nano /opt/blackbox-exporter/config.yml
```

Contenu du fichier `config.yml` :

```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      method: GET
      preferred_ip_protocol: "ip4"
      tls_config:
        insecure_skip_verify: false
  http_2xx_verify_cert:
    prober: http
    timeout: 5s
    http:
      method: GET
      preferred_ip_protocol: "ip4"
      fail_if_ssl: false
      fail_if_not_ssl: true
      tls_config:
        insecure_skip_verify: false
  tcp_tls:
    prober: tcp
    timeout: 5s
    tcp:
      preferred_ip_protocol: "ip4"
      tls: true
      tls_config:
        insecure_skip_verify: false
```

Création du service systemd pour Blackbox :

```bash
sudo nano /etc/systemd/system/blackbox-exporter.service
```

Contenu du fichier `blackbox-exporter.service` :

```ini
[Unit]
Description=Blackbox Exporter
After=network.target

[Service]
Type=simple
User=nobody
Group=nogroup
ExecStart=/opt/blackbox-exporter/blackbox_exporter --config.file=/opt/blackbox-exporter/config.yml
Restart=always
RestartSec=1
StartLimitInterval=0

[Install]
WantedBy=multi-user.target
```

```bash
# Activation des permissions
sudo chmod +x /opt/blackbox-exporter/blackbox_exporter

# Activation et démarrage du service
sudo systemctl daemon-reload
sudo systemctl enable blackbox-exporter
sudo systemctl start blackbox-exporter

# Ouverture du port
sudo ufw allow 9115/tcp
```

### 2.3. Script personnalisé pour les certificats internes

Pour les certificats internes ou ceux qui ne sont pas accessibles directement, créez un script personnalisé :

```bash
sudo nano /opt/ssl-exporter/check_certs.sh
```

Contenu du script `check_certs.sh` :

```bash
#!/bin/bash

# Répertoire des certificats internes
CERT_DIR="/etc/ssl/certs/internal"

# Fichier de sortie au format Node Exporter
OUTPUT_FILE="/var/lib/node_exporter/textfile_collector/ssl_certificates.prom"

# Créez le répertoire si nécessaire
mkdir -p /var/lib/node_exporter/textfile_collector/

# Supprimez l'ancien fichier
> $OUTPUT_FILE

# Parcourez tous les certificats
for cert_file in $CERT_DIR/*.crt; do
  cert_name=$(basename $cert_file)
  
  # Obtenir la date d'expiration et calculer les jours restants
  expiry_date=$(openssl x509 -enddate -noout -in $cert_file | cut -d= -f2)
  expiry_epoch=$(date -d "$expiry_date" +%s)
  current_epoch=$(date +%s)
  days_left=$(( (expiry_epoch - current_epoch) / 86400 ))
  
  # Écrire les métriques
  echo "ssl_certificate_expiry_days{certificate=\"$cert_name\"} $days_left" >> $OUTPUT_FILE
  
  # Vérifier la validité (1 si valide, 0 si expiré)
  if [ $days_left -gt 0 ]; then
    echo "ssl_certificate_valid{certificate=\"$cert_name\"} 1" >> $OUTPUT_FILE
  else
    echo "ssl_certificate_valid{certificate=\"$cert_name\"} 0" >> $OUTPUT_FILE
  fi
done
```

Rendez le script exécutable et configurez un cron pour l'exécuter régulièrement :

```bash
sudo chmod +x /opt/ssl-exporter/check_certs.sh

# Ajouter au cron pour une exécution toutes les heures
(crontab -l 2>/dev/null; echo "0 * * * * /opt/ssl-exporter/check_certs.sh") | crontab -
```

### 2.4. Configuration de Node Exporter pour les métriques textuelles

Si vous utilisez déjà Node Exporter pour d'autres métriques, configurez-le pour collecter les métriques textuelles :

```bash
# Modifiez le service Node Exporter
sudo systemctl stop node_exporter
sudo nano /etc/systemd/system/node_exporter.service
```

Ajoutez l'option `--collector.textfile.directory` :

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter --collector.textfile.directory=/var/lib/node_exporter/textfile_collector

[Install]
WantedBy=multi-user.target
```

Redémarrez Node Exporter :

```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
```

## Intégration avec Prometheus

### 3.1. Configuration de Prometheus pour SSL Exporter

Ajoutez les cibles dans la configuration de Prometheus :

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Ajoutez les configurations suivantes :

```yaml
scrape_configs:
  # Configuration pour SSL Exporter
  - job_name: 'ssl_exporter'
    static_configs:
      - targets: ['localhost:9219']
    metrics_path: /metrics

  # Configuration pour les certificats HTTPS
  - job_name: 'ssl_https'
    metrics_path: /probe
    params:
      module: [https]
    file_sd_configs:
      - files:
          - /opt/ssl-exporter/targets.yml
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9219

  # Configuration pour Blackbox Exporter
  - job_name: 'blackbox_tls'
    metrics_path: /probe
    params:
      module: [http_2xx_verify_cert]
    file_sd_configs:
      - files:
          - /opt/ssl-exporter/targets.yml
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115
        
  # Configuration pour les métriques textuelles (certificats internes)
  - job_name: 'node_exporter_textfile'
    static_configs:
      - targets: ['localhost:9100']
```

Rechargez la configuration de Prometheus :

```bash
sudo systemctl reload prometheus
```

## Configuration des alertes

### 4.1. Création des règles d'alerte

```bash
sudo nano /etc/prometheus/rules/certificate_alerts.yml
```

Contenu du fichier `certificate_alerts.yml` :

```yaml
groups:
- name: certificates
  rules:
  # Alerte pour certificat expirant dans moins de 30 jours
  - alert: CertificateExpiringInNext30Days
    expr: ssl_tls_connect_success == 1 and ssl_tls_connect_tls_version_info * on(instance) group_right ssl_tls_connect_cert_not_after < (time() + 86400 * 30)
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Certificat expirant dans les 30 jours"
      description: "Le certificat pour {{ $labels.instance }} expire dans moins de 30 jours (expire le {{ $value | humanizeTimestamp }})"

  # Alerte pour certificat expirant dans moins de 7 jours
  - alert: CertificateExpiringInNext7Days
    expr: ssl_tls_connect_success == 1 and ssl_tls_connect_tls_version_info * on(instance) group_right ssl_tls_connect_cert_not_after < (time() + 86400 * 7)
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Certificat expirant dans les 7 jours"
      description: "Le certificat pour {{ $labels.instance }} expire dans moins de 7 jours (expire le {{ $value | humanizeTimestamp }})"

  # Alerte pour certificat expiré
  - alert: CertificateExpired
    expr: ssl_tls_connect_success == 1 and ssl_tls_connect_tls_version_info * on(instance) group_right ssl_tls_connect_cert_not_after < time()
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Certificat expiré"
      description: "Le certificat pour {{ $labels.instance }} a expiré le {{ $value | humanizeTimestamp }}"

  # Alerte pour certificat utilisant un algorithme faible
  - alert: CertificateWeakCipher
    expr: ssl_tls_connect_success == 1 and ssl_tls_connect_tls_version_info{cipher=~".*RC4.*|.*MD5.*|.*SHA1.*"} > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Certificat utilisant un algorithme faible"
      description: "Le certificat pour {{ $labels.instance }} utilise un algorithme cryptographique faible: {{ $labels.cipher }}"
      
  # Alerte pour certificat avec chaîne de confiance incorrecte
  - alert: CertificateBrokenChain
    expr: ssl_tls_connect_cert_verify_error_code > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Chaîne de certificats incomplète ou incorrecte"
      description: "Le certificat pour {{ $labels.instance }} a une chaîne de confiance incorrecte ou incomplète (code d'erreur: {{ $value }})"

  # Alerte pour certificats internes
  - alert: InternalCertificateExpiring
    expr: ssl_certificate_expiry_days < 30
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Certificat interne expirant bientôt"
      description: "Le certificat interne {{ $labels.certificate }} expire dans {{ $value }} jours"
```

Mettez à jour la configuration de Prometheus pour inclure ces règles :

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Ajoutez la référence au fichier de règles :

```yaml
rule_files:
  - "rules/certificate_alerts.yml"
```

Rechargez Prometheus :

```bash
sudo systemctl reload prometheus
```

## Tableaux de bord Grafana

### 5.1. Création d'un tableau de bord pour les certificats

1. Connectez-vous à Grafana
2. Cliquez sur "+" puis "Dashboard"
3. Cliquez sur "Add new panel"

Voici quelques panneaux utiles à ajouter :

#### Panneau 1: Vue d'ensemble des certificats

```
# Requête
ssl_tls_connect_cert_not_after - time()
# Transformation en jours
/ 86400
# Filtrer les résultats
> 0
```

Configuration du panneau :
- Type: Gauge
- Min: 0, Max: 90
- Thresholds: 0-7 (rouge), 7-30 (orange), 30-90 (vert)

#### Panneau 2: Liste des certificats par date d'expiration

```
# Format de table
ssl_tls_connect_cert_not_after
```

Configuration du panneau :
- Type: Table
- Colonnes: Instance, Issuer, Not After
- Transformation: Format des dates en format lisible

#### Panneau 3: Certificats utilisant des algorithmes faibles

```
ssl_tls_connect_tls_version_info{cipher=~".*RC4.*|.*MD5.*|.*SHA1.*"}
```

Configuration du panneau :
- Type: Table
- Colonnes: Instance, Cipher, Protocol

#### Panneau 4: Certificats internes

```
ssl_certificate_expiry_days
```

Configuration du panneau :
- Type: Table
- Colonnes: Certificate, Expiry Days

### 5.2. Importation du tableau de bord

Alternativement, vous pouvez importer un tableau de bord prédéfini. Plusieurs sont disponibles dans la communauté Grafana :

1. Dans Grafana, allez à "+" puis "Import"
2. Entrez l'ID 1301 (SSL Exporter dashboard) ou 13230 (Certificate Expiry dashboard)
3. Cliquez sur "Load"
4. Sélectionnez votre source de données Prometheus
5. Cliquez sur "Import"

## Gestion des renouvellements

### 6.1. Processus de renouvellement manuel

Voici les étapes générales pour renouveler un certificat SSL/TLS :

1. **Génération d'une demande de signature de certificat (CSR)** :

```bash
openssl req -new -newkey rsa:2048 -nodes -keyout domaine.key -out domaine.csr
```

2. **Soumission de la CSR à l'autorité de certification** :
   - Suivez les instructions spécifiques à votre autorité de certification
   - Téléchargez le certificat signé

3. **Installation du nouveau certificat** :
   - Copiez le certificat et la clé dans le répertoire approprié
   - Mettez à jour les configurations du service concerné

4. **Redémarrage du service** :

```bash
sudo systemctl restart apache2  # Exemple pour Apache
# OU
sudo systemctl restart nginx    # Exemple pour Nginx
```

### 6.2. Automatisation avec certbot pour Let's Encrypt

#### Installation de certbot

```bash
sudo apt update
sudo apt install -y certbot
```

Pour Apache :

```bash
sudo apt install -y python3-certbot-apache
```

Pour Nginx :

```bash
sudo apt install -y python3-certbot-nginx
```

#### Obtention d'un certificat

Pour Apache :

```bash
sudo certbot --apache -d example.com -d www.example.com
```

Pour Nginx :

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

Pour un certificat autonome :

```bash
sudo certbot certonly --standalone -d example.com -d www.example.com
```

#### Configuration du renouvellement automatique

Certbot installe automatiquement une tâche cron ou un timer systemd pour le renouvellement. Vous pouvez vérifier que tout fonctionne correctement avec :

```bash
sudo certbot renew --dry-run
```

#### Script de notification post-renouvellement

Créez un script pour envoyer des notifications après le renouvellement :

```bash
sudo nano /etc/letsencrypt/renewal-hooks/post/notification.sh
```

Contenu du script :

```bash
#!/bin/bash

# Variables
ADMIN_EMAIL="admin@votredomaine.com"
SLACK_WEBHOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

# Envoyer un e-mail de notification
echo "Certificat Let's Encrypt renouvelé pour $RENEWED_DOMAINS" | mail -s "Renouvellement de certificat" $ADMIN_EMAIL

# Envoyer une notification Slack
curl -X POST -H 'Content-type: application/json' --data "{\"text\":\"Certificat Let's Encrypt renouvelé pour $RENEWED_DOMAINS\"}" $SLACK_WEBHOOK_URL
```

Rendez le script exécutable :

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/post/notification.sh
```

### 6.3. Gestion des certificats auto-signés

Pour les services internes, vous pouvez utiliser des certificats auto-signés :

```bash
# Génération d'une autorité de certification (CA) interne
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt

# Génération d'une clé pour le service
openssl genrsa -out service.key 2048

# Création d'une demande de signature (CSR)
openssl req -new -key service.key -out service.csr

# Création d'un fichier de configuration
cat > service.ext << EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = service.internal.domain
EOF

# Signature du certificat par la CA interne
openssl x509 -req -in service.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out service.crt -days 365 -sha256 -extfile service.ext
```

## Dépannage

### 7.1. Diagnostic des problèmes de certificats

Voici quelques commandes utiles pour diagnostiquer les problèmes de certificats :

#### Vérification d'un certificat distant

```bash
# Vérifier un certificat HTTPS
openssl s_client -connect example.com:443 -servername example.com

# Examiner les détails d'un certificat
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | openssl x509 -text

# Vérifier la date d'expiration
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | openssl x509 -noout -dates
```

#### Vérification d'un certificat local

```bash
# Afficher le contenu d'un certificat
openssl x509 -in certificate.crt -text -noout

# Vérifier la correspondance entre clé privée et certificat
openssl x509 -noout -modulus -in certificate.crt | openssl md5
openssl rsa -noout -modulus -in private.key | openssl md5
```

#### Vérification d'une chaîne de certificats

```bash
# Vérifier la chaîne complète
openssl verify -CAfile chain.pem certificate.crt
```

### 7.2. Problèmes courants et solutions

#### Erreur de nom d'hôte non concordant

**Problème** : Le nom d'hôte dans le certificat ne correspond pas au nom de domaine utilisé.

**Solution** : 
1. Vérifiez les SAN (Subject Alternative Names) dans le certificat
2. Assurez-vous d'utiliser le bon nom d'hôte
3. Générez un nouveau certificat avec le bon nom d'hôte

#### Certificat expiré

**Problème** : Le certificat a dépassé sa date de validité.

**Solution** : 
1. Renouvelez immédiatement le certificat
2. Configurez des alertes préventives (comme décrit dans ce document)
3. Mettez en place un renouvellement automatique

#### Erreur de chaîne de confiance

**Problème** : La chaîne de certificats est incomplète ou incorrecte.

**Solution** : 
1. Vérifiez que tous les certificats intermédiaires sont correctement installés
2. Téléchargez la chaîne complète auprès de votre autorité de certification
3. Vérifiez l'ordre des certificats dans le fichier de chaîne

#### Problème de clé privée

**Problème** : Clé privée ne correspondant pas au certificat ou mal protégée.

**Solution** : 
1. Vérifiez la correspondance entre la clé et le certificat
2. Vérifiez les permissions du fichier de clé (600 ou 400)
3. Si nécessaire, générez une nouvelle paire clé/certificat

## Références

- [SSL Exporter GitHub](https://github.com/ribbybibby/ssl_exporter)
- [Blackbox Exporter GitHub](https://github.com/prometheus/blackbox_exporter)
- [Certbot Documentation](https://certbot.eff.org/docs/)
- [Let's Encrypt](https://letsencrypt.org/docs/)
- [OpenSSL Documentation](https://www.openssl.org/docs/)
- [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) 