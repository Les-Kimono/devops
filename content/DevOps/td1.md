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

## Déployer une application avec un PaaS : Render


J'ai connecté mon compte GitHub à Render et je m'apprête à créer le web service dans un tier gratuit pour mon application Hello World.

![Dashboard Render](images/devops/td1/1.png)


### Déploiement

Voilà l'application déployée, après quelque temps, voici les logs :

![Logs de déploiement](images/devops/td1/2.png)

### Résultat

L'application est maintenant accessible via l'URL fournie par Render

![Site déployé](images/devops/td1/3.png)

---

## Déployer une application avec IaaS : AWS

### Création d'un utilisateur IAM

J'ai créé avec succès un utilisateur nommé `lucas` dont je pourrais me servir tout au long de ce TD.

![Utilisateur IAM créé](images/devops/td1/user_iam_created.png)

### Création et configuration d'une instance EC2

Ensuite, j'ai procédé à la création d'une instance EC2. Lors de la configuration de l'application, j'ai rencontré un problème, en essayant d'accéder à mon application via l'adresse IP publique fournie, celle ci ne répondait pas.

Intuitivement, j'ai consulté les logs de la console de l'instance pour comprendre ce qui se passait et il s'est avéré que le code fourni dans le TD contenait une erreur de syntaxe, les commentaires en JavaScript ne sont pas déterminés par les symboles `#`, donc l'application Node plantait en rencontrant un `#`.

![Code copié ne marche pas](images/devops/td1/code_copie_ne_marche_pas.png)

J'ai donc arrêté l'exécution de l'instance pour modifier les données utilisateur (`user data`), qui contiennent dans ce TD le code de l'application complète. Après avoir corrigé le problème de syntaxe dans le code JavaScript, j'ai relancé l'instance. Cependant, cela ne fonctionnait toujours pas, ce qui est un comportement normal et attendu.

En effet, j'ai pu comprendre que les données utilisateur ne sont exécutées qu'au premier démarrage de l'instance, cela signifie donc que si je souhaite que l'application JavaScript corrigée soit accessible depuis cette instance, je devrais recréer une nouvelle instance (avec les données utilisateur contenant le code corrigé). 

Dans AWS, les logs sont bien plus bruts et techniques, donc ils offrent plus de précision et d'informations détaillées sur ce qui se passe réellement derrière les instances.

Finalement, j'ai stoppé l'instance pour conclure ce TD.
![Instance EC2 stoppée](images/devops/td1/instance_stoppe.png)