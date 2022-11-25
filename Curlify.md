[[WRITE UPS]]




### DGHACK CTF
Novembre 2022 
*Challenge Web*


## Introduction 

Dans le cadre d'un programme de bug bounty nous devons auditer l'application **Curlify** en pré-prod afin de valider sa robustesse avant son déploiement en production. 

```
L'objectif du challenge est de lire le flag se trouvant dans le fichier flag.php 
```

En premier lieu on peut deviner grâce à son appellation que l'application utilise l'interface en ligne de commande cURL qui permet notamment de récupérer le contenu d'une ressource en y indiquant son URL.



## Curlify

L'application se résume en une simple page contenant un formulaire et un bouton "Curl it !"

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-01.png">

Après de nombreuses essais, peu importe l'adresse renseignée l'application nous retourne *Not Implemented Yet*. Aucune adresse web ni aucun paramètre cURL n'est pris en compte. Tout cela pouvant nous laisser croire que l'application n'est pas du tout fonctionnelle. 



## dev.php

En analysant le code source de la page on peut remarquer qu'il existe une page **/dev.php** et que le lien pour y accéder n'est pas visible puisqu'il a été commenté. 

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-02.png">

Malheureusement la page **/dev.php** nous retourne un message d'erreur nous indiquant qu'elle est accessible qu'en interne. 

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-03.png">

On se retrouve avec une application Curl qui serait non fonctionnelle et une page /dev inacessible...

**Et si il était possible de demander à Curlify de récupérer une page locale et non web ?**


<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-04.png">

Bingo ! 
On arrive obtenir le contenu de la page /dev.php nous avons donc affaire à une faille **SSRF** 



## admin_panel

L'adresse **/536707b92** nous permet de télécharger une archive zip du code source de ce qui semblerait être une autre application disponible à l'adresse **/admin_panel/

**admin_panel/index.php**

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-05.png">

Et **admin_panel/task.php**

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-06.png">



### Code source obtenu à partir de l'archive zip

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-07.png">

Dans le backup du code source on peut remarquer le fameux fichier **flag.php** présent dans le dossier user_prefs qui une fois interpreté par le serveur fera apparaître le flag pour valider le challenge. 

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-08.png">

Il faudra donc probablement ***include*** ce fichier avec une LFI d'une manière ou d'une autre pour faire apparaître le flag. 



## Exploiter le code

On a pu voir auparavant comment bypass le message d'erreur **Internal Access Only** en utilisant Curlify pour obtenir le code source de la page en local donc on recommence pour **admin_panel/index.php** : 

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-09.png">

