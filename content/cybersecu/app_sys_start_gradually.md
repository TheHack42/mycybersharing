---
title: "App-Sys: Débuter progressivement"
slug: "app_sys_start_gradually"
date: 2019-02-17
description: "Commencer pas-à-pas dans le domaine de l'exploitation de failles applicatives systèmes."
---

## "App-Sys", vous avez dit ?
Oui, "App-Sys" :) C'est le principe d'exploiter une vulnérabilité applicative principalement liée à une erreur de développement et qui peut finir par aboutir à des corruptions de différentes zones mémoire.
\
\
L'objectif ici c'est d'aborder pas-à-pas ce domaine en passant par les registres, la structure d'un exécutable, la pile, le tas... bref, des choses compliquées pour un débutant qui souhaite se lancer dans l'exploitation de failles systèmes.

## Les débuts du Intel 8086

L'Intel 8086 est un microprocesseur (unité central de traitement) sortie en 1978 dont son objectif est d'effectuer des calculs. C'est le premier processeur de la famille X86, qui est devenu à ce jour une grande famille et la plus répandue dans le monde des ordinateurs et serveurs informatiques.

![C8086](/img/Intel_C8086.jpg)

Cette petite bébête exécute des instructions machines permettant d'effectuer des opérations (l'addition, la soustraction, multiplication, ET logique...), c'est ce qui fait fonctionner nos programmes et notre système d'exploitation. Cette liste d'instructions est appelée un "jeu d'instructions".\
Depuis ce jour (1978), les processeurs de la famille X86 ont gardé la rétrocompatibilité avec la précédente version. Cela veut dire que le jeu d'instructions du 8086 est également présent dans nos dernières générations de processeurs. Il est donc possible de faire fonctionner un programme datant de 1978 sur les derniers processeurs :). Pas mal, non?

## Et ces instructions, ça ressemble à quoi exactement ?

Une instruction c'est une suite de bits représentant un ordre pour le microprocesseur. Comme le binaire est difficilement compréhensible pour les humains, le programmeur utilise une abréviation, un simple mot-clé suivis des arguments qui va désigner l'instruction à exécuter. Par la suite, ces mots-clés sont convertis en binaire avant d'être envoyé au microprocesseur.

```
mov ah, 5		; déplace 5 dans "ah" 	: 	ah = 5
add ah, 3		; effectue une addition : 	ah = ah + 3 ("ah" vaut 8 du coup)
```

Prenons exemple avec la première instruction :

```
[mov ah, 5] = [0xb405] = [10110100 00000101] <-- envoyé au microprocesseur
```

Les instructions possèdent une taille qui sera variable suivant les arguments passés à celle-ci mais également de l'architecture. Si nous reprenons l'instruction "mov" utilisée ci-dessus, elle fait très exactement 2 octets (0xb4 0x05).\
\
En réalité, le petit mot-clé au début de l'instruction est appellé un "**opcode**" (code opération) et permet de déterminer la nature de l'instruction. "mov" est donc un opcode parmis tant d'autres.

![C8086](/img/instruction.jpg)

## Les registres pour nos calcules

Chaque microprocesseur inclut une suite de plusieurs registres, un emplacement mémoire interne au microprocesseur. Il s'agit de la mémoire la plus rapide d'un ordinateur dû fait qu'elle soit présente directement dans l'unité de calcul.\
Ces petites zones de mémoire ont commencé par faire 16 bits (à l'époque du 8086), puis 32 bits et maintenant 64 bits pour les processeurs x64.\
Suivant la version que nous souhaitons, le préfixe change : **E** pour obtenir la version 32 bits et **R** pour la version 64 bits du registre.\
Voici la liste des registres les plus importants:

 - **AX** (16 bits) -> **EAX** (32 bits) -> **RAX** (64 bits)
 - **BX** (16 bits) -> **EBX** (32 bits) -> **RBX** (64 bits)
 - **CX** (16 bits) -> **ECX** (32 bits) -> **RCX** (64 bits)
 - **DX** (16 bits) -> **EDX** (32 bits) -> **RDX** (64 bits)
 - **SI** (16 bits) -> **ESI** (32 bits) -> **RSI** (64 bits)
 - **DI** (16 bits) -> **EDI** (32 bits) -> **RDI** (64 bits)
 - **BP** (16 bits) -> **EBP** (32 bits) -> **RBP** (64 bits) : *adresse de la partie Basse de la Pile*
 - **SP** (16 bits) -> **ESP** (32 bits) -> **RSP** (64 bits) : *adresse de la partie Supérieure de la Pile*
 - **IP** (16 bits) -> **EIP** (32 bits) -> **RIP** (64 bits) : *adresse de la prochaine instruction à exécuter*

*La pile est expliquée plus bas, ne vous inquiétez pas ;)*

