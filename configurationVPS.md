# Configuration d'un VPS

## Sommaire

* Pré-requis
* Se connecter au VPS
* Sécruriser le VPS
* Installer LAMP sur Ubuntu
* Installer phpMyAdmin
* Faire pointer un nom de domaine vers le VPS
* Créer un VirtualHost (ou Hôte Virtuel)
* Téléverser des fichiers sur son VPS
* Activer le HTTPS sur son site Internet avec Let's Encrypt
* Annexes
* Sources

## Pré-requis

* VPS chez OVH (l'installation et la configuration diffère peu selon les prestataires).
* Installation de la distribution Ubuntu 18.04 Server (version 64 bits) sur le VPS.

## Se connecter au VPS

Lors de l’installation (ou de la réinstallation) du VPS, OVH envoie un e-mail avec un mot de passe **en clair** pour l’accès SSH en tant qu'utilisateur root.
Le SSH est un protocole de communication sécurisé. L’accès se fait via un terminal de commande (Linux ou MAC) ou par l’intermédiaire d’un logiciel tiers sur Windows (Putty, par exemple).

Voici la commande à taper pour se connecter au VPS :
* `ssh root@IPv4_de_votre_VPS` ou `ssh root@vpsXXXX.ovh.net`

Si erreur **WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!** => Consulter l'annexe en bas de page.

## Sécuriser le VPS

**Toutes les opérations qui suivent doivent être effectuées avec les droits root.**

### Mettre à jour le système

* La mise à jour de la liste des paquets : `apt-get update`
* La mise à jour des paquets eux-mêmes : `apt-get upgrade` + éventuellement `apt-get full-upgrade` en cas de mises à jour restantes malgré la première commande. Cette dernière commande permet de gérer la mise à jour des dépendances.

### Modifier le port d’écoute par défaut du service SSH

Voici la commande à effectuer pour modifier le fichier de configuration du service :
> `nano /etc/ssh/sshd_config`

Il faut ensuite visualiser la ligne suivante (ou équivalente, faire une recherche du mot clé **port** avec `Ctrl + W`) :
`# What ports, IPs and protocols we listen for Port 22`

Décommenter la ligne en supprimant le `# ` et remplacer le nombre 22 (port par défaut de SSH) par le port de votre choix (par exemple 8787).
Enregistrer la modification.

Il faut ensuite redémarrer le service SSH : `/etc/init.d/ssh restart`

À présent, lors des prochianes connexions SSH, il faudra obligatoirement renseigner le nouveau port :
* `ssh root@votrevps.ovh.net -p NouveauPort` Ex : `ssh root@votrevps.ovh.net -p 8787`

### Modifier le mot de passe associé à l’utilisateur “root”

Le mot de passe est créé automatiquement lors de l’installation d’une distribution pour l’accès principal (root). Il s'agit du mot de passe envoyé **en clair** par email.
Il est fortement conseillé de le modifier en tapant la commande suivante :
* `passwd root`
* Rentrer le nouveau mot de passe deux fois pour le valider.
* Attention, **celui-ci ne s’affichera pas lors de l’écriture**, par mesure de sécurité.
* Effectif dès la prochaine connexion SSH.

### Créer un utilisateur avec des droits restreints et agir sur le système avec les droits root

Créer un nouvel utilisateur avec la commande suitvante :
* `adduser NouveauUtilisateur`
* Remplir ensuite les différentes informations demandées par le système (mot de passe, nom, etc).
* Excepté pour le mot de passe, il est possible de valider tous les champs à blanc.

Une fois connecté au système avec ce nouvel utilisateur, il faudra taper la commande suivante pour avoir les droits root :
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

LAMP = Linux - Apache - MySQL - PHP

**Toutes les opérations qui suivent doivent être effectuées avec les droits root.**

Installer les paquets nécessaires pour Apache, PHP et MySQL :
* `apt install apache2 php libapache2-mod-php mysql-server php-mysql`

*N.B. Sans précision de version, c'est actuellement PHP 7.2 qui est installé par défaut.*

La plupart des scripts PHP (CMS, forums, applications web en tout genre) utilisent des modules de PHP pour bénéficier de certaines fonctionnalités. Voici comment installer les modules les plus courants :
* `apt install php-curl php-gd php-intl php-json php-mbstring php-xml php-zip`

Une fois les paquets installés, ouvrir un des liens suivants dans le navigateur :
* http://127.0.0.1/
* http://localhost

Une page ayant pour titre **Apache2 Ubuntu Default Page** et sous-titre **It works!** s'affiche si le serveur LAMP est correctement installé.

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
| Directive | Description |
| - | - |
| Options +FollowSymLinks | Apache suivra les liens symboliques qu'il trouvera dans ce répertoire (et ses descendants). |
| AllowOverride all | On pourra inclure une configuration personnalisée via un fichier .htaccess. |
| Require all granted | Tous les visiteurs pourront accéder au contenu de ce répertoire. Pour des raisons de sécurité ou de privacité on peut par exemple limiter l'accès au serveur à seulement une ou certaines adresses IP avec une directive du type `Require ip 192.168.1.10.`  |


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

Activation du module "rewrite" (permet de réécrire des URL) :
* `a2enmod rewrite`

Activation du module SSL.
Pour que le protocole TLS (successeur du SSL) puisse fonctionner avec Apache2, il faut activer le module ssl avec la commande :
* `a2enmod ssl`

Toujours relancer ensuite Apache2 :
* `systemctl restart apache2`

Activer le module PHP 7.2
* `a2enmod php7.2` => Si vous avez bien suivi les précédentes étapes, vous devriez normalement obtenir un message vous indiquant que le module est déjà activé.

N.B. Pour lister les modules d'Apache chargés : `apache2ctl -M`

### Modification des propriétaires du dossier www/html

Commande à taper en tant que root : `chown -R nouveauUtilisateur:www-data /var/www/html`
Cette étape permettra de donner les droits sur le dossier www/html au nouvel utilisateur (créé avec la commande `adduser`) et au groupe du serveur Apache (www-data).

Cette étape est nécessaire afin de pouvoir par la suite téléverser des fichiers dans le dossier **www/html** en tant que `nouveauUtilisateur` (le user créé précédemment pour l'accès SSH) et non pas seulement avec `root` dont nous avons bloqué l'accès direct.

## Installer phpMyAdmin

**Toutes les opérations qui suivent doivent être effectuées avec les droits root.**

Installer phpMyAdmin :
* `apt install phpmyadmin`
* Lors de l'installation, il vous sera posé quelques questions auxquelles il faut répondre :
  * Créer la base de données phpMyAdmin : **oui** (sauf si l'on souhaite procéder soi-même à cette configuration). 
  * Définir un mot de passe pour l'utilisateur MySQL phpmyadmin : laisser le champ vide et valider **OK**. Cela aura pour effet de générer un mot de passe aléatoire. Etant donné que nous n'avons pas besoin de connaître le mot de passe de connexion pour l'utilisateur phpmyadmin, cela n'est pas gênant.
  * Choisir le serveur web à configurer automatiquement : Appuyer sur la touche Espace pour vérifier qu'une étoile est bien positionné entre les crochets associés à **apache2**, et valider **OK**. Sinon modifier la sélection pour **apache2** et valider **OK**.

### Accès root

Avec MySQL depuis Bionic 18.04, et MariaDB depuis Xenial 16.04, l'authentification de l'utilisateur root de MySQL se fait au moyen du plugin auth_socket, donc avec sudo.
Cette méthode ne permet pas de se connecter avec phpMyAdmin, mais il est vivement déconseillé de modifier ce comportement.

Si besoin d'un accès global aux bases de données depuis un même compte, la solution conseillée est donc de créer un nouvel utilisateur et de lui attribuer tous les privilèges :
* `mysql` (avec les droits root)

Puis dans la console MySQL :
```
GRANT ALL ON *.* TO 'nom_utilisateur_choisi'@'localhost' IDENTIFIED BY 'mot_de_passe_solide' WITH GRANT OPTION;
FLUSH PRIVILEGES;
QUIT;
```

## Faire pointer un nom de domaine vers le VPS

La procédure exacte diffère selon les registrars, mais de manière générale il est conseillé de :
1. Ne pas modifier les paramètres DNS associé au nom de domaine enregistrés par défaut par le Registrar.
2. Modifier l'enregistrement DNS de type **A** du domaine **example.com** pour y ajouter l'adresse IP du VPS. L’enregistrement DNS de type A permet de relier un nom d’hôte (domaine ou sous-domaine) à l’adresse IP d’un serveur.
3. Si ce n'est pas déjà fait automatiquement par le Registrar, créer le sous-domaine www.example.com. Puis modifier l'enregistrement DNS de type **CNAME** de ce sous-domaine pour y ajouter la valeur **example.com**. L’enregistrement DNS de type CNAME permet de relier un nom d’hôte vers l’enregistrement DNS d’un autre nom d’hôte, sur le principe de l’alias.

Pour vérifier la propagation des DNS : https://www.whatsmydns.net/ (généralement entre quelques minutes à 24h max).

## Créer un VirtualHost (ou Hôte Virtuel)

**Toutes les opérations qui suivent doivent être effectuées avec les droits root.**

Créer un fichier de configuration dans lequel est défini un hôte virtuel pour chaque site ou application web dans le répertoire **/etc/apache2/sites-available/** :
* `nano /etc/apache2/sites-available/example.com.conf`

Copier dans le fichier créé les lignes suivantes (apporter les modifications nécessaires) :
```
<VirtualHost *:80>
	ServerName example.com
	ServerAlias www.example.com
	DocumentRoot "/var/www/html/votreChemin"
	<Directory "/var/www/html/votreChemin">
		Options +FollowSymLinks
		AllowOverride all
		Require all granted
	</Directory>
	ErrorLog /var/log/apache2/error.example.com.log
	CustomLog /var/log/apache2/access.example.com.log combined
</VirtualHost>
```
| Directive | Description |
| - | - |
| ServerAlias www.example.com | L'hôte sera aussi appelé pour le sous-domaine www.example.com. On peut aussi utiliser *.example.com pour inclure tous les sous-domaines. |
| ErrorLog /var/log/apache2/error.example.com.log **et** CustomLog /var/log/apache2/access.example.com.log combined | Il est pratique d'avoir des logs séparés pour chaque hôte virtuel, afin de ne pas mélanger toutes les informations. |

* Enregistrer puis fermer le fichier.

Il faut ensuite activer cette configuration avec la commande `a2ensite [nom du fichier]`, puis relancer le serveur Apache pour que la configuration soit prise en compte avec la commande `systemctl reload apache2`.

Le fichier sera alors visible dans **/etc/apache2/sites-enabled/**.

Taper dans votre navigateur `http://example.com` pour vérifier la bonne configuration de votre VH.

*N.B. Pour désactiver un fichier de configuration VH, il faut utiliser la commande `a2dissite [nom du fichier]`, puis relancer le serveur Apache `systemctl reload apache2`. Le fichier ne sera alors plus présent dans **/etc/apache2/sites-enabled/** mais toujours dans **/etc/apache2/sites-available/**.*

Pour la redirection 301 du sous-domaine www.example.com vers le domaine example.com, ajouter ces lignes dans le fichier de configuration du VH :
```
RewriteEngine On
RewriteCond %{HTTP_HOST} !^example.com [NC]
RewriteRule (.*) http://example.com/$1 [QSA,R=301,L]
```
Nous obtenons donc un fichier de configuration complet du VH :
```
<VirtualHost *:80>
        ServerName example.com
        ServerAlias www.example.com
        DocumentRoot "/var/www/html/votreChemin"

        RewriteEngine On
        RewriteCond %{HTTP_HOST} !^example.com [NC]
        RewriteRule (.*) http://example.com/$1 [QSA,R=301,L]

        <Directory "/var/www/html/votreChemin">
                Options +FollowSymLinks
                AllowOverride all
                Require all granted
        </Directory>
        ErrorLog /var/log/apache2/error.example.com.log
        CustomLog /var/log/apache2/access.example.com.log combined
</VirtualHost>
```

### Téléverser des fichiers sur son VPS

Installer FileZilla : `apt install filezilla` (avec les droits root)

1. Exécuter Filezilla, puis aller dans le **Gestionnaire de Sites** => onglet situé juste en dessous du menu **Fichier**.
2. Cliquer sur **Nouveau Site**.
3. Rentrer les informations suivantes : `Hôte = adresse IP du VPS` ; `Port = numéro du port modifié` ; `Protocole = SFTP - SSH File Transfer Protocol` ; `Type d'authentification = Normale` ; `Identifiant = nouveauUtilisateur` (précédemment créé) ; `Mot de passe = MDP du nouveauUtilisateur`.
4. Puis **Connexion**.
5. Message d'avertissement lors de la première connexion => **Cocher Toujours faire confiance à cet hôte, ajouter cette clé au cache**.
6. Il est maintenant possible de transmettre des fichiers vers ses différents VH enregistrés dans le dossier **/var/www/html** côté serveur.
7. Pour se déconnecter => Menu **Serveur** et cliquer sur **Déconnexion Ctrl+D**.

## Activer le HTTPS sur son site Internet avec Let's Encrypt

Let’s Encrypt est une autorité de certification gratuite, automatisée et ouverte.
Pour la génération et la gestion des certificats, Let's Encrypt recommande l'utilisation du client ACME **Certbot**.

**Toutes les opérations qui suivent doivent être effectuées avec les droits root.**

### Pré-requis

Vérifier que le serveur Apache soit fontionnel avec les modules SSL et Rewrite.

Pour voir les modules chargés :
* `apache2ctl -M`

Si besoin, activer les modules :
* `a2enmod rewrite` et/ou
* `a2enmod ssl`
* Puis, relancer le serveur Apache : `systemctl reload apache2`

## Installer Certbot

Taper les lignes de commandes qui suivent une à une :
```
apt-get update
apt-get install software-properties-common
add-apt-repository universe
add-apt-repository ppa:certbot/certbot
apt-get update
apt-get install python-certbot-apache 
```
### Générer des certificats Let's Encrypt

Il existe plusieurs manières de générer des certificats Let's Encrypt.
L'exécution de la commande `certbot --apache` permet d'obtenir un certificat et autorise Certbot à modifier automatiquement la configuration Apache pour configurer les certificats.

Une fois la commande `certbot --apache` lancée, suivre les différentes instructions à l'écran, et notamment :
* Saisir une adresse e-mail valide.
* Sélectionner les domaines et sous-domaines pour lesquels générer un certificat.
* Choisir l'option accès HTTPS :
	* [1] Easy - Permet l'accès à HTTP et HTTPS.
	* [2] Secure - Redirige toutes les réquêtes vers l'accès HTTPS sécurisé (à privilégier, d'autant plus si le site est nouveau).
* Les certificats générés sont stockés dans le dossier **/etc/letsencrypt/live/CERTNAME/**

A ce niveau, nous constatons que Certbot a procédé aux configurations suivantes dans le dossier **/etc/apache2/sites-available/** (celles-ci peuvent varier suivant la configuration demandée) :
* Edition du fichier VH example.com.conf en ajoutant à la fin du fichier les lignes suivantes pour forcer la redirection de tous les domaines et sous-domaines vers le HTTPS :
```
RewriteCond %{SERVER_NAME} =example.com [OR]
RewriteCond %{SERVER_NAME} =www.example.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
```
Le fichier example.com.conf entier est donc :
```
<VirtualHost *:80>
        ServerName example.com
        ServerAlias www.example.com
        DocumentRoot "/var/www/html/votreChemin"

        RewriteEngine On
        RewriteCond %{HTTP_HOST} !^example.com [NC]
        RewriteRule (.*) http://example.com/$1 [QSA,R=301,L]

        <Directory "/var/www/html/votreChemin">
                Options +FollowSymLinks
                AllowOverride all
                Require all granted
        </Directory>
        ErrorLog /var/log/apache2/error.example.com.log
        CustomLog /var/log/apache2/access.example.com.log combined
RewriteCond %{SERVER_NAME} =example.com [OR]
RewriteCond %{SERVER_NAME} =www.example.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>
```
* Création d'un fichier **example.com-le-ssl.conf** :
```
<IfModule mod_ssl.c>
<VirtualHost *:443>
        ServerName example.com
        ServerAlias www.example.com
        DocumentRoot "/var/www/html/votreChemin"

        RewriteEngine On
        RewriteCond %{HTTP_HOST} !^example.com [NC]
        RewriteRule (.*) http://example.com/$1 [QSA,R=301,L]

        <Directory "/var/www/html/votreChemin">
                Options +FollowSymLinks
                AllowOverride all
                Require all granted
        </Directory>
        ErrorLog /var/log/apache2/error.example.com.log
        CustomLog /var/log/apache2/access.example.com.log combined

Include /etc/letsencrypt/options-ssl-apache.conf
SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem
SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem
</VirtualHost>
</IfModule>
```
Ce dernier fichier apparait déjà comme activé dans le dossier **/etc/apache2/sites-enabled/**.
Le port 443 correspond au HTTPS.

Pour vérifier la bonne configuration du serveur SSL avec le nom de domaine et ses sous-domaines : https://www.ssllabs.com/ssltest/

### Renouvellement des certificats

Les certificats Let's Encrypt ont une validité de 90 jours.
Pour renouveller les certificats, utiliser cette commande :
* `certbot renew`

Cette commande tente de renouveler tous les certificats précédemment obtenus expirant dans moins de 30 jours. 

Il est possible d'automatiser le renouvellement automatique des certificats via le **crontab** (= table de planification).
*Cette fiche n'explique pas cette procédure.*

Il est alors possible de tester le renouvellement automatique des certificats avec la commande suivante :
* ``certbot renew --dry-run``

### Révocation et suppression des certificats

Pour révoquer des certificats :
* `certbot revoke --cert-path /etc/letsencrypt/live/CERTNAME/cert.pem`

On peut également spécifier le motif de révocation du certificat à l'aide de l'argument **--reason**. Les différents motifs comprennent **unspecified** (la valeur par défaut), ainsi que **keycompromise**, **affiliationchanged**, **superseded**, et **cessationofoperation** :
* `certbot revoke --cert-path /etc/letsencrypt/live/CERTNAME/cert.pem --reason keycompromise`


Une fois qu'un certificat est révoqué, tous les fichiers associés peuvent être supprimés du système à l'aide de la commande `delete` :
`certbot delete --cert-name example.com`

Ensuite, penser à :
1. Taper la commande `a2dissite example.com-le-ssl.conf`
2. Dans **/etc/apache2/sites-available**, supprimer le fichier de configuration associé : `rm example-le-ssl.conf`
3. Supprimer les lignes ajoutées par certbot dans le fichier de **config example.com.conf**. Par exemple à la fin du fichier example.com.conf :
```
RewriteCond %{SERVER_NAME} =example.com [OR]
RewriteCond %{SERVER_NAME} =www.example.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
```
4. Relancer Apache : `systemctl restart apache2`
5. Si aucune différence dans le navigateur, fermer le navigateur et supprimer le cache et les cookies.

## Annexes

### Erreur : phpMyAdmin non accessible

Bien vérifier que phpMyAdmin est installé:

Entrer la commande suivante : `whereis phpmyadmin`

Si le terminal retourne :
* `phpmyadmin: /etc/phpmyadmin /usr/share/phpmyadmin`
=> c'est que phpMyAdmin est bien installé.

Dans le cas contraire, s'assurer auparavant que, lors de l'installation du paquet phpmyadmin, le serveur web souhaité (généralement Apache) a bien été sélectionné.
Taper la commande suivante pour reconfigurer phpmyadmin :
* `dpkg-reconfigure phpmyadmin` (toujours avec les droits root).
L'option Apache peut sembler sélectionnée alors qu'elle ne l'est pas. Il faut appuyer sur la barre d'espace et s'assurer d'avoir une astérisque * au niveau d'Apache.

## Erreur : phpMyAdmin affiche “Warning in ./libraries/sql.lib.php#613 count(): Parameter must be an array or an object that implements Countable”

**Toutes les opérations qui suivent doivent être effectuées avec les droits root.**

* Effectuer une copie du fichier concerné : `cp /usr/share/phpmyadmin/libraries/sql.lib.php /usr/share/phpmyadmin/libraries/sql.lib.php.bak`
* Editer le fichier avec les droits root : `nano /usr/share/phpmyadmin/libraries/sql.lib.php`
* Rechercher la ligne `count($analyzed_sql_results['select_expr'] == 1)` et la remplacer par `((count($analyzed_sql_results['select_expr']) == 1)`
* Enregistrer et quitter

## Erreur : phpMyAdmin affiche “Warning in ./libraries/plugin_interface.lib.php#551”

**Toutes les opérations qui suivent doivent être effectuées avec les droits root.**

* Effectuer une copie du fichier concerné : `cp /usr/share/phpmyadmin/libraries/plugin_interface.lib.php /usr/share/phpmyadmin/libraries/plugin_interface.lib.php.bak`
* Editer le fichier avec les droits root : `nano /usr/share/phpmyadmin/libraries/plugin_interface.lib.php`
* Rechercher la ligne `if (! is_null($options) && count($options) > 0) {` ou `if ($options != null && count($options) > 0) {` et la remplacer par `if (!empty($options)) {`
* Enregistrer et quitter


### Erreur : @    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!  @

Suite à la réinstallation du VPS, l'erreur suivante apparaît lors de la connexion au VPS : **WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!** (le message d'erreur sera différent sous Windows).

Dans ce cas, il convient de supprimer l'ancienne clé SSH.
Sous Linux, la clé est généralement enregistré dans le fichier `/home/[user]/.ssh/known_hosts`
Le terminal invite normalement à supprimer l'ancienne clé SSH en communiquant une ligne de commande similaire à celle-ci : `ssh-keygen -f "/home/user/.ssh/known_hosts" -R [192.168.x.x]:22`

Pour supprimer une clé SSH sur Windows avec Putty :
* Accéder à la base de registre : 
  * Cliquer avec le bouton droit sur Démarrer,
  * Sélectionner Exécuter,
  * Entrer `regedit` dans la zone *Ouvrir :*, puis sélectionnez *OK*.
* Aller dans `HKEY_CURRENT_USER\Software\SimonTatham\PuTTY\SshHostKeys`
  * Sélectionner le certificat à supprimer,
  * Cliquer avec le bouton droit, puis cliquer sur Supprimer.

## Sources

Débuter avec un VPS : https://docs.ovh.com/fr/vps/debuter-avec-vps/
Sécuriser un VPS : https://docs.ovh.com/fr/vps/conseils-securisation-vps/
Exemple d'installation complète depuis un environnement Gandi : https://github.com/O-clock-Alumni/fiches-recap/blob/master/serveur/installation-gandi.md
How to solve the incompatibility problem between PhpMyAdmin and PHP7.2 : https://1netwiki.com/wiki/27
Procédure de mise en place d'un domaine local : https://github.com/O-clock-Alumni/fiches-recap/blob/master/ldc/local_vhost.md
Résoudre le duplicate content (avec et sans www) : https://www.webrankinfo.com/dossiers/techniques/redirection-301-www
Let's Encrypt : https://letsencrypt.org/
Certbot Apache on Ubuntu 18.04 LTS (bionic) : https://certbot.eff.org/lets-encrypt/ubuntubionic-apache
User Guide Certbot : https://certbot.eff.org/docs/using.html
