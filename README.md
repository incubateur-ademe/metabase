![Metabase](metabase.png)

# Metabase sur Scalingo — déploiement automatisé

Ce dépôt est partagé par plusieurs SE (équipes produit), chacune ayant sa
propre instance Metabase sur [Scalingo](https://scalingo.com/). Pas besoin de
forker : chaque SE ajoute un environnement GitHub dédié, et le déploiement se
fait via un buildpack qui récupère Metabase à chaque déploiement — il n'y a
pas de code applicatif à compiler dans ce dépôt.

Trois GitHub Actions automatisent le cycle de vie des instances :

| Workflow | Rôle |
| --- | --- |
| `check-metabase-release.yml` | Vérifie chaque jour s'il existe une nouvelle version de [Metabase](https://github.com/metabase/metabase/releases). Si oui, déclenche un déploiement pour chaque environnement GitHub du dépôt (un job par SE, dans la même exécution). Chaque déploiement pousse le code vers l'app Scalingo de la SE ciblée et nécessite une **validation manuelle** par les reviewers de son environnement — plusieurs SE en attente peuvent être approuvées depuis le même écran d'exécution. |
| `sync-upstream.yml` | Vérifie chaque semaine si le dépôt d'origine (`Scalingo/metabase-scalingo`) a de nouveaux commits (buildpack, configuration…) et ouvre une PR si besoin. |

Chaque SE dispose de son propre environnement GitHub, nommé `<slug>`
(par exemple `quefairedemesobjetsetdechets`), qui porte :
- ses propres secrets (`SCALINGO_SSH_PRIVATE_KEY`, `SCALINGO_GIT_REMOTE`) pointant vers son app Scalingo,
- ses propres reviewers, chargés de valider les déploiements de leur instance,
- sa propre variable `LAST_DEPLOYED_METABASE_VERSION`, pour ne redéployer que si une nouvelle version est disponible.

La clé SSH peut être partagée entre SE (c'est juste l'identifiant technique
utilisé pour pousser le code), mais chaque environnement doit pointer vers
sa propre app Scalingo via `SCALINGO_GIT_REMOTE`.

## Ajouter une nouvelle SE

Ces étapes sont à réaliser par la SE qui souhaite héberger sa propre instance
Metabase.

⚠️ Tout environnement GitHub créé sur ce dépôt est considéré comme une SE à
déployer : ne pas créer d'environnement pour un autre usage.

### 1. Créer l'application sur Scalingo

```bash
scalingo create <slug>
scalingo --app <slug> addons-add postgresql postgresql-starter-512
scalingo --app <slug> env-set 'BUILDPACK_URL=https://github.com/Scalingo/multi-buildpack'
```

### 2. Générer une clé SSH dédiée au déploiement automatique

Ne pas réutiliser une clé SSH personnelle : une clé dédiée évite de lier le
déploiement à un compte individuel. Cette étape peut être mutualisée entre SE
si une clé de déploiement partagée existe déjà pour ce dépôt.

```bash
ssh-keygen -t ed25519 -f ~/.ssh/scalingo_deploy_ci -N "" -C "github-actions-<slug>"
scalingo login
scalingo keys-add github-actions-<slug> ~/.ssh/scalingo_deploy_ci.pub
```

### 3. Créer l'environnement GitHub `<slug>` avec validation obligatoire

```bash
gh api repos/<owner>/<repo>/environments/<slug> -X PUT --input - <<EOF
{
  "reviewers": [
    {"type": "User", "id": <id_github_reviewer_1>},
    {"type": "User", "id": <id_github_reviewer_2>}
  ]
}
EOF
```

L'identifiant numérique d'un utilisateur GitHub s'obtient avec :

```bash
gh api users/<login> --jq .id
```

### 4. Configurer les secrets et la variable de version, scopés à cet environnement

```bash
gh secret set SCALINGO_SSH_PRIVATE_KEY --repo <owner>/<repo> --env <slug> < ~/.ssh/scalingo_deploy_ci
gh secret set SCALINGO_GIT_REMOTE --repo <owner>/<repo> --env <slug> --body "git@ssh.osc-secnum-fr1.scalingo.com:<slug>.git"
gh variable set LAST_DEPLOYED_METABASE_VERSION --repo <owner>/<repo> --env <slug> --body "v0.0.0"
```

Puis supprimer la clé privée locale, elle n'a plus besoin d'exister que dans
le secret GitHub :

```bash
rm ~/.ssh/scalingo_deploy_ci
```

À partir de là, la SE est autonome : chaque jour, si une nouvelle version de
Metabase est disponible, un déploiement sera proposé et attendra la
validation d'un des reviewers de son environnement, depuis l'onglet
**Actions** du dépôt.

## Mettre à jour manuellement

Pour forcer un redéploiement sur une SE donnée sans attendre la vérification
quotidienne :

```bash
gh workflow run check-metabase-release.yml --repo <owner>/<repo> -f environment=<slug>
```

## Variables d'environnement de l'application

| Nom | Description | Valeur par défaut |
| --- | --- | --- |
| `BUILDPACK_URL` | Buildpack utilisé pour le déploiement | `https://github.com/Scalingo/multi-buildpack.git` |
| `DATABASE_URL` | URL de l'addon PostgreSQL (fournie automatiquement par Scalingo) | — |
| `MAX_METASPACE_SIZE` | Mémoire maximale allouée au Metaspace Java | `512m` |

Metabase supporte également de nombreuses [variables d'environnement propres à l'application](https://www.metabase.com/docs/latest/operations-guide/environment-variables.html).
