---
title: "TD1 - Déploiement d'applications"
description: "Déploiement d'une application Hello World avec PaaS et IaaS"
tags:
  - devops
  - deployment
  - paas
  - iaas
  - td1
---

## 1. Introduction

Ce rapport présente le **TD1** sur le déploiement d'applications. L'objectif était de découvrir deux approches différentes pour déployer une application : le PaaS (Platform as a Service) et l'IaaS (Infrastructure as a Service). Nous avons travaillé sur :
1. **Déploiement avec PaaS** : Utilisation de Render pour déployer rapidement une application Hello World.
2. **Déploiement avec IaaS** : Configuration d'une instance EC2 sur AWS et déploiement manuel de l'application.

## 2. Déployer une application avec un PaaS : Render

J'ai connecté mon compte GitHub à Render et je m'apprête à créer le web service dans un tier gratuit pour mon application Hello World.

![Dashboard Render](devops/images/td1/1.png)

### Déploiement

Voilà l'application déployée, après quelque temps, voici les logs :

![Logs de déploiement](devops/images/td1/2.png)

### Résultat

L'application est maintenant accessible via l'URL fournie par Render

![Site déployé](devops/images/td1/3.png)

---

## Déployer une application avec IaaS : AWS

### Création d'un utilisateur IAM

Nous avons créé avec succès un utilisateur nommé `lucas` dont nous pourrions nous servir tout au long de ce TD.

![Utilisateur IAM créé](devops/images/td1/user_iam_created.png)

### Création et configuration d'une instance EC2

Ensuite, nous avons procédé à la création d'une instance EC2. Lors de la configuration de l'application, nous avons rencontré un problème, en essayant d'accéder à notre application via l'adresse IP publique fournie, celle ci ne répondait pas.

Intuitivement, nous avons consulté les logs de la console de l'instance pour comprendre ce qui se passait et il s'est avéré que le code fourni dans le TD contenait une erreur de syntaxe, les commentaires en JavaScript ne sont pas déterminés par les symboles `#`, donc l'application Node plantait en rencontrant un `#`.

![Code copié ne marche pas](devops/images/td1/code_copie_ne_marche_pas.png)

Nous avons donc arrêté l'exécution de l'instance pour modifier les données utilisateur (`user data`), qui contiennent dans ce TD le code de l'application complète. Après avoir corrigé le problème de syntaxe dans le code JavaScript, nous avons relancé l'instance. Cependant, cela ne fonctionnait toujours pas, ce qui est un comportement normal et attendu.

En effet, nous avons pu comprendre que les données utilisateur ne sont exécutées qu'au premier démarrage de l'instance, cela signifie donc que si nous souhaitons que l'application JavaScript corrigée soit accessible depuis cette instance, nous devrions recréer une nouvelle instance (avec les données utilisateur contenant le code corrigé).

Dans AWS, les logs sont bien plus bruts et techniques, donc ils offrent plus de précision et d'informations détaillées sur ce qui se passe réellement derrière les instances.

Finalement, nous avons stoppé l'instance pour conclure ce TD.

![Instance EC2 stoppée](devops/images/td1/instance_stoppe.png)

---

## Conclusion

Ce TD nous a permis de comparer deux approches de déploiement : le PaaS avec Render et l'IaaS avec AWS. 

Avec Render, le déploiement est très simple et rapide. Il suffit de connecter le dépôt GitHub et l'application se déploie automatiquement. C'est pratique pour démarrer rapidement, mais on a moins de contrôle sur l'infrastructure.

Avec AWS EC2, c'est plus complexe mais on a un contrôle total. On doit gérer l'instance, les logs, et comprendre comment fonctionnent les données utilisateur. C'est plus technique mais ça permet de mieux comprendre ce qui se passe sous le capot.

Les deux approches ont leurs avantages selon le contexte et les besoins du projet.
