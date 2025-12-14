# DevOps Report - Site Quartz

## 🚀 Démarrage rapide

### Première installation


1. Cloner le repository :

   ```bash
   git clone <url-du-repo>
   cd devops
   ```

2. Récupérer le fichier `.env` manuellement (demandez-le à un collaborateur)

**⚠️ Important : Toutes les commandes doivent être exécutées dans Linux (c'est mieux) ou WSL (Windows Subsystem for Linux), pas dans PowerShell ou CMD.**

3. Installer les dépendances système :

   ```bash
   sudo apt-get install jq git curl
   ```

4. Lancer l'installation :

   ```bash
   make site/setup
   ```

### Développement local

Pour travailler sur le site localement :

```bash
# 1. Servir le site localement pour prévisualiser
make site/serve

# 2. Déployer les changements (il faut faire ça sur la branche main)
make site/update
```

## 📝 Workflow de développement

1. **Modifier le contenu** : Éditez les fichiers Markdown dans `content/`
2. **Prévisualiser** : `make site/serve` pour voir les changements localement

## 📚 Comment fonctionne Quartz 4 ?

Quartz 4 est un générateur de site statique qui transforme vos fichiers Markdown en site web.

### Le processus

1. **Source** : Vos fichiers Markdown dans `content/`
2. **Build** : Quartz lit `content/` et génère des fichiers HTML/CSS/JS dans `public/`
3. **Déploiement** : Le dossier `public/` est déployé sur Cloudflare Pages

### Éditer le contenu

#### Créer un nouveau rapport

1. **Créer le fichier Markdown** dans `content/DevOps/` :
   - Nommez-le de manière claire : `td1.md`, `td2.md`, `tp1.md`, etc.
   - Utilisez des **minuscules** et des **tirets** pour les noms de fichiers : `td1.md` ✅, pas `TD1.md` ❌

2. **Ajouter le frontmatter YAML** en haut du fichier :

   ```markdown
   ---
   title: "TD1 - Déploiement d'applications"
   description: "Description courte du rapport"
   tags:
     - devops
     - deployment
     - paas
     - iaas
     - td1
   ---
   ```

   **Champs importants :**
   - `title` : Titre affiché sur la page (utilisez des guillemets si le titre contient des caractères spéciaux)

3. **Ajouter des images** :
   - Créez un dossier `content/devops/images/nom-du-td/` (ex: `images/td1/`)
   - Placez vos images dans ce dossier (PNG, JPG, etc.)
   - Référencez-les dans le Markdown avec un chemin relatif :

     ```markdown
     ![Description de l'image](devops/images/td1/mon-image.png)
     ```

4. **Structurer votre contenu** :
   - Utilisez des titres de section (`##`, `###`) pour organiser
   - Incluez des captures d'écran pour illustrer vos étapes
   - Documentez les problèmes rencontrés et leurs solutions

#### Exemple de structure complète

```text
content/
└── devops/
    ├── td1.md
    └── images/
        └── td1/
            ├── 1.png
            ├── 2.png
            └── screenshot.png
```

## 📖 Documentation

- [Quartz 4 Documentation](https://quartz.jzhao.xyz/)
- [Markdown Guide](https://quartz.jzhao.xyz/features/creating-content)
