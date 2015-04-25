---
title: La programmation système en Python
author: "Arnaud Calmettes (nohar)"
date: \today
papersize: a4paper
documentclass: scrreprt
classoption:
    - 11pt
header-includes:
    - \usepackage[left=2cm, right=2cm, bottom=3cm]{geometry}
    - \usepackage{fancyhdr}
#    - \pagestyle{fancy}
    - \usepackage[french]{babel}
...

# Avant-propos {.unnumbered}

Savez-vous comment fonctionne votre système d'exploitation ?

Par exemple, savez-vous de quelle façon les programmes qui tournent sur votre
ordinateur sont capables de communiquer entre eux ? Connaissez-vous la
différence entre un *thread* et un processus ? Ou même sans aller jusque là,
avez-vous une idée de ce qu'il se passe *réellement* lorsque vous ouvrez un
fichier pour le lire ?

Si ce n'est pas le cas, **comment voulez-vous comprendre ce que font vraiment
les programmes que vous écrivez ?**

Même si cela ne saute pas aux yeux, la *programmation système* n'est pas
seulement utile aux barbus qui bidouillent leur Linux dans une cave. Elle
permet à n'importe quel développeur d'avoir une bonne intuition de l'efficacité
de ses programmes, et de comprendre le fonctionnement des mécanismes
de son système d'exploitation.

Contrairement à tous les autres cours que vous pourrez trouver sur la
programmation système, celui-ci n'utilisera pas directement le langage C, mais
*Python*. Son but n'est pas de vous livrer un savoir encyclopédique (il existe
déjà des kilomètres de documentation pour ça), mais plutôt de découvrir toutes
ces notions *utiles*, dont certaines que vous avez sûrement déjà utilisées dans
vos programmes sans forcément le savoir ni les comprendre.

Ainsi, l'objectif de ce cours est de compléter votre culture informatique avec
des notions qui feront de vous un meilleur développeur.

Inévitablement, on ne peut pas faire de programmation système sans se
concentrer sur un système d'exploitation particulier. Dans ce cours, nous nous
pencherons de près sur le fonctionnement des systèmes qui suivent la norme
POSIX, parmi lesquels on trouvera la famille des UNIX (comme BSD ou Mac OS X)
ainsi que GNU/Linux.

Pour suivre ce cours, vous devrez :

* Connaître les bases de la programmation en Python. Nous prendrons Python 3.4
  comme version de référence.
* Disposer d'un ordinateur (ou d'un Raspberry Pi, ou même d'une machine
  virtuelle) sous GNU/Linux (ou l'un des descendants d'UNIX).
* Ne pas prendre peur à la vue d'une ligne de commande.

Alors, êtes-vous prêts à ne plus être un *utilisateur lambda* de votre OS ? La
curiosité de percer ses petits secrets vous a-t'elle piqué au vif ? Ne
frissonnez-vous pas de plaisir à l'idée de mettre les mains dans le cambouis ?
Suivez-moi, et vous ne serez pas déçus !
