```markdown

---
title: "TD5 - CI/CD, OpenTofu, AWS, OIDC & GitHub Actions"
description: "Mise en œuvre d'une chaîne CI/CD complète avec GitHub Actions"
tags:
  - devops
  - AWS
  - GitHub Actions
  - CI/CD
  - OpenTofu
  - td1
---

## 1. Introduction

Ce rapport documente la réalisation du **Lab 5**, dont l'objectif principal était de maîtriser les pratiques modernes de CI/CD (Continuous Integration / Continuous Delivery) dans un contexte cloud. Le laboratoire s'est concentré sur cinq piliers fondamentaux :
1. **Intégration Continue (CI)** avec GitHub Actions et tests automatisés.
2. **Authentification sécurisée** avec AWS via OpenID Connect (OIDC).
3. **Infrastructure as Code (IaC)** avec OpenTofu et modules réutilisables.
4. **Tests d'infrastructure** automatisés pour valider les déploiements.
5. **Stratégies de déploiement** et concepts GitOps.

## 2. Intégration Continue (CI) avec GitHub Actions

Cette section visait à automatiser l'exécution des tests à chaque modification du code, garantissant la qualité et la stabilité de l'application.

### 2.1 Préparation du Projet

Nous avons préparé l'environnement de travail en créant une nouvelle structure pour le Lab 5 :
* **Création du dossier td5** : `mkdir -p td5/scripts`
* **Copie de l'application** : `cp -r td4/scripts/sample-app td5/scripts/sample-app`
* Cette duplication permet d'isoler les expérimentations du Lab 5 tout en conservant la base de travail du Lab 4.

### 2.2 Configuration du Workflow GitHub Actions

Un workflow GitHub Actions a été créé pour automatiser les tests de l'application Node.js. Le fichier `.github/workflows/app-tests.yml` définit :

```yaml
name: Sample App Tests
on: push
jobs:
  sample_app_tests:
    name: "Run Tests Using Jest"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install dependencies
        working-directory: td5/scripts/sample-app
        run: npm install
      - name: Run tests
        working-directory: td5/scripts/sample-app
        run: npm test
```

**Explication du workflow :**
* **`name`** : Nom du workflow visible dans l'onglet Actions de GitHub.
* **`on: push`** : Déclenchement automatique à chaque push sur n'importe quelle branche.
* **`runs-on: ubuntu-latest`** : Environnement d'exécution (runner Ubuntu).
* **`actions/checkout@v3`** : Action officielle pour récupérer le code source du dépôt.
* **`working-directory`** : Spécifie le répertoire de travail pour chaque étape.
* **`npm install`** : Installation des dépendances définies dans `package.json`.
* **`npm test`** : Exécution des tests Jest configurés dans le projet.

### 2.3 Test du Workflow avec Erreur Intentionnelle

Pour valider le bon fonctionnement de la CI, nous avons introduit une erreur intentionnelle :
* **Création d'une branche de test** : `git checkout -b test-workflow`
* **Modification du code** : Changement du message de réponse dans `app.js` de `"Hello, World!"` à `"DevOps Labs!"`
* **Commit et push** : Les modifications ont été poussées sur la branche `test-workflow`
* **Observation du résultat** : Le workflow s'est exécuté automatiquement et a **échoué** car le test attendait toujours `"Hello, World!"`
* **Correction** : Mise à jour du test dans `app.test.js` pour correspondre au nouveau message
* **Validation** : Après correction, le workflow s'est réexécuté et a **réussi**, confirmant que la CI fonctionne correctement

Cette approche démontre l'importance d'une **self-testing build** : chaque commit déclenche automatiquement les tests, garantissant que seul le code validé est intégré.

### 2.4 Principes de l'Intégration Continue

Nous avons appliqué les principes fondamentaux de la CI :
* **Trunk-based development** : Développement sur la branche principale avec des branches de fonctionnalité courtes.
* **Self-testing build** : Tests automatisés exécutés après chaque commit.
* **CI server** : GitHub Actions automatise le processus de build et de test.

## 3. Authentification Sécurisée avec AWS via OIDC

Cette section visait à sécuriser l'accès à AWS sans utiliser de clés d'accès statiques, en exploitant OpenID Connect (OIDC).

### 3.1 Problématique des Credentials

Pour exécuter des tests d'infrastructure qui déploient des ressources AWS, une authentification est nécessaire. Deux approches existent :

**Machine User Credentials** :
* Compte utilisateur dédié à l'automatisation avec permissions limitées.
* **Inconvénients** : Gestion manuelle des credentials, credentials à longue durée de vie, risque de compromission.

**Automatically-Provisioned Credentials (OIDC)** :
* Credentials générées dynamiquement, souvent à courte durée de vie.
* **Avantages** : Plus sécurisé, pas de gestion manuelle de secrets, recommandé lorsque possible.

Nous avons choisi l'approche **OIDC** pour sa sécurité supérieure.

### 3.2 Configuration du Provider OIDC

Un module OpenTofu a été créé pour configurer GitHub comme provider OIDC dans AWS. Le module `github-aws-oidc` (`td5/scripts/tofu/modules/github-aws-oidc/main.tf`) définit :

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url                   = var.provider_url
  client_id_list        = ["sts.amazonaws.com"]
  thumbprint_list       = ["ffffffffffffffffffffffffffffffffffffffff"]
}
```

