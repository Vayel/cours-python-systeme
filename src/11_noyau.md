# Le noyau et la coquille

Commençons par le commencement. Dans ce chapitre, nous allons survoler
rapidement la façon dont sont structurées les couches basses de votre système
d'exploitation. Il s'agit principalement d'un rappel des notions sur lesquelles
nous construirons les chapitres suivants.

## Au plus proche du matériel : le noyau

Rassurez-vous, je vous ferai grâce d'un long couplet sur l'architecture des
micro-processeurs. Ce n'est pas ce qui nous intéresse dans ce cours.
Simplement, vous le savez comme moi, un ordinateur, c'est d'abord tout un tas
de composants matériels soudés sur des cartes et connectés entre eux par des
bus de données. Et puis n'oublions pas tous les périphériques reliés à ces
cartes par des câbles. Bref, un ordinateur, c'est d'abord et avant tout *du
matériel*.

En fait, *tous* les programmes que vous exécutez vont forcément, à un moment
donné, devoir intéragir avec ce matériel. Au minimum, le microprocesseur devra
exécuter les instructions du programme en allant piocher des données dans la
mémoire vive (ou RAM). Mais pour qu'un programme serve réllemment à quelque
chose, il faut bien qu'il ait des **entrées** et des **sorties**.

Par exemple, vous allez peut-être saisir des données sur votre *clavier*, ou
bien utiliser une *souris* pour cliquer sur des boutons. Et pour voir ce que
vous êtes en train de faire, ce même programme devra afficher quelque chose sur
un *écran*.  Peut-être même qu'il ira lire ou écrire des fichiers sur un
*disque dur*, ou encore se connecter à internet pour échanger des données avec
d'autres programmes qui tournent sur d'autres ordinateurs, en passant par *la
carte réseau*...

Vous imaginez bien que tout cela ne se fait pas par magie. Pour qu'un programme
utilise un périphérique matériel, **il doit forcément faire appel à du code qui
sert à manipuler ce matériel.**

J'entends quelqu'un murmurer le terme "pilote", là bas au fond. J'y viens,
patience !

### Interagir avec le matériel. Oui, mais…

Imaginons un instant que vous ayez écrit un programme capable de communiquer
directement avec l'un de vos périphériques. Par exemple votre disque dur. Vous
avez mis quelques bonnes semaines et englouti des douzaines de cafetières, vous
avez plongé votre nez dans la documentation technique du matériel et failli
vous décourager une bonne vingtaine de fois, mais ça y est, votre programme
sait lire et sauvegarder des données au bon endroit sur votre disque dur et il
fonctionne à merveille. Ouf !

Vous avez donc quelque chose de semblable à la figure 1.1.

![Interaction directe avec le matériel](src/img/interaction_directe.png)



Imaginons maintenant que vous vouliez créer *un autre* programme qui ait besoin
d'interagir avec ce disque dur. Manque de chance, le code de votre premier
programme ne sait écrire que des fichiers de type `A` sur le disque, mais le
second a besoin de lire des fichiers de type `B`. Vous voilà obligé de tout
recommencer : les quelques semaines, la cafetière, la documentation technique…
Sauf que cette fois-ci, vous ne vous laisserez pas avoir ! Vous allez écrire
une *bibliothèque de fonctions* qui sera capable de lire et d'écrire n'importe
quelle donnée sur votre disque dur. De cette manière, les programmes n'auront
qu'à mettre en forme leurs données selon le type de fichier `A` ou `B`, et
laisser le soin à votre bibliothèque de les charger au bon endroit sur le
disque. Cette bibliothèque, c'est ce que l'on peut appeler un **pilote de
périphérique** (on parle également de *pilote* tout court, ou bien de *driver*
en anglais).

Vous obtenez alors une architecture similaire à la figure 1.2.

