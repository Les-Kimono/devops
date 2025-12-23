---
title: "TD2 - Gestion de l'IAC (Infrastructure as Code)"
description: "Déploiement d'infrastructure AWS via Bash, Ansible, Packer et OpenTofu"
tags:
  - devops
  - aws
  - iac
  - ansible
  - packer
  - opentofu
  - td2
---

## 1. Introduction

Ce rapport documente la réalisation du **Lab 2**, dont l'objectif principal était de comprendre les différentes approches de la gestion d'infrastructure (Infrastructure as Code). Nous avons exploré l'évolution des pratiques de déploiement, en passant par :
1.  **Scripting impératif** avec Bash.
2.  **Gestion de configuration** avec Ansible.
3.  **Infrastructure immuable** avec Packer.
4.  **Provisioning déclaratif** avec OpenTofu (successeur open-source de Terraform).

## 2. Authentification et Déploiement par Script (Bash)

La première étape consistait à configurer l'accès à AWS via la CLI et à tenter un déploiement manuel scripté.

### 2.1 Configuration AWS
Nous avons généré des clés d'accès (Access Key ID et Secret Access Key) dans la console IAM d'AWS pour permettre l'interaction programmatique.

![Clé d'accès](devops/images/td2/access_key_aws.png)

### 2.2 Déploiement Bash et Limites
Nous avons exécuté le script `./deploy-ec2-instance.sh` qui utilise AWS CLI pour créer un Security Group et lancer une instance EC2.
* **Résultat :** L'instance a été créée avec succès (`i-08f89eeae23714e07`) et l'application "Hello, World!" était accessible via l'IP publique.

![Succès déploiement](devops/images/td2/deploy_script_success.png)

### 2.3 Problème d'Idempotence (Exercice 1)
Lors de la seconde exécution du script, nous avons rencontré l'erreur suivante :
`An error occurred (InvalidGroup.Duplicate) when calling the CreateSecurityGroup operation`

**Explication :** Le script Bash est **impératif**. Il tente de créer le Security Group "sample-app" sans vérifier s'il existe déjà. Comme AWS impose des noms uniques au sein d'un VPC, la commande échoue. Cela démontre que les scripts simples manquent souvent d'idempotence.

### 2.4 Gestion de Multiples Instances (Exercice 2)
Pour déployer plusieurs instances, nous avons dû modifier le script en ajoutant le paramètre `--count` à la commande `run-instances` :

```bash
instance_id=$(aws ec2 run-instances \
    --image-id "ami-0900fe555666598a2" \
    --instance-type "t3.micro" \
    --count 2 \
    ...
)
```

## 3. Gestion de Configuration avec Ansible

Nous avons ensuite utilisé Ansible pour effectuer le même déploiement, mais de manière plus robuste.

Résultat :

![Succès Ansible](devops/images/td2/ansible_instance_success.png)

En allant sur le lien http://18.219.176.140:8080 on a bien le message "Hello World!"

### 3.1 Idempotence Native (Exercice 3)

Contrairement au script Bash, la ré-exécution du playbook Ansible a donné le résultat suivant :

![Succès Exercice 3](devops/images/td2/exo3_success.png)

**Explication :** Ansible est **déclaratif** et **idempotent**. Il vérifie l'état actuel de l'infrastructure. Si Node.js est déjà installé et que l'instance tourne, il ne fait rien. Le statut `ok` indique que la configuration souhaitée est déjà en place.

### 3.2 Adaptation pour Multiples Instances (Exercice 4)

Pour déployer 2 instances avec Ansible, nous avons modifié le module `amazon.aws.ec2_instance` dans le playbook en utilisant le paramètre `exact_count` :

```yaml
- name: Create EC2 instance
  amazon.aws.ec2_instance:
    name: sample-app-ansible
    exact_count: 2
    instance_type: t3.micro
    # ...
```

Résultat :

![Succès Exercice 4](devops/images/td2/exo4_success.png)

## 4. Création d'Images Immuables avec Packer

Pour éviter de configurer le serveur à chaque démarrage (comme fait précédemment avec `user-data`), nous avons utilisé Packer pour créer une **AMI (Amazon Machine Image)** pré-configurée.

### 4.1 Configuration du Build

Nous avons adapté le fichier Packer pour :
* Utiliser une instance `t3.micro`.
* Installer Node.js 16.
* Générer un nom unique via `uuidv4()`.

Résultat :

![Succès Packer](devops/images/td2/packer_success.png)

### 4.2 Unicité des Builds (Exercice 5)

L'exécution multiple de `packer build` fonctionne sans erreur, créant à chaque fois une nouvelle AMI.
**Explication :** Grâce à la fonction `ami_name = "sample-app-packer-${uuidv4()}"`, chaque image possède un nom unique sur AWS, évitant les conflits de nommage qui interdisent deux AMIs identiques.

### 4.3 Packer pour d'autres fournisseurs (Exercice 6)

Packer est agnostique au cloud. Pour créer une image locale (ex: VirtualBox), on ajouterait une source `virtualbox-iso` :

```hcl
source "virtualbox-iso" "local_vm" {
  vm_name = "sample-app-vm"
  iso_url = "https://releases.ubuntu.com/20.04/ubuntu-20.04.6-live-server-amd64.iso"
  cpus    = 2
  memory  = 2048
}
```

## 5. Orchestration avec OpenTofu

Enfin, nous avons utilisé OpenTofu pour déployer l'AMI créée par Packer.

Résultat : 

![Succès OpenTofu 1](devops/images/td2/opentofu_success_1.png)

### 5.1 Cycle de Vie (Exercice 7)

Si l'on exécute `tofu apply` après un `tofu destroy`, OpenTofu recrée intégralement les ressources.

**Explication :** OpenTofu compare l'état désiré (fichiers `.tf`) avec l'état réel (State). Si les ressources n'existent plus, son rôle est de les ramener à l'existence pour correspondre à la configuration.

### 5.2 Boucles et Variables (Exercice 8)

Pour déployer plusieurs instances sans dupliquer le code, nous avons utilisé le méta-argument `count` :

```hcl
resource "aws_instance" "sample_app" {
  count = 2
  ami   = var.ami_id
  tags  = {
    Name = "sample-app-tofu-${count.index + 1}"
  }
}
```

## 6. Modularisation et Bonnes Pratiques

Pour rendre le code réutilisable, nous avons refactorisé la configuration en modules.

Résultat, 2 instances distinctes créées à partir du même module local :

![Succès OpenTofu 2](devops/images/td2/opentofu_success_2.png)

### 6.1 Paramétrage du Module (Exercice 9)

Nous avons rendu le module flexible en ajoutant des variables dans `variables.tf` (`instance_type`, `server_port`) et en les utilisant dans `main.tf` :

```hcl
resource "aws_security_group_rule" "allow_http_inbound" {
  type              = "ingress"
  security_group_id = aws_security_group.sample_app.id
  from_port         = var.server_port
  to_port           = var.server_port
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
}

resource "aws_instance" "sample_app" {
  instance_type = var.instance_type
  # ...
}
```

Résultat : 

![Succès Exercice 9](devops/images/td2/exo9_success.png)

### 6.2 Appel de Module avec Count (Exercice 10)
Dans le fichier racine (live/sample-app), nous avons instancié le module 2 fois et récupéré les IPs via une sortie groupée :

```hcl
module "sample_app" {
  source = "../../modules/ec2-instance"
  count  = 2
  name   = "sample-app-tofu-${count.index + 1}"
  # ...
}

output "public_ips" {
  value = module.sample_app[*].public_ip
}
```

Résultat :

![Succès Exercice 10](devops/images/td2/exo10_success.png)

## 7. Modules Distants et Versionning

### 7.1 Utilisation de GitHub

Nous avons configuré OpenTofu pour télécharger un module directement depuis un dépôt GitHub plutôt que localement.

OpenTofu utilise bien le module distant : 

![OpenTofu Module GitHub](devops/images/td2/opentofu_module_git.png)

Résultat : 

![Succès OpenTofu GitHub](devops/images/td2/opentofu_success_3.png)

### 7.2 Versionning (Exercice 11)

Pour garantir la stabilité de l'infrastructure de production, il est crucial de fixer les versions. Nous avons utilisé le paramètre `ref` pour cibler un tag Git précis :

```hcl
module "sample_app" {
  source = "github.com/AdrienB23/my-terraform-module?ref=v1.0.0"
  # ...
}
```
Résultat :

![Succès Exercice 11](devops/images/td2/exo11_success.png)

## 8. Conclusion

Ce laboratoire a mis en évidence la progression des outils DevOps :

* Les **scripts Bash** sont rapides mais fragiles (manque d'idempotence).
* **Ansible** apporte l'idempotence et la gestion de configuration.
* **Packer** permet de créer des serveurs immuables, réduisant les temps de démarrage et les erreurs de configuration ("Configuration Drift").
* **OpenTofu** offre une approche déclarative puissante pour gérer le cycle de vie de l'infrastructure, rendue modulaire et versionnée grâce à Git.