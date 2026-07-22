![Metabase](metabase.png)

# Metabase sur Scalingo - déploiement automatisé

Ce dépôt permet à n'importe quelle startup d'État (SE) de déployer sa propre
instance [Metabase](https://www.metabase.com/) sur [Scalingo](https://scalingo.com/),
sans avoir à forker le dépôt ni à maintenir de code applicatif : le
déploiement se fait via un buildpack qui récupère Metabase à chaque mise à
jour.

## Déployer Metabase pour votre startup d'État

Ces étapes sont à réaliser par la SE qui souhaite héberger sa propre instance
Metabase.

Tout environnement GitHub créé sur ce dépôt est considéré comme une SE à
déployer : ne pas créer d'environnement pour un autre usage.

### 1. Créer l'application sur Scalingo

```bash
scalingo create <slug>
scalingo --app <slug> addons-add postgresql postgresql-starter-512
scalingo --app <slug> env-set 'BUILDPACK_URL=https://github.com/Scalingo/multi-buildpack'
```

### 2. Demander à un admin d'ajouter votre app Scalingo sous forme d'environnement GitHub

Un admin du dépôt doit créer l'environnement `<slug>`, y ajouter les
reviewers de la SE, et configurer la variable `LAST_DEPLOYED_METABASE_VERSION`
ainsi que le secret `SCALINGO_GIT_REMOTE` pointant vers votre app (`SCALINGO_SSH_PRIVATE_KEY`
n'est pas à configurer ici, voir plus bas).

La clé SSH de déploiement (`SCALINGO_SSH_PRIVATE_KEY`) n'est pas portée par
votre environnement : elle est définie une seule fois au niveau du dépôt par
un admin et partagée entre toutes les SE. Ce n'est qu'un identifiant
technique utilisé pour pousser le code, chaque environnement restant
cloisonné via son propre `SCALINGO_GIT_REMOTE`. Vous n'avez rien à fournir
de ce côté.

À partir de là, la SE est autonome : chaque jour, si une nouvelle version de
Metabase est disponible, un déploiement sera proposé et attendra la
validation d'un des reviewers de son environnement, depuis l'onglet Actions
du dépôt.

### Mettre à jour manuellement

Pour forcer un redéploiement sur votre SE sans attendre la vérification
quotidienne :

```bash
gh workflow run check-metabase-release.yml --repo <owner>/<repo> -f environment=<slug>
```

## Fonctionnement du dépôt

Deux GitHub Actions automatisent le cycle de vie des instances :

| Workflow | Rôle |
| --- | --- |
| `check-metabase-release.yml` | Vérifie chaque jour s'il existe une nouvelle version de [Metabase](https://github.com/metabase/metabase/releases). Si oui, déclenche un déploiement pour chaque environnement GitHub du dépôt (un job par SE, dans la même exécution). Chaque déploiement pousse le code vers l'app Scalingo de la SE ciblée et nécessite une validation manuelle par les reviewers de son environnement : plusieurs SE en attente peuvent être approuvées depuis le même écran d'exécution. |
| `sync-upstream.yml` | Vérifie chaque semaine si le dépôt d'origine (`Scalingo/metabase-scalingo`) a de nouveaux commits (buildpack, configuration...) et ouvre une PR si besoin. |

Chaque SE dispose de son propre environnement GitHub, nommé `<slug>`
(par exemple `quefairedemesobjetsetdechets`), qui porte :
- le secret `SCALINGO_GIT_REMOTE`, pointant vers son app Scalingo,
- ses propres reviewers, chargés de valider les déploiements de leur instance,
- sa propre variable `LAST_DEPLOYED_METABASE_VERSION`, pour ne redéployer que si une nouvelle version est disponible.

Le secret `SCALINGO_SSH_PRIVATE_KEY` n'est pas défini par SE ni porté par un
environnement : c'est un secret défini une seule fois au niveau du dépôt par
un admin, et réutilisé pour toutes les SE. Seul `SCALINGO_GIT_REMOTE`
distingue une app d'une autre.

## Variables d'environnement de l'application

| Nom | Description | Valeur par défaut |
| --- | --- | --- |
| `BUILDPACK_URL` | Buildpack utilisé pour le déploiement | `https://github.com/Scalingo/multi-buildpack.git` |
| `DATABASE_URL` | URL de l'addon PostgreSQL (fournie automatiquement par Scalingo) | - |
| `MAX_METASPACE_SIZE` | Mémoire maximale allouée au Metaspace Java | `512m` |

Metabase supporte également de nombreuses [variables d'environnement propres à l'application](https://www.metabase.com/docs/latest/operations-guide/environment-variables.html).
