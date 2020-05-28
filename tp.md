# TP Windows Server

## I. Mise en place d'une structure hiérarchique des domaines

### A. Mise en place d'un serveur DNS

Il faut tout d'abord attribuer une IP statique pour le serveur (et par la même occasion et pour des raisons de simplicité, une IP statique pour le client). Mes deux VMs sont dans le réseau `192.168.1.0/24`. Pour configurer une IP statique il faut réaliser les étapes suivantes :

- Aller dans `Panneau de configuration -> Réseau et internet -> Centre Réseau et partage -> Modifier les paramètres de la carte -> Sélectionner la carte souhaitée -> Clic droit -> Propriétés -> IPv4`
- Modifier l'IP statique

_Serveur_
![image](https://raw.githubusercontent.com/NorthBlue333/B2-Windows-Server/master/ipstatic.png)

Pour mettre en place un serveur DNS, il faut suivre les étapes suivantes sur le serveur :

- Aller dans `Gestionnaire de serveur -> Ajouter des rôles et des fonctionnalités`
- A l'étape `Rôles de serveurs` sélectionner `Serveur DNS` (et par la même occasion `AD DS` pour les prochaines étapes)
- Installer le tout

![image](https://raw.githubusercontent.com/NorthBlue333/B2-Windows-Server/master/addroles.png)

### B. Paramétrage de la zone de recherche directe

Pour cette étape il faut se rendre dans `Gestionnaire de serveur -> DNS -> Clic droit sur le serveur dans la liste -> Gestionnaire DNS`

- Double clic sur le serveur
- Zones directes
- Menu Action
- Nouvelle zone
- Zone principale
- Remplir le nom de la zone (pour moi `ltabbara.local`)
- Créer un nouveau fichier nommé `ltabbara.local.dns`
- Ne pas autoriser les mises à jour dynamiques
- Terminer

### C. Paramétrage de la zone de recherche inversée

Pour cette étape il faut se rendre dans `Gestionnaire de serveur -> DNS -> Clic droit sur le serveur dans la liste -> Gestionnaire DNS`

- Double clic sur le serveur
- Zones inversées
- Menu Action
- Nouvelle zone
- Zone principale
- Zone de recherche inversée IPv4
- Remplir le champ ID réseau (pour moi `192.168.1`)
- Créer un nouveau fichier nommé `1.168.192.in-addr.arpa.dns`
- Ne pas autoriser les mises à jour dynamiques
- Terminer

![image](https://raw.githubusercontent.com/NorthBlue333/B2-Windows-Server/master/configuredns.png)

## II. Mise en place d'Active Directory

L'installer d'AD DS est déjà fait suite à l'étape précédente.

### A. Chaque serveur devra, au final, être contrôleur de son propre domaine

Promouvoir le serveur en contrôleur de domaine :

![image](https://raw.githubusercontent.com/NorthBlue333/B2-Windows-Server/master/promoteserver.png)

- Ajouter une nouvelle forêt en la nommant (pour moi `ltabbara.local`)
- Créer une nouveau mot de passe
- Installer le tout

### B. Les postes clients seront membres du domaine qui leur correspond

Exécuter le script suivant depuis le dossier où se trouve le CSV sur le serveur permet d'importer automatiquement tous les utilisateurs et toutes les OU :

```powershell
Import-Module ActiveDirectory
Import-Module 'Microsoft.Powershell.Security'

$users = Import-Csv -Delimiter ";" -Path ".\Liste_nom_prenom.csv"

# ------Create OU------

New-ADOrganizationalUnit -Name "CPI" -Path "dc=ltabbara, dc=local"
New-ADOrganizationalUnit -Name "L3" -Path "dc=ltabbara, dc=local"
New-ADOrganizationalUnit -Name "Master" -Path "dc=ltabbara, dc=local"

New-ADOrganizationalUnit -Name "2008" -Path "ou=CPI,dc=ltabbara, dc=local"
New-ADOrganizationalUnit -Name "2009" -Path "ou=CPI,dc=ltabbara, dc=local"

New-ADOrganizationalUnit -Name "2009" -Path "ou=L3,dc=ltabbara, dc=local"

New-ADOrganizationalUnit -Name "2007" -Path "ou=Master,dc=ltabbara, dc=local"
New-ADOrganizationalUnit -Name "2008" -Path "ou=Master,dc=ltabbara, dc=local"
New-ADOrganizationalUnit -Name "2009" -Path "ou=Master,dc=ltabbara, dc=local"

foreach ($user in $users)
{
    $name = $user.NOM + " " + $user.Prenom
    $fname = $user.Prenom
    $lname = $user.NOM
    $login = $user.Prenom.Substring(0,1) + "." + $user.NOM
    $Uoffice = $user.NomEtablissement
    $dept = $user.Region
    $annee = $user.AnneeDemarrage
    $cycle = $user.Cycle
    $password = "Toortoor1234!"

    if ($cycle -eq "CPI" -and $annee -eq "2008")
    {
        $office = "OU=2008,OU=CPI,DC=ltabbara, DC=local"
    }

    ElseIf ($cycle -eq "CPI" -and $annee -eq "2009")
    {
        $office = "OU=2009,OU=CPI,DC=ltabbara, DC=local"
    }

    ElseIf ($cycle -eq "L3" -and $annee -eq "2009")
    {
        $office = "OU=2009,OU=L3,DC=ltabbara, DC=local"
    }

    ElseIf ($cycle -eq "Master" -and $annee -eq "2007")
    {
        $office = "OU=2007,OU=Master,DC=ltabbara, DC=local"
    }

    ElseIf ($cycle -eq "Master" -and $annee -eq "2008")
    {
        $office = "OU=2008,OU=Master,DC=ltabbara, DC=local"
    }

    ElseIf ($cycle -eq "Master" -and $annee -eq "2009")
    {
        $office = "OU=2009,OU=Master,DC=ltabbara, DC=local"
    }

    try
    {
        New-ADUser -Name $name -SamAccountName $login -UserPrincipalName $login -DisplayName $name -GivenName $fname -Surname $lname -AccountPassword (ConvertTo-SecureString $password -AsPlainText -Force) -City $Uoffice -Path $office -Department $dept -Enabled $true
        echo "Utilisateur ajoute : $name"
    } catch [Exception]{

        echo "Utilisateur non ajoute : $name"
        Write-Warning -Message "$($_.Exception.Message)"
    }

}
```

![image](https://raw.githubusercontent.com/NorthBlue333/B2-Windows-Server/master/powershell-1.png)
![image](https://raw.githubusercontent.com/NorthBlue333/B2-Windows-Server/master/powershell-2.png)

Depuis le centre d'administration on peut vérifier :

![image](https://raw.githubusercontent.com/NorthBlue333/B2-Windows-Server/master/allusers.png)

Il faut maintenant ajouter le PC client au domaine : Depuis le client, faire les étapes suivantes :

- `Paramètres > Système > Informations Système > Se connecter à Professionnel ou scolaire > Connecter > Joindre cet appareil à un Active Directory local`
- Rentrer le nom du domaine (ici LTABBARA)
- Rentrer les identifiants d'un utilisateur
- Utilisateur standard
- Redémarrer maintenant

![image](https://raw.githubusercontent.com/NorthBlue333/B2-Windows-Server/master/joindomain.png)

### C. Les types de zones seront changées afin d'obtenir des zones intégrées Active Directory

![image](https://raw.githubusercontent.com/NorthBlue333/B2-Windows-Server/master/checkdns.png)

## III. Gestion des comptes

### A. Vous devrez automatiser la création des comptes en créant les scripts adaptés

Voir au-dessus

### B. Ces comptes seront placés dans les OU correspondantes

Voir au-dessus

### C. Chaque utilisateur aura à sa disposition un répertoire de base stocké sur le contrôleur de son domaine sous la lettre U

Il faut ajouter une fonctionnalité :

- `Ajouter des rôles et des fonctionnalités > Gestionnaire de ressources du serveur de fichier`
- Installer le tout
  ![image](https://raw.githubusercontent.com/NorthBlue333/B2-Windows-Server/master/addrolefiles.png)

On peut ensuite aller dans l'outil correspondant. On crée un modèle de quota :

![image](https://raw.githubusercontent.com/NorthBlue333/B2-Windows-Server/master/quotas.png)

Puis on crée un quota :

![image](https://raw.githubusercontent.com/NorthBlue333/B2-Windows-Server/master/quotas-2.png)

On peut enfin ajouter le dossier de base pour l'utilisateur (ici A.BERNARD)

![image](https://raw.githubusercontent.com/NorthBlue333/B2-Windows-Server/master/userfiles.png)
