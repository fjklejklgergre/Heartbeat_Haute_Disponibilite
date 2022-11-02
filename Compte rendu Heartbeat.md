# Compte rendu haute disponibilité 
### Enzo SOHIER
### Matheo FOUCAULT
### Antonin LAMOUREUX


# Qu'est-ce que Heartbeat

- Heartbeat est un système permettant, sous Linux, la mise en cluster (en groupe) de plusieurs serveurs. C’est plus clairement un outil de haute disponibilité qui va permettre à plusieurs serveurs d’effectuer entre eux un processus de « fail-over ». Le principe du « fail-over » (ou « tolérance de panne ») est le fait qu’un serveur appelé « passif » ou « esclave » soit en attente et puisse prendre le relais d’un serveur « actif » ou « maitre » si ce dernier serait amené à tomber en panne ou à ne plus fournir un service.
- Le principe d’Heartbeat est donc de mettre nos serveurs dans un cluster qui détiendra et sera représenté par une IP « virtuelle » par laquelle les clients vont passer plutôt que de passer par l’IP d’un serveur (actif ou passif).
- Le processus Heartbeat se chargera de passer les communications aux serveur actif si celui-ci est vivant et au serveur passif le cas échéant.

<img> <img src="https://www.it-connect.fr/wp-content-itc/uploads/2013/07/HA01.png" style="display: block; margin-right: auto; margin-left: auto;">

# Création des deux machine virtuelles

### Pour commencer j'ai crée deux machines virtuelle sous Debian **"pc1"** et **"pc2"**

#### **pc1 = 192.168.10.50**
#### **pc2 = 192.168.10.51**

J'ai commencé par faire les mises à jours sur les deux machines

```sh
sudo apt update && sudo apt full-upgrade -y
```

<img src="https://cdn.discordapp.com/attachments/1029113801003511859/1036734062628446320/unknown.png">

Installé Apache sur les deux machines

```sh
sudo apt install apache2 -y
```

<img src="https://cdn.discordapp.com/attachments/1029113801003511859/1036957365855072357/unknown.png">

Pour vérifier si Apache fonctionne il suffit de rentrer l'IP dans un navigateur 

### pc1

<img src="https://cdn.discordapp.com/attachments/1029113801003511859/1036957692436164618/unknown.png">

### pc2

<img src="https://cdn.discordapp.com/attachments/1029113801003511859/1036957729207615508/unknown.png">

Ici on peut voir la page par defaut de Apache sous Debian 


# Installation de Heartbeat

```sh
sudo apt-get install heartbeat -y
```

<img src="https://cdn.discordapp.com/attachments/1029113801003511859/1037137311223259206/unknown.png">

```php
sudo nano /etc/heartbeat/ha.cf
```

Dans mon cas j'ai édité le fichier de cette manière 

```php
# Indication du fichier de log
logfile /var/log/heartbeat.log
# Les logs heartbeat seront gérés par syslog, dans la catégorie daemon
logfacility daemon
# On liste tous les membres de notre cluster heartbeat (par les noms de préférences)
node pc1
node pc2
# On défini la périodicité de controle des noeuds entre eux (en seconde)
keepalive 1
# Au bout de combien de seconde un noeud sera considéré comme "mort"
deadtime 10
# Quelle carte résau utiliser pour les broadcasts Heartbeat (eth0 dans mon cas)
bcast eth0
# Adresse du routeur pour vérifier la connexion au net
ping 51.159.198.27
# Rebascule-t-on automatiquement sur le primaire si celui-ci redevient vivant ? oui
auto_failback yes
```

<img src="https://cdn.discordapp.com/attachments/1029113801003511859/1037138160083275856/unknown.png">

Maintenant je passe maintenant au fichier "**/etc/heartbeat/authkeys**", ce fichier va définir la clé qui permettra d'authentifier un minimum les serveurs membres du cluster entre eux

```php
sudo nano /etc/heartbeat/authkeys
```

```php
auth 1
# Ce MDP sera présent sur les deux serveurs
1 sha1 SuperMDP
```

Je passe maintenant au fichier "**/etc/heartbeat/haresources**" qui va contenir les actions à mener au moment d'un basculement. Quand un serveur passera du status "**passif**" à "**actif**", il ira lire ce fichier pour voir ce qu'il doit faire. Dans notre cas nous allons dire à notre serveur de prendre l'IP virtuelle **192.168.10.55**

```php
sudo nano /etc/heartbeat/haresources
```

```php
pc1 IPaddr::192.168.10.55/24/eth0
```

<img src="https://cdn.discordapp.com/attachments/1029113801003511859/1037144595127074916/unknown.png">

Je rappel que le contenu du fichier doit être le même sur les deux serveurs. On indique donc ici le nom du serveur primaire du cluster (pc1 est pour moi "**192.168.10.50**") puis l'IP virtuelle du cluster : "**192.168.10.55**" dans mon cas.

Une fois la configuration d'**Heartbeat** terminé j'ai donc créé une page qui différencie les serveurs lors du basculement 

# Configuration de Apache

Pour démarrer, j'ai créé une copie en **"/var/www/html/"** que j'ai appelée "**site**" 

```php
sudo mkdir /var/www/html/site
```

J'ai ensuite fais un document en "**.html**" avec "site1" pour le pc1 et "site2" pour le pc2

Par la suite j'ai edité le fichier de config "**/etc/apache2/sites-available/default-ssl.conf**", pourquoi "default-ssl.conf" car j'ai voulu activé le "HTTPS"

```php
sudo nano /etc/apache2/sites-available/default-ssl.conf
```

<img src="https://cdn.discordapp.com/attachments/1029113801003511859/1037148449931526224/unknown.png">

Ici il suffit juste de mettre le chemin du site dans "DocumentRoot"

Pour activé le SSL sur Apache il suffit d'activer le mode "SSL" et de passé la config "default-ssl" en site actif

```php
sudo a2enmod ssl
sudo a2ensite default-ssl.conf
sudo systemctl reload apache2
```

<img src="https://cdn.discordapp.com/attachments/1029113801003511859/1037152628565610516/unknown.png">
<img src="https://cdn.discordapp.com/attachments/1029113801003511859/1037152807683367002/unknown.png">

<img src="https://cdn.discordapp.com/attachments/1029113801003511859/1037153349394518096/unknown.png">
<img src="https://cdn.discordapp.com/attachments/1029113801003511859/1037153396832084059/unknown.png">


# Hearteat start service

Pour tester le TP j'ai donc démarré Hearteat et j'ai vérifié si j'avais bien l'interface virtuelle

```php
sudo service heartbeat start
```

<img src="https://cdn.discordapp.com/attachments/1029113801003511859/1037154778620375192/unknown.png">

<img src="https://cdn.discordapp.com/attachments/1029113801003511859/1037155649294962818/Sans_titre.png">

Ce qu'il faut savoir c'est que l'interface virtuelle s'active que quand l'esclave remarque que le pc1 est mort

# Test des services

<img src="https://cdn.discordapp.com/attachments/1029113801003511859/1037156051851694140/unknown.png" style="display: block; margin-right: auto; margin-left: auto;">
<img src="https://cdn.discordapp.com/attachments/1029113801003511859/1037156296857763900/unknown.png" style="display: block; margin-right: auto; margin-left: auto;">
Ici j'ai étain le Pc1 pour vérifié si le Pc2 prend bien le relaie et oui ça fonctionne.