![C8086](/img/register.jpg)

Ces registres sont utilisés par les différentes instructions du programme.

## Un programme comment ça marche sinon ?

Suivant le système d'exploitation, un programme va avoir une structure différente mais chacune reste équivalente. Nous allons nous pencher sur la structure du format de fichier ELF (Executable and Linkable Format) qui est le format des applications sous linux.
\
\
Avant qu'un programme soit exécuté, il est chargé en mémoire et ensuite la première instruction se trouvant au point d'entrer du programme (**EP** pour Entry Point) est exécutée.

### Le système d'adressage mémoire

Un programme contient une zone mémoire divisée en octets. Chaque octet de cette zone contient une adresse représentée en [hexadécimale](https://fr.wikipedia.org/wiki/Syst%C3%A8me_hexad%C3%A9cimal) permettant de l'utiliser. La première adresse est la plus petite et la dernière la plus grande.

*Mémoire du programme*

| Adresse mémoire | Valeur |
|-----------------|--------|
| 4000            | B      |
| 4001            | o      |
| 4002            | n      |
| ....            | ...    |

Ces adresses sont codées soit en 32 ou 64 bits suivant l'architecture. Un programme compilé pour une architecture 32 bits aura des adresses 32 bits et un programme 64 bits, des adresses 64 bits. Il n'est pas possible de faire fonctionner un programme 64 bits sur une architecture de processeur 32 bits. En revanche, l'inverse est possible en simulant une architecture 32 bits et ainsi un programme 32 bits pourra fonctionner sur un processeur 64 bits.
\
\
Deux instances d'un même programme peuvent utiliser les mêmes adresses sans que cela pose problème... Ce qui ne devrait pas être possible.\
Cela a été rendu possible grâce à l'utilisation de la mémoire virtuelle. Sur un système, deux types d'adresses existent : les adresses virtuelles et les adresses physiques.\
Pour faire simple, un programme a l'impression qu'il possède toute la mémoire à lui seul parce-qu'on lui a attribué une zone mémoire virtuelle et non réelle :

 - Adresse virtuelle : elles sont utilisées à l'intérieur d'un programme
 - Adresse physique : c'est les adresses utilisées physiquement par les puces présentes sur les barrettes de RAM

![C8086](/img/virtual_memory.jpg)

Voilà pourquoi un programme peut utiliser les mêmes adresses virtuelles mais pas les mêmes adresses physiques.

### Différents segments

Un programme contient plusieurs segments (sous-zone mémoire) qui sont des espaces d'adressage virtuel contenant toutes les informations permettant de mener à bien l'exécution du programme (des chaînes de caractères, des données, les instructions du programme...).\
Les segments sont attachés à des droits d'accès (lecture/écriture/exécution) permettant ainsi de les protéger.\
\
Les principaux segments sont :

- **.text** : contient les instructions du programme (le code)
- **.data** : contient toutes les variables globales ou statiques possédant une valeur prédéfinie et pouvant être modifiées
- **.rodata** : à l'opposition au segment .data, ce segment est uniquement en lecture seule (**ro** pour read-only)
- **.bss** : contient toutes les variables globales ou statiques initialisées à zéro ou n'ayant pas d'initialisation explicite dans le code source
- **heap** : le tas contient toutes les variables dynamiquement allouées au cours de l'exécution du programme
- **stack** : la pile est une structure [LIFO](https://fr.wikipedia.org/wiki/Last_in,_first_out)

Cette liste n'est pas complète mais les principaux segments y sont. Ne vous inquiétez pas si vous n'avez pas très bien compris à quoi servaient les segments. Par la suite, avec la pratique cela viendra :)

![C8086](/img/segments.jpg)

Ce schéma illustre une représentation de la mémoire virtuelle d'un programme. La position des segments ne change pas d'une exécution à l'autre et reste toujours dans cet ordre.\
Nous pouvons constater que la pile grossie du haut vers le bas et que le tas grossit du bas vers le haut. La taille de ces deux segments n'est donc pas fixe.

### Mais cette pile, c'est quoi en fait ?

Ça!

![C8086](/img/stack_1.jpg)

Bon d'accord, pas exactement mais il y a des points communs avec la pile de notre programme. La pile est une structure LIFO, c'est-à-dire que le dernier élément ajouté sera le premier à être retiré. Quand on empile des assiettes les unes sur les autres, il faut d'abord retirer la première pour ensuite retirer la deuxième assiette de la pile.

![C8086](/img/stack_2.jpg)

La pile est principalement utilisée pour stocker les données nécessaires à l'exécution d'une fonction ainsi que la position actuelle de notre pointeur d'exécution (registre EIP) 