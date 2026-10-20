# Django on k3s

A minimal Django app containerized with Docker, pushed to Docker Hub, and deployed
to a single-node k3s cluster on an AWS EC2 instance, exposed through ingress-nginx.

```
Browser
   |
   v
ingress-nginx (port 80)
   |
   v
Service (cluster IP, port 80 -> 8000)
   |
   v
Deployment (2 pods, gunicorn, port 8000)
```

## Project structure

```
.
├── Dockerfile
├── requirements.txt
├── manage.py
├── core/               # Django project (settings, urls, wsgi)
├── app/                # Django app (index view)
└── k3s/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

## 1. Install packages locally

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

Verify the project runs:

```bash
python manage.py migrate
python manage.py runserver
# open http://127.0.0.1:8000/
```

## 2. Build and push the Docker image

> IMPORTANT: On Apple Silicon (M1/M2/M3) you MUST cross-build for `linux/amd64`,
> because the EC2 instance is x86_64. Skip `--platform` and the image will fail to
> run on EC2 (`ImagePullBackOff` / `exec format error`).

```bash
# Build for the EC2 architecture
docker build --platform linux/amd64 -t mrfahad0907/django-k3s:latest .

# Log in to Docker Hub (use a Personal Access Token, not your password)
docker login -u mrfahad0907

# Push
docker push mrfahad0907/django-k3s:latest
```

Optional sanity check that the published image is amd64:

```bash
docker buildx imagetools inspect mrfahad0907/django-k3s:latest
```

## 3. Prepare the EC2 instance (Ubuntu)

Open inbound port **80** (HTTP) in the EC2 security group.

Install k3s (Traefik disabled because we use ingress-nginx):

```bash
curl -sfL https://get.k3s.io | sh -s - --disable traefik --write-kubeconfig-mode 644
mkdir -p ~/.kube && cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
kubectl get nodes
```

Install Helm and ingress-nginx:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

Check the controller is up:

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

## 4. Deploy the application

Clone the repo on EC2:

```bash
git clone git@github.com:Fahad0907/k3s-practice.git
cd k3s-practice
```

Apply the manifests:

```bash
kubectl apply -f k3s/
```

Watch the rollout:

```bash
kubectl get pods -w
# wait for 2 pods with 1/1 Running
```

Inspect the ingress:

```bash
kubectl get ingress django-k3s -o wide
```

## 5. Access the app

The ingress has **no host rule**, so the bare EC2 public IP works:

```bash
curl http://<EC2_PUBLIC_IP>/
# Hello, World from Django on k3s!

curl -I http://<EC2_PUBLIC_IP>/admin/
# HTTP/1.1 302 Found (redirect to /admin/login/)
```

Open in a browser:

- App: `http://<EC2_PUBLIC_IP>/`
- Admin: `http://<EC2_PUBLIC_IP>/admin/`

### Optional: use a hostname instead of the IP

If the ingress is configured with `host: django.local`, add an entry to your machine:

```bash
echo "<EC2_PUBLIC_IP> django.local" | sudo tee -a /etc/hosts
```

Then visit `http://django.local/`. Note: only YOUR machine knows this hostname —
other devices must use the IP or a real DNS record.

## 6. Update / redeploy

After changing code:

```bash
docker build --platform linux/amd64 -t mrfahad0907/django-k3s:latest .
docker push mrfahad0907/django-k3s:latest
```

On EC2:

```bash
git pull
kubectl apply -f k3s/
kubectl rollout restart deployment django-k3s
kubectl get pods -w
```

`imagePullPolicy: Always` ensures the node actually pulls the newly pushed
`:latest` image instead of reusing its cached copy.

## Challenges faced and how they were solved

### 1. GitHub push failed: "remote origin already exists" + password auth rejected

**Symptom:** `git remote add origin` failed; HTTPS push asked for a password and
GitHub refused it ("Password authentication is not supported").

**Cause:** An `origin` remote already existed with an HTTPS URL; GitHub requires a
token or SSH, not an account password.

**Solution:** Repoint the existing remote to SSH:

```bash
git remote set-url origin git@github.com:Fahad0907/k3s-practice.git
git push -u origin main
```

### 2. Pods stuck in ImagePullBackOff / ErrImagePull

