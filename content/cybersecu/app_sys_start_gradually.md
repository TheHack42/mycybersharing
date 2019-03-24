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

![instruction](/img/instruction.jpg)

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

![register](/img/register.jpg)

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

Ces adresses sont codées suivant l'architecture de destination. Un programme compilé pour une architecture 32 bits aura des adresses 32 bits et un programme 64 bits, des adresses 64 bits. Il n'est pas possible de faire fonctionner un programme 64 bits sur une architecture de processeur 32 bits. En revanche, l'inverse est possible en simulant une architecture 32 bits et ainsi un programme 32 bits pourra fonctionner sur un processeur 64 bits.
\
\
Deux instances d'un même programme peuvent utiliser les mêmes adresses sans que cela pose problème... Ce qui ne devrait pas être possible.\
Cela a été rendu possible grâce à l'utilisation de la mémoire virtuelle. Sur un système, deux types d'adresses existent : les adresses virtuelles et les adresses physiques.\
Pour faire simple, un programme a l'impression qu'il possède toute la mémoire à lui seul parce-qu'on lui a attribué une zone mémoire virtuelle et non réelle :

 - Adresse virtuelle : elles sont utilisées à l'intérieur d'un programme
 - Adresse physique : c'est les adresses utilisées physiquement par les puces présentes sur les barrettes de RAM

![virtual memory](/img/virtual_memory.jpg)

