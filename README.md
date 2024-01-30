# lab-crowdsec

- [Déploiement du serveur Web](#déploiement-du-serveur-web)
- [Déploiement de crowdsec](#déploiement-de-crowdsec)
- [Premier scan](#premier-scan)
- [Second scan](#second-scan)

Pour ce lab nous allons utiliser deux VMs distinctes:

* l'attaque via Kali (et principalement l'outil nikto)
* le serveur web sous une Debian11 standart

## Préparation du TP (maj 2024)

// les VMS doivent avoir le mode pont de paramétré sous VirtualBox

cf les options

// un export du $PATH doit être effectué sous Debian

Pour cela dans le répertoire du root éditez:

```
nano /root/.bashrc 
```

et rajoutez à la fin du fichier:

```
export PATH=/usr/sbin:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
```

Puis sourcez le fichier:

```
source /root/.bashrc
```

## Déploiement du serveur Web

### Apache

Installer Apache sous Debian 11

On commence par mettre à jour le cache des paquets et installation des paquets:

```
apt update
apt install -y apache2 apache2-utils
```

Pour qu'Apache démarre automatiquement en même temps que Debian, il faut activer le service au démarrage :

```
systemctl enable apache2
```

On test un accès depuis votre navigateur préféré en vous rendant sur http://localhost ou sur l'ip:

```
apt install curl
```

Puis test d'accès:

```
curl http://localhost
```

Plusieurs modes seront nécessaires:

   * "deflate" pour la gestion de la compression, notamment en gzip, pour utiliser la mise en cache des pages sur votre site
   * "headers" afin de pouvoir agir sur les en-têtes HTTP
   * "ssl" pour gérer les certificats SSL et donc l'utilisation du protocole HTTPS
   * "rewrite" pour la réécriture d'URI

```
a2enmod rewrite
a2enmod deflate
a2enmod headers
a2enmod ssl
```

Puis enfin on redémarre pour que ces mods soient pris en compte:

```
systemctl restart apache2
```

### PHP

PHP va venir se greffer sur notre serveur Apache, comme une extension, afin de pouvoir traiter les scripts intégrés aux pages ".php". 

```
apt install -y php php-pdo php-mysql php-zip php-gd php-mbstring php-curl php-xml php-pear php-bcmath
```

Pour vérifier que php est bien installé, exécutez la commande suivante :


```
php -v
```

Maintenant, pour nous assurer que notre moteur de script PHP est bien actif, nous allons créer un fichier "phpinfo.php" (ou un autre nom) à la racine de notre site Web :

```
nano /var/www/html/phpinfo.php

```

Dans ce fichier, indiquez le code suivant :

```
<?php
phpinfo();
?>
```

Et tester d'accéder à l'url suivante: http://localhost/phpinfo.php

### Une page PHP

On va se contenter d'une simple page d'accueil "index.php" avec un message "Hello !". Pour créer cette page:

```
nano /var/www/html/index.php
```

Insérez le code suivant:

```
<?php
echo "<h1>Hello !</h1>";
?>
```

Enfin, supprimez la page d'accueil d'origine d'Apache.

```
rm /var/www/html/index.html
```

## Déploiement de crowdsec

Tout d'abord on va récupérer les dépôts de crowdsec:

```
curl -s https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh | sudo bash
```

Puis installation de l'outil:

```
apt install crowdsec
```

Lors de l'installation, CrowdSec va analyser votre machine afin de détecter les services présents et de télécharger les collections associées. Ces composants vont permettre à CrowdSec de détecter les attaques (sans les bloquer).

On peut lister les collections avec la commande suivante :

```
cscli collections list
```

Si la collection "base-http-scenarios" est présente dans la liste, ce qui normalement le cas si vous avez déjà installé Apache sur votre serveur, cela va notamment permettre de bloquer les mauvais User Agents, comme ceux utilisés par certains outils de scans.

--> Dans le cadre d'un réseau local, il faut désactiver les ranges d'ip privées, sinon crowdsec ne bloque pas les ips du même réseau.

Il faut éditer le fichier YAML:

```
nano /etc/crowdsec/parsers/s02-enrich/whitelists.yaml
```

Puis supprimer le range d'ip qui concerne votre réseau local

Ensuite redémarrez l'outil

```
systemctl restart crowdsec
```

On peut regarder si nous avons des décisions actives au niveau de notre instance CrowdSec. 

```
cscli decisions list
```

Pour l'instant il n'y a rien? c'est normal!


## Premier scan

### Nikto

Nikto est un outil open source qui sert à analyser un serveur Web à la recherche de vulnérabilités ou de défaut de configuration.

Déployons l'outil depuis la VM Kali:

```
apt install nikto
```

Puis effectuons un premier scan sur la machine cible:

```
nikto -h ip_debian11
```

Suite au scan avec Nikto, si vous retournez sur votre machine Debian, crowdsec a dû repérer le scan:

```
cscli decisions list
```

On peut voir aussi que mon analyse avec Nikto a généré plusieurs alertes:

```
cscli alerts list
```

Nous voyons ici que crowdsec remonte des alertes, mais il n'y a pas d'action menées. Pour cela nous allons devoir installer un _bouncer_ (videur) qui traitera les alertes

### Installation de PHP Composer

Pour déployer le bouncer PHP sur son serveur, il faut installer Composer sinon il ne s'installera pas correctement.

Déployons donc au préalable:


```
apt install php-cli unzip
```

Ensuite, il faut se positionner dans son répertoire racine et récupérer l'installeur avec Curl :

```
cd ~
curl -sS https://getcomposer.org/installer -o composer-setup.php
```

Une fois qu'il est récupéré, il faut vérifier que le hash SHA-384 correspond bien à l'installeur que l'on vient de télécharger:

```
HASH=`curl -sS https://composer.github.io/installer.sig`
```

Puis, on lance la commande fournie sur le site officiel de Composer qui va permettre de vérifier le hash avant que l'installation soit lancée :

```
php -r "if (hash_file('SHA384', 'composer-setup.php') === '$HASH') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
```

Vous devez obtenir le message "Installer verified". Sinon, cela signifie que votre installeur est corrompu, vous devez relancer le téléchargement.

Enfin, lancez l'installation de Composer :

```
php composer-setup.php --install-dir=/usr/local/bin --filename=composer
```

Pour vérifier que l'installation est opérationnelle, exécutez simplement :

```
composer
```

Nous avons besoin de récupérer le bouncer sur mon dépot en local :

```
wget https://sandbox.scourzic.net/td/bouncer-php.tar

```

Puis on extrait le répertoire:

```
tar -xzf bouncer-php.tar
```

On obtient un dossier nommé "cs-php-bouncer" dans lequel on va se positionner puis on lance le script en précisant qu'on s'appuie sur apache:

```
./install.sh --apache
```

Enfin, on suit les instructions qui s'affichent à l'écran pour finaliser l'installation. On définit comme propriétaire sur le dossier "/usr/local/php/crowdsec/" l'utilisateur associé à Apache. Par défaut, il s'agit de l'utilisateur "www-data":

```
sudo chown www-data /usr/local/php/crowdsec/
```

Puis, on termine en rechargeant la configuration d'Apache :

```
systemctl reload apache2
```

---------------------------> 

Si à la suite de l'installation du bouncer Apache2 est instable (page blanche et erreur500), il est nécessaire de faire un upgrade du bouncer. Pour cela toujours sous le répertoire _cs-php-bouncer_ il faut tout d'abord éditer le script pour qu'il puisse se lancer en tant que _root_: 

```
nano upgrade.sh
```

Puis modifier la ligne :

```
if [ $(id -u) = 0 ]; then
```

Par: 

```
if [ $(id -u) = 1 ]; then
```

Enfin lancez l'upgrade

```
./upgrade.sh
```

Normalement vous pouvez à nouveau voir votre page PHP, plus d'erreur 500

<---------------------------

Tentons de lister les bouncers installés sur notre serveur:

```
cscli bouncers list
```

## Second scan

Avant de lancer un scan, supprimer le ban en cours sur l'ip de Kali

```
cscli decisions delete --ip ip_de_kali
```

Relançons un second scan nikto depuis Kali:

```
nikto -h ip_Debian11
```

Le scan se déroule, et normalement tombe en erreur au bout d'un certain temps sur une erreur 403. Ce qui indique qu'il y a bien un ban coté client.

Si j'essaie d'accéder au site avec un navigateur, on peut voir qu'un message s'affiche pour indiquer que mon adresse IP a été bloquée par CrowdSec, le bouncer PHP est entré en action!

On peut débannir manuellement une adresse IP avec la commande suivante:

```
cscli decisions delete --ip X.X.X.X
```

De la même façon, on peut aussi bannir manuellement une adresse IP :

```
cscli decisions add --ip X.X.X.X
```

### Mise en place d'un captcha

-> Pour simulier un captcha, procédez de la manière suivante;

```
cscli decisions add --ip ip_kali --duration 5mn --type captcha
```

Avec la configuration actuelle, l'adresse IP sera bloquée pour une durée de 4 heures. Pour limiter l'impact des éventuels faux positifs sur la règle "bad user agents", nous allons présenter un captcha à l'utilisateur bloqué, plutôt qu'un blocage direct.

Pour cela, il faut éditer le fichier de configuration nommé "profiles.yaml", qui est au format YAML, comme son nom l'indique.

```
nano /etc/crowdsec/profiles.yaml
```

Ce fichier contient une configuration par défaut qui sert à bloquer les adresses IP pendant 4 heures, peu importe le type d'attaque détectée. Nous souhaitons conserver ce comportement, à l'exception de deux types d'alertes :

    http-crawl-non_statics
    http-bad-user-agent

Voici le code à ajouter **au tout début** du fichier :

```
# Bad User agents + Crawlers - Captcha 4H
name: crawler_captcha_remediation
filters:
- Alert.Remediation == true && Alert.GetScenario() in ["crowdsecurity/http-crawl-non_statics", "crowdsecurity/http-bad-user-agent"]
decisions:
- type: captcha
duration: 4h
on_success: break
---
```

Enfin il faut relancer crowdsec:

```
systemctl restart crowdsec
```

Avant de refaire des tests, pensez bien à supprimer votre ip dans celles bannies. On peut utiliser la commande curl pour établir une session tcp en précisant un user agent moisi, par exemple _Nikto_:

```
curl -I ip_Debian11 -H "User-Agent: Nikto"
```

Répétez trois fois la commande, puis tentez de naviguer depuis votre navigateur internet favori, vous tomberez sur le captcha. Vous pourrez également voir dans les ips bannis votre ip, avec comme règle _captcha_.
