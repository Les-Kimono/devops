# Rapport de Lab 4 : Version Control, Build Systems, and Automated Testing

**Sujet :** Mise en œuvre d'une chaîne CI/CD complète (Git, NPM, Docker, Jest, OpenTofu)

## 1. Introduction

Ce rapport documente la réalisation du **Lab 4**, dont l'objectif principal était de maîtriser les outils et pratiques essentiels du DevOps moderne. Le laboratoire s'est concentré sur quatre piliers fondamentaux :
1.  **Gestion de version** avec Git et GitHub.
2.  **Systèmes de build** et gestion de dépendances via NPM.
3.  **Conteneurisation** d'une application via Docker.
4.  **Tests automatisés** pour l'application (Jest) et l'infrastructure (OpenTofu).


## 2. Gestion de Version (Git & GitHub)

Cette section visait à comprendre le cycle de vie des modifications de code et la collaboration en équipe.

### 2.1 Initialisation et Flux Local
Nous avons mis en place un dépôt Git local pour suivre les changements du projet :
* **Initialisation :** Création du dépôt via `git init`.
* **Suivi des modifications :** Utilisation de `git status` pour visualiser l'état des fichiers et `git add` pour les placer dans la zone de staging.
* **Commit :** Enregistrement des versions avec des messages clairs via `git commit -m "..."`.

### 2.2 Stratégie de Branches
Pour isoler les développements, nous avons adopté un flux basé sur les branches :
* Création d'une branche dédiée `testing` avec `git checkout -b testing`.
* Modification de fichiers (ex: `example.txt`) en isolation.
* Fusion des changements dans la branche principale `main` via `git merge testing`.

### 2.3 Collaboration Distante (GitHub)
Le dépôt a été connecté à GitHub pour simuler un environnement collaboratif :
* Ajout du dépôt distant : `git remote add origin ...`.
* Utilisation des **Pull Requests** pour proposer des changements (ex: branche `update-readme`) et les faire valider avant fusion.


## 3. Système de Build et Conteneurisation (NPM & Docker)

Nous avons structuré une application Node.js simple pour automatiser son cycle de vie.

### 3.1 Automatisation avec NPM
Le projet a été initialisé avec un fichier `package.json` via `npm init -y`. Des scripts personnalisés ont été définis pour standardiser les opérations :

```json
"scripts": {
  "start": "node server.js",
  "dockerize": "./build-docker-image.sh",
  "test": "jest --verbose"
}
```

### 3.2 Conteneurisation
Un `Dockerfile` a été créé pour encapsuler l'application, assurant la portabilité :
* **Base Image :** `node:21.7`.
* **Exposition :** Port 8080.
* **Commande :** `CMD ["npm", "start"]`.

Le script `build-docker-image.sh` automatise la construction via `docker buildx`, en taguant l'image dynamiquement avec le nom et la version extraits du `package.json`.


## 4. Tests Applicatifs Automatisés (Jest)

Pour garantir la qualité du code, nous avons intégré des tests unitaires et d'intégration.

### 4.1 Refactorisation du Code
L'application a été scindée pour améliorer la testabilité :
* **`app.js`** : Définit et exporte l'application Express sans démarrer le serveur (`module.exports = app`).
* **`server.js`** : Importe `app` et écoute sur le port 8080.

Cette séparation permet à la librairie de test de lancer l'application en mémoire sans conflit de port.

### 4.2 Exécution des Tests
Nous avons utilisé **Jest** et **SuperTest** pour valider les endpoints HTTP.
Exemple de test pour la route racine `/` dans `app.test.js` :

```javascript
describe('Test the root path', () => {
    test('It should respond to the GET method', async () => {
        const response = await request(app).get('/');
        expect(response.statusCode).toBe(200);
        expect(response.text).toBe('Hello, World!');
    });
});
```

Les tests sont exécutés via `npm test`, validant le bon fonctionnement de l'API avant tout déploiement.


## 5. Tests d'Infrastructure (OpenTofu)

Au-delà du code applicatif, nous avons appliqué les principes de test à l'infrastructure (IaC).

### 5.1 Configuration du Test
Un fichier de test `deploy.tftest.hcl` a été créé pour valider le déploiement. Il utilise un module `test-endpoint` qui effectue des requêtes HTTP réelles sur l'infrastructure provisionnée.

### 5.2 Scénario de Validation
Le test suit les étapes suivantes définies dans le fichier `.tftest.hcl` :
1.  **Deploy :** Déploiement de l'infrastructure via `command = apply`.
2.  **Validate :** Exécution du module de test avec l'URL de sortie (`run.deploy.api_endpoint`).
3.  **Assertions :** Vérification stricte des résultats.
    * Code HTTP attendu : `200`.
    * Corps de réponse attendu : `"Hello, World!"`.

```hcl
assert {
    condition = data.http.test_endpoint.status_code == 200
    error_message = "Unexpected status code: ${data.http.test_endpoint.status_code}"
}
```

L'exécution se fait via la commande `tofu test`, qui gère le cycle de vie complet (setup, test, teardown).


## 6. Conclusion et Retours d'Expérience

Ce laboratoire a permis de consolider une chaîne DevOps complète. Nous avons appris à :
* Gérer efficacement l'historique et la collaboration avec **Git**.
* Automatiser les tâches répétitives (build, docker) via **NPM**.
* Sécuriser les évolutions du code grâce aux tests **Jest**.
* Valider le déploiement d'infrastructure avec **OpenTofu**.

Ces pratiques garantissent une meilleure qualité logicielle et réduisent les risques lors des déploiements en production.
