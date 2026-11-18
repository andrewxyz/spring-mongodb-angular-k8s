# Publishing the images to Docker Hub

How to push this project's two images to Docker Hub (account `andrimardiana`). Run the commands from the repo root.

## Which images get pushed

| Local image | Docker Hub name |
|---|---|
| `belajar-backend:1.0.0` | `andrimardiana/belajar-backend:1.0.0` |
| `belajar-frontend:1.0.0` | `andrimardiana/belajar-frontend:1.0.0` |

The other images in the project (`mongo:7`, Traefik, nginx and the Maven/Node/JRE build images) are public. Anyone can pull them, so there's no need to copy them to your account.

## Steps

**1. Check the images exist locally.**
```bash
docker images | grep belajar            # PowerShell: docker images | Select-String belajar
```
You should see both images with tag `1.0.0`. If one is missing, build it:
```bash
docker build -t belajar-backend:1.0.0  spring-boot-data-mongodb
docker build -t belajar-frontend:1.0.0 angular-crud
```

**2. Create an access token** (use this instead of your account password).
- Docker Hub → your avatar → **Account settings** → **Personal access tokens** → **Generate new token**.
- Choose **Read & Write** access.
- Copy the token right away: Docker Hub shows it only once.

**3. Log in.**
```bash
docker login -u andrimardiana
```
Paste the token when it asks for a password. You should see `Login Succeeded`. Docker Desktop remembers it, so you don't need to log in again next time.

**4. Tag the images with your username.** Docker Hub only accepts names that start with your username. A tag adds a second name to the same image; nothing is copied.
```bash
docker tag belajar-backend:1.0.0  andrimardiana/belajar-backend:1.0.0
docker tag belajar-frontend:1.0.0 andrimardiana/belajar-frontend:1.0.0
```

**5. Push.**
```bash
docker push andrimardiana/belajar-backend:1.0.0
docker push andrimardiana/belajar-frontend:1.0.0
```

**6. Check it worked.**
- Open https://hub.docker.com/repositories/andrimardiana. Both repositories should be there with the tag `1.0.0`.
- Or test a real download: remove the local copy and pull it back.
  ```bash
  docker rmi andrimardiana/belajar-backend:1.0.0
  docker pull andrimardiana/belajar-backend:1.0.0
  ```

## Public or private repositories

Docker Hub creates a repository automatically on the first push, and new repositories are **public**. For a private one, create it first on the website (Repositories → **Create repository** → Private), then push. Free accounts have a limited number of private repositories.

## Optional: make Kubernetes use the Docker Hub images

At the moment the manifests use `belajar-backend:1.0.0`, which only works because `kind load` copies the image into the nodes. Once the images are on Docker Hub, the cluster can pull them itself, so you can skip `kind load`.

1. Change the `image:` lines:
   - [backend/deployment.yaml](../backend/deployment.yaml): `image: andrimardiana/belajar-backend:1.0.0`
   - [frontend/deployment.yaml](../frontend/deployment.yaml): `image: andrimardiana/belajar-frontend:1.0.0`
2. Apply and watch the rollout:
   ```bash
   kubectl apply -k k8s/
   kubectl rollout status deploy/backend  -n demo-namespace
   kubectl rollout status deploy/frontend -n demo-namespace
   ```
3. **If the repositories are private**, the cluster needs to log in as well. Create a pull secret:
   ```bash
   kubectl create secret docker-registry dockerhub-cred -n demo-namespace \
     --docker-username=andrimardiana --docker-password=<your-token>
   ```
   Then add this under `spec.template.spec` in both Deployment files:
   ```yaml
   imagePullSecrets:
     - name: dockerhub-cred
   ```
   Never put the token in a file that goes into git.

## Things to know

- **Use a new tag for every change** (`1.0.1`, `1.0.2`, ...) and avoid `latest`. With `imagePullPolicy: IfNotPresent`, Kubernetes won't download an image whose tag it already has.
- **CPU architecture:** the images were built on a normal Windows PC, so they're `amd64`. They won't run on ARM machines such as an Apple Silicon Mac. To publish both architectures in one step:
  ```bash
  docker buildx build --platform linux/amd64,linux/arm64 \
    -t andrimardiana/belajar-backend:1.0.0 --push spring-boot-data-mongodb
  ```
  Do the same for the frontend with `angular-crud` and its own image name. This takes longer.
- **No secrets inside the images.** The database credentials arrive as environment variables when the pod starts, so it's safe to publish the images publicly.
