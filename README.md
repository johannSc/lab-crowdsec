# lab-crowdsec

- [Déploiement du serveur Web](#déploiement-du-serveur-web)
- [Déploiement de crowdsec: déploiement-de-crowdsec](#nmap)
- [ZAP: Le truc de pro](#zap)
- [DefectDojo: La représentation par graphes](#defectdojo)

Pour ce lab nous allons utiliser deux VMs distinctes:

* l'attaque via Kali (et principalement l'outil nikto)
* le serveur web sous une Debian11 standart

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

On test un accès depuis votre navigateur préféré en vous rendant sur http://localhost ou sur l'ip

Plusieurs modes seront nécessaires:

    "deflate" pour la gestion de la compression, notamment en gzip, pour utiliser la mise en cache des pages sur votre site
    "headers" afin de pouvoir agir sur les en-têtes HTTP
    "ssl" pour gérer les certificats SSL et donc l'utilisation du protocole HTTPS
    "rewrite" pour la réécriture d'URI

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

Sur Debian 11, CrowdSec est directement dans les dépôts, ce qui va nous faciliter la vie. Il suffit de mettre à jour le cache des paquets et de lancer l'installation :

sudo apt-get update
sudo apt-get install -y crowdsec

Le cas échéant, si le paquet n'est pas disponible, vous devez utiliser cette méthode pour réaliser l'installation :

curl -s https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh | sudo bash 
sudo apt install crowdsec

Lors de l'installation, CrowdSec va analyser votre machine afin de détecter les services présents et de télécharger les collections associées. Ces composants vont permettre à CrowdSec de détecter les attaques (sans les bloquer).

On peut lister les collections avec la commande suivante :

cscli collections list

Si la collection "base-http-scenarios" est présente dans la liste, ce qui normalement le cas si vous avez déjà installé Apache sur votre serveur, cela va notamment permettre de bloquer les mauvais User Agents, comme ceux utilisés par certains outils de scans. Ceci n'est qu'un exemple, car cette collection va détecter d'autres événements comme la recherche de backdoors, etc.

On peut regarder si nous avons des décisions actives au niveau de notre instance CrowdSec. En toute logique, non. Vérifions que ce soit bien le cas avec la commande ci-dessous issue de "cscli", l'ensemble de commandes associées à CrowdSec.

sudo cscli decisions list

## L'attaquant

appel : Nikto est un outil open source qui sert à analyser un serveur Web à la recherche de vulnérabilités ou de défaut de configuration.

Avant toute chose, il faut installer l'outil Nikto sur la machine qui sert à réaliser le scan. Pour ma part, j'utilise Kali Linux via WSL (Windows Subsystem for Linux).

L'installation s'effectue très simplement :

sudo apt-get update
sudo apt-get install nikto

Avant d'exécuter le scan Nikto, vous pouvez vérifier que votre machine Kali Linux parvient à charger la page d'accueil de votre site :

curl -I it-connect.tech

Si vous obtenez un résultat avec un code de retour HTTP égal à 200, c'est tout bon ! Maintenant, on va lancer un scan de notre serveur Web avec Nikto. Pour cela, on spécifie l'adresse IP de l'hôte cible ou le nom de domaine, et on laisse tourner. Comme ceci :

nikto -h it-connect.tech

Suite au scan avec Nikto, mon adresse IP est bien dans le viseur de CrowdSec puisqu'il a décidé de bannir mon adresse IP. Cependant, l'adresse IP n'est pas bloquée. En effet, CrowdSec doit s'appuyer sur un Bouncer pour appliquer la décision et bannir l'adresse IP.

sudo cscli decisions list

On peut voir aussi que mon analyse avec Nikto a généré plusieurs alertes, ce qui donne des indications sur les types d'attaques détectés (reason).

cscli alerts list

B. Installation de PHP Composer

Pour déployer le Bouncer PHP sur son serveur, il faut installer Composer sinon il ne s'installera pas correctement. Pour l'installer, nous avons besoin de deux paquets : php-cli et unzip, que l'on va installer sans plus attendre :

sudo apt-get update
sudo apt-get install php-cli unzip

Ensuite, il faut se positionner dans son répertoire racine et récupérer l'installeur avec Curl :

cd ~
curl -sS https://getcomposer.org/installer -o composer-setup.php

Une fois qu'il est récupéré, il faut vérifier que le hash SHA-384 correspond bien à l'installeur que l'on vient de télécharger. Cela permet de s'assurer de l'intégrité du fichier. On stocke le hash dans une variable nommée "HASH" :

HASH=`curl -sS https://composer.github.io/installer.sig`

Puis, on lance la commande fournie sur le site officiel de Composer qui va permettre de vérifier le hash avant que l'installation soit lancée :

php -r "if (hash_file('SHA384', 'composer-setup.php') === '$HASH') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"

Vous devez obtenir le message "Installer verified". Sinon, cela signifie que votre installeur est corrompu, vous devez relancer le téléchargement.

Enfin, lancez l'installation de Composer :

sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer

Pour vérifier que l'installation est opérationnelle, exécutez simplement :

composer

Vous devez obtenir ceci :

Composer est correctement installé, vous pouvez passer à l'installation du Bouncer en lui-même.
C. Mise en place du Bouncer PHP

Souvenez-vous du premier tutoriel sur CrowdSec où j'utilisais un serveur NginX. J'avais utilisé le Bouncer NginX directement pour bloquer les adresses IP que CrowdSec souhaitait bannir. Cette fois-ci, comme nous n'utilisons pas NginX mais Apache, on va se tourner vers un autre Bouncer : PHP.

    Note : le Bouncer PHP développé par CrowdSec prend en charge de nombreuses versions (PHP 7.2.x, 7.3.x, 7.4.x et 8.0.x).

    Page du Bouncer PHP de CrowdSec
    Page du Bouncer PHP sur GitHub

Nous avons besoin de Git pour installer ce Bouncer afin de cloner le projet. Pour installer Git :

sudo apt-get install git

Ensuite, on récupère le projet en le clonant en local :

git clone https://github.com/crowdsecurity/cs-php-bouncer.git

On obtient un dossier nommé "cs-php-bouncer" dans lequel on va se positionner :

cd cs-php-bouncer/

Puis, on lance le script d'installation (il ne doit pas être lancé avec le compte "root", ni "sudo" !) en spécifiant que l'on utilise le serveur Web "Apache" :

./install.sh --apache

Installation du Bouncer PHP sur CrowdSec
Installation du Bouncer PHP sur CrowdSec

Enfin, on suit les instructions qui s'affichent à l'écran pour finaliser l'installation. On définit comme propriétaire sur le dossier "/usr/local/php/crowdsec/" l'utilisateur associé à Apache. Par défaut, il s'agit de l'utilisateur "www-data".

sudo chown www-data /usr/local/php/crowdsec/

Puis, on termine en rechargeant la configuration d'Apache :

sudo systemctl reload apache2

Si on liste les Bouncer installés sur notre serveur, on va voir qu'il apparaît dans la liste :

sudo cscli bouncers list

D. Deuxième scan avec Nikto : le bouncer PHP va-t-il se montrer intraitable ?

Cette fois-ci, on devrait être en mesure de détecter l'attaque, de décider de bannir l'adresse IP de l'hôte "Nikto" et surtout d'appliquer la décision grâce au Bouncer PHP. C'est ce que nous allons vérifier.

On peut commencer par utiliser Curl pour vérifier que l'on obtient bien un code de retour HTTP 200.

curl -I it-connect.tech

Puis, on lance notre scan Nikto :

nikto -h it-connect.tech

Cette fois-ci, le scan mouline un peu... Comme s'il était bloqué par quelque chose. Si on force son arrêt (CTRL+C) et que l'on relance la commande Curl, on peut voir que le code de retour HTTP a changé : HTTP/1.0 403 Forbidden. Ce qui correspond à un accès refusé sur la page, y compris si l'on essaie d'accéder au serveur avec l'adresse IP au lieu du nom.

Si j'essaie d'accéder au site avec un navigateur, on peut voir qu'un message s'affiche pour indiquer que mon adresse IP a été bloquée par CrowdSec ! Le Bouncer PHP est entré en action !

Je vous rappelle que l'on peut débannir manuellement une adresse IP avec la commande suivante (nous allons utiliser cette commande dans la suite du tutoriel) :

sudo cscli decisions delete --ip X.X.X.X

De la même façon, on peut aussi bannir manuellement une adresse IP :

sudo cscli decisions add --ip X.X.X.X

V. Bouncer PHP : mise en place d'un captcha

Avec la configuration actuelle, l'adresse IP sera bloquée pour une durée de 4 heures et l'utilisateur n'a pas de possibilité d'être débloqué pendant ce temps. À moins de prendre la peine de contacter le Webmaster du site...

Pour limiter l'impact des éventuels faux positifs sur la règle "bad user agents", nous allons présenter un captcha à l'utilisateur bloqué, plutôt qu'un blocage pur. De cette façon, un robot sera bloqué tandis qu'un utilisateur réel pourra  déverrouiller l'accès au site. Nous allons réserver le même sort au crawlers malveillants (robots d'exploration).

Pour cela, il faut éditer le fichier de configuration nommé "profiles.yaml", qui est au format YAML, comme son nom l'indique.

sudo nano /etc/crowdsec/profiles.yaml

Ce fichier contient une configuration par défaut qui sert à bloquer les adresses IP pendant 4 heures, peu importe le type d'attaque détectée. Nous souhaitons conserver ce comportement, à l'exception de deux types d'alertes :

    http-crawl-non_statics
    http-bad-user-agent

Il faut que l'on ajoute notre configuration au début du fichier, et non pas à la fin. À ce jour, CrowdSec gère la priorité par rapport à l'ordre dans le fichier puisque dès que l'on match sur une règle, on ne vérifie pas la suite (un peu sur le même principe que les ACL sur un routeur, finalement) compte tenu de la présence de la directive "on_success: break" (voir ci-dessous).

Autrement dit, si l'on ajoute notre configuration spécifique à la suite dans le fichier profiles.yaml, cela ne fonctionnera pas, car la première règle correspondra (matchera) toujours, et donc notre règle ne sera pas prise en compte.

Voici le bout de code à ajouter au tout début du fichier :

# Bad User agents + Crawlers - Captcha 4H
name: crawler_captcha_remediation
filters:
- Alert.Remediation == true && Alert.GetScenario() in ["crowdsecurity/http-crawl-non_statics", "crowdsecurity/http-bad-user-agent"]
decisions:
- type: captcha
duration: 4h
on_success: break
---

La partie "filters" sert à spécifier que l'on cible seulement les scénarios d'alertes "http-crawl-non_statics" et "http-bad-user-agent". Ensuite, au niveau du "type", on spécifie "captcha" au lieu de "ban" qui est l'action standard.

Enfin, la partie "duration" sert à spécifier la durée pendant laquelle on présente un captcha à l'adresse IP associée au "blocage". En l'occurrence, 4h par défaut, mais vous pouvez modifier cette valeur.

Au final, vous obtenez ceci :

Sauvegardez le fichier et fermez-le. Il ne reste plus qu'à relancer CrowdSec :

sudo systemctl restart crowdsec

Avant de refaire des tests, on va supprimer la décision qui s'applique actuellement sur notre adresse IP, c'est-à-dire "ban". On utilise la commande ci-dessous en remplaçant "X.X.X.X" par l'IP publique correspondante.

sudo cscli decisions delete --ip X.X.X.X

Ensuite, il faut que l'on simule un accès depuis un user-agent non autorisé, car si j'utilise Firefox, Chrome, Brave, Edge, etc... cela ne fonctionnera pas, car ce sont des user-agent légitimes. Pour cela, on peut se référer à la liste de user-agents malveillants relayée par CrowdSec sur GitHub. Pour l'outil à utiliser, on partira sur l'excellent Curl que l'on a déjà utilisé précédemment.

Nous allons utiliser le modèle de commande suivant :

curl -I it-connect.tech -H "User-Agent: <nom-user-agent>"

Voici quelques exemples :

curl -I it-connect.tech -H "User-Agent: Cocolyzebot"
curl -I it-connect.tech -H "User-Agent: Mozlila"
curl -I it-connect.tech -H "User-Agent: OpenVAS"
curl -I it-connect.tech -H "User-Agent: Nikto"

En effectuant deux-trois requêtes Curl à destination de notre site, on se retrouve rapidement avec un code d'erreur HTTP : 401 Unauthorized. Comprenez accès non autorisé.

Maintenant que je suis bloqué par CrowdSec, on va utiliser un navigateur classique pour voir comment réagit le serveur Web.

Aaaaah ! Bonne nouvelle ! Une page avec un captcha s'affiche ! Pour outrepasser le blocage, il suffit de compléter le captcha, de cliquer sur "Continue", afin d'accéder au site.
CrowdSec - Page de blocage avec captcha
CrowdSec - Page de blocage avec captcha

Si on liste les décisions actives, on peut voir que mon adresse IP est bien associée à l'action "captcha" au lieu de l'action "ban". On peut voir aussi que mon instance CrowdSec a banni quelqu'un d'autre, qui visiblement, s'en prend à mon serveur Web.

sudo cscli decisions list

Si l'on effectue un scan avec Nikto comme nous l'avons fait à plusieurs reprises dans cette démonstration, nous n'aurons pas le captcha, nous allons être directement bannis. En fait, il y a bien un user-agent Nikto, mais le scan avec cet outil va générer d'autres alertes avant celle sur le user-agent, ce qui implique que c'est l'action "bannir" qui va s'appliquer avant l'action "captcha". En soi, ce n'est pas plus mal, car quelqu'un qui scanne notre serveur Web, on préfère qu"il soit bloqué