**Explication :**
* **`url`** : URL du provider OIDC GitHub (`https://token.actions.githubusercontent.com`)
* **`client_id_list`** : Audience autorisée (AWS STS)
* **`thumbprint_list`** : Empreinte du certificat GitHub (à mettre à jour avec la valeur réelle)

### 3.3 Création des Rôles IAM

Un module `gh-actions-iam-roles` a été développé pour créer des rôles IAM avec des permissions spécifiques pour les tests et déploiements. Le module configure :

**Trust Policy** :
```hcl
assume_role_policy = jsonencode({
  Version = "2012-10-17"
  Statement = [{
    Effect = "Allow"
    Principal = {
      Federated = var.oidc_provider_arn
    }
    Action = "sts:AssumeRoleWithWebIdentity"
    Condition = {
      StringEquals = {
        "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        "token.actions.githubusercontent.com:sub" = "repo:${var.github_repo}:ref:refs/heads/main"
      }
    }
  }]
})
```

Cette politique permet uniquement aux workflows GitHub Actions du dépôt spécifié d'assumer le rôle.

**Permissions Policy** :
Le rôle dispose de permissions pour :
* **S3** : Lecture/écriture sur le bucket de state OpenTofu (`s3:GetObject`, `s3:PutObject`, `s3:ListBucket`)
* **DynamoDB** : Accès à la table de verrouillage du state (`dynamodb:GetItem`, `dynamodb:PutItem`, `dynamodb:UpdateItem`)
* **Lambda** : Invocation et lecture des fonctions (`lambda:InvokeFunction`, `lambda:GetFunction`)

### 3.4 Déploiement de l'Infrastructure

La configuration complète est définie dans `td5/scripts/tofu/live/ci-cd-permissions/main.tf` :

```hcl
provider "aws" {
  region = "eu-west-3"
}

module "oidc_provider" {
  source = "github.com/Aaa2122/Lab4-Version-Control-Build-Systems-and-Automated-Testing//td5/scripts/tofu/modules/github-aws-oidc"
  provider_url = "https://token.actions.githubusercontent.com"
}

module "iam_roles" {
  source = "github.com/Aaa2122/Lab4-Version-Control-Build-Systems-and-Automated-Testing//td5/scripts/tofu/modules/gh-actions-iam-roles"
  name = "lambda-sample"
  oidc_provider_arn = module.oidc_provider.oidc_provider_arn
  enable_iam_role_for_testing = true
  enable_iam_role_for_plan = true
  enable_iam_role_for_apply = true
  github_repo = "Aaa2122/Lab4-Version-Control-Build-Systems-and-Automated-Testing"
  lambda_base_name = "lambda-sample"
  tofu_state_bucket = "lab5-tofu-state-bucket"
  tofu_state_dynamodb_table = "lab5-tofu-state-lock"
}
```

**Commandes de déploiement :**
```bash
cd td5/scripts/tofu/live/ci-cd-permissions
tofu init
tofu apply
```

Les outputs (`lambda_test_role_arn`, `lambda_deploy_plan_role_arn`, `lambda_deploy_apply_role_arn`) sont ensuite utilisés dans les secrets GitHub.

## 4. Tests d'Infrastructure Automatisés

Au-delà des tests applicatifs, nous avons automatisé les tests d'infrastructure pour valider les déploiements cloud.

### 4.1 Configuration du Workflow de Tests d'Infrastructure

Un workflow GitHub Actions dédié a été créé (`.github/workflows/infra-tests.yml`) pour exécuter les tests OpenTofu :

```yaml
name: Infra OpenTofu Tests
on:
  push:
    paths:
      - 'td5/scripts/tofu/live/ci-cd-permissions/**'
jobs:
  tofu_infra_tests:
    runs-on: ubuntu-latest
    env:
      AWS_REGION: eu-west-3
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
      - name: Install OpenTofu
        run: |
          curl -sSL https://github.com/opentofu/opentofu/releases/download/v1.7.0/tofu_1.7.0_linux_amd64.tar.gz | tar xz
          sudo mv tofu /usr/local/bin/
      - name: Set up AWS credentials with OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ${{ env.AWS_REGION }}
          role-to-assume: ${{ secrets.AWS_OIDC_ROLE_ARN }}
      - name: Initialize tofu
        working-directory: td5/scripts/tofu/live/ci-cd-permissions
        run: tofu init
      - name: Run tofu plan
        working-directory: td5/scripts/tofu/live/ci-cd-permissions
        run: tofu plan
      - name: Run tofu test
        working-directory: td4/scripts/tofu/live/lambda-sample
        run: tofu test
```

