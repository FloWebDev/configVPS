# Configuration d'un VPS

## Sommaire

* Pré-requis
* Se connecter au VPS
* Sécruriser le VPS
* Installer LAMP sur Ubuntu
* Installer phpMyAdmin
* Résoudre les problèmes d'incompatibilité entre phpMyAdmin et PHP 7.2
* Créer un Virtual Host
* Rediriger un nom de domaine vers le VPS
* Redirecton 301 de http://www.example.com vers http://example.com
* Activer le HTTPS sur son site Internet avec Let's Encrypt
* Annexes
* Sources

## Pré-requis

* Avoir un VPS chez OVH (l'installation et la configuration diffèrera peu selon les prestataires).
* Avoir installé la distribution Ubuntu 18.04 Server (version 64 bits) sur son VPS.

## Se connecter au VPS

Lors de l’installation (ou de la réinstallation) du VPS, OVH envoie un e-mail avec un mot de passe **en clair** pour l’accès SSH en tant qu'utilisateur root.
Le SSH est un protocole de communication sécurisé. L’accès se fait via un terminal de commande (Linux ou MAC) ou par l’intermédiaire d’un logiciel tiers sur Windows (Putty, par exemple).

Voici la commande à taper pour se connecter à son VPS :
* `ssh root@IPv4_de_votre_VPS` ou `ssh root@vpsXXXX.ovh.net`

Si erreur **WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!** => Consulter l'annexe en bas de page.

## Sécuriser le VPS

** Toutes les opératiosn qui suivent nécessitent d'ête fait en tant qu'utilisateur root **

### Mettre à jour le système

* La mise à jour de la liste des paquets : `apt-get update`
* La mise à jour des paquets eux-mêmes : `apt-get upgrade` + éventuellement `apt-get full-upgrade` en cas de mises à jour restantes malgré la première commande. Cette dernière commande permet également de gérer la mise à jour des dépendances.

### Modifier le port d’écoute par défaut du service SSH

Voici la commande à effectuer pour modifier le fichier de configuration du service :
> `nano /etc/ssh/sshd_config`

Il faut ensuite visualiser la ligne suivante (ou équivalente, faire une recherche du mot clé **port** avec `Ctrl + W`) :
`# What ports, IPs and protocols we listen for Port 22`
Décommenter la ligne en supprimant le `# ` et remplacer le nombre 22 (port par défaut de SSH) par le port de votre choix (par exemple 8787).
Enregistrer la modification.

Il faut ensuite redémarrer le service SSH : `/etc/init.d/ssh restart`

À présent, lors des prochianes connexions SSH, il faudra obligatoirement renseigner le nouveau port :
* `ssh root@votrevps.ovh.net -p NouveauPort` Ex : `ssh root@votrevps.ovh.net -p 4646`

### Modifier le mot de passe associé à l’utilisateur “root”

ÀU mot de passe est créé automatiquement lors de l’installation d’une distribution pour l’accès principal (root). Il s'agit du mot de passe envoyé **en clair** par email.
Il est fortement conseillé de le personnaliser en le modifiant en tapant la commande suivante :
* `passwd root`
* Rentrer le nouveau mot de passe deux fois pour le valider.
* Attention, **celui-ci ne s’affichera pas lors de l’écriture**, par mesure de sécurité. Vous ne pourrez donc pas voir les caractères saisis.
* Effectif dès la prochaine connexion SSH.

### Créer un utilisateur avec des droits restreints et agir sur le système avec les droits root

Créer un nouvel utilisateur avec la commande suitvante :
* `adduser NouveauUtilisateur`
* Remplir ensuite les différentes informations demandées par le système (mot de passe, nom, etc).
* Excepté pour le mot de passe, il est possible de valider tous les champs à blanc.

Une fois connecté à votre système avec ce nouvel utulisateur, il faudra taper la commande suivante pour avoir les droits root :
* `su` ou `su root`

### Désactiver l’accès au serveur via l’utilisateur root

Il est recommandé de désactiver l'accès direct de l'utilisateur root via le protocole SSH.
Pour effectuer cette opération, il faut modifier le fichier de configuration SSH :
* `nano /etc/ssh/sshd_config`
* Repérer la ligne `PermitRootLogin yes` et rempalcer la valeur **yes** par **no**.
* Redémarrer le service SSH : `/etc/init.d/ssh restart`

### Installer et configurer un pare-feu

La procédure n'est pas indiquée dans cette fiche.

## Installer LAMP sur Ubuntu

**Toutes les opérations qui suivent doivent être effectuées avec les droits root**

Installer les paquets nécessaires pour Apache, PHP et MySQL :
* `apt install apache2 php libapache2-mod-php mysql-server php-mysql`

La plupart des scripts PHP (CMS, forums, applications web en tout genre) utilisent des modules de PHP pour bénéficier de certaines fonctionnalités. Voici comment installer les modules les plus courants :
* `apt install php-curl php-gd php-intl php-json php-mbstring php-xml php-zip`

*N.B. Sans précision de version, c'est actuellement PHP 7.2 qui est installé par défaut.*

### Configuration par défaut du serveur Apache

* Ouvrir et éditer le fichier de configuration par défaut du serveur Apache. Il s'agit du premier Virtual Host.
* `nano /etc/apache2/sites-available/000-default.conf`
* Après la ligne `DocumentRoot /var/www/html` sauter une ligne et ajouter :
```
<Directory "/var/www/html">
	Options +FollowSymLinks
	AllowOverride all
	Require all granted
</Directory>
```

On obtient donc la configuration de base suivante pour le fichier 000-default.conf :
```
<VirtualHost *:80>
	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html
      <Directory "/var/www/html">
        Options +FollowSymLinks
        AllowOverride all
        Require all granted
      </Directory>
	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
### Activer des modules complémentaires sur Apache

Activer le module "rewrite" (permet de réécrire des URL) :
* `a2enmod rewrite`

AActivation du module SSL.
Pour que le protocole TLS (successeur du SSL) puisse fonctionner avec Apache2, il faut activer le module ssl avec la commande :
* `a2enmod ssl`

Toujours relancer ensuite Apache2 :
* `systemctl restart apache2`

Activer le module PHP 7.2
* `a2enmod php7.2` Si vous avez bien suivi les précédentes étapes, vous devriez normalement obtenir un message vous indiquant que le module est déjà activé.

N.B. Pour lister les modules d'Apache chargés : `apache2ctl -M`

## Installer phpMyAdmin

Installer phpMyAdmin :
* `apt install phpmyadmin`



## Annexes

### Erreur : @    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!  @

Suite à la réinstallation de votre VPS, vous pouvez rencontrer l'erreur suivante lors de la connexion au VPS : **WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!** (le message d'erreur sera différent sur Windows).
Dans ce cas, il convient de supprimer l'ancienne clé SSH.
Sous Linux, la clé est généralement enregistré dans le fichier `/home/[user]/.ssh/known_hosts`
Le terminal invite normalement à supprimer l'ancienne clé SSH en communiquant une ligne de commande similaire à celle-ci : `ssh-keygen -f "/home/user/.ssh/known_hosts" -R [192.168.x.x]:22`

Pour supprimer une clé SSH sur Windows avec Putty :
* Accéder à la base de registre : 
** Cliquer avec le bouton droit sur Démarrer,
** Sélectionner Exécuter,
** Entrer `regedit` dans la zone Ouvrir :, puis sélectionnez OK.
* Aller dans `HKEY_CURRENT_USER\Software\SimonTatham\PuTTY\SshHostKeys`
** Sélectionner le certificat à supprimer,
** Cliquer avec le bouton droit, puis cliquer sur Supprimer.






