![Utilisation d'un pilote](src/img/pilote.png)

Vous pouvez être fier de vous ! Vous avez partagé le code de votre pilote
avec vos amis qui désirent interagir avec ce disque dur : ils n'ont même pas
besoin de mettre le nez dans la documentation technique, ils n'ont plus qu'à se
concentrer sur leurs programmes à eux.

Tout va bien dans le meilleur des mondes, les gens utilisent votre pilote pour
lire et écrire des programmes sur le disque et celui-ci fonctionne
parfaitement. En fait, il fonctionne tellement bien qu'un beau jour, au milieu
de l'été, *le disque est rempli à ras bord de données*.  Qu'à cela ne tienne,
vous direz-vous, il suffit d'aller en acheter un deuxième.  Et puis comme il y
a une promo dans votre magazin, vous en trouvez un deux fois plus grand, deux
fois plus rapide, deux fois plus joli et deux fois moins cher, une affaire en
or ! Satisfait, vous rentrez chez vous en toute hâte, puis déballez ce petit
bijou pour l'installer sur votre ordinateur, mais... il s'avère que votre
nouveau disque dur n'est pas compatible avec le pilote que vous aviez développé
pour le premier. La poisse !

Par chance, ce nouveau disque dur est livré avec une disquette[^disquette]
d'installation qui contient son pilote. Voilà qui vous fera gagner du temps.
Mais le problème c'est que les programmes que vous avez écrits jusqu'à présent
savent utiliser le pilote de votre premier disque dur, mais pas celui-ci, qui
n'a absolument rien à voir.

[^disquette]: Ah oui, j'avais oublié de le préciser, mais nous sommes au début
des années 1970. Vous n'imaginiez pas que quelqu'un développerait son propre
pilote de disque dur de nos jours, tout de même !

Résigné, vous vous laissez choir dans votre fauteuil : tout est à refaire.
Décidément, **interagir directement avec le matériel, c'est un véritable
cauchemar**.  Quand ce n'est pas le code de pilotage du disque qui est mélangé
aux programmes, ce sont les programmes qui ne sont pas compatibles avec les
autres pilotes de disque et qu'il faut modifier chaque fois qu'un nouveau
modèle arrive sur le marché. Même avec la meilleure volonté du monde, ce n'est
pas gérable. En fait, de nos jours, les programmes n'interagissent jamais
directement avec le matériel, sauf à la rigueur sur certains systèmes
embarqués.


### Un noyau pour les gouverner tous, et dans l'abstraction les lier

Vous l'avez compris : un système d'exploitation digne de ce nom doit fournir
aux programmes une interface commune, de façon à ce que ceux-ci n'aient jamais
besoin de se soucier du matériel sur lequel ils agissent. Cette interface, on
l'appelle *HAL* pour *Hardware Abstraction Layer* (*couche d'abstraction
matérielle*). C'est cette couche qui s'occupe de charger et d'utiliser le bon
pilote de périphérique en fonction du matériel qui est effectivement branché à
l'ordinateur.

Néanmoins, cela n'est pas encore tout à fait suffisant. En effet, c'est bien
joli de fournir aux développeurs une interface commune pour accéder au
matériel, mais encore faut-il penser à *protéger* l'ordinateur contre le code
qui fait n'importe quoi.

Imaginons qu'un premier programme demande à HAL de sauvegarder des données sur
le disque dur. HAL va faire bien attention à trouver une adresse libre sur le
matériel pour écrire ces données. Maintenant, devinez ce qu'il se passe si un
second programme ne demande pas l'avis de HAL, et accède directement au disque
dur pour écrire des données *à la même adresse*…

C'est pour cela que les systèmes d'exploitation modernes disposent
d'un **noyau** (ou kernel) qui :

* **protège** le matériel en étant *le seul* autorisé à interagir avec lui,
* fournit aux programmes qui tournent sur la machine une **interface** leur
  permettant d'utiliser le matériel de façon contrôlée.

Ainsi, le système d'exploitation va faire la distinction entre deux types de
codes :

