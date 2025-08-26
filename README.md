# test-app

## Application: Gitea (Swarm)

Gitea est une application Git auto-hébergée, légère et prête pour la prod. Nous déployons 3 instances (alice, bob, charlie) derrière Traefik, chacune avec sa base Postgres.

### Déploiement rapide

1. Créer le réseau overlay partagé

   ```bash
   docker network create --driver overlay traefik-public
   ```

2. Déployer Traefik

   ```bash
   docker stack deploy -c traefik-stack.yml traefik
   ```

3. Déployer les applications

   ```bash
   docker stack deploy -c docker-stack-alice.yml alice
   docker stack deploy -c docker-stack-bob.yml bob
   docker stack deploy -c docker-stack-charlie.yml charlie
   ```

### Hôtes

Les stacks exposent:

- Alice: `alice.192.168.101.100.nip.io`
- Bob: `bob.192.168.101.100.nip.io`
- Charlie: `charlie.192.168.101.100.nip.io`

Comptes/init: Gitea s’initialise au premier lancement. La base Postgres utilise par défaut `gitea / gitea` (utilisateur/mot de passe) définis dans les stacks.

---

## Section legacy (LinkedIn clone)

- `linkedin-clone-docker/` (frontend statique servi par Nginx + backend Node + Mongo)
  - `build/` (assets frontend précompilés copiés par `Dockerfile.fe`)
  - `server/` (backend Express, `Dockerfile.be`)
  - `.gitignore` et `.dockerignore` configurés pour ignorer le superflu

## Déploiement Traefik (Swarm)

1. Créer le réseau overlay partagé:

   ```bash
   docker network create --driver overlay traefik-public
   ```

2. Déployer Traefik:

   ```bash
   docker stack deploy -c traefik-stack.yml traefik
   ```

## Déploiement de l'app

Attention: `docker stack deploy` (Swarm) ne construit pas les images à partir de `build:`. Deux options:

- Option A (local/Compose): utiliser `docker-compose.yaml` dans `linkedin-clone-docker/` pour tester en local.
- Option B (recommandé en Swarm): construire et pousser des images, puis référencer ces images dans les stacks `docker-stack-*.yml` via `image:` (à la place de `build:`).

Exemple (schématique) pour construire/pousser:

```bash
# Frontend (copie de linkedin-clone-docker/build/ dans Nginx)
docker build -t <registry>/linkedin-fe:latest -f linkedin-clone-docker/Dockerfile.fe linkedin-clone-docker
docker push <registry>/linkedin-fe:latest

# Backend
docker build -t <registry>/linkedin-be:latest -f linkedin-clone-docker/server/Dockerfile.be linkedin-clone-docker/server
docker push <registry>/linkedin-be:latest
```
Ensuite, remplacer les sections `build:` par `image: <registry>/linkedin-fe:latest` et `image: <registry>/linkedin-be:latest` dans `docker-stack-*.yml`.

## Hôtes

Les stacks exposent:

- Alice: `alice.192.168.101.100.nip.io`
- Bob: `bob.192.168.101.100.nip.io`
- Charlie: `charlie.192.168.101.100.nip.io`

Le backend est exposé sur le même host (port interne 9000) via Traefik.

## Variables frontend

Le frontend utilise `process.env.REACT_APP_*` (injectées au build CRA). Si vous devez changer les URLs API, reconstruisez le dossier `build/` avec les variables souhaitées, puis reconstruisez l'image frontend.

## Nettoyage/ignores

- `.gitignore` dans `linkedin-clone-docker/` ignore `node_modules/`, dossiers redondants (`dev/`, `linked-in-clone-cp/`), logs et fichiers d’environnement.
- `.dockerignore` à la racine frontend inclut uniquement `build/` et `Dockerfile.fe` dans le contexte de build.
- `.dockerignore` dans `server/` exclut `node_modules/`, logs, `.env`, etc.