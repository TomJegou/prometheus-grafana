# Procédure d'Installation et Configuration - Serveur de Connexion Linux

Ce document détaille les étapes d'installation et de configuration d'un serveur de connexion Linux, offrant soit des fonctionnalités de VPN, soit de reverse proxy, pour sécuriser l'accès aux ressources du réseau.

## Table des matières

1. [Prérequis](#prérequis)
2. [Option 1: Installation d'un serveur VPN (OpenVPN)](#option-1-installation-dun-serveur-vpn-openvpn)
3. [Option 2: Installation d'un reverse proxy (Nginx)](#option-2-installation-dun-reverse-proxy-nginx)
4. [Configuration du monitoring avec Prometheus](#configuration-du-monitoring-avec-prometheus)
5. [Dépannage](#dépannage)
6. [Sécurisation](#sécurisation)
7. [Références](#références)

## Prérequis

- Une machine virtuelle Linux (Ubuntu Server 22.04 LTS recommandé)
- Accès SSH à la machine avec privilèges sudo
- Adresse IP publique ou routage correctement configuré sur le réseau
- Ports nécessaires ouverts sur le pare-feu (selon la solution choisie)

## Option 1: Installation d'un serveur VPN (OpenVPN)

### 1.1. Préparation du système

```bash
# Mise à jour du système
sudo apt update && sudo apt upgrade -y

# Installation des outils de base
sudo apt install -y net-tools curl wget
```

### 1.2. Installation d'OpenVPN

#### Utilisation du script easy-rsa

```bash
# Installation des paquets OpenVPN et easy-rsa
sudo apt install -y openvpn easy-rsa

# Copie des fichiers easy-rsa dans le répertoire OpenVPN
mkdir -p /etc/openvpn/easy-rsa/
cp -r /usr/share/easy-rsa/* /etc/openvpn/easy-rsa/

# Se placer dans le répertoire easy-rsa
cd /etc/openvpn/easy-rsa/
```

#### Création de l'autorité de certification (CA)

```bash
# Initialisation du PKI
./easyrsa init-pki

# Création de l'autorité de certification
./easyrsa build-ca nopass

# Génération du certificat et de la clé du serveur
./easyrsa build-server-full server nopass

# Génération des paramètres Diffie-Hellman
./easyrsa gen-dh

# Génération d'une clé TLS pour renforcer la sécurité
openvpn --genkey --secret /etc/openvpn/ta.key
```

### 1.3. Configuration du serveur OpenVPN

```bash
# Copie des fichiers de certificats et clés
cp pki/ca.crt /etc/openvpn/
cp pki/issued/server.crt /etc/openvpn/
cp pki/private/server.key /etc/openvpn/
cp pki/dh.pem /etc/openvpn/

# Création du fichier de configuration du serveur
sudo nano /etc/openvpn/server.conf
```

Contenu du fichier `/etc/openvpn/server.conf`:

```
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
tls-auth ta.key 0
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
keepalive 10 120
cipher AES-256-CBC
user nobody
group nogroup
persist-key
persist-tun
status openvpn-status.log
log-append openvpn.log
verb 3
```

### 1.4. Configuration du routage et du pare-feu

```bash
# Activation du forwarding IP
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Configuration du pare-feu
# Remplacez eth0 par votre interface réseau
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
sudo iptables -A INPUT -i tun0 -j ACCEPT
sudo iptables -A FORWARD -i tun0 -j ACCEPT
sudo iptables -A FORWARD -i tun0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o tun0 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Rendre les règles persistantes
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

### 1.5. Démarrage du service OpenVPN

```bash
# Démarrage du service
sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server

# Vérification du statut
sudo systemctl status openvpn@server
```

### 1.6. Création de clients VPN

```bash
# Génération d'un certificat client
cd /etc/openvpn/easy-rsa/
./easyrsa build-client-full client1 nopass

# Création du fichier de configuration client
mkdir -p ~/client-configs/files
cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~/client-configs/base.conf

nano ~/client-configs/base.conf
```

Modifiez `base.conf` pour qu'il contienne :

```
client
dev tun
proto udp
remote your_server_ip 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-CBC
verb 3
```

Créez un script pour générer des configurations client :

```bash
nano ~/client-configs/make_config.sh
```

Contenu du script :

```bash
#!/bin/bash

# First argument: Client identifier

KEY_DIR=/etc/openvpn/easy-rsa/pki
OUTPUT_DIR=~/client-configs/files
BASE_CONFIG=~/client-configs/base.conf

cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/issued/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/private/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    /etc/openvpn/ta.key \
    <(echo -e '</tls-auth>') \
    > ${OUTPUT_DIR}/${1}.ovpn

echo "Configuration client générée : ${OUTPUT_DIR}/${1}.ovpn"
```

Rendez le script exécutable et générez un profil client :

```bash
chmod 700 ~/client-configs/make_config.sh
~/client-configs/make_config.sh client1
```

## Option 2: Installation d'un reverse proxy (Nginx)

### 2.1. Préparation du système

```bash
# Mise à jour du système
sudo apt update && sudo apt upgrade -y

# Installation des outils de base
sudo apt install -y curl gnupg2 ca-certificates lsb-release
```

### 2.2. Installation de Nginx

```bash
# Ajout du dépôt Nginx officiel
echo "deb https://nginx.org/packages/ubuntu/ $(lsb_release -cs) nginx" | sudo tee /etc/apt/sources.list.d/nginx.list
curl -fsSL https://nginx.org/keys/nginx_signing.key | sudo apt-key add -

# Installation de Nginx
sudo apt update
sudo apt install -y nginx

# Démarrage et activation du service
sudo systemctl start nginx
sudo systemctl enable nginx

# Vérification du statut
sudo systemctl status nginx
```

### 2.3. Configuration du pare-feu

```bash
# Ouverture des ports HTTP et HTTPS
sudo ufw allow 'Nginx Full'
sudo ufw enable
sudo ufw status
```

### 2.4. Obtention d'un certificat SSL avec Let's Encrypt

```bash
# Installation de Certbot
sudo apt install -y certbot python3-certbot-nginx

# Obtention d'un certificat
sudo certbot --nginx -d example.com -d www.example.com
```

### 2.5. Configuration du reverse proxy

Création d'un fichier de configuration pour chaque site :

```bash
sudo nano /etc/nginx/conf.d/example.com.conf
```

Exemple de configuration de reverse proxy pour une application web interne :

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name example.com www.example.com;
    
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    
    # Configuration du buffer
    proxy_buffer_size 128k;
    proxy_buffers 4 256k;
    proxy_busy_buffers_size 256k;
    
    # Configurations des en-têtes de proxy
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    
    location / {
        proxy_pass http://internal-app-server:8080;
    }
}
```

Tester et recharger la configuration :

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 2.6. Configurations avancées pour la sécurité

```bash
sudo nano /etc/nginx/conf.d/security.conf
```

Ajoutez les configurations suivantes :

```nginx
# Configuration des en-têtes de sécurité
add_header X-Content-Type-Options nosniff;
add_header X-Frame-Options SAMEORIGIN;
add_header X-XSS-Protection "1; mode=block";
add_header Content-Security-Policy "default-src 'self'";
add_header Referrer-Policy no-referrer-when-downgrade;

# Désactivation de l'affichage de la version de Nginx
server_tokens off;

# Limitation du taux de requêtes
limit_req_zone $binary_remote_addr zone=ratelimit:10m rate=10r/s;
```

## Configuration du monitoring avec Prometheus

### Configuration d'exporteurs pour OpenVPN

#### 1. Installation de l'exporteur OpenVPN

```bash
# Installation des dépendances
sudo apt install -y python3-pip
pip3 install prometheus-client

# Téléchargement de l'exporteur
wget https://raw.githubusercontent.com/kumina/openvpn_exporter/master/openvpn_exporter.py -O /usr/local/bin/openvpn_exporter.py
chmod +x /usr/local/bin/openvpn_exporter.py

# Création d'un service systemd
sudo nano /etc/systemd/system/openvpn-exporter.service
```

Contenu du fichier de service :

```ini
[Unit]
Description=Prometheus OpenVPN Exporter
After=network.target

[Service]
User=nobody
Group=nogroup
Type=simple
ExecStart=/usr/local/bin/openvpn_exporter.py --status-file /etc/openvpn/openvpn-status.log

[Install]
WantedBy=multi-user.target
```

```bash
# Activation du service
sudo systemctl daemon-reload
sudo systemctl start openvpn-exporter.service
sudo systemctl enable openvpn-exporter.service
```

### Configuration d'exporteurs pour Nginx

#### 1. Installation de nginx-prometheus-exporter

```bash
# Téléchargement de l'exporteur
cd /tmp
wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v0.11.0/nginx-prometheus-exporter_0.11.0_linux_amd64.tar.gz
tar -xvf nginx-prometheus-exporter_0.11.0_linux_amd64.tar.gz
sudo mv nginx-prometheus-exporter /usr/local/bin/

# Activation des métriques stub_status dans Nginx
sudo nano /etc/nginx/conf.d/status.conf
```

Ajoutez la configuration suivante :

```nginx
server {
    listen 127.0.0.1:8080;
    server_name localhost;

    location /nginx_status {
        stub_status on;
        access_log off;
        allow 127.0.0.1;
        deny all;
    }
}
```

```bash
# Création d'un service systemd
sudo nano /etc/systemd/system/nginx-exporter.service
```

Contenu du fichier de service :

```ini
[Unit]
Description=Prometheus Nginx Exporter
After=network.target

[Service]
User=nobody
Group=nogroup
Type=simple
ExecStart=/usr/local/bin/nginx-prometheus-exporter -nginx.scrape-uri=http://127.0.0.1:8080/nginx_status

[Install]
WantedBy=multi-user.target
```

```bash
# Activation du service
sudo systemctl daemon-reload
sudo systemctl start nginx-exporter.service
sudo systemctl enable nginx-exporter.service
```

### Configuration de Prometheus

Ajoutez les cibles dans la configuration de Prometheus :

```yaml
scrape_configs:
  - job_name: 'openvpn'
    static_configs:
      - targets: ['vpn-server-ip:9176']

  - job_name: 'nginx'
    static_configs:
      - targets: ['reverse-proxy-ip:9113']
```

## Métriques Surveillées

### Pour OpenVPN

- **openvpn_server_up**: État du serveur OpenVPN
- **openvpn_status_update_time**: Dernier timestamp de mise à jour du statut
- **openvpn_connected_clients**: Nombre de clients connectés
- **openvpn_client_received_bytes_total**: Total des octets reçus par client
- **openvpn_client_sent_bytes_total**: Total des octets envoyés par client

### Pour Nginx

- **nginx_up**: État du serveur Nginx
- **nginx_connections_active**: Nombre de connexions actives
- **nginx_connections_reading**: Connexions en lecture
- **nginx_connections_writing**: Connexions en écriture
- **nginx_connections_waiting**: Connexions en attente
- **nginx_http_requests_total**: Nombre total de requêtes HTTP

## Dépannage

### Problèmes courants avec OpenVPN

1. **Le serveur ne démarre pas** :
   ```bash
   sudo journalctl -u openvpn@server
   ```

2. **Les clients ne peuvent pas se connecter** :
   - Vérifiez le statut du serveur : `sudo systemctl status openvpn@server`
   - Vérifiez les logs : `sudo tail -f /var/log/openvpn.log`
   - Vérifiez la configuration du pare-feu : `sudo iptables -L -n -v`

3. **Problèmes de routage** :
   - Vérifiez le forwarding IP : `cat /proc/sys/net/ipv4/ip_forward`
   - Vérifiez les routes : `ip route`

### Problèmes courants avec Nginx

1. **Erreurs 502 Bad Gateway** :
   - Vérifiez si le service backend est accessible
   - Vérifiez les logs Nginx : `sudo tail -f /var/log/nginx/error.log`

2. **Problèmes de certificats SSL** :
   - Vérifiez les dates d'expiration : `sudo certbot certificates`
   - Renouvelez les certificats : `sudo certbot renew --dry-run`

3. **Performances lentes** :
   - Vérifiez la charge du serveur : `top` ou `htop`
   - Ajustez les paramètres de worker et de buffer dans `nginx.conf`

## Sécurisation

### Sécurisation d'OpenVPN

1. **Hardening des paramètres de chiffrement** :
   ```bash
   sudo nano /etc/openvpn/server.conf
   ```
   Ajoutez ou modifiez :
   ```
   cipher AES-256-GCM
   auth SHA512
   tls-cipher TLS-DHE-RSA-WITH-AES-256-GCM-SHA384
   tls-version-min 1.2
   ```

2. **Utilisation d'une liste de révocation de certificats (CRL)** :
   ```bash
   cd /etc/openvpn/easy-rsa/
   ./easyrsa gen-crl
   cp pki/crl.pem /etc/openvpn/
   ```
   Ajoutez à server.conf : `crl-verify crl.pem`

### Sécurisation de Nginx

1. **Configuration SSL renforcée** :
   ```bash
   sudo nano /etc/nginx/conf.d/ssl.conf
   ```
   ```nginx
   ssl_protocols TLSv1.2 TLSv1.3;
   ssl_prefer_server_ciphers on;
   ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
   ssl_session_cache shared:SSL:10m;
   ssl_session_timeout 10m;
   ssl_session_tickets off;
   ssl_stapling on;
   ssl_stapling_verify on;
   ```

2. **Protection contre les attaques DDoS** :
   ```nginx
   # Limitation du nombre de connexions par IP
   limit_conn_zone $binary_remote_addr zone=conn_limit_per_ip:10m;
   limit_conn conn_limit_per_ip 10;
   
   # Limitation du taux de requêtes
   limit_req_zone $binary_remote_addr zone=req_limit_per_ip:10m rate=5r/s;
   limit_req zone=req_limit_per_ip burst=10 nodelay;
   ```

## Références

- [Documentation officielle OpenVPN](https://openvpn.net/community-resources/)
- [Documentation officielle Nginx](https://nginx.org/en/docs/)
- [Let's Encrypt](https://letsencrypt.org/docs/)
- [OpenVPN Exporter pour Prometheus](https://github.com/kumina/openvpn_exporter)
- [Nginx Prometheus Exporter](https://github.com/nginxinc/nginx-prometheus-exporter)
- [Documentation Prometheus](https://prometheus.io/docs/introduction/overview/) 