![Metabase](metabase.png)

# Metabase sur Scalingo — déploiement automatisé

Ce dépôt est partagé par plusieurs SE (équipes produit), chacune ayant sa
propre instance Metabase sur [Scalingo](https://scalingo.com/). Pas besoin de
forker : chaque SE ajoute un environnement GitHub dédié, et le déploiement se
fait via un buildpack qui récupère Metabase à chaque déploiement — il n'y a
pas de code applicatif à compiler dans ce dépôt.

Deux GitHub Actions automatisent le cycle de vie des instances :

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

### 2. Demander à un admin d'ajouter votre app Scalingo sous forme d'environnement GitHub

Un admin du dépôt doit créer l'environnement `<slug>`, y ajouter les
reviewers de la SE, et configurer les secrets (`SCALINGO_SSH_PRIVATE_KEY`,
`SCALINGO_GIT_REMOTE`) et la variable `LAST_DEPLOYED_METABASE_VERSION`.

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
