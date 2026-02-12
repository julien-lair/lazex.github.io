---
title: Write-up keygen 1
date: 2026-01-12
categories: [Reverse, x86]
tags: [REVERSE,X86, KEYGEN]     # TAG names should always be lowercase
---
Aujourd’hui, j’ai écrit mon premier write-up.  
J’ai découvert le reverse il y a quelques mois, et je m’en suis passionné.

J’ai appris à analyser et reverser des applications iOS et Android. Puis, j’ai souhaité en apprendre davantage sur le reverse d’applications x86.  
C’est pour cela qu’aujourd’hui je vous partage mon premier write-up sur un keygen.

J’ai beaucoup apprécié cette façon de raisonner : savoir trouver des briques utiles pour, petit à petit, reconstruire le programme.

# Keygenme

## Présentation

Nom du programme : Keygenme  
Plateforme : Windows  
Architecture : x86 (32 bits)  
Objectif : Trouver une clé permettant d’activer la licence  

# Analyse

## Point de départ

La première étape consiste à analyser les chaînes de caractères présentes dans le programme.  
Pour cela, sur IDA, ouvrir la vue *Strings*.

![alt text](/images/first-keygen/image.png)

Nous avons deux chaînes de caractères intéressantes : « License valid ! » et « License invalid ! ».  
En regardant où ces chaînes sont utilisées, nous pouvons trouver le point de départ de la vérification de clé.

## Conditionnement des clés

![alt text](/images/first-keygen/image-1.png)

À la fin du premier bloc, nous avons une première condition : le programme doit avoir trois arguments : le nom de l’exécutable + 2 clés.

![alt text](/images/first-keygen/image-2.png)

La seconde condition, en fin du second bloc, impose que le premier argument passé au programme ait une longueur strictement supérieure à 4.

![alt text](/images/first-keygen/image-3.png)

La troisième condition impose que le premier argument ait une taille strictement inférieure à 15 caractères.  
Donc : 4 < len(argument 1) < 15.

![alt text](/images/first-keygen/image-4.png)

Sur ce nouveau bloc d’instructions, une opération `& 1` est effectuée sur la longueur de la chaîne de caractères du second argument.  
Ainsi, le second argument passé au programme doit contenir un nombre pair d’éléments.

![alt text](/images/first-keygen/image-5.png)

Par la suite, une condition impose que le second argument possède exactement 40 caractères.

Pour résumer jusqu’à maintenant :  
Le programme prend 2 chaînes de caractères en paramètres.  
La première a une taille comprise entre 5 et 14 caractères.  
Le second argument a une taille de 40 caractères.

![alt text](/images/first-keygen/image-6.png)

Le second argument est ensuite converti en hexadécimal pour la suite des opérations.

![alt text](/images/first-keygen/image-7.png)

Ces trois blocs correspondent à une boucle `for`.  
Le premier, en haut, permet de tester si le compteur est ≤ 4.  
Si oui, on va dans le bloc de gauche, sinon dans celui de droite.

Le bloc de gauche appelle une fonction `XOR_PREVIOUS`, qui prend en paramètre le second argument du programme converti en hexadécimal.  
À chaque itération suivante, la fonction prend le résultat du XOR précédent.

![alt text](/images/first-keygen/image-8.png)

La fonction `XOR_PREVIOUS` applique un XOR entre chaque octet d’un buffer source et une clé constante sur un octet.  
Le résultat est écrit dans un second buffer de même taille.

Lorsque la fonction se termine et est rappelée (boucle for de 0 à 4), le principe reste le même mais avec une clé différente à chaque appel.

Les différentes clés XOR appliquées sont :  
0x45, 0x37, 0x06, 0x13, 0x26.

![alt text](/images/first-keygen/image-8.png)

Une fois la boucle terminée, nous passons dans le bloc de droite.

![alt text](/images/first-keygen/image-9.png)

Dans ce bloc, une nouvelle condition compare les deux premiers octets du résultat obtenu après les 5 XOR consécutifs avec la valeur 13 37 (présente dans EAX).  
Le résultat obtenu après les 5 XOR est désormais appelé Buf2.

![alt text](/images/first-keygen/image-10.png)

Ensuite, nous avons les trois étapes d’une fonction de hachage : init, update, puis digest (calcul du hash).

Nous pouvons identifier l’algorithme SHA1 en observant, dans `sha1_init`, la présence des 5 constantes spécifiques à SHA1  
(RFC 3174, page 7).

Le bloc contenant ces trois étapes effectue les actions suivantes :

- Calcul de Buf1 = sha1(argument 1)
- Vérification que Buf1 est équivalent à Buf2

Pour valider cette étape, il est nécessaire d’utiliser un langage de programmation tel que Python.

Nous souhaitons générer deux chaînes de caractères : arg1 et arg2.

Conditions :
- 4 < len(arg1) < 15  
- len(arg2) = 40  
- XOR_5_Fois(arg2) commence par 13 37  
- sha1(arg1) == XOR_5_Fois(arg2)

Nous devons donc trouver un sha1(X) commençant par 13 37.

```python 
import hashlib 
import threading 
import sys

def shal(clair):
    # fonction de hashage
    return hashlib.shal(clair.encode()).hexdigest()

alphabet = "abcdefghijklmnopqrstuvwxyz0123456789"

def increment_string(s):
    # générateur de mot de passe
    s = list(s)
        for i in range(len(s) - 1, -1, -1):
        pos = alphabet.index(s[i])
        if pos < len(alphabet) - 1:
            s[li] = alphabet[pos + 11]
            return "". join(s)
        s[i] = alphabet[0]
    return None

password = "aaaaaaaaaa" #valeur de départ

while True:
    hasher = shal(password)
    if hasher.startswith("1337"): #si hash commence par 1337
        print("Mot de passe :", password)
        print( "SHAl :", hasher)
        break
    password = increment_string (password)
```

Le programme nous retourne :

Mot de passe : aaaaaaebf9  
SHA1 : 1337d8bbfc3d9cc7ee17587bb4abf84d1f015537  

Nous avons donc pour valeur de l’argument 1 : aaaaaaebf9

Pour calculer la valeur de l’argument 2, il faut inverser l’opération XOR_5_Fois sur le hash obtenu.

```python 
import codecs
final = [0x13,0x37,0xd8, Oxbb, Oxfc, Ox3d, 0x9c, Oxc7, Oxee, Ox17, 0x58, 0x7b, 0xb4, Oxab, 0xf8,0x4d,0x1f,0x01, 0x55, 0x37]
for i in final:
    res = hex(i ^ 0x26 ^ 0x13 ^ 0x6 ^ 0x37 ^ 0x45)
    print( res, end=" ")
```

Le programme retourne :

0x52 0x76 0x99 0xfa 0xbd 0x7c 0xdd 0x86 0xaf 0x56  
0x19 0x3a 0xf5 0xea 0xb9 0x0c 0x5e 0x40 0x14 0x76  

Nous obtenons alors pour l’argument 2 :  
527699fabd7cdd86af56193af5eab90c5e401476

![alt text](/images/first-keygen/image-11.png)

Nous arrivons maintenant au dernier bloc imposant une condition.

Dans ce bloc :
- Le programme calcule la somme des valeurs ASCII de l’argument 1
- Il récupère le dernier octet du sha1(argument 1)
- Il compare ces deux valeurs, qui doivent être équivalentes

Nous ajoutons donc cette condition dans notre script Python.

```python 
import hashlib 
import threading 
import sys

def shal(clair):
    # fonction de hashage
    return hashlib.shal(clair.encode()).hexdigest()

alphabet = "abcdefghijklmnopqrstuvwxyz0123456789"

def increment_string(s):
    # générateur de mot de passe
    s = list(s)
        for i in range(len(s) - 1, -1, -1):
        pos = alphabet.index(s[i])
        if pos < len(alphabet) - 1:
            s[li] = alphabet[pos + 11]
            return "". join(s)
        s[i] = alphabet[0]
    return None
def somme(password):
    return sum(ord(c) for c in password)

password = "aaaaaaaaaa" #valeur de départ

while True:
    hasher = shal(password)
    if hasher.startswith("1337"): #si hash commence par 1337
    hex_sum = somme(password) #calcul la somme
    hex_sum_str = format( hex_sum, 'X') # sans 0x
    last_octet = hex_sum_str[-2:].zfill(2)
        if last_octet == hasher[-2:]:
        print("Mot de passe:", password)
        print("SHAl :", hasher)
    break
    password = increment_string (password)
```

Nous obtenons :

Mot de passe : aaaaaaidmd  
SHA1 : 1337fddc0aa6beb5b71717ab28cf0b3156cbefe4  

En inversant de nouveau le XOR_5_Fois, nous obtenons :

5276bc9d4be7fff4f65656ea698e4a70178aaea5  

Ainsi :
- argument 1 = aaaaaaidmd  
- argument 2 = 5276bc9d4be7fff4f65656ea698e4a70178aaea5  

![alt text](/images/first-keygen/image-12.png)