Voilà pourquoi un programme peut utiliser les mêmes adresses virtuelles mais pas les mêmes adresses physiques. On place en quelque sorte notre programme dans des bacs à sable ([sandbox](https://fr.wikipedia.org/wiki/Sandbox_(s%C3%A9curit%C3%A9_informatique))) et c'est le noyau du système d'exploitation qui gère les opérations de plus bas niveau en relation avec le matériel.

Les adresses sont utilisées partout et pour tout. Toutes les cases mémoires sont rattachées à une adresse : les fonctions, les variables, les instructions... Elles permettent d'accéder à une zone de la mémoire en utilisant un identifiant, un nombre entier naturel. C'est le microprocesseur qui va s'occuper d'accéder à cette zone mémoire en lui envoyant son adresse.

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

![segments](/img/segments.jpg)

Ce schéma illustre une représentation de la mémoire virtuelle d'un programme. La position des segments ne change pas d'une exécution à l'autre et reste toujours dans cet ordre.\
Nous pouvons constater que la pile grossie du haut vers le bas et que le tas grossit du bas vers le haut. La taille de ces deux segments n'est donc pas fixe.

### Mais cette pile, c'est quoi en fait ?

Ça!

![stack](/img/stack_1.jpg)

Bon d'accord, pas exactement mais il y a des points communs avec la pile de notre programme. La pile est une structure LIFO, c'est-à-dire que le dernier élément ajouté sera le premier à être retiré. Quand on empile des assiettes les unes sur les autres, il faut d'abord retirer la première pour ensuite retirer la deuxième assiette de la pile.

![stack](/img/stack_2.jpg)

La pile est principalement utilisée pour stocker les données nécessaires à l'exécution d'une fonction et également préserver le pointeur d'exécution (registre EIP) afin de reprendre l'exécution de cette fonction. On y retrouve les arguments de notre fonction mais également les variables locales à celle-ci. Toutes ces choses-là sont appelées la **stack frame** (cadre de pile).

### Un exemple concret
Prenons l'exemple d'un programme très basic possédant deux fonctions : "main" et "addition".\
Le programme va réaliser la somme entre deux nombres et afficher le résultat à l'écran :

```
#include <stdio.h>

void addition(int a, int b) {
    printf("La somme de %d et %d est %d\n", a, b, a + b);
}

int main() {
    addition(4, 8); // <= Appel de la fonction "addition"
    return 0;
}
```

Lançons la compilation du programme et exécutons-le sans plus attendre :

```
[julien@hack42]$ gcc main.c -o test -m32
[julien@hack42]$ ./test 
La somme de 4 et 8 est 12
```

La commande "gcc" (GNU Compiler Collection) est un ensemble de compilateurs capables de compiler divers langages de programmation. Ici, nous l'utilisons pour compiler un programme en C. L'argument "-m32" permet de compiler notre programme en 32 bits et ainsi notre programme possédera des adresses de 4 octets.

Mais du coup qu'est-ce qu'il se passe concrètement pendant l'exécution du programme ?\
Tout d'abord, le programme commence par exécuter la fonction principale "main" et dès le début, la fonction "addition" est appelée (opcode "call" en assembleur). Regardons à quoi ressemble le code assembleur permettant d'effectuer cet appel :

```
[-------------------------------------code-------------------------------------]
   0x804846f <main+20>:	push   0x8
   0x8048471 <main+22>:	push   0x4
=> 0x8048473 <main+24>:	call   0x8048436 <addition>
   0x8048478 <main+29>:	add    esp,0x10
Guessed arguments:
arg[0]: 0x4 
arg[1]: 0x8
[------------------------------------stack-------------------------------------]
0000| 0xffffc560 --> 0x4 
0004| 0xffffc564 --> 0x8
```

On constate que les arguments sont poussés (push) sur le haut de la pile et c'est seulement après cela que la fonction est appelée.\
***Petite remarque : les arguments sont poussés en commençant par le dernier afin d'avoir le premier en haut de la pile.***

Maintenant allons voir comment ça se passe pour la fonction "addition" :

```
[-------------------------------------code-------------------------------------]
=> 0x8048436 <addition>:	  push   ebp
   0x8048437 <addition+1>:	mov    ebp,esp
   0x8048439 <addition+3>:	sub    esp,0x8
   0x804843c <addition+6>:	mov    edx,DWORD PTR [ebp+0x8]
   0x804843f <addition+9>:	mov    eax,DWORD PTR [ebp+0xc]
[------------------------------------stack-------------------------------------]
0000| 0xffffc55c --> 0x8048478 (<main+29>:	add    esp,0x10)
0004| 0xffffc560 --> 0x4 
0008| 0xffffc564 --> 0x8
```

Mais... il y a un élément supplémentaire en haut de la pile !\
Eh oui. A votre avis, comment le programme peut-il reprendre l'exécution de la fonction "main" sans sauvegarder sa position actuelle ? Ce n'est pas possible. Pour pouvoir y parvenir, à l'exécution de l'instruction "call", le processeur empile automatiquement le registre EIP (contenant l'instruction suivante). Dans notre exemple, "0x8048478" (main+29).

#### Initialiser la stack frame (prologue)

**Chaque fonction possède sa propre stack frame** et la fonction "addition" n'y échappe pas. Cela veut dire qu'elle doit donc se réserver sa stack frame. Regardez ces deux instructions :

```
push   ebp
mov    ebp,esp
```

Ces deux instructions au début de la fonction "addition" sont appelées le **prologue**. Elles permettent dans un premier temps de sauvegarder le bas de la pile courante et ensuite d'en initialiser une nouvelle. Attendez, laissez-moi vous faire un schéma pour que ça soit plus simple !

![stack frame](/img/stack_frame.jpg)

Vous voyez que la stack frame de "addition" est vide, parce qu'elle n'a pas encore été utilisée. De ce fait, le pointeur ESP est identique au pointeur EBP (ils sont confondus). Quand la stack frame de "addition" va être consommée, alors le pointeur ESP va monter vers les adresses les plus basses.\
Maintenant voyons comment récupérer les arguments qui ont été passés à la fonction "addition".

#### Récupérer les arguments

```
mov    edx,DWORD PTR [ebp+0x8]
mov    eax,DWORD PTR [ebp+0xc]
```

En français, cela donnerait :

- "Copier la valeur (DWORD pour [double mot](https://fr.wikipedia.org/wiki/Mot_(architecture_informatique))) pointant en EBP+8 vers le registre EDX" (**argument 1**)
- "Copier la valeur pointant en EBP+12 vers le registre EAX" (**argument 2**)

Comme EBP pointe sur le haut de la stack frame de "main" alors il suffit de l'utiliser comme référence. Pour récupérer les arguments, nous devons donc faire **"EBP+0x8" pour le premier** et **"EBP+0xc" (0xc = 12) pour le second** (regardez le schéma plus haut).

#### Restaurer la stack frame de main (épilogue)

A la fin de la fonction "addition", il faut restaurer la stack frame de main et reprendre l'exécution de cette dernière. Regardons maintenant les dernières instructions de la fonction "addition" pour comprendre comment cela fonctionne :

```
leave  
ret 
``` 

Eh oui, c'est ces deux petites instructions qui permettent de restaurer la stack frame de main et ainsi reprendre son exécution normalement. On appelle cette phase, l'**épilogue**.

---

Vous vous souvenez que dans le prologue EBP a été poussé en haut de la pile ? L'instruction "leave" va maintenant utiliser cette valeur pour restaurer la stack frame de main :

![stack frame restore](/img/stack_frame_restore.jpg)

Dans l'ordre, l'instruction leave va effectuer ceci :

1. Restauration de ESP. Pour cela, ESP = EBP+0x4
2. Restauration de EBP. Pour cela, il utilise la valeur poussée sur la pile tout au début (valeur en bleue sur le schéma). La valeur de EBP vaudra 0xffffc578.

L'instruction "leave" est équivalente à ceci :

```
lea    esp,[ebp+0x4]				; esp = ebp + 4
mov    ebp,DWORD PTR [ebp]	; ebp = valeur de ebp
```

---

Par la suite, l'instruction "ret" (pour retour) va permettre de reprendre l'exécution de la fonction "main" et ceci est possible parce que EIP a été poussé sur la pile (en rouge sur le schéma) par l'instruction "call".

L'instruction "ret" est équivalente à ceci :

```
jmp    DWORD PTR [esp]	; saute à la valeur (adresse) présente sur le haut de la pile
add    esp,0x4 					; ajoute 4 à esp
```

---

Et voilà, la stack frame de main a été restaurée correctement et nous avons repris l'exécution de main là où on c'était arrêté au moment du "call" (main+29) :

```
[----------------------------------registers-----------------------------------]
EBP: 0xffffc578 --> 0x0 
ESP: 0xffffc560 --> 0x4 
EIP: 0x8048478 (<main+29>:	add    esp,0x10)
[-------------------------------------code-------------------------------------]
   0x804846f <main+20>:	push   0x8
   0x8048471 <main+22>:	push   0x4
   0x8048473 <main+24>:	call   0x8048436 <addition>
=> 0x8048478 <main+29>:	add    esp,0x10
[------------------------------------stack-------------------------------------]
0000| 0xffffc560 --> 0x4 
0004| 0xffffc564 --> 0x8
```

![stack call](/img/stack_call.gif)

### Le mot de la fin

J'espère que ce tuto d'introduction sur les failles systèmes vous aura plu et que vous l'avez compris dans sa globalité. Si vous l'avez apprécié, n'hésitez pas à mettre un petit "j'aime" en bas de la page. Si vous avez des questions sur une partie que vous n'avez pas comprise ou tout simplement partager votre avis, alors n'hésitez pas à commenter la page. Et même (surtout) si j'ai fait une boulette.

Maintenant vous êtes prêt pour passer à la suite, l'exploitation d'un buffer overflow (arrive prochainement) :)