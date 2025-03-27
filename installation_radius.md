# Procédure d'Installation et Configuration d'un Serveur RADIUS (FreeRADIUS)

## Prérequis
- Une machine virtuelle Linux (Ubuntu Server 22.04 LTS recommandé)
- Accès SSH à la machine
- Droits d'administration (sudo)
- Accès réseau aux équipements à authentifier

## Étapes d'Installation

### 1. Préparation du Système

```bash
# Mise à jour du système
sudo apt update
sudo apt upgrade -y

# Installation des dépendances
sudo apt install -y build-essential net-tools
```

### 2. Installation de FreeRADIUS

```bash
# Installation du serveur FreeRADIUS et des utilitaires
sudo apt install -y freeradius freeradius-utils freeradius-mysql
```

### 3. Configuration de Base

```bash
# Arrêt du service pour modification
sudo systemctl stop freeradius.service
```

#### Configuration des Clients RADIUS

Éditez le fichier de configuration des clients (/etc/freeradius/3.0/clients.conf) :

```bash
sudo nano /etc/freeradius/3.0/clients.conf
```

Ajoutez une entrée pour chaque client (équipement réseau) qui doit communiquer avec le serveur RADIUS :

```
client switch01 {
    ipaddr = 192.168.1.10
    secret = votre_clé_secrète_partagée
    shortname = switch01
    nastype = other
}
```

#### Configuration des Utilisateurs

Pour une configuration simple avec des utilisateurs locaux, éditez le fichier users :

```bash
sudo nano /etc/freeradius/3.0/users
```

Ajoutez les utilisateurs au format suivant :

```
utilisateur1  Cleartext-Password := "mot_de_passe1"
              Reply-Message = "Bienvenue, %{User-Name}"

utilisateur2  Cleartext-Password := "mot_de_passe2"
              Reply-Message = "Bienvenue, %{User-Name}"
```

### 4. Intégration avec Active Directory (Facultatif)

Pour utiliser l'AD comme source d'authentification :

```bash
# Installation des dépendances pour l'intégration AD
sudo apt install -y freeradius-ldap libpam-winbind

# Configuration de l'intégration avec Samba/Winbind
sudo nano /etc/samba/smb.conf
```

Ajoutez la configuration suivante à smb.conf :

```
[global]
    workgroup = VOTRE_DOMAINE
    realm = VOTRE_DOMAINE.COM
    security = ads
    winbind enum users = yes
    winbind enum groups = yes
    winbind use default domain = yes
    winbind refresh tickets = yes
```

Joindre le domaine :

```bash
sudo net ads join -U administrateur
```

Configurez le module LDAP de FreeRADIUS :

```bash
sudo nano /etc/freeradius/3.0/mods-available/ldap
```

Modifiez les paramètres suivants :

```
ldap {
    server = "ldap://votre-serveur-ad.domaine.com"
    identity = "CN=Service RADIUS,OU=Service Accounts,DC=domaine,DC=com"
    password = mot_de_passe
    base_dn = "DC=domaine,DC=com"
    user_dn = "OU=Users,DC=domaine,DC=com"
    filter = "(sAMAccountName=%{%{Stripped-User-Name}:-%{User-Name}})"
}
```

Activez le module LDAP :

```bash
sudo ln -s /etc/freeradius/3.0/mods-available/ldap /etc/freeradius/3.0/mods-enabled/
```

### 5. Configuration des Méthodes d'Authentification

#### Configuration de PEAP/MSCHAPv2 (pour le Wi-Fi d'entreprise)

```bash
sudo nano /etc/freeradius/3.0/sites-available/default
```

Assurez-vous que les sections EAP et MS-CHAP sont activées.

#### Génération des Certificats (pour EAP-TLS)

```bash
cd /etc/freeradius/3.0/certs/
sudo ./bootstrap
```

### 6. Test et Démarrage du Service

```bash
# Test de la configuration
sudo freeradius -X

# Si le test est réussi, démarrez le service et activez-le au démarrage
sudo systemctl start freeradius.service
sudo systemctl enable freeradius.service
```

