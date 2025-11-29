# DevOps Report - Site Quartz

## ⚠️ Important : Utiliser WSL

**Toutes les commandes `make` doivent être exécutées dans WSL (Windows Subsystem for Linux), pas dans PowerShell ou CMD.**

## 🚀 Démarrage rapide

### Première installation

1. Cloner le repository :

   ```bash
   git clone <url-du-repo>
   cd devops
   ```

2. Récupérer le fichier `.env` manuellement (demandez-le à un collaborateur)

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

# 2. Mettre à jour et déployer les changements
make site/update
```

Le site sera accessible sur `http://localhost:8080` (ou le port indiqué).

## 📝 Workflow de développement

1. **Modifier le contenu** : Éditez les fichiers Markdown dans `content/`
2. **Mettre à jour** : `make site/update` pour builder et déployer
3. **Prévisualiser** : `make site/serve` pour voir les changements localement

## 📚 Comment fonctionne Quartz 4 ?

Quartz 4 est un générateur de site statique qui transforme vos fichiers Markdown en site web.

### Le processus

1. **Source** : Vos fichiers Markdown dans `content/`
2. **Build** : Quartz lit `content/` et génère des fichiers HTML/CSS/JS dans `public/`
3. **Déploiement** : Le dossier `public/` est déployé sur Cloudflare Pages

### Éditer le contenu

- Ajoutez/modifiez des fichiers `.md` dans `content/`
- Utilisez le [frontmatter YAML](https://quartz.jzhao.xyz/features/frontmatter) pour la métadonnée :

  ```markdown
  ---
  title: Mon Titre
  tags: [devops, tutorial]
  ---
  
  # Contenu de la page
  ```

### Configuration

Le fichier `quartz.config.ts` contient la configuration du site (thème, plugins, etc.).



## 📖 Documentation

- [Quartz 4 Documentation](https://quartz.jzhao.xyz/)
- [Markdown Guide](https://quartz.jzhao.xyz/features/creating-content)
