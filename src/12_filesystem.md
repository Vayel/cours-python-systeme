# Le système de fichiers

En tant qu'utilisateurs réguliers de nos ordinateurs, on pourrait se demander
ce qu'il y a de *si* important à propos du système de fichiers pour lui dédier
un chapitre entier. En effet, c'est plutôt intuitif : c'est une arborescence
composée de répertoires avec des fichiers dedans...

Cependant, vous avez sûrement déjà entendu quelqu'un déclarer que « Unix c'est
génial parce que *tout est fichier* ». Vous n'y avez sûrement pas prêté
attention sur l'instant, mais cela fait pourtant toute la différence.
Figurez-vous que le système de fichiers Unix va très au-delà de la simple
représentation du contenu d'un disque. C'est **une puissante abstraction
matérielle** à la fois intuitive pour l'utilisateur et extrêmement commode pour
le développeur : pour preuve, il vous suffit de savoir lire ou écrire dans un
fichier pour être capable d'écouter les évènements liés aux périphériques
d'entrée, interagir avec les périphériques de sortie, et même surveiller les
programmes qui sont en train de s'exécuter !

Cela vaut bien la peine de jeter un oeil sous le capot, vous ne trouvez pas ?

## Arborescence des fichiers

### L'interface utilisateur classique

Commençons par enfoncer des portes ouvertes. Un système de fichiers
(*filesystem*) se présentera toujours à l'utilisateur sous la forme d'une
structure arborescente :

* Les noeuds de l'arbre sont appelés des **répertoires** (*directories* en
  anglais), quoique l'appellation "dossier" (*folder*) a été largement
  popularisée par les systèmes d'exploitation de chez Microsoft.

* Les feuilles de l'arbre s'appellent des **fichiers** (*files*).

* La **racine** du système de fichiers (*filesystem root*), est désignée
  conventionnellement par une simple barre oblique (`/`).

![Représentation abstraite du système de fichiers](src/img/filesystem_tree.png)

