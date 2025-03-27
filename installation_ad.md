# Procédure d'Installation et Configuration - Active Directory

Ce document détaille les étapes d'installation et de configuration d'un serveur Active Directory (AD) pour la gestion centralisée des identités et des politiques de sécurité, ainsi que son intégration avec Prometheus pour le monitoring.

## Table des matières

1. [Prérequis](#prérequis)
2. [Installation de Windows Server](#installation-de-windows-server)
3. [Installation d'Active Directory](#installation-dactive-directory)
4. [Configuration post-installation](#configuration-post-installation)
5. [Sécurisation d'Active Directory](#sécurisation-dactive-directory)
6. [Configuration du monitoring avec Prometheus](#configuration-du-monitoring-avec-prometheus)
7. [Dépannage](#dépannage)
8. [Références](#références)

## Prérequis

- Une machine virtuelle (minimum 4 Go de RAM, 2 vCPU, 50 Go de disque)
- ISO Windows Server 2019 ou 2022
- Adresse IP statique pour le serveur AD
- Accès réseau pour les clients AD
- DNS correctement configuré

## Installation de Windows Server

### 1.1. Configuration de la VM dans Proxmox

1. Dans l'interface Proxmox, créez une nouvelle VM :
   - Cliquez sur "Create VM"
   - ID VM et Nom : Choisissez un identifiant et un nom descriptif (ex : "AD-Server")
   - OS : Sélectionnez Microsoft Windows, version Windows 2022
   - Disque : Minimum 50 GB, stockage SSD si disponible
   - CPU : Au moins 2 vCPU
   - Mémoire : Minimum 4 GB (8 GB recommandé)
   - Réseau : Sélectionnez le bridge correspondant à votre réseau

2. Démarrez la VM et montez l'ISO Windows Server.

### 1.2. Installation de l'OS

1. Suivez l'assistant d'installation Windows Server :
   - Choisissez "Windows Server 2022 Standard Edition (Desktop Experience)"
   - Acceptez les termes de licence
   - Choisissez "Installation personnalisée" 
   - Sélectionnez le disque préalablement créé

2. Une fois l'installation terminée, définissez le mot de passe administrateur.

### 1.3. Configuration initiale du serveur

1. Configurez l'adresse IP statique :
   - Accédez aux Paramètres réseau
   - Cliquez-droit sur l'adaptateur réseau > Propriétés
   - Sélectionnez IPv4 > Propriétés
   - Configurez une adresse IP statique appropriée pour votre réseau
   - Définissez l'adresse du serveur DNS sur 127.0.0.1 (le serveur AD sera son propre serveur DNS)

2. Renommez le serveur :
   - Ouvrez les Propriétés système (clic droit sur "Ce PC" > Propriétés)
   - Cliquez sur "Modifier les paramètres" puis "Modifier"
   - Entrez un nom de serveur descriptif (ex: "AD-SERVER")
   - Redémarrez le serveur

## Installation d'Active Directory

### 2.1. Installation du rôle AD DS

1. Ouvrez le "Gestionnaire de serveur"
2. Cliquez sur "Ajouter des rôles et fonctionnalités"
3. Cliquez sur "Suivant" jusqu'à "Rôles de serveurs"
4. Cochez "Services de domaine Active Directory"
5. Dans la fenêtre popup, cliquez sur "Ajouter des fonctionnalités"
6. Continuez avec "Suivant" puis "Installer"
7. Une fois l'installation terminée, cliquez sur "Fermer"

### 2.2. Promotion du serveur en contrôleur de domaine

1. Dans le "Gestionnaire de serveur", cliquez sur la notification pour "Promouvoir ce serveur en contrôleur de domaine"
2. Sélectionnez "Ajouter une nouvelle forêt"
3. Entrez le nom de domaine racine (ex: "entreprise.local")
4. Cliquez sur "Suivant"
5. Choisissez le niveau fonctionnel de la forêt et du domaine (Windows Server 2016 recommandé pour la compatibilité)
6. Définissez le mot de passe du mode de restauration des services d'annuaire (DSRM)
7. Cliquez sur "Suivant" jusqu'à ce que vous atteigniez la page "Vérification des prérequis"
8. Si les vérifications sont réussies, cliquez sur "Installer"
9. Le serveur redémarrera automatiquement pour terminer l'installation

## Configuration post-installation

### 3.1. Création des unités d'organisation (UO)

1. Ouvrez "Utilisateurs et ordinateurs Active Directory" depuis le menu "Outils"
2. Cliquez-droit sur le domaine > Nouveau > Unité d'organisation
3. Créez les UO suivantes :
   - Utilisateurs
   - Ordinateurs
   - Groupes
   - Serveurs
   - Administrateurs

```powershell
# Création des UO via PowerShell
New-ADOrganizationalUnit -Name "Utilisateurs" -Path "DC=entreprise,DC=local"
New-ADOrganizationalUnit -Name "Ordinateurs" -Path "DC=entreprise,DC=local"
New-ADOrganizationalUnit -Name "Groupes" -Path "DC=entreprise,DC=local"
New-ADOrganizationalUnit -Name "Serveurs" -Path "DC=entreprise,DC=local"
New-ADOrganizationalUnit -Name "Administrateurs" -Path "DC=entreprise,DC=local"
```

### 3.2. Création des utilisateurs et groupes

1. Création d'un utilisateur :
   - Cliquez-droit sur l'UO "Utilisateurs" > Nouveau > Utilisateur
   - Remplissez les informations de l'utilisateur (nom, prénom, nom de connexion)
   - Définissez un mot de passe sécurisé
   - Désélectionnez "L'utilisateur doit changer de mot de passe à la prochaine ouverture de session" si nécessaire

```powershell
# Création d'un utilisateur via PowerShell
$securePassword = ConvertTo-SecureString "MotDePasse1!" -AsPlainText -Force
New-ADUser -Name "Jean Dupont" -GivenName "Jean" -Surname "Dupont" -SamAccountName "jdupont" -UserPrincipalName "jdupont@entreprise.local" -Path "OU=Utilisateurs,DC=entreprise,DC=local" -AccountPassword $securePassword -Enabled $true
```

2. Création d'un groupe :
   - Cliquez-droit sur l'UO "Groupes" > Nouveau > Groupe
   - Entrez le nom du groupe (ex: "Techniciens")
   - Sélectionnez le type de groupe (Sécurité) et l'étendue (Global)

```powershell
# Création d'un groupe via PowerShell
New-ADGroup -Name "Techniciens" -GroupScope Global -GroupCategory Security -Path "OU=Groupes,DC=entreprise,DC=local"
```

3. Ajout d'un utilisateur à un groupe :
   - Double-cliquez sur le groupe créé
   - Cliquez sur l'onglet "Membres" puis sur "Ajouter"
   - Entrez le nom de l'utilisateur et cliquez sur "Vérifier les noms"
   - Cliquez sur "OK" puis "Appliquer"

```powershell
# Ajout d'un utilisateur à un groupe via PowerShell
Add-ADGroupMember -Identity "Techniciens" -Members "jdupont"
```

### 3.3. Configuration des stratégies de groupe (GPO)

1. Ouvrez la "Gestion des stratégies de groupe" depuis le menu "Outils"
2. Cliquez-droit sur le domaine > Créer un objet GPO dans ce domaine
3. Nommez la GPO (ex: "Politique de sécurité")
4. Cliquez-droit sur la GPO créée et sélectionnez "Modifier"
5. Configurez les paramètres de la GPO selon vos besoins

Exemple de configuration des stratégies de mot de passe :

```powershell
# Configuration des stratégies de mot de passe via PowerShell
Set-ADDefaultDomainPasswordPolicy -Identity "entreprise.local" -ComplexityEnabled $true -MinPasswordLength 12 -PasswordHistoryCount 24 -MaxPasswordAge (New-TimeSpan -Days 90) -MinPasswordAge (New-TimeSpan -Days 1) -LockoutThreshold 5 -LockoutDuration (New-TimeSpan -Minutes 30) -LockoutObservationWindow (New-TimeSpan -Minutes 30)
```

## Sécurisation d'Active Directory

### 4.1. Bonnes pratiques de sécurité

1. **Protection des comptes privilégiés** :
   - Créez des comptes administratifs dédiés
   - Utilisez l'authentification à deux facteurs (MFA) pour les comptes administratifs
   - Évitez l'utilisation du compte "Administrator" pour les tâches quotidiennes

2. **Segmentation des privilèges** :
   - Appliquez le principe du moindre privilège
   - Créez des rôles administratifs spécifiques à des tâches

3. **Surveillance des activités** :
   - Activez l'audit des événements de sécurité
   - Centralisez les journaux d'événements

### 4.2. Configuration de l'audit

1. Ouvrez l'éditeur de stratégie de groupe :
   - Créez une nouvelle GPO ou modifiez-en une existante
   - Naviguez vers Configuration ordinateur > Paramètres Windows > Paramètres de sécurité > Stratégies locales > Stratégie d'audit
   - Configurez l'audit des accès aux objets, des connexions, des modifications de stratégie, etc.

```powershell
# Activation de l'audit des événements de réussite et d'échec
auditpol /set /subcategory:"Ouverture de session de compte" /success:enable /failure:enable
auditpol /set /subcategory:"Gestion de compte" /success:enable /failure:enable
auditpol /set /subcategory:"Accès au service d'annuaire" /success:enable /failure:enable
auditpol /set /subcategory:"Modification de stratégie" /success:enable /failure:enable
```

### 4.3. Mise en place d'une stratégie de sauvegarde

1. Planifiez des sauvegardes régulières du système d'état Active Directory
2. Configurez le service de sauvegarde Windows :
   - Installez la fonctionnalité "Sauvegarde Windows Server"
   - Configurez des sauvegardes programmées de l'état du système
   - Stockez les sauvegardes sur un support externe ou une destination réseau

```powershell
# Installation de la fonctionnalité de sauvegarde
Install-WindowsFeature Windows-Server-Backup

# Configuration d'une sauvegarde programmée
$policy = New-WBPolicy
$systemState = New-WBSystemState
Add-WBSystemState -Policy $policy
$backupTarget = New-WBBackupTarget -NetworkPath "\\backup-server\backups" -Credential (Get-Credential)
Add-WBBackupTarget -Policy $policy -Target $backupTarget
Set-WBSchedule -Policy $policy -Schedule 02:00
Set-WBPolicy -Policy $policy
```

## Configuration du monitoring avec Prometheus

### 5.1. Installation de Windows Exporter

Pour collecter des métriques Windows et AD avec Prometheus, il faut installer Windows Exporter sur le serveur AD :

```powershell
# Téléchargement de Windows Exporter
Invoke-WebRequest -Uri "https://github.com/prometheus-community/windows_exporter/releases/download/v0.20.0/windows_exporter-0.20.0-amd64.msi" -OutFile "C:\Temp\windows_exporter.msi"

# Installation de Windows Exporter
Start-Process -FilePath "msiexec.exe" -ArgumentList "/i", "C:\Temp\windows_exporter.msi", "ENABLED_COLLECTORS=ad,process,service,memory,cpu,dns,net,tcp,logical_disk", "/qn" -Wait
```

### 5.2. Configuration des métriques Active Directory

Pour activer les métriques spécifiques à Active Directory dans Windows Exporter, configurez le service pour collecter les compteurs de performance AD :

```powershell
# Configuration du service Windows Exporter pour les métriques AD
$registryPath = "HKLM:\SYSTEM\CurrentControlSet\Services\windows_exporter"
Set-ItemProperty -Path $registryPath -Name "ImagePath" -Value "`"C:\Program Files\windows_exporter\windows_exporter.exe`" --collectors.enabled ad,process,service,memory,cpu,dns,net,tcp,logical_disk"
Restart-Service windows_exporter
```

### 5.3. Configuration du pare-feu Windows

Pour permettre à Prometheus d'accéder aux métriques exportées, configurez le pare-feu Windows :

```powershell
# Ouverture du port pour Windows Exporter dans le pare-feu
New-NetFirewallRule -DisplayName "Windows Exporter (9182)" -Direction Inbound -LocalPort 9182 -Protocol TCP -Action Allow
```

### 5.4. Configuration de Prometheus

Ajoutez la cible Windows Exporter dans la configuration de Prometheus :

```yaml
scrape_configs:
  - job_name: 'windows_ad'
    static_configs:
      - targets: ['ad-server-ip:9182']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '(.*)(:.*)?'
        replacement: '$1'
```

## Métriques Surveillées

### Active Directory

- **ad_ds_dra_inbound_values** : Taux de réplication AD entrant
- **ad_ds_dra_outbound_values** : Taux de réplication AD sortant
- **ad_ds_security_descriptor_propagations_event_queue_size** : Taille de la file d'attente de propagation des descripteurs de sécurité
- **ad_ds_suboperations** : Nombre de sous-opérations AD
- **ad_ds_threads_in_use** : Nombre de threads utilisés par AD
- **ad_ds_name_cache_hit_rate** : Taux de succès du cache de noms
- **ad_ldap_bind_time** : Temps de liaison LDAP
- **ad_ldap_successful_binds** : Nombre de liaisons LDAP réussies
- **ad_ldap_bind_last_successful_time** : Horodatage de la dernière liaison LDAP réussie

### Windows Server

- **windows_cpu_time_total** : Utilisation du CPU
- **windows_memory_available_bytes** : Mémoire disponible
- **windows_logical_disk_free_bytes** : Espace disque disponible
- **windows_net_bytes_sent_total** : Octets réseau envoyés
- **windows_net_bytes_received_total** : Octets réseau reçus
- **windows_service_state** : État des services Windows, y compris les services AD

## Dépannage

### Problèmes courants d'Active Directory

1. **Problèmes de réplication** :
   - Vérifiez les erreurs de réplication : `repadmin /showrepl`
   - Forcez la réplication : `repadmin /syncall /Ade`
   - Consultez les journaux d'événements : Observateur d'événements > Journaux Windows > Réplication du service d'annuaire

2. **Problèmes de résolution DNS** :
   - Vérifiez la configuration DNS : `ipconfig /all`
   - Testez la résolution DNS : `nslookup domaine.local`
   - Vérifiez les enregistrements SRV : `nslookup -type=srv _ldap._tcp.domaine.local`

3. **Problèmes d'authentification** :
   - Vérifiez les journaux de sécurité : Observateur d'événements > Journaux Windows > Sécurité
   - Testez l'authentification avec le compte de test : `runas /user:domaine\utilisateur cmd`
   - Vérifiez l'état du compte utilisateur : `Get-ADUser utilisateur -Properties *`

### Réparation des problèmes

1. **Réparation NTDS** :
   - Lancez une vérification de la base de données AD : `ntdsutil "files" "integrity" q q`
   - Effectuez une maintenance sémantique : `ntdsutil "semantic database analysis" "go" q q`

2. **Problèmes de démarrage des services AD** :
   - Vérifiez le statut des services AD : `Get-Service NTDS, DNS, Netlogon, KDC`
   - Redémarrez les services AD : `Restart-Service NTDS`

## Alertes Prometheus

### Configuration des alertes pour Active Directory

```yaml
groups:
- name: active_directory_alerts
  rules:
  - alert: ADReplicationFailure
    expr: increase(windows_ad_replication_sync_failures_total[1h]) > 0
    for: 15m
    labels:
      severity: critical
    annotations:
      summary: "Échec de réplication AD sur {{ $labels.instance }}"
      description: "Le contrôleur de domaine {{ $labels.instance }} a rencontré des échecs de réplication au cours de la dernière heure."

  - alert: ADHighCPUUsage
    expr: windows_cpu_time_total{mode="idle"} < 10
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Utilisation élevée du CPU sur {{ $labels.instance }}"
      description: "Le contrôleur de domaine {{ $labels.instance }} utilise plus de 90% du CPU depuis 15 minutes."

  - alert: ADLowDiskSpace
    expr: 100 * windows_logical_disk_free_bytes / windows_logical_disk_size_bytes < 10
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Espace disque faible sur {{ $labels.instance }}"
      description: "Le contrôleur de domaine {{ $labels.instance }} a moins de 10% d'espace disque disponible."

  - alert: ADServiceDown
    expr: windows_service_state{name=~"NTDS|DNS|Netlogon|Kdc", state!="running"} == 1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Service AD arrêté sur {{ $labels.instance }}"
      description: "Le service {{ $labels.name }} n'est pas en cours d'exécution sur {{ $labels.instance }}."
```

## Références

- [Documentation Microsoft Active Directory](https://docs.microsoft.com/fr-fr/windows-server/identity/ad-ds/active-directory-domain-services)
- [Guide des meilleures pratiques de sécurité AD](https://docs.microsoft.com/fr-fr/windows-server/identity/ad-ds/plan/security-best-practices/best-practices-for-securing-active-directory)
- [Windows Exporter pour Prometheus](https://github.com/prometheus-community/windows_exporter)
- [Documentation Prometheus](https://prometheus.io/docs/introduction/overview/)
- [Grafana Dashboard pour Active Directory](https://grafana.com/grafana/dashboards/11993)
- [Outils de surveillance AD Microsoft](https://docs.microsoft.com/fr-fr/windows-server/identity/ad-ds/manage/troubleshoot/troubleshooting-active-directory-replication-problems) 