* le code *utilisateur* (ou *non-privilégié*), qui n'a pas le droit d'accéder
  au matériel,
* le code *noyau* (ou *privilégié*) qui a tous les droits.

Lorsqu'un programme utilisateur a besoin d'exécuter du code privilégié,
celui-ci va réaliser un **appel système**. Lors de cet appel système, le
programme va passer en mode privilégié et exécuter le code correspondant puis,
au retour de la fonction, il repassera automatiquement en mode utilisateur.

Ce fonctionnement est résumé sur la figure 1.3.

![Accès au matériel sur un système moderne](src/img/noyau_syscall.png)

Les appels systèmes en eux-mêmes sont réalisés au moyen d'une interruption
matérielle. En somme, leur nature et la façon dont ils sont réalisés dépend à
la fois de la machine et du système d'exploitation.

## Cas des Unixoïdes

La famille Unix est la plus grande famille de systèmes d'exploitation qui
existe à ce jour. Ces systèmes ont tous un fonctionnement commun, qui
reproduit celui du système UNIX développé par AT&T durant les années 1970.

Dans cette famille de systèmes, on trouvera notamment GNU/Linux[^linuxunix] et
OpenBSD, qui sont des systèmes d'exploitation libres, mais également Mac OS X,
qui peuple les ordinateurs de la marque Apple, ainsi qu'une myriade d'autres
systèmes moins connus tels que Minix, Plan9 ou GNU/Hurd.

[^linuxunix]: En toute rigueur, GNU/Linux est une réécriture complète d'UNIX,
mais ça ne l'empêche pas de fonctionner de la même façon et de suivre en très
grande partie les mêmes standards. C'est la raison pour laquelle on le place
très volontiers dans la famille des systèmes Unix.

### Appels systèmes sous Unix

Les systèmes Unix proposent tous, dans leur bibliothèque C standard (que l'on
appelle la `libc`), des fonctions permettant de réaliser des appels système.

La plus basique d'entre elles est la fonction `syscall()` dont la signature est
la suivante :

```c
int syscall (int number, ...)
```

En somme, pour réaliser un appel système depuis un programme C, il suffit de
passer à cette fonction le *numéro* de l'appel système en question, ainsi que
les arguments attendus par celui-ci (si tel est le cas), et celle-ci retournera
un entier.

Essayons d'utiliser cette fonction depuis Python. Pour commencer, sachez qu'il
est possible de charger la `libc` et d'en utiliser les fonctions directement
depuis Python, en nous servant du module standard `ctypes`. Sous GNU/Linux, la
`libc` sera nommée `libc.so.6`. Chargeons-la :

```python
>>> import ctypes
>>> libc = ctypes.CDLL('libc.so.6')
>>> libc.syscall
<_FuncPtr object at 0x7f634a29a5c0>
```

La dernière ligne nous indique que Python a bel et bien accès à la fonction
standard `syscall`. Il nous suffit maintenant de trouver un numéro d'appel
système à tester. Prenons-en un qui n'a besoin d'aucun argument. Par exemple,
récupérons le numéro du processus dans lequel notre console Python est en train
de tourner au moyen de l'appel système `getpid`. Cet appel système correspond
sur ma machine (Linux 3.13 sur une architecture `x86_64`) au numéro 39.

Essayons !

```python
>>> libc.syscall(39)
7827
```

Cela semble avoir fonctionné, mais comment en être certain ?

### Les *stubs* de la `libc` et le module standard `os` de Python

En fait, dans la plupart des cas, nous n'aurons jamais besoin d'utiliser cette
fonction `syscall()` directement. Déjà, sachez que la `libc` définit un certain
nombre de raccourcis, appelés *stubs*, pour la plupart des appels systèmes
courants. Pour en revenir à notre exemple, il existe effectivement une fonction
standard `getpid()` en C, qui réalise ce même appel système :

```python
>>> libc.getpid()
7827
```