La figure 2.1 illustre cette arborescence. Nous y voyons trois fichiers
différents, dont les chemins sont respectivement `/usr/bin/python3.4`
(l'interpréteur Python), `/bin/bash` (le shell `bash`) et
`/home/arnaud/.zshrc` (mon fichier de configuration personnel pour le shell
`zsh`).

Il y a d'énormes chances pour que vous manipuliez déjà des arborescences de
fichiers depuis des années. Ainsi, je ne vous apprendrai rien en vous disant
que la commande `cd` permet de se déplacer dans cette arborescence, que `pwd`
permet d'afficher le chemin du répertoire courant et que `ls` permet de lister
le contenu d'un répertoire :

```
$ cd /home/arnaud/doc/cours-systeme
$ pwd
/home/arnaud/doc/cours-systeme
$ ls
makefile  src
```

Peut-être vous apprendrai-je en revanche que Python permet de réaliser les
mêmes opérations, notamment grâce aux appels systèmes `os.chdir()` et
`os.getcwd()`, ainsi que la fonction utilitaire `os.listdir()` :

```python
>>> os.chdir('/home/arnaud/doc/cours-systeme')
>>> os.getcwd()
'/home/arnaud/doc/cours-systeme'
>>> os.listdir()
['src', 'makefile']
```

Toutes ces notions consistent à manipuler la représentation abstraite du
système de fichier que le noyau nous expose. C'est largement suffisant pour un
utilisateur occasionnel d'un ordinateur, mais je suis certain que le *hacker*
qui sommeille en vous est bien plus curieux que ça, et désire en savoir plus
sur ce qui se cache derrière cette abstraction.

### La représentation bas niveau : les *inodes*

Peut-être avez-vous déjà utilisé l'option `-l` de la commande standard `ls`
afin d'afficher le résultat sous la forme d'une liste détaillée :

```bash
$ ls -l
total 628
-rw-rw-r-- 1 arnaud arnaud    477 avril 17 22:51 makefile
drwxrwxr-x 3 arnaud arnaud   4096 avril 21 22:44 src
```

Cette commande fait apparaître pour chaque noeud fils du répertoire courant des
informations telles que l'utilisateur et le groupe propriétaires du noeud, sa
taille et sa date de dernière modification. On peut légitimement se demander
d'où la commande `ls` tire toutes ces infos !

En réalité, tous les noeuds du système de fichiers (qu'il s'agisse de fichiers
simples, de répertoires, ou d'autres choses encore) sont associés en interne
à une structure que l'on appelle un *inode* (de l'anglais *index node* :
*noeud d'index*). Cette structure recelle de nombreuses informations, et sert à
*indiquer* où se trouvent les données du noeud correspondant.

Ces données peuvent prendre plusieurs formes. Dans le cas d'un système de
fichiers *réel* (c'est-à-dire qui correspond vraiment au contenu d'un disque
dur), on pourra trouver :

* Des données de fichiers brutes, qui ne sont rien d'autre que des séquences
  d'octets.
* Des données de répertoire. On peut se représenter un répertoire comme
  une collection d'**entrées de répertoire**. Chaque entrée associe le *nom*
d'un noeud fils du répertoire au *numéro* de son inode.

Pour bien se représenter cette structure à bas niveau, la figure 2.2 détaille
le chemin `/bin/bash` qui consiste à traverser deux répertoires (`/` et `/bin`)
pour retrouver les données du fichier `bash`.

![Représentation physique du chemin `/bin/bash`](src/img/filesystem_inodes.png)

Le *numéro d'inode* est un nombre permettant d'identifier un noeud de
l'arborescence de façon unique sur le périphérique auquel il appartient. Pour
l'afficher, il suffit de passer l'option `-i` à la commande `ls`

```
$ ls -i
6422940 makefile  6422937 src
```

Ce dernier exemple nous montre que dans le répertoire courant, nous avons une
entrée nommée `makefile` pointant sur l'inode numéro 6422940 et une entrée
nommée `src` pointant sur l'inode numéro 6422937. Notez que je parle bien
d'*entrées* : lorsque l'on traverse un répertoire, nous n'avons aucun moyen de
savoir si une entrée donnée correspond à un fichier ou à un sous-répertoire.
Pour avoir cette info, il faut aller lire les données des inodes.

## Manipulation des *inodes*

La structure réelle des inodes dépend du type de système de fichiers sur lequel
ils stont stockés (et il en existe des tonnes : `ext4`, `reiserfs`, `ramfs`,
`fat32`, `ntfs`…). Cela dit, quelle que soit la structure adoptée par le
système de fichiers, il est possible d'accéder aux données qu'elle recèle en
invoquant l'appel système `stat`, ou bien la commande POSIX du même nom.

### L'appel système `stat`

Nous nous contenterons de réaliser cet appel système depuis Python, en
utilisant la fonction standard `os.stat()`.

```python
>>> stat_res = os.stat('/home/arnaud/doc/cours-systeme/makefile')
>>> stat_res
os.stat_result(st_mode=33204, st_ino=6422940, st_dev=64513, st_nlink=1,
               st_uid=1000, st_gid=1000, st_size=477, st_atime=1429727097,
               st_mtime=1429303879, st_ctime=1429303879)

```

Le retour de cet appel système est une structure complexe que Python représente
par un tuple nommé. Prenons le temps de détailler ses champs dans le tableau
ci-dessous.

-------------------------------------------------------------------------------
Champ       Signification
----------- -------------------------------------------------------------------
`st_mode`   Nombre codé sur 16 bits, décrivant le type et les permissions du
            fichier.

`st_ino`    Numéro d'inode. Identifiant unique de l'inode sur le périphérique.

`st_dev`    Numéro de *device*, qui identifie le périphérique de façon unique
            sur le système.

`st_nlinks` Nombre d'entrées de répertoire qui pointent vers cet inode.

`st_uid`    Identifiant de l'utilisateur propriétaire du fichier.

`st_gid`    Identifiant du groupe propriétaire du fichier.

`st_size`   Taille du fichier en octets.

`st_atime`  *Access time*. Date du dernier accès, sous la forme d'un
            *timestamp Unix*. Ce champ est mis à jour chaque fois qu'un
            programme lit les données du fichier.

`st_mtime`  *Modification time*. Date de la dernière modification. Ce champ
            est mis à jour chaque fois qu'un programme écrit des données dans
            le fichier.

`st_ctime`  *Status change time*. Date de la dernière modification de cet
            inode. **Attention, ce n'est PAS la date de création du
            fichier.**
------------------------------------------------------------------------------
Table: Résultat de l'appel système `stat`

Un *timestamp Unix* est une date représentée sur 32 bits. Il s'agit ni plus ni
moins du nombre de secondes écoulées depuis le premier janvier 1970 à minuit
(le timestamp 0), date que l'on surnomme *Epoch*. Pour décoder un timestamp en
Python, il suffit d'utiliser le module `datetime` :

```python
>>> from datetime import datetime
>>> d = datetime.fromtimestamp(1429303879)
>>> str(d)
'2015-04-17 22:51:19'
```

En somme, on peut en déduire qu'à l'exception du nom des fichiers, toutes les
infos utiles se trouvent dans les inodes.

### Type et permissions d'un fichier

Le champ le plus dense en informations que nous retourne l'appel système
`os.stat()` est sans aucun doute `st_mode` : il y a tellement de choses
encodées sur ces 16 bits que Python dédie un module standard entier
([`stat`](https://docs.python.org/3.4/library/stat.html)) à leur manipulation !

Concrètement, chaque bit de ce nombre est un *flag*, c'est-à-dire une option
qui peut être activée ou non. Manipuler ce style de données est monnaie
courante en C, mais reste assez anecdotique chez les Pythonistes[^flagmanip].

[^flagmanip]: Il y a une explication très simple à cela. Les opérateurs
bit-à-bit sont extrêmement rapides en C car les microprocesseurs savent les
exécuter en une seule opération, mais pas en Python où les opérations sur les
entiers, quelles qu'elles soient, sont comparativement très lentes.

Rappelons rapidement le fonctionnement des flags binaires. Imaginons un champ
*vierge*, où tous les bits sont à 0 :

```python
>>> mode = 0
```

Nous pouvons vérifier si le flag `stat.S_IRGRP` (nous verrons sa signification
plus loin) est activé au moyen d'un *ET bit-à-bit*, et l'activer au moyen du
*OU bit-à-bit*:

```python
>>> import stat
>>> bool(mode & stat.S_IRGRP)
False
>>> mode |= stat.S_IRGRP      # Activation du bit
>>> bool(mode & stat.S_IRGRP)
True
```

Le système va différencier trois types d'utilisateurs pour chaque fichier :

* son **propriétaire**, dont l'UID est égal à celui du fichier (`st_uid`),
* son **groupe propriétaire**, appartenant au groupe désigné par le GID du
  fichier (`st_gid`),
* les **autres**.

Pour chacun de ces types d'utilisateurs, on va définir trois *permissions*
possibles :

* accès en lecture (l'utilisateur a le droit de lire le contenu du fichier),
* accès en écriture (l'utilisateur a le droit de modifier le fichier),
* accès en exécution (l'utilisateur a le droit d'exécuter le fichier).

Le tableau suivant établit la correspondance entre les valeurs des *flags* du
module `stat`, l'indice visuel que l'on retrouvera un peu partout, et leur
signification.

-------------------------------------------------------------------------------
Flag      Mode         Octal Signifiaction
--------- ------------ ----- --------------------------------------------------
`S_IRUSR` `-r--------` 0400  Le propriétaire du fichier a le droit d'y accéder
                             **en lecture**.

`S_IWUSR` `--w-------` 0200  Le propriétaire du fichier a le droit d'y accéder
                             **en écriture**.

`S_IXUSR` `---x------` 0100  Le propriétaire du fichier a le droit
                             **d'exécuter** le fichier.

`S_IRGRP` `----r-----` 0040  Les membres du groupe propriétaire du fichier y ont
                             accès **en lecture**.

`S_IWGRP` `-----w----` 0020  Les membres du groupe propriétaire du fichier y ont
                             accès **en écriture**.

`S_IXGRP` `------x---` 0010  Les membres du groupe propriétaire du fichier ont
                             le droit de l'**exécuter**.

`S_IROTH` `-------r--` 0004  Les autres utilisateurs ont accès au fichier **en
                             lecture**.

`S_IWOTH` `--------w-` 0002  Les autres utilisateurs ont accès au fichier **en
                             écriture**.

`S_IXOTH` `---------x` 0001  Les autres utilisateurs ont le droit
                             d'**exécuter** le fichier.
-------------------------------------------------------------------------------
Table: Masque des permissions sur un fichier

Bien évidemment, ces valeurs sont cumulatives. Ainsi, si l'on regarde celles
de mon fichier `makefile` :

```
$ ls -l makefile
-rw-rw-r-- 1 arnaud arnaud 477 avril 17 22:51 makefile
```

Ce fichier m'appartient et appartient à mon groupe primaire. J'ai le droit de
le lire et de le modifier, mon groupe primaire également, mais tous les autres
utilisateurs ont seulement le droit de le lire. La représentation de ce mode,
en octal, sera :

$$ 0400 + 0200 + 0040 + 0020 + 0004 = 0664 $$

Cette représentation en octal vous semble peut-être sortir de nulle part, mais
sachez qu'elle est utilisée aussi souvent que la notation visuelle dans les
appels systèmes et les commandes shell qui permettent de manipuler le mode des
fichiers, à l'instar de `chmod`[^chmod].

[^chmod]: RTFM ! Non, je plaisante, on va en parler juste après. ;)

Notez le module `stat` nous propose une fonction `filemode()` qui sert à
représenter les permissions du fichier de façon visuelle :

```python
>>> stat.filemode(os.stat('makefile').st_mode)
'-rw-rw-r--'
```

Les plus observateurs d'entre vous l'auront remarqué : cette représentation
visuelle est préfixée d'un dixième symbole alors que nous n'en avons pour
l'instant lu que 9. Qu'est-il donc supposé représenter ?

Voyons voir :

```python
>>> stat.filemode(os.stat('.').st_mode)
'drwxrwxr-x'
```

En appelant `stat` sur le répertoire courant, ce dixième symbole se transforme
en un `d`. D comme ? D comme di… dir… ? *Directory*, oui ! Ce dixième symbole
nous donne le *type* de l'inode. Ainsi, on trouvera les flags suivants dans le
module `stat` :

* `S_IFREG`: l'inode désigne un *fichier régulier* (un fichier, quoi),
  représenté par le préfixe `-`.
* `S_IFDIR`: l'inode désigne un *répertoire*, représenté par le préfixe `d`.

Il existe encore (plein) d'autres flags et d'autres informations encodées dans
ce champ `st_mode`, mais nous allons nous contenter de ceux-ci pour
l'instant[^manstat]. Nous découvrirons toutes les autres au fur et à mesure que
nous toucherons dans ce cours aux notions qu'elles font intervenir, promis !

[^manstat]: Par contre, si vous êtes curieux et voulez absolument en avoir un
aperçu dès maintenant, alors cette fois je vous le dis très sérieusement :
RTFM !  `man 2 stat` vous donnera la page du "*fucking manual*" à propos de l'appel système
`stat`. Alternativement, [la documentation du module Python
`stat`](https://docs.python.org/3.4/library/stat.html) vous dira...
la même chose, en fait, mais sans vous préciser quels bits sont concernés.

### Modifier les permissions d'un fichier

J'ai mentionné un peu plus haut la commande `chmod`. Voyons comment l'utiliser
(ainsi que la fonction Python équivalente, évidemment).

Créons un fichier vide pour faire nos tests.

```
$ touch monfichier
$ stat -c "%A (%a) %U:%G" monfichier
-rw-rw-r-- (664) arnaud:arnaud
```

L'option "`-c "%A (%a) %U:%G"`" de la commande `stat` sert à lui dire que je
veux afficher uniquement les droits (en version lisible puis en octal), le
propriétaire et le groupe du fichier.

La façon la plus frontale de modifier les droits d'un fichier sera de passer à
`chmod` la valeur octale du masque de droits. Par exemple, si je souhaite
retirer les droits en écriture au groupe, je vais calculer le nouveau masque en
prenant l'actuel (0664) auquel je retranche la valeur de `S_IRGRP` (0020), ce
qui me donne 0644 :

```
$ chmod 644 monfichier
$ stat -c "%A (%a) %U:%G" monfichier
-rw-r--r-- (644) arnaud:arnaud
```

L'autre syntaxe acceptée par `chmod` consiste à spécifier :

* le(s) type(s) d'utilisateur(s) (`u` pour le propriétaire, `g` pour le groupe
  et `o` pour les autres),
* l'ajout (`+`) ou le retrait (`-`) du droit,
* la valeur du droit (`r` pour la lecture, `w` pour l'écriture, `x` pour
  l'exécution)

Par exemple, pour rajouter les droits en exécution au propriétaire et ceux en
écriture au groupe et aux autres :

```
$ chmod u+x,go+w monfichier
$ stat -c "%A (%a) %U:%G" monfichier
-rwxrw-rw- (766) arnaud:arnaud
```

Oh et puis non, en fait, je ne veux pas exécuter ce fichier ni que les autres
le lisent ni écrivent dedans, finalement :

```
$ chmod u-x,o-rw monfichier
$ stat -c "%A (%a) %U:%G" monfichier
-rw-rw---- (660) arnaud:arnaud
```

En Python, cependant, cette syntaxe *user-friendly* n'existe pas. La fonction
`os.chmod()` n'accepte que la valeur du masque, que vous pouvez spécifier soit
de façon verbeuse en assemblant celui-ci via les flags du module `stat`.

```python
>>> os.chmod("monfichier",
...       stat.S_IRUSR | stat.S_IWUSR | stat.S_IRGRP
... )
>>> stat.filemode(os.stat("monfichier").st_mode)
'-rw-r-----'
```

Soit, ce que personnellement je trouve beaucoup plus simple, en utilisant
directement la valeur octale :

```python
>>> os.chmod("monfichier", 0o664)
>>> stat.filemode(os.stat("monfichier").st_mode)
'-rw-rw-r--'
```

## Lecture et écriture de fichiers au niveau système

Maintenant que nous avons une abstraction pour représenter les fichiers sur un
périphérique de stockage, voyons un peu comment le noyau va présenter ces
fichiers aux programmes qui sont en train de s'exécuter.

### Inodes et descripteurs de fichiers

Bien sûr, manipuler des fichiers est une chose très courante en programmation.
Après tout, une fois que l'on a compris le fonctionnement des fonctions de base
`open()`, `read()`, `write()` et `close()`, on peut se dire qu'on a fait le
tour de la question… La figure 2.3 montre qu'en réalité, ces quelques appels
système ne sont que la partie émergée de l'iceberg, et qu'il y a bien plus à en
dire que d'expliquer leur signature !

![Accès aux fichiers en mémoire](src/img/fd_table.png)

Alors, que se passe-t'il *réellement* lorsque vous ouvrez un fichier dans un
programme ?

Pour commencer, le noyau va aller chercher l'inode correspondant dans le
système de fichiers. Une fois qu'il a trouvé cet inode (disque), il va le
charger en mémoire. Ensuite, il va rajouter une entrée dans sa *table des
fichiers ouverts*, qui contient en particulier un pointeur sur cet inode
(mémoire). Cette entrée va également contenir d'autres données intéressantes,
comme le *mode d'ouverture* du fichier (lecture, écriture, ajout...) et un
indicateur de position (pour savoir où le processus[^processfwddecl] en est
dans la lecture du fichier).

[^processfwddecl]: Nous n'avons pas encore vu la notion de *processus* dans ce
cours. Si cela vous perturbe, considérez qu'un processus est une instance d'un
programme qui est en train de tourner sur l'ordinateur.

Cette table des fichiers ouverts est globale à tous les processus qui tournent
sur le système. Cela permet, dans certains cas particuliers, que plusieurs
processus manipulent la même entrée de la table des fichiers ouverts, mais nous
verrons ce genre de choses beaucoup plus loin dans ce cours.

Le processus qui va chercher à manipuler ce fichier, quant à lui, dispose (côté
noyau) d'une *table des descripteurs de fichiers*, qui associe, grosso-modo, un
indice entier (le **descripteur de fichier**, qu'on abrègera FD dans la suite,
pour *file descriptor*) à une entrée de la table des fichiers ouverts. Ce qu'il
est intéressant de noter, c'est que cette table est *locale* au processus, tout
comme les descripteurs de fichiers (puisque ce sont tout simplement les indices
de cette table). Ainsi, comme le schéma nous le montre, on peut très bien avoir
un programme ayant ouvert un fichier et que le noyau aura placé dans l'entrée
numéro 5 de sa table des FD, et un second programme qui aura ouvert le même
fichier mais en l'associant localement au FD 12.

Le fait que la table des descripteurs des fichiers du processus se trouve *côté
noyau* n'est pas anodin, puisque ça signifie qu'on ne peut pas la manipuler
directement (et donc que l'on ne peut pas faire n'importe quoi avec). Par
contre, les descripteurs de fichiers sont exposés côté utilisateur, puisque
ce sont eux que le programme passe en argument aux appels système `read`,
`write` et `close` (et tout une palanquée d'autres dont nous nous garderons
bien d'établir une liste aussi fastidieuse que superflue).

Ainsi, de notre point de vue d'utilisateurs des facilités du noyau, ce sont
surtout ces descripteurs de fichiers qui nous intéressent.

### Interactions de base

Nous allons travailler sur un exemple simple. Nous allons copier le fichier
`menu` dont le contenu est le suivant :

```
$ cat menu
* spam
* eggs
* bacon
* spam
* sausage
* spam
* ham
```

Pour cela, nous allons l'ouvrir, en lire le contenu, puis ouvrir un second
fichier (qui n'existe pas encore) pour écrire le contenu à l'intérieur.

Ouvrir un fichier à bas niveau se fait grâce à l'appel système `os.open()`.
Cet appel système prend au minimum en argument :

* le chemin vers le fichier,
* un *flag* parmi :
    * `os.O_RDONLY` pour ouvrir le fichier en lecture seule,
    * `os.O_WRONLY` pour ouvrir le fichier en écriture seule,
    * `os.O_RDWR` pour ouvrir le fichier à la fois en lecture et en écriture.

Sa valeur de retour est, bien sûr, le descripteur de fichier qui aura été
créé dans la table locale du processus.

```python
>>> fd = os.open("menu", os.O_RDONLY)
>>> fd
3
```

Comme vous le constatez, le fichier est associé au descripteur `3` dans la
table des descripteurs de fichiers. Les fd `0`, `1` et `2` étant déjà occupés
par les trois flux d'entrée/sortie standard du processus, que nous découvrirons
plus loin.

Pour lire des données dans un fichier, on appellera la fonction `os.read()` qui
réalisera l'appel système du même nom, en lui passant en arguments :

* le descripteur de fichier à lire,
* le nombre d'octets que nous comptons lire. Ce nombre peut être supérieur à la
  quantité de données restantes à lire dans le fichier, auquel cas la valeur
retournée sera la totalité des données du fichier, ou bien inférieur, auquel
cas un appel ultérieur nous permettra de continuer à lire les données.

```python
>>> data = os.read(fd, 4096)
>>> data
b'* spam\n* eggs\n* bacon\n* spam\n* sausage\n* spam\n* ham\n'
```

Remarquez que la fonction `read` nous a retourné ici un objet de type `bytes`
et non une chaîne de caractères Unicode (`str`) : il s'agit de données brutes,
non décodées.

Ici, nous avons lu la totalité du contenu de notre fichier. Ainsi, si l'on
appelle une nouvelle fois la fonction `read`, nous obtiendrons en retour une
donnée vide :

```python
>>> os.read(fd, 4096)
b''
```

Fermons notre fichier pour libérer le descripteur avec `os.close()` :

```python
>>> os.close(fd)
```

Bien, maintenant, ouvrons le fichier `copie` en écriture :

```python
>>> fd = os.open("copie", os.O_WRONLY)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
FileNotFoundError: [Errno 2] No such file or directory: 'copie'
```

Python nous crache au visage que ce fichier n'existe pas. Contrairement à
l'interface que vous connaissez déjà sûrement, l'appel système `open` ne va pas
créer de nouveau fichier lorsque celui-ci n'existe pas. Et si celui-ci existe,
il ne saura pas s'il doit l'effacer ou écrire à sa suite. C'est la raison pour
laquelle les *flags* de cet appel système, lorsque l'on ouvre le fichier en
écriture, doivent être combinés avec une ou plusieurs des valeurs suivantes :

* `os.O_APPEND` : Écrire à la fin du fichier (en mode "ajout") si celui-ci
  existe déjà,
* `os.O_TRUNC` : Tronquer le fichier (supprimer ses données) si celui-ci existe
  déjà,
* `os.O_EXCL` : Échouer (en Python, lever une exception) si le fichier existe
  déjà,
* `os.O_CREAT`: Créer le fichier si celui-ci n'existe pas encore.

Lorsque l'on utilise cette dernière valeur, il est également nécessaire de
fournir un argument supplémentaire à `os.open` pour spécifier le `mode` du
fichier créé, avec exactement la même sémantique que la fonction `os.chmod`.

Ainsi, nous pouvons ouvrir ce nouveau fichier `copie` en écriture de la façon
suivante :

```python
>>> fd = os.open("copie", os.O_WRONLY | os.O_CREAT | os.O_TRUNC, mode=0o640)
>>> fd
3
```

Remarquez que `fd` vaut ici une nouvelle fois 3 : le descripteur que nous avons
fermé plus haut est réutilisé.

Pour écrire des données dans ce nouveau fichier, rien de plus simple :

```python
>>> os.write(fd, data)
52
```

La fonction nous retourne le nombre d'octets effectivement écrits dans le
fichier, soit 52 (la taille de la donnée). Nous pouvons maintenant refermer le
fichier pour éviter de charger inutilement la table des descripteurs et libérer
la ressource.

```python
>>> os.close(fd)
```

Nous pouvons remarquer que le fichier `copie` a bien été créé, que j'en suis
évidemment propriétaire puisque c'est moi qui l'ai créé, et qu'il s'agit bien
d'une copie du `menu` :

```
$ ls -l copie
-rw-r----- 1 arnaud arnaud 52 avril 26 21:57 copie
$ cat copie
* spam
* eggs
* bacon
* spam
* sausage
* spam
* ham
```

Rien de bien compliqué, en somme !

Mais que sont donc ces trois descripteurs de fichier 0, 1 et 2 ?
Conventionnellement, le système va attribuer à chaque processus trois *flux*
standard :

* `STDIN` (0) : L'entrée standard du processus, qui permet par exemple de lire des
  données que l'utilisateur saisit au clavier,
* `STDOUT` (1) : La sortie standard, que l'on utilise
  conventionnellement pour afficher des données dans la console,
* `STDERR` (2) : La sortie d'erreur standard, sur laquelle on écrit les
  descriptions des erreurs lorsqu'elles se produisent.

Ainsi, les FD 1 et 2 se comporteront comme des fichiers ouverts en écriture :

```python
>>> os.write(1, b'Hello, World!\n')
Hello, World!
14
>>> os.write(2, b"Une erreur s'est produite\n")
Une erreur s'est produite
26
```

Quant à l'entrée standard, celle-ci peut être lue avec `read()`: faites l'essai
en tapant du texte puis en validant avec la touche `<Entrée>`.

```python
>>> os.read(0, 4096)
Ceci est une saisie au clavier.
b'Ceci est une saisie au clavier.\n'
```

Comme vous le constatez, les "fichiers" que le noyau nous expose peuvent
également être des abstractions pour le terminal, le clavier, ou un
périphérique matériel ou d'autres choses encore… Je suppose que vous comprenez,
maintenant, le potentiel des quatres appels que nous venons de voir !

## L'interface haut niveau de Python (parce qu'il qui vous veut du bien)

Dans « la vraie vie », on n'a pratiquement jamais besoin d'utiliser les appels
système que nous venons d'évoquer en Python. Et pour cause ! Celui-ci expose au
développeur une interface beaucoup plus confortable pour manipuler des
fichiers. Cette interface consiste à envelopper un *file descriptor*  dans ce
que Python nomme des *file objects*, et est implémentée dans le module standard
[`io`](https://docs.python.org/3.4/library/io.html).

L'apport majeur de cette interface à haut niveau est qu'elle présuppose
plusieurs types d'interactions différents avec les fichiers :

* l'interaction en mode **binaire** assez bas niveau, qui permet de lire et
  écrire des données brutes dans un fichier,
* l'interaction en mode **texte**, plus haut niveau, qui présuppose que les
  données du fichier sont du texte.

Cela permet au développeur de profiter d'un grand nombre de facilités sans même
avoir besoin de soupçonner l'existence de tous les mécanismes que son langage
lui apporte.

### La toute-puissante builtin `open()`

    TODO

### Les flux de données binaires

    TODO

### Les fichiers texte

    TODO

### Formatage des données textuelles

Le mode texte besoin d'au moins deux paramètres pour fonctionner : l'encodage
du fichier, et le délimiteur de fin de ligne, que l'on peut spécifier en
argument de la builtin `open()`. Par défaut, Python utilisera ceux qui sont
configurés sur le système. On peut retrouver l'encodage par défaut en nous
servant du module `locale`, et le délimiteur de fin de ligne via `os.linesep` :

```python
>>> locale.getlocale()
('en_US', 'UTF-8')
>>> os.linesep
'\n'
```

Ces quelques lignes m'apprennent que mon système est configuré en anglais, et
qu'il utilise l'encodage `utf-8` par défaut, en délimitant les lignes du
symbole ASCII `LINE_FEED`.

Ces considérations sur l'encodage sont importantes car ni Python, ni votre
système d'exploitation ne sont magiques : ils ne chercheront pas à deviner
l'encodage des fichiers texte[^guessencoding]. Par exemple, si je cherche à
ouvrir un fichier texte rédigé sous Windows, je risque fort de me
heurter à un problème :

[^guessencoding]: Mais certains modules non-standard de Python comme
[chardet](https://pypi.python.org/pypi/chardet/2.3.0) en sont capables.

```python
>>> fobj = open('exemple.txt', 'r')
>>> print(fobj.read())
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/lib/python3.4/codecs.py", line 313, in decode
    (result, consumed) = self._buffer_decode(data, self.errors, final)
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xe9 in position 18: ...
```

En effet, il y a de grandes chances que ce fichier ait été encodé en `latin-1`,
comme la commande `file` nous le confirme :

```
$ file exemple.txt
exemple.txt: ISO-8859 text, with CRLF line terminators
```

Il faudra donc spécifier cet encodage à la fonction `open()`[^supportedcodecs]:

[^supportedcodecs]: Pour déterminer l'argument attendu par Python, vous pouvez
vous référer à [ce
tableau](https://docs.python.org/3.4/library/codecs.html#standard-encodings)
qui décrit les encodages qu'il supporte.

```python
>>> fobj = open('exemple.txt', 'r', encoding='iso-8859-1')
>>> fobj.read()
'Donnée encodée\nen latin-1'
```

Pour comparaison, relisons ce fichier, mais cette fois-ci en mode binaire :

```python
>>> fobj = open('exemple.txt', 'rb')
>>> fobj.read()
b'Donn\xe9e encod\xe9e\r\nen latin-1'
```

Au-delà des accents (qui sont encodés de façon propre au `latin-1`), nous
pouvons également remarquer que le passage à la ligne est effectué via la
séquence CRLF (`CARRIAGE_RETURN LINE_FEED`): `'\r\n'`, alors que dans la
version décodée, celui-ci est remplacé par `'\n'`.

Il s'agit du comportement par défaut de Python en mode texte : les retours à la
ligne sont standardisés pour correspondre à un caractère `'\n'` dans les
chaînes de caractères, puis remplacés par la séquence désignée par `os.linesep`
lorsqu'ils sont écrits dans un fichier.