**Explication du workflow :**
* **`on: push: paths`** : Déclenchement uniquement lors de modifications dans le dossier d'infrastructure.
* **Installation d'OpenTofu** : Téléchargement et installation de la version 1.7.0.
* **Configuration AWS via OIDC** : Utilisation du secret `AWS_OIDC_ROLE_ARN` pour s'authentifier sans credentials statiques.
* **`tofu plan`** : Vérification des changements prévus sans application réelle.
* **`tofu test`** : Exécution des tests d'infrastructure définis dans `deploy.tftest.hcl`.

### 4.2 Configuration des Secrets GitHub

Pour que le workflow fonctionne, le secret `AWS_OIDC_ROLE_ARN` doit être configuré dans GitHub :
* **Settings > Secrets and variables > Actions > New repository secret**
* **Nom** : `AWS_OIDC_ROLE_ARN`
* **Valeur** : ARN du rôle IAM généré par `tofu apply` (exemple : `arn:aws:iam::123456789012:role/lambda-sample-test`)

### 4.3 Scénario de Test d'Infrastructure

Les tests d'infrastructure suivent un scénario défini dans `deploy.tftest.hcl` :
1. **Deploy** : Déploiement de l'infrastructure via `command = apply`
2. **Validate** : Exécution du module de test avec l'URL de sortie (`run.deploy.api_endpoint`)
3. **Assertions** : Vérification stricte des résultats (code HTTP 200, corps de réponse attendu)

```hcl
run "validate" {
  command = apply
  module {
    source = "../../modules/test-endpoint"
  }
  variables {
    endpoint = run.deploy.api_endpoint
  }
  assert {
    condition = data.http.test_endpoint.status_code == 200
    error_message = "Unexpected status code: ${data.http.test_endpoint.status_code}"
  }
  assert {
    condition = data.http.test_endpoint.response_body == "Hello, World!"
    error_message = "Unexpected response body: ${data.http.test_endpoint.response_body}"
  }
}
```

## 5. Organisation Modulaire de l'Infrastructure

Nous avons structuré l'infrastructure en modules réutilisables pour faciliter la maintenance et la réutilisation.

### 5.1 Structure des Modules

L'organisation suit une architecture modulaire :
* **`td5/scripts/tofu/modules/github-aws-oidc`** : Module pour créer le provider OIDC
* **`td5/scripts/tofu/modules/gh-actions-iam-roles`** : Module pour créer les rôles IAM avec policies
* **`td5/scripts/tofu/live/ci-cd-permissions`** : Configuration racine utilisant les modules

Cette séparation permet :
* **Réutilisabilité** : Les modules peuvent être utilisés dans d'autres projets
* **Maintenabilité** : Modifications isolées dans les modules
* **Testabilité** : Tests unitaires possibles sur chaque module

### 5.2 Variables et Outputs

Les modules exposent des variables configurables et des outputs pour l'intégration :
* **Variables** : `name`, `github_repo`, `tofu_state_bucket`, `tofu_state_dynamodb_table`, etc.
* **Outputs** : `oidc_provider_arn`, `lambda_test_role_arn`, etc.

Cette approche permet une configuration flexible selon l'environnement (dev, staging, prod).

## 6. Stratégies de Déploiement et GitOps

Le Lab 5 introduit également les concepts de stratégies de déploiement et de GitOps.

### 6.1 Stratégies de Déploiement

Différentes stratégies existent selon le type d'application :
* **Blue/Green** : Déploiement instantané avec basculement, adapté aux applications stateless
* **Canary** : Déploiement progressif sur un sous-ensemble d'instances
* **Progressive** : Déploiement progressif avec remplacement des instances
* **Feature Toggles** : Déploiement du code sans activer les fonctionnalités

### 6.2 Concept GitOps

GitOps est une approche où Git devient la source de vérité pour l'infrastructure et les applications. Les changements sont appliqués automatiquement via des outils comme Flux, garantissant que l'état réel correspond à l'état déclaré dans Git.

## 7. Conclusion et Retours d'Expérience

Ce laboratoire a permis de consolider une chaîne CI/CD moderne et sécurisée. Nous avons appris à :
* **Automatiser efficacement** les tests applicatifs avec GitHub Actions, garantissant la qualité du code à chaque commit.
* **Sécuriser les accès cloud** sans gestion de secrets statiques grâce à OIDC, réduisant les risques de compromission.
* **Structurer l'infrastructure** en modules réutilisables avec OpenTofu, facilitant la maintenance et l'évolution.
* **Valider les déploiements** avec des tests d'infrastructure automatisés, assurant la fiabilité des ressources cloud.

Ces pratiques garantissent une meilleure qualité logicielle, une sécurité renforcée, et réduisent significativement les risques lors des déploiements en production. L'automatisation complète du cycle de vie (code → test → déploiement) permet des livraisons fréquentes et fiables, essentielles dans un contexte DevOps moderne.

**Cette expérience démontre l'importance cruciale des pipelines automatisés, de l'IaC, et de la sécurité dans les pratiques DevOps contemporaines.**
