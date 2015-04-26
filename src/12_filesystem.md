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

### L'appel système `stat` : lire le contenu des inodes

La structure réelle des inodes dépend du type de système de fichiers sur lequel
ils stont stockés (et il en existe des tonnes : `ext4`, `reiserfs`, `ramfs`,
`fat32`, `ntfs`…). Cela dit, quelle que soit la structure adoptée par le
système de fichiers, il est possible d'accéder aux données qu'elle recèle en
invoquant l'appel système `stat`, ou bien la commande POSIX du même nom.

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

## Les "*file-like objects*" en Python