Avouez que c'est quand même plus pratique et portable que d'appeler un numéro
arbitraire.

Maintenant, sachez que ces *stubs* sont également portés en Python. On en
trouvera un très grand nombre, par exemple, dans le module `os` *dont c'est
précisément le rôle*[^osrole]. Ainsi, au lieu de charger la libc comme des
barbares, nous aurions tout simplement pu utiliser la fonction `getpid()` de
Python, qui réalise l'appel système du même nom !

```python
>>> import os
>>> os.getpid()
7827
```

[^osrole]: Je parie que c'est la première fois que l'on vous présente le module
`os` pour ce qu'il est vraiment : une collection d'appels systèmes.

Constatez que nous obtenons bien le même résultat, que nous utilisions la
fonction `syscall()`, le *stub* `getpid()` de la `libc` ou la fonction
`os.getpid()` de la bibliothèque standard Python.

Gardez cependant cette fonction `syscall()` dans un coin de votre mémoire. Il
nous arrivera peut-être un jour (ou dans un futur chapitre de ce cours) de
devoir nous rappatrier sur cette fonction pour réaliser un appel système dont
il n'existe ni fonction équivalente dans le module `os`, ni *stub* dans la
`libc`. Dans ce cas, nous serons bien contents de pouvoir recourir à cette
solution.

## Le shell Unix

Un système d'exploitation, ce n'est pas seulement un noyau. C'est d'ailleurs
pour cette raison que l'on ne devrait jamais parler de « Linux » tout seul
lorsque l'on désigne le système d'exploitation GNU/Linux : pour qu'un système
soit utilisable, il faut envelopper le noyau d'une couche logicielle qui sert
*d'interface* entre celui-ci et l'utilisateur. Cette interface, dans le
vocabulaire Unix, porte le nom de *shell* (comme le mot anglais qui désigne une
*coquille*).

La notion de *shell*, en raison de nombreux abus de langages, est finalement
assez vague, car elle prend un sens différent selon le contexte dans lequel on
y fait référence. Dans le contexte d'un système d'exploitation, le shell est
l'interface qui enveloppe le noyau. Cette interface peut être *graphique* ou en
*ligne de commande*, tant qu'elle désigne la couche logicielle qui permet à
l'utilisateur d'interagir avec son OS. On s'accordera toutefois à dire qu'un
*shell Unix* désigne bel et bien une interface en ligne de commande, puisque la
norme POSIX (qui sert à uniformiser le fonctionnement des systèmes Unix)
définit le format et le comportement d'un certain nombre de commandes standard.

En revanche, il existe plusieurs *shells* qui permettent
tous de travailler avec ces mêmes commandes standard. Le plus répandu à l'heure
actuelle est `bash` (pour *Bourne-Again SHell*), puisque c'est celui qui vient
par défaut avec les distributions de GNU/Linux et Mac OS X.

À la rigueur, pour suivre ce cours, vous pouvez bien utiliser le shell que vous
voudrez. Personnellement, j'utilise `zsh`, mais toutes les lignes de commande
que nous taperons dans ce cours seront compatibles avec `bash`. De toute
manière, dès que nous aurons besoin de réaliser des choses élaborées, nous
utiliserons Python.

Ce cours ne portant pas sur le shell mais sur la programmation système, toutes
les commandes utilisées ne seront pas nécessairement détaillées. Le lecteur est
invité à découvrir la fonction de celles qu'il ne connaît pas en explorant de
lui-même, soit en affichant les messages d'aide (via `<cmd> --help`), soit en
se retournant vers le manuel standard (`man <cmd>`).

### Conventions de notation

Dans toute la suite de ce cours, nous distinguerons trois notations
particulières pour différencier les lignes de commande. Les commandes à taper
dans une console Python seront précédées, comme dans l'interpréteur standard,
par trois chevrons fermants (`>>>`) :

```python
>>> print("Hello, World!")
Hello, World!
```

