---
title: IA pour les rapports d'audit & pentest
date: 2026-01-25
categories: [Dev, Python]
tags: [python, ia, pentest]
---

# IA pour générer automatiquement des rapports de Pentest

## Pourquoi ce projet ?

Récemment, j’ai réalisé un pentest sur une application web et un client lourd (dans le cadre d’un exercice avec un client fictif). Après avoir identifié plusieurs vulnérabilités, j’ai dû rédiger un rapport complet. C’est là que l’idée m’est venue : **et si l’on automatisait complètement la génération de rapports de pentest grâce à l’intelligence artificielle ?**  

Les rapports d’audit demandent beaucoup de temps et d’attention, et chaque étape (collecte de données, organisation, rédaction et mise en forme) peut être automatisée avec des modèles d’IA. Le projet est donc né de cette volonté de **gagner du temps tout en garantissant la qualité et la confidentialité des informations**.

---

## Comment j’ai imaginé le projet

L’idée initiale était simple :

1. **Créer un template de projet** pour structurer les informations : description de l’application, scope, screenshots, vulnérabilités, etc.  
2. **Parser toutes les données** collectées durant le pentest.  
3. **Appeler différents modèles d’IA** pour générer les descriptions de vulnérabilités, les recommandations, et même intégrer les images pertinentes.  
4. **Assembler toutes les informations** dans un rapport final au format Markdown et PDF.  

L’objectif : un flux automatisé où l’utilisateur fournit les données brutes, et le système sort un rapport prêt à l’emploi, personnalisable si nécessaire.

---

## Architecture du projet

Voici le workflow global du projet :  

![Pipeline IA pour rapport de Pentest](/images/AI-pentest/pipeline.png)

---

## Comment j’ai fait

Pour respecter la confidentialité et permettre une utilisation sur Mac (sans GPU puissant), j’ai choisi d’utiliser **Ollama** comme orchestrateur pour les modèles d’IA. Voici la stack utilisée :  

- **Python 3.10+**  
- **Ollama** pour gérer et exécuter les modèles localement  
- **Modèles IA utilisés** :  
  - `gpt-oss` pour le texte général  
  - `functiongemma` pour le traitement d’images et textes combinés  
  - `deepseek-ocr` pour extraire le texte depuis les captures d’écran  
- **Librairies Python** : Pandas, Markdown, PDF generation, etc.  

Le workflow concret :  

1. Lancer le serveur Ollama : `ollama serve`  
2. Créer le template projet : `python3 main.py -create`  
3. Générer le rapport depuis les données : `python3 main.py -generate projects/_customerName/_appName/`  
4. Modifier le Markdown si nécessaire et regénérer le PDF : `python3 main.py -generate_from_md projects/_customerName/_appName/`  

Chaque vulnérabilité est stockée dans un sous-dossier avec description et captures d’écran, ce qui permet à l’IA de les traiter et de les intégrer automatiquement dans le rapport final.  

---

## Points d’attention et améliorations futures

- **Performance sur Mac** : sans GPU, certains modèles sont un peu lents. Le projet reste cependant utilisable et fonctionnel.  
- **Personnalisation des rapports** : via `conf/agent.py` pour les prompts et `conf/style.css` pour le style du PDF.  
- **Extension IA** : possibilité d’intégrer `llama3.2-vision` ou `gemma3:4b` pour mieux gérer les images et les graphiques dans les rapports.  

---

## Conclusion

Ce projet est un **exemple concret d’automatisation d’un processus complexe grâce à l’IA**. Il permet non seulement de gagner un temps précieux, mais aussi de créer des rapports plus cohérents et visuellement structurés. L’IA ne remplace pas l’auditeur, mais elle devient un assistant puissant pour toutes les tâches répétitives et de mise en forme.  

Pour explorer le projet, le code est disponible sur GitHub : [AI-audit-report](https://github.com/julien-lair/AI-audit-report/).
