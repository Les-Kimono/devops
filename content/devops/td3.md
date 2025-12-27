---
title: "TD3 - Comment déployer vos applications"
description: "Exploration des paradigmes DevOps sur AWS : IaC avec Ansible et OpenTofu, conteneurisation via Kubernetes et déploiement Serverless avec Lambda."
tags:
  - devops
  - aws
  - iac
  - ansible
  - packer
  - opentofu
  - kubernetes
  - docker
  - td3
---

## 1. Introduction
Ce rapport documente la réalisation du **TD 3**, dont l'objectif était de maîtriser la chaîne complète de déploiement d'une application moderne, en passant d'une gestion d'infrastructure classique à des architectures cloud-natives. Nous avons exploré une progression logique des abstractions :

1.  **Gestion de configuration** avec Ansible.
2.  **Infrastructure immuable** avec Packer et OpenTofu.
3.  **Orchestration de conteneurs** avec Docker et Kubernetes.
4.  **Architecture Serverless** avec AWS Lambda.

## 2. Orchestration de serveurs avec Ansible

Dans cette partie, nous avons abandonné les scripts Bash manuels ("User Data") au profit d'**Ansible**, un outil de gestion de configuration déclaratif et sans agent.

### 2.1 Objectif
Configurer une machine virtuelle vierge (EC2) pour qu'elle puisse exécuter notre application Docker, de manière répétable et idempotente.

### 2.2 Réalisation
Nous avons créé un **Playbook** (`playbook.yml`) qui décrit l'état désiré du serveur :
1.  Mise à jour des dépôts de paquets.
2.  Installation du moteur Docker.
3.  Démarrage du conteneur de l'application (Nginx/WebApp) sur le port 80.

**Vérification des instances sur AWS :**
Les instances EC2 ont été correctement provisionnées et tagguées via Ansible, comme visible dans la console AWS :

![Instances Running dans la Console AWS](devops/images/td3/Ansible_instances_running.png)

Nous avons ensuite récupéré les adresses IP publiques dynamiques via l'inventaire dynamique ou le CLI :

![Récupération des IPs des instances](devops/images/td3/Instances_IP.png)

### 2.3 Résultat
En exécutant la commande `ansible-playbook`, nous pouvons configurer n'importe quelle machine accessible via SSH en quelques secondes. Cependant, cette méthode configure la machine *après* son démarrage (Runtime configuration), ce qui peut être lent lors d'un scaling horizontal.