Les commandes *shell* sous un utilisateur quelconque seront précédées d'un
symbole *dollar* (`$`) :

```
$ whoami
arnaud
```

Les commandes *shell* tapées sous l'utilisateur spécial `root` seront précédées
d'un symbole *pourcent* (`%`) :

```
% whoami
root
```

### Utilisateurs et groupes

Tous les utilisateurs d'un système Unix sont identifiés par un numéro
d'utilisateur unique : leur UID (User IDentifier).

Par exemple, sur ma machine, je dispose de l'UID 1000 :

```
$ id -u
1000
```

En fait, cet UID unique est associé à un *nom d'utilisateur* dans une base de
données spéciale, ce qui est beaucoup plus facile à manipuler pour nous autres,
simples mortels :

```
$ id -un
arnaud
```

Par ailleurs, chaque utilisateur fait partie d'un ou plusieurs **groupes**. La
plupart des utilisateurs ont un *groupe primaire* qui leur est propre (leur
GID), et appartiennent possiblement à plusieurs *groupes secondaires*. De la
même manière que les utilisateurs, les groupes sont désignés par un numéro de
groupe unique, et associés à un nom de groupe dans une base de données. Ces
groupes servent à identifier les utilisateurs qui partagent des permissions
communes (nous en reparlerons plus tard).

```
$ id
uid=1000(arnaud) gid=1000(arnaud) groups=1000(arnaud),4(adm),27(sudo),...
```

Dans cet exemple, on s'aperçoit que j'ai l'UID 1000, que mon groupe primaire
(mon GID) est le numéro 1000, et que j'appartiens également aux groupes 4 et 27
(et plein d'autres que j'ai éludés).

La base de données des utilisateurs s'appelle généralement `passwd`. Elle est
écrite en clair dans le fichier `/etc/passwd`. La base de données des groupes,
quant à elle, se trouve classiquement dans le fichier `/etc/groups`.

### Les modules `pwd` et `grp` de la bibliothèque standard

Plutôt que d'interagir directement avec ces fichiers, il est plus pratique
d'utiliser respectivement les modules standard `pwd` et `grp` pour accéder en
lecture aux bases de données des utilisateurs et des groupes. Leur
documentation ([pwd](https://docs.python.org/3.4/library/pwd.html),
[grp](https://docs.python.org/3.4/library/grp.html)) est suffisamment concise
pour que je me permette de vous y renvoyer directement.

Dans l'exemple suivant, on utilise l'appel système `getuid` pour récupérer
le UID de l'utilisateur du programme, et le saluer par son login que l'on
retrouve dans la base de données `passwd` via le module `pwd` :

```python
>>> import os, pwd
>>> def say_hello():
...     pwd_struct = pwd.getpwuid(os.getuid())
...     print("Hello, {}!".format(pwd_struct.pw_name))
...
>>> say_hello()
Hello, arnaud!
```

#### Exercices

1. Compléter cette fonction `say_hello` de façon qu'elle affiche également le
   GID de l'utilsateur (via l'appel système `getgid`) ainsi que le nom de ce
groupe en vous servant du module `grp`.

2. Utiliser ces modules pour essayer de reproduire le comportement de la
   commande shell `id` (au moins l'appel de base, sans option). Pour connaître
tous les groupes auxquels un utilisateur appartient, vous pouvez vous servir de
l'appel système
[`os.getgrouplist`](https://docs.python.org/3.4/library/os.html#os.getgrouplist)
qui prend en arguments le nom d'utilisateur en toutes lettres ainsi que son
GID.

```python
>>> os.getgrouplist('arnaud', 1000)
[1000, 4, 24, 27, 30, 46, 102, 108, 124]
```

Nous aurons l'occasion de revenir sur les notions d'utilisateurs et de groupes
dans plusieurs chapitres de ce cours. En particulier, chaque fois que nous
parlerons des permissions, qui sont le principal mécanisme de sécurité des
systèmes Unix.


