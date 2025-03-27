# Projet de Monitoring - Équipe SOC

Ce dépôt contient la documentation et les configurations pour la mise en place de l'infrastructure de monitoring avec Prometheus et Grafana pour l'équipe SOC.

## Objectifs du Projet

Mise en place d'une solution de monitoring basée sur Prometheus et Grafana pour surveiller les composants suivants :
- Serveur Radius
- Serveur de connexion Linux (VPN/Reverse-Proxy)
- Active Directory (AD)
- GLPI
- Validité des certificats

## Infrastructure

L'infrastructure est hébergée sur une dedibox contenant Proxmox, permettant la création et la gestion des machines virtuelles nécessaires.

## Composants

### 1. Serveur Radius
- **Description** : Service d'authentification réseau
- **Métriques surveillées** : Tentatives d'authentification, échecs, latence
- **Exporteurs utilisés** : [radius_server_exporter](https://github.com/bvantagelimited/radius_server_exporter)
- **Procédure d'installation** : [Voir documentation détaillée](installation_radius.md)

### 2. Serveur de Connexion Linux (VPN/Reverse-Proxy)
- **Description** : Sécurisation des accès aux ressources
- **Métriques surveillées** : Connexions actives, tentatives de connexion, bande passante
- **Exporteurs utilisés** : [openvpn_exporter](https://github.com/kumina/openvpn_exporter) ou [nginx-prometheus-exporter](https://github.com/nginxinc/nginx-prometheus-exporter)
- **Procédure d'installation** : [Voir documentation détaillée](installation_vpn_proxy.md)

### 3. Active Directory
- **Description** : Gestion des utilisateurs et des politiques de sécurité
- **Métriques surveillées** : Activité des utilisateurs, modifications des comptes, réplication
- **Exporteurs utilisés** : [windows_exporter](https://github.com/prometheus-community/windows_exporter)
- **Procédure d'installation** : [Voir documentation détaillée](installation_ad.md)

### 4. GLPI
- **Description** : Gestion des actifs informatiques
- **Métriques surveillées** : Tickets ouverts, temps de résolution, inventaire
- **Exporteurs utilisés** : [Détail des exporteurs]

### 5. Gestion des Certificats
- **Description** : Surveillance de la validité des certificats SSL/TLS
- **Métriques surveillées** : Date d'expiration, validité, alertes
- **Exporteurs utilisés** : [Détail des exporteurs]

## Architecture

```
┌─────────────┐     ┌────────────┐     ┌─────────────┐
│ Exporteurs  │────▶│ Prometheus │───▶│   Grafana   │
└─────────────┘     └────────────┘     └─────────────┘
       │                   │                  │
       │                   │                  │
       ▼                   ▼                  ▼
┌──────────────────────────────────────────────────┐
│                  Infrastructure                  │
│  (Radius, VPN/Reverse-Proxy, AD, GLPI, Certs)    │
└──────────────────────────────────────────────────┘
```

## Installation et Configuration

### Prérequis
- Accès à la dedibox Proxmox
- Templates Linux et Windows disponibles
- Droits d'administration sur les VM

### Étapes d'Installation
1. [Installation et configuration du serveur Radius](installation_radius.md)
2. [Installation et configuration du serveur VPN/Reverse-Proxy](installation_vpn_proxy.md)
3. [Installation et configuration d'Active Directory](installation_ad.md)
4. [Instructions détaillées pour configurer GLPI]
5. [Instructions détaillées pour la gestion des certificats]

### Configuration de Prometheus
1. [Instructions pour installer Prometheus]
2. [Instructions pour configurer les targets]
3. [Instructions pour configurer les règles d'alerte]

### Configuration de Grafana
1. [Instructions pour installer Grafana]
2. [Instructions pour connecter Prometheus comme source de données]
3. [Instructions pour créer les tableaux de bord]

## Tableaux de Bord

- **Tableau de bord général** : Vue d'ensemble de l'infrastructure
- **Tableau de bord Sécurité** : Focus sur les événements de sécurité
- **Tableau de bord Certificats** : Surveillance des certificats SSL/TLS
- **Tableau de bord GLPI** : Suivi des tickets et de l'inventaire
- **Tableau de bord Active Directory** : Surveillance des utilisateurs et des événements
- **Tableau de bord RADIUS** : Suivi des authentifications et de la performance du service
- **Tableau de bord VPN/Reverse-Proxy** : Suivi des connexions, des requêtes et de la performance

## Alertes

- Alerte en cas d'expiration imminente d'un certificat
- Alerte en cas d'échecs d'authentification répétés sur le serveur RADIUS
- Alerte en cas de problème de réplication AD
- Alerte en cas de surcharge des serveurs
- Alerte en cas d'indisponibilité d'un service
- Alerte en cas de nombre élevé de connexions VPN rejetées
- Alerte en cas d'erreurs HTTP 5xx sur le reverse proxy
- Alerte en cas d'échec de réplication Active Directory

## Procédures

- [Procédure d'installation et configuration du serveur RADIUS](installation_radius.md)
- [Procédure d'installation et configuration du serveur VPN/Reverse-Proxy](installation_vpn_proxy.md)
- [Procédure d'installation et configuration d'Active Directory](installation_ad.md)
- [Procédure de déploiement d'un nouvel exporteur]
- [Procédure de création d'un nouveau tableau de bord]
- [Procédure de configuration d'une nouvelle alerte]
- [Procédure de résolution des problèmes courants]

## Métriques Surveillées

### Serveur RADIUS
- **radius_server_up** : État du serveur RADIUS
- **radius_authentications_total** : Nombre total d'authentifications
- **radius_authentication_success_total** : Nombre d'authentifications réussies
- **radius_authentication_failure_total** : Nombre d'authentifications échouées
- **radius_authentication_latency** : Temps de réponse aux requêtes
- **radius_active_sessions** : Nombre de sessions actives

### Serveur VPN (OpenVPN)
- **openvpn_server_up** : État du serveur OpenVPN
- **openvpn_connected_clients** : Nombre de clients connectés
- **openvpn_client_received_bytes_total** : Total des octets reçus par client
- **openvpn_client_sent_bytes_total** : Total des octets envoyés par client

### Reverse Proxy (Nginx)
- **nginx_up** : État du serveur Nginx
- **nginx_connections_active** : Nombre de connexions actives
- **nginx_connections_reading** : Connexions en lecture
- **nginx_connections_writing** : Connexions en écriture
- **nginx_http_requests_total** : Nombre total de requêtes HTTP

### Active Directory
- **ad_ds_dra_inbound_values** : Taux de réplication AD entrant
- **ad_ds_dra_outbound_values** : Taux de réplication AD sortant
- **ad_ldap_bind_time** : Temps de liaison LDAP
- **ad_ldap_successful_binds** : Nombre de liaisons LDAP réussies
- **windows_service_state** : État des services Windows, y compris les services AD

## Références

- [Documentation officielle Prometheus](https://prometheus.io/docs/introduction/overview/)
- [Documentation officielle Grafana](https://grafana.com/docs/)
- [Documentation officielle FreeRADIUS](https://freeradius.org/documentation/)
- [Exporteur RADIUS pour Prometheus](https://github.com/bvantagelimited/radius_server_exporter)
- [Documentation OpenVPN](https://openvpn.net/community-resources/)
- [Documentation Nginx](https://nginx.org/en/docs/)
- [Documentation Microsoft Active Directory](https://docs.microsoft.com/fr-fr/windows-server/identity/ad-ds/active-directory-domain-services)
- [Windows Exporter pour Prometheus](https://github.com/prometheus-community/windows_exporter)