### 7. Test d'Authentification

Testez l'authentification à l'aide de l'utilitaire radtest :

```bash
radtest utilisateur1 mot_de_passe1 localhost 0 testing123
```

Si l'authentification réussit, vous devriez voir "Access-Accept".

## 8. Configuration pour le Monitoring Prometheus

### Installation de l'Exporteur RADIUS

```bash
# Cloner le dépôt de l'exporteur
git clone https://github.com/bvantagelimited/radius_server_exporter.git
cd radius_server_exporter

# Installation des dépendances
pip3 install -r requirements.txt

# Configuration
cp config.yml.example config.yml
nano config.yml
```

Modifiez le fichier config.yml avec vos informations :

```yaml
radius_server:
  address: localhost
  port: 1812
  secret: testing123
  metrics_port: 9812

log:
  level: INFO
```

Créez un service systemd pour l'exporteur :

```bash
sudo nano /etc/systemd/system/radius_exporter.service
```

Contenu du fichier :

```
[Unit]
Description=RADIUS Prometheus Exporter
After=network.target

[Service]
Type=simple
User=radius
ExecStart=/usr/bin/python3 /chemin/vers/radius_server_exporter/radius_exporter.py
WorkingDirectory=/chemin/vers/radius_server_exporter
Restart=always

[Install]
WantedBy=multi-user.target
```

Démarrez et activez le service :

```bash
sudo systemctl daemon-reload
sudo systemctl start radius_exporter.service
sudo systemctl enable radius_exporter.service
```

### Configuration de Prometheus

Ajoutez la cible de l'exporteur dans la configuration de Prometheus :

```yaml
scrape_configs:
  - job_name: 'radius'
    static_configs:
      - targets: ['radius-server-ip:9812']
```

## Métriques de Surveillance

L'exporteur RADIUS collecte généralement les métriques suivantes :

- **radius_server_up** : Indique si le serveur RADIUS répond
- **radius_authentications_total** : Nombre total de tentatives d'authentification
- **radius_authentication_success_total** : Nombre d'authentifications réussies
- **radius_authentication_failure_total** : Nombre d'authentifications échouées
- **radius_authentication_latency** : Temps de réponse aux requêtes d'authentification
- **radius_accounting_requests_total** : Nombre de requêtes de comptabilité
- **radius_active_sessions** : Nombre de sessions actives

## Dépannage

### Problèmes Courants

1. **Échec de l'authentification** :
   - Vérifiez les logs : `sudo tail -f /var/log/freeradius/radius.log`
   - Vérifiez la configuration des clients et des secrets partagés

2. **Problèmes de certificats pour EAP** :
   - Régénérez les certificats : `cd /etc/freeradius/3.0/certs && sudo ./bootstrap`

3. **Problèmes d'intégration AD** :
   - Vérifiez la connexion au domaine : `wbinfo -t`
   - Testez la résolution d'utilisateurs : `wbinfo -u`

4. **Service qui ne démarre pas** :
   - Vérifiez les erreurs : `sudo systemctl status freeradius.service`
   - Exécutez en mode debug : `sudo freeradius -X`

## Sécurisation du Serveur RADIUS

1. **Restriction des accès réseau** :
   ```bash
   sudo ufw allow from 192.168.1.0/24 to any port 1812 proto udp
   sudo ufw allow from 192.168.1.0/24 to any port 1813 proto udp
   ```

2. **Rotation des Logs** :
   ```bash
   sudo nano /etc/logrotate.d/freeradius
   ```

3. **Mise à jour régulière** :
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

## Références

- [Documentation officielle FreeRADIUS](https://freeradius.org/documentation/)
- [Guide d'intégration FreeRADIUS-AD](https://wiki.freeradius.org/guide/Active_Directory)
- [Exporteur Prometheus pour RADIUS](https://github.com/bvantagelimited/radius_server_exporter)
- [RFC 2865 - RADIUS Protocol](https://tools.ietf.org/html/rfc2865) 