Mais on est bloqué par un WAF homemade, on va donc analyser le code source du fichier **index.php** (que j'ai commenté) pour bypass ce message d'erreur. 

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-10.png">

La première sécurité est déjà bypass grâce à curlify mais la seconde sera exploitable dans le fichier firewall.php dont le code est présent ci-dessous.

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-11.png">

Après avoir modifié notre user agent pour correspondre au format "DGHACK/1.0 (Curlify)" requis par le WAF nous voici bloqués une étape plus loin dans l'exploitation de l'application. 

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-12.png">

Par la suite l'équipe de développement à choisi d'utiliser la fonction PHP extract() pour récupérer les paramètres envoyés par l'utilisateur dans l'URL. 

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-13.png">

Le paramètre **$source** ici nous permettra d'indiquer une page et d'en afficher le code source **seulement** si l'adresse ne contient pas la chaine de caractère "flag.php". 

Il est donc possible d'afficher le code source de tous les fichiers dans l'arborescence du serveur directement à travers Curlify, on remarquera notamment qu'un fichier n'était pas présent dans l'archive zip back_up récupérée et c'est le fichier **config.php** qui nous sera utile très prochainement. 

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-14.png">

On va pouvoir en obtenir le code source grâce au paramètre **$source** : 

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-15.png">

On remarque ici qu'il y a plusieurs variables définies dont le langage par défaut et une secret_key. 



## Requêtes POST 

Prochaine étape dans le code, l'application nous demande impérativement de POST un **username**. Or comme indiqué au début de ce WU, l'application Curlyfier ne prend aucun paramètre et ne permet pas de changer le type de requête de GET à POST. 

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-16.png">

Après une longue recherche, il est possible d'exploiter la fonction extract($\_GET) vue précédemment pour bypass la requête POST. Cette *erreur* de conception offre la possibilité de surcharger n'importe qu'elle variable tout simplement via l'URL comme ceci : 

```php
127.0.0.1/admin_panel/index.php?_POST[username]=admin
```

Cet exploit nous permettra de surcharger toutes les variables nécéssaires pour obtenir le flag. 



## Craft d'un Cookie 

Une fois le nom d'utilisateur surchargé, nous avons besoin d'un cookie "remember_me" et ce dernier doit correspondre au *return* de la fonction **generate_remember_me_cookie($username)**

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-17.png">

Cette fonction **generate_remember_me_cookie()** présente dans le fichier **utils.php** est détaillée ci-dessous. 

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-18.png">

Ici l'erreur du développeur est que le md5 sera généré avec la chaîne de caractère '$SECRET_KEY' et non avec la variable $SECRET_KEY trouvée dans le fichier **config.php**..

Le cookie crafté est le suivant :

```
admin7a988e11680f9e151f6f46808690d5ca
```

La payload est donc :

```
127.0.0.1/admin_panel/index.php?_POST[username]=admin&_COOKIE[remember_me]=admin7a988e11680f9e151f6f46808690d5ca
```

Résultat :

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-19.png">



### Ticketting 

C'est bien beau une fois qu'on reconnu comme admin mais qu'est-ce qu'on peut faire maintenant ? 

Et bien après avoir bypass la vérification du cookie **remember_me**, le serveur a créé une variable de session côté serveur nommée $\_SESSION["is_connected"]  = true & $\_SESSION["username"] = admin. 

Ce qui va nous être utile pour la suite du challenge puisque sur la page **task.php** on vérifie qu'une de ces deux variables serveur est bien présente. 

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-20.png">

On est donc à partir maintenant plus obligé de passer par Curlify pour afficher la page **task.php** puisque le serveur nous reconnait (erreur *Not Authorized* précédemment) et on tombe sur une page de création de ticket. 

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-21.png">

A la lecture du code source : 

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-22.png">
<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-23.png">

L'application s'attend à ce que l'auteur et *l'assignee* soit un username présent dans la base de données, le type d'incident pourra être par exemple **bug** et la description est libre. 

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-24.png">

Une fois que toutes les données entrées sont correctes, un ticket "task_XXXX.txt" sera généré avec un **id aléatoire** dans le repertoire /admin_panel/tasks/. 

Le code précédent nous informe par ailleurs qu'il sera possible d'inclure les préférences de l'utilisateur si les variables $\_SESSION["userid"] et $\_SESSION["user_prefs"] existent. Ce qui est une bonne nouvelle puisqu'on cherche à inclure le fichier **flag.php** présent dans les préférences de l'utilisateur. 

Voici la logique permettant d'inclure les préférences de l'utilisateur dans le fichier **prefs.php**

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-25.png">

On peut en conclure que si la variable **$prefs** ne correspond à rien dans le switch case on incluera un fichier correspondant à $lang

Cette fonction get_prefs est appelée dans **/admin_panel/index.php** lorsque le serveur vérifie le cookie **remember_me** (*vu précédemment*). 

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-26.png">

On retourne donc sur Curlify pour surcharger toutes les variables nécéssaires à savoir _SERVER[HTTP_ACCEPT_LANGUAGE] (prefs) & \$DEFAULT_LANGUAGE (lang)

Voici la payload finale : 

```php
127.0.0.1/admin_panel/index.php?_POST[username]=admin&_COOKIE[remember_me]=admin7a988e11680f9e151f6f46808690d5ca&_SESSION[userid]=1&_SERVER[HTTP_ACCEPT_LANGUAGE]=n0ne&DEFAULT_LANGUAGE=flag.php
```


Mais ce n'est pas terminé ! 

Puisqu'il faut maintenant créer un ticket qui une fois généré incluera nos préférences à savoir le fichier **flag.php**

![[curly27.png]]

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-28.png">

La description du challenge indiquait "La DSI nous indique que les administrateurs sont très réactifs dans le traitement des tickets", on comprend alors pourquoi car effectivement 5 secondes pour traiter un ticket c'est vraiment rapide..

Le ticket généré est consultable via Curlify à l'adresse **/admin_panel/tasks/task_9532.txt** et est automatiquement supprimé au bout de quelques secondes. 

Son id étant aléatoire sur 4 digits il faut être rapide pour l'attraper. 

Afin d'obtenir le précieux graal deux choix sont possibles, être assez fast pour afficher le ticket avant sa suppression ou créer un script permettant à coup sûr de l'obtenir... 



## Graal 

Une fois le ticket en votre possession, c'est enfin terminé pour le challenge Curlify ;) 

```
127.0.0.1/admin_panel/tasks/task_7295.txt
```

<img align="center" src="https://raw.githubusercontent.com/xStrat0S/CTF-Write-Ups/main/img/curlify/curly-29.png">