![Test Curl sur l'instance applicative](devops/images/td3/Instance_success.png)

![Test Curl sur l'instance Nginx](devops/images/td3/nginx_success.png)

Nous avons également validé que la mise à jour de l'application (version "DevOps Base") était bien prise en compte sur les serveurs existants :

![Validation de la mise à jour application via IP](devops/images/td3/rolling_updates_success.png)

## 3. Orchestration de VM avec Packer et OpenTofu

Pour pallier la lenteur de configuration au démarrage, nous sommes passés à une stratégie d'**Infrastructure Immuable**. Cette partie combine deux outils complémentaires : Packer pour l'image et OpenTofu pour l'infrastructure.

### 3.1 Création d'image (AMI) avec Packer
Au lieu de configurer un serveur vivant, nous avons utilisé **Packer** pour construire une image disque (AMI AWS) qui contient *déjà* notre application configurée.
* **Provisioning :** Packer lance une instance temporaire et utilise notre playbook Ansible (de la partie 2) pour y installer Docker.
* **Artifact :** Le résultat est une "Golden Image" prête à l'emploi.

**Preuve de la construction de l'AMI :**
Packer a réussi à provisionner l'instance et à créer l'AMI dans la région `us-east-2` :

![Succès du build Packer et création de l'AMI](devops/images/td3/Packer_build_success.png)

### 3.2 Provisionnement avec OpenTofu
Une fois l'AMI créée, nous avons utilisé **OpenTofu** (fork open-source de Terraform) pour décrire notre infrastructure AWS sous forme de code (HCL).
* **Définition des ressources :** Le fichier `main.tf` déclare l'instance EC2, le groupe de sécurité (pare-feu) et la clé SSH.
* **Déploiement :** OpenTofu utilise l'ID de l'AMI générée par Packer pour lancer l'instance.

**Application de l'infrastructure :**
La commande `tofu apply` a créé l'ensemble des 10 ressources demandées sans erreur :

![Résultat du Tofu Apply](devops/images/td3/tofu_apply_success.png)

### 3.3 Bilan
Nous avons maintenant une infrastructure "élastique" : nous pouvons détruire et recréer notre environnement complet (Réseau + VM + App) avec une seule commande (`tofu apply`), et l'application est disponible immédiatement au boot.

![Test Curl sur le Load Balancer OpenTofu](devops/images/td3/tofu_alb_success.png)

Nous avons également testé la capacité de "Rolling Update" (mise à jour sans interruption) en changeant la version de l'application. La boucle de requêtes ci-dessous montre la transition fluide entre les deux versions :

![Test du Rolling Update](devops/images/td3/tofu_rolling_update_success.png)

## 4. Orchestration de conteneurs avec Docker et Kubernetes

Après avoir automatisé la création de machines virtuelles (Packer/OpenTofu), nous sommes passés à une granularité plus fine : le conteneur. Cette dernière partie vise à découpler totalement l'application de l'infrastructure sous-jacente.

### 4.1 Dockerisation et Registre
Bien que nous ayons utilisé Docker dans les étapes précédentes, nous avons ici formalisé le cycle de vie de l'image :
1.  **Build :** Construction de l'image de l'application via un `Dockerfile` optimisé.
2.  **Push :** Envoi de l'image vers un registre d'images (Docker Hub ou AWS ECR) pour la rendre accessible par le cluster.

![Build et Test Local Docker](devops/images/td3/docker_build_success.png)

### 4.2 Déploiement sur Kubernetes
Nous avons utilisé Kubernetes (K8s) pour orchestrer nos conteneurs, remplaçant la gestion manuelle des conteneurs sur des VMs.

**Manifestes utilisés :**
Nous avons défini l'état désiré de l'application via des fichiers YAML :
* **Deployment :** Définit quelle image utiliser et le nombre de répliques (Pods) souhaitées pour assurer la haute disponibilité.
* **Service :** Expose l'application (LoadBalancer ou NodePort) pour la rendre accessible de l'extérieur tout en gérant la répartition de charge entre les Pods.

**Exemple de définition (Deployment) :**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: mon-user/mon-app:latest
        ports:
        - containerPort: 80
```

Etat des pods :

![Etats des pods](devops/images/td3/kubectl_pods_running_success.png)

Résultats avec rolling update : 

![Succès du Rolling Update avec kubectl](devops/images/td3/kubectl_rolling_update_success.png)

### 4.3 Avantages de cette architecture
- **Auto-guérison (Self-healing)** : Si un Pod crashe, Kubernetes le redémarre automatiquement.

- **Scalabilité** : Augmenter le nombre d'instances de l'application se fait instantanément en modifiant le nombre de répliques, sans avoir à provisionner de nouvelles VMs lourdes.

- **Abstraction** : L'application ne se soucie plus du serveur sur lequel elle tourne.

## 5. Serverless Orchestration with AWS Lambda

Après avoir exploré le déploiement sur machines virtuelles (EC2), nous avons abordé une approche radicalement différente : le **Serverless** (FaaS - Function as a Service).

L'objectif de cette partie était de déployer une application sans provisionner le moindre serveur, en utilisant **AWS Lambda** couplé à **API Gateway**, le tout orchestré par **OpenTofu**.

### 5.1 Architecture et Concepts

Contrairement aux parties précédentes où nous payions pour des serveurs qui tournent 24/7 (même inactifs), ici nous utilisons une architecture événementielle :
1.  **API Gateway :** Reçoit la requête HTTP de l'utilisateur (le déclencheur).
2.  **AWS Lambda :** Un conteneur éphémère démarre, exécute notre code (Node.js), renvoie la réponse, et s'éteint.

**Avantage majeur :** Nous ne payons que pour les millisecondes d'exécution (Compute time) et n'avons aucune maintenance d'OS à faire.

#### 5.2 Infrastructure as Code avec OpenTofu

Pour déployer cette fonction, nous avons dû adapter notre code HCL (main.tf). Le défi principal est que Lambda a besoin d'un fichier .zip du code.

Points clés de la configuration OpenTofu :

- **Zippage automatique** : Nous avons utilisé le provider archive pour créer le zip automatiquement avant le déploiement.
```hcl
data "archive_file" "zip" {
  type        = "zip"
  source_file = "index.js"
  output_path = "lambda_function.zip"
}
```
- **Rôle IAM** : Contrairement à une VM où l'on se connecte en SSH, la Lambda a besoin d'un rôle d'exécution (IAM Role) pour avoir le droit de s'exécuter et d'écrire des logs dans CloudWatch.
- **Définition de la Ressource Lambda** :
```hcl
resource "aws_lambda_function" "terraform_lambda" {
  filename         = "lambda_function.zip"
  function_name    = "my-td3-lambda"
  role             = aws_iam_role.iam_for_lambda.arn
  handler          = "index.handler"
  runtime          = "nodejs18.x"
  source_code_hash = data.archive_file.zip.output_base64sha256
}
```

### 5.3 Déploiement et Test
Nous avons suivi le workflow standard OpenTofu :
- tofu init : Téléchargement du provider AWS et Archive.
- tofu apply : Création de l'API Gateway et de la Lambda.

![Succès fonction lambda](devops/images/td3/lambda_function_success.png)

## 6. Conclusion

Ce laboratoire a mis en évidence la progression des paradigmes DevOps :

* **Ansible** est idéal pour la gestion fine de la configuration des OS.
* **Packer & OpenTofu** permettent une infrastructure immuable et robuste pour les VMs.
* **Kubernetes** s'impose pour la résilience et l'orchestration complexe de micro-services.
* **Serverless (Lambda)** offre l'abstraction ultime, permettant aux développeurs de se concentrer uniquement sur le code métier en déléguant toute l'infrastructure au Cloud Provider.