---
title: Automatiser les rapports de Pentest grâce à l’IA
date: 2026-01-25
categories: [Dev, Python]
tags: [python, ia, pentest]
---

# Automatiser les rapports de Pentest grâce à l’IA

Récemment, j’ai eu l’occasion de réaliser un pentest sur une application web et un client lourd, dans le cadre d’un exercice avec un client fictif. Après avoir identifié plusieurs vulnérabilités, je me suis retrouvé face à la partie la plus fastidieuse : **rédiger le rapport d’audit complet**.  

C’est là que l’idée m’est venue : et si je pouvais utiliser l’intelligence artificielle pour automatiser l’ensemble du processus ? De la collecte des informations à la mise en page finale, tout en conservant la confidentialité des données.

---

## L’idée derrière le projet

L’objectif était simple : créer un outil capable de transformer des données brutes issues d’un pentest en un rapport professionnel, prêt à être remis à un client.  

Pour y arriver, j’ai imaginé un flux en plusieurs étapes :  

1. **Structurer les informations** dans un template projet : description de l’application, scope, captures d’écran, vulnérabilités…  
2. **Parser les données collectées** durant le pentest.  
3. **Exploiter différents modèles d’IA** pour générer automatiquement les descriptions de vulnérabilités, les recommandations et intégrer les images.  
4. **Assembler le tout** dans un rapport final en Markdown et PDF, modifiable si besoin.  

L’objectif : passer de la saisie manuelle et répétitive à un processus automatisé et fiable.

---

## Architecture du projet

Le pipeline que j’ai mis en place suit ce schéma :  

![Pipeline IA pour rapport de Pentest](/images/AI-pentest/pipeline.png)

Concrètement, l’utilisateur fournit les données brutes du pentest et le programme s’occupe du reste : parsing, génération de texte, intégration des images et export en PDF.

---

## Technologies et choix techniques

Pour ce projet, il était essentiel de **travailler localement**, sans dépendre d’un service cloud, afin de garantir la confidentialité des informations sensibles. J’ai donc choisi Ollama comme orchestrateur pour mes modèles d’IA.  

La stack utilisée :  

- **Python 3.10+**  
- **Ollama** pour gérer les modèles localement  
- **Modèles IA** :  
  - `gpt-oss` pour la génération de texte  
  - `functiongemma` pour le traitement combiné texte + image  
  - `deepseek-ocr` pour extraire du texte depuis les captures d’écran  
- Librairies Python pour manipuler Markdown et générer le PDF  

Le workflow d’utilisation :  

1. Lancer le serveur Ollama : `ollama serve`  
2. Créer le template projet : `python3 main.py -create`  
3. Générer le rapport à partir des données : `python3 main.py -generate projects/_customerName/_appName/`  
4. Modifier le Markdown si nécessaire et regénérer le PDF : `python3 main.py -generate_from_md projects/_customerName/_appName/`  

Chaque vulnérabilité dispose d’un sous-dossier contenant la description, l’exploitation et les captures d’écran, ce qui permet à l’IA de les traiter et de produire un rapport complet automatiquement.

---

## Challenges rencontrés

Travailler sur Mac sans GPU puissant a imposé certaines contraintes : les modèles étaient plus lents et certains traitements d’images prenaient du temps. Cependant, l’utilisation d’Ollama permet de gérer ces limitations et de rester fonctionnel même sans carte graphique.  

Le projet reste très flexible :  

- Les prompts peuvent être personnalisés via `conf/agent.py` pour améliorer les résultats.  
- Le style du PDF peut être modifié via `conf/style.css`.  
- À terme, il sera possible d’intégrer des modèles plus avancés (`llama3.2-vision`, `gemma3:4b`) pour traiter les images et générer des graphiques plus complexes.

---

## Ce que j’en retire

Ce projet est un excellent exemple de **comment l’IA peut assister un expert en sécurité** :  

- Gain de temps considérable  
- Rapport plus structuré et cohérent  
- Possibilité de personnaliser et d’adapter selon le client ou l’exercice  

L’IA n’a pas remplacé le travail d’audit, mais elle a été un assistant puissant pour automatiser les tâches répétitives et générer un produit final professionnel.  

Le code est disponible sur GitHub si vous voulez explorer ou tester : [AI-audit-report](https://github.com/julien-lair/AI-audit-report/).

