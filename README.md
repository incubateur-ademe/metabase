![Metabase](metabase.png)

# Metabase sur Scalingo — déploiement automatisé

Ce dépôt est un fork de [Scalingo/metabase-scalingo](https://github.com/Scalingo/metabase-scalingo),
utilisé pour déployer et maintenir à jour une instance Metabase hébergée sur
[Scalingo](https://scalingo.com/). Le déploiement se fait via un buildpack qui
récupère Metabase à chaque déploiement : il n'y a pas de code applicatif à
compiler dans ce dépôt.

Trois GitHub Actions automatisent le cycle de vie de l'instance :

| Workflow | Rôle |
| --- | --- |
| `check-metabase-release.yml` | Vérifie chaque jour s'il existe une nouvelle version de [Metabase](https://github.com/metabase/metabase/releases). Si oui, déclenche `deploy.yml`. |
| `deploy.yml` | Pousse le code vers Scalingo pour forcer un redéploiement. Nécessite une **validation manuelle** (environnement GitHub `metabase`). |
| `sync-upstream.yml` | Vérifie chaque semaine si le dépôt d'origine (`Scalingo/metabase-scalingo`) a de nouveaux commits (buildpack, configuration…) et ouvre une PR si besoin. |

## Mise en place initiale

Ces étapes ne sont à faire qu'une seule fois, lors de la création du dépôt ou
si l'application Scalingo est recréée.

### 1. Créer l'application sur Scalingo

```bash
scalingo create mon-metabase
scalingo --app mon-metabase addons-add postgresql postgresql-starter-512
scalingo --app mon-metabase env-set 'BUILDPACK_URL=https://github.com/Scalingo/multi-buildpack'
```

### 2. Générer une clé SSH dédiée au déploiement automatique

Ne pas réutiliser votre clé SSH personnelle : une clé dédiée évite de lier le
déploiement à un compte individuel.

```bash
ssh-keygen -t ed25519 -f ~/.ssh/scalingo_deploy_ci -N "" -C "github-actions-metabase"
scalingo login
scalingo keys-add github-actions-metabase ~/.ssh/scalingo_deploy_ci.pub
```

### 3. Configurer les secrets et variables GitHub

```bash
gh secret set SCALINGO_SSH_PRIVATE_KEY --repo <owner>/<repo> < ~/.ssh/scalingo_deploy_ci
gh secret set SCALINGO_GIT_REMOTE --repo <owner>/<repo> --body "git@ssh.osc-secnum-fr1.scalingo.com:mon-metabase.git"
gh variable set LAST_DEPLOYED_METABASE_VERSION --repo <owner>/<repo> --body "v0.0.0"
```

Puis supprimer la clé privée locale, elle n'a plus besoin d'exister que dans
le secret GitHub :

```bash
rm ~/.ssh/scalingo_deploy_ci
```

### 4. Créer l'environnement GitHub `metabase` avec validation obligatoire

```bash
gh api repos/<owner>/<repo>/environments/metabase -X PUT
gh api repos/<owner>/<repo>/environments/metabase -X PUT \
  -f 'reviewers[][type]=User' \
  -F 'reviewers[][id]=<id_github_du_validateur>'
```

L'identifiant numérique d'un utilisateur GitHub s'obtient avec :

```bash
gh api users/<login> --jq .id
```

À partir de là, chaque déploiement (automatique ou manuel via
`workflow_dispatch`) sera bloqué tant qu'un des validateurs configurés ne
l'aura pas approuvé depuis l'onglet **Actions** du dépôt.

## Mettre à jour manuellement

Pour forcer un redéploiement sans attendre la vérification quotidienne :

```bash
gh workflow run deploy.yml --repo <owner>/<repo>
```

## Variables d'environnement de l'application

| Nom | Description | Valeur par défaut |
| --- | --- | --- |
| `BUILDPACK_URL` | Buildpack utilisé pour le déploiement | `https://github.com/Scalingo/multi-buildpack.git` |
| `DATABASE_URL` | URL de l'addon PostgreSQL (fournie automatiquement par Scalingo) | — |
| `MAX_METASPACE_SIZE` | Mémoire maximale allouée au Metaspace Java | `512m` |

Metabase supporte également de nombreuses [variables d'environnement propres à l'application](https://www.metabase.com/docs/latest/operations-guide/environment-variables.html).
