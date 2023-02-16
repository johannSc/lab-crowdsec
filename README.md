# lab-crowdsec

- [Déploiement du serveur Web](#déploiement-du-serveur-web)
- [Déploiement de crowdsec](#déploiement-de-crowdsec)
- [L'attaquant](#l-attaquant)
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

On peut regarder si nous avons des décisions actives au niveau de notre instance CrowdSec. 

```
sudo cscli decisions list
```

Pour l'instant il n'y a rien? c'est normal!

## L'attaquant

### Nikto

Nikto est un outil open source qui sert à analyser un serveur Web à la recherche de vulnérabilités ou de défaut de configuration.

Déployons l'outil depuis la VM Kali:

```
apt install nikto
```

Puis effectuons un premier scan sur la machine cible:

```
nikto -h ip_debian1&
```

Suite au scan avec Nikto, si vous retournez sur votre machine Debian, crowdsec a dû repérer le scan:

```
sudo cscli decisions list
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

Nous avons besoin de Git pour installer ce Bouncer afin de cloner le projet. Pour installer Git :

```
apt install git
```

Ensuite, on récupère le projet en le clonant en local :

```
git clone https://github.com/crowdsecurity/cs-php-bouncer.git
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

Tentons de lister les bouncers installés sur notre serveur:

```
cscli bouncers list
```

### Nikto le retour

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