**Symptom:** `kubectl get pods` showed `ImagePullBackOff` then `ErrImagePull`.

**Cause:** The image was built on an Apple Silicon Mac (arm64), but the EC2
instance is amd64. Docker Hub had only an `arm64` manifest, so containerd could not
find a matching platform.

**Solution:** Rebuild for the target architecture:

```bash
docker build --platform linux/amd64 -t mrfahad0907/django-k3s:latest .
docker push mrfahad0907/django-k3s:latest
```

### 3. `/` returned 404 and readiness/liveness probes failed

**Symptom:** Probes hit `/`, but the root URL raised 404, so pods never became Ready.

**Cause:** `core/urls.py` only routed `admin/`; `app/urls.py` (the index view) was
never included.

**Solution:**

```python
# core/urls.py
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('app.urls')),
]
```

### 4. Bare IP returned "page not found"

**Symptom:** `http://<IP>/` gave 404 while `http://django.local/` worked.

**Cause:** The ingress rule had `host: django.local`, so requests whose Host header
was the IP matched nothing.

**Solution (choose one):**

- Remove the `host:` entry to accept any host/IP (what we did), or
- Keep the host and use `/etc/hosts` locally (only works on your own machine), or
- Point a real domain's DNS at the EC2 IP.

### 5. Admin CSS/JS broken (static files 404)

**Symptom:** `/admin/` loaded but unstyled; `/static/admin/css/base.css` returned 404.

**Causes:**

- `nginx.ingress.kubernetes.io/rewrite-target: /` rewrote **every** path to `/`,
  including `/static/...`.
- gunicorn does not serve static files. Django only serves them under `runserver`.

**Solution:**

- Removed the `rewrite-target` annotation from `k3s/ingress.yaml`.
- Added WhiteNoise and collected static files at build time:

```python
# requirements.txt
whitenoise==6.6.0

# core/settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',
    ...
]
STATIC_ROOT = BASE_DIR / 'staticfiles'
```

```dockerfile
# Dockerfile
COPY . .
RUN python manage.py collectstatic --noinput
```

### 6. New image pushed, but EC2 kept running the old one

**Symptom:** CSS still 404 after pushing a fixed `:latest`; pod image digest was the
old one.

**Cause:** `imagePullPolicy: IfNotPresent` plus a reused `:latest` tag meant the node
used its cached image.

**Solution:** Set `imagePullPolicy: Always` in `k3s/deployment.yaml`, re-apply, and
restart:

```bash
kubectl apply -f k3s/deployment.yaml
kubectl rollout restart deployment django-k3s
```

### 7. Docker Hub login rejected the account password

**Symptom:** `unauthorized: incorrect username or password`.

**Cause:** Docker Hub no longer accepts plain account passwords for CLI login.

**Solution:** Create an Access Token at
<https://hub.docker.com/settings/security> and use it:

```bash
docker login -u <username>   # paste the token as password
```

### 8. `venv/` accidentally committed to git

**Symptom:** A commit included hundreds of `venv/` and `__pycache__` files.

**Cause:** `.gitignore` did not exist / did not list them.

**Solution:**

```bash
git rm -r --cached venv __pycache__ .idea
printf 'venv/\n__pycache__/\n*.pyc\nstaticfiles/\n.idea/\n' > .gitignore
git add .gitignore && git commit -m "gitignore venv, pycache, ide files"
```

## Known limitations / next steps

- **SQLite is ephemeral.** Each pod has its own copy of `db.sqlite3`; data resets on
  restart and is not shared between the 2 replicas. Use a managed database
  (PostgreSQL) with a Kubernetes Secret for real persistence.
- **No superuser.** The admin login page renders but no account exists. Create one
  with `python manage.py createsuperuser` (and a persistent DB to keep it).
- **`DEBUG = True` and a hardcoded `SECRET_KEY`.** Move these to environment
  variables / a Secret and set `DEBUG = False` for production.
- **HTTP only.** No TLS. Add cert-manager + a real domain for HTTPS.
- **Single node.** k3s runs on one EC2 instance; there is no high availability.

## Useful commands

```bash
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs -f deployment/django-k3s
kubectl rollout restart deployment django-k3s
kubectl delete -f k3s/
kubectl get events --sort-by=.lastTimestamp
```
