# Repozytoria
- https://github.com/KrzysztofCaban/lab10_source
- https://github.com/KrzysztofCaban/lab10_config

# Task 1A
## Plik index.html
```html
<!DOCTYPE html>
<html lang="pl">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Lab10</title>
</head>

<body>
    <h1>Lab10</h1>
    <ul>
        <li>Author Name: Krzysztof Caban</li>
        <li>App Version: 1.0.3</li>
        <li>Dockerfile: <br><code>
            FROM nginx:alpine <br>
            WORKDIR /usr/share/nginx/html <br>
            COPY ./index.html . <br>
            EXPOSE 80 <br>
            CMD ["nginx", "-g", "daemon off;"] <br>
        </code></li>
    </ul>

</body>

</html>
```

## Dockerfile
```dockerfile
FROM nginx:alpine 
WORKDIR /usr/share/nginx/html
COPY ./index.html .
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

# Task 1B
## lab10-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lab10-deployment
spec:
  replicas: 4
  selector:
    matchLabels:
      app: lab10-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 5
      maxUnavailable: 2
  template:
    metadata:
      labels:
        app: lab10-app
    spec:
      containers:
        - name: lab10-app
          image: kcaban99/lab10-app:1.0.0
```

## lab10-ingress.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lab10-ingress
spec:
  rules:
  - host: zad2.lab
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: lab10-service
            port:
              number: 8080
```

## lab10-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: lab10-service
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 80
      nodePort: 30007
  selector:
    app: lab10-app
```

# Task 2
## .github/workflows
```yaml
name: Docker CI

on:
  workflow_dispatch:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
        
  dockerCI:
    runs-on: ubuntu-latest
    outputs:
      version_number: ${{ steps.vars.outputs.version_number }}

    steps:
    - name: Check out the repo
      uses: actions/checkout@v4
    
    - name: Set up QEMU
      uses: docker/setup-qemu-action@v3
      
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: set-output
      id: vars
      run: |
        version_number=$(sed -n 's/.*App Version: \(.*\)<\/li>/\1/p' index.html)
        echo "version_number=${version_number}" >> "$GITHUB_OUTPUT"
      shell: bash
      
    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_HUB_LOGIN }}
        password: ${{ secrets.DOCKER_HUB_PASSWORD }}
        
    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        platforms: linux/amd64,linux/arm64
        push: true
        tags: kcaban99/lab10-app:${{ steps.vars.outputs.version_number}}
        
  kubernetesCI:
    needs: dockerCI
    runs-on: ubuntu-latest

    steps:
      - name: Check out the repo
        uses: actions/checkout@v4
        with:
          repository: KrzysztofCaban/lab10_config
          token: ${{ secrets.ACTIONS_TOKEN }}
      - run: |
          sed -i 's/kcaban99\/lab10-app:.*/kcaban99\/lab10-app:${{ needs.dockerCI.outputs.version_number }}/g' lab10-deployment.yaml
          git config user.name github-actions
          git config user.email github-actions@github.com
          git add -u
          git commit -m "updated app version ${{ needs.dockerCI.outputs.version_number }}"
          git push
```

## <code style="color : red">Uwaga</code>
Z powodu pracy na platformie z chipsetem ARM, podczas budowania obrazu wymagane było ustawienie dodatkowego parametru dla stepu `Build and Push` - opcja `platforms` z wprowadzonymi architekturami `linux/amd64,linux/arm64`. Bez tego kroku, obraz budowany był jedynie na architekturę x86 co w dalszej części powodowało błąd przy pobieraniu obrazu z Docker Hub i stawianiu na nim deploymentu.

```yaml
- name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        platforms: linux/amd64,linux/arm64
        push: true
        tags: kcaban99/lab10-app:${{ steps.vars.outputs.version_number}}
```

Bez ustawienia tej opcji budowane obrazy były przeznaczone jedynie na platformę `linux/amd64`:

![dokcer_hub_amd_only](./screens/dockerhub_image_amd.png)

Po dodaniu informacji, aby obraz był także kompatybilny z architekturą ARM - `linux/arm64`:

![docker_hub_amd_arm](./screens/dockerhub_image_amd_arm.png)

# Task 3A
## gitops dockerfile
```dockerfile
FROM alpine:latest

RUN apk update && \
    apk add --no-cache git curl && \
    apk add --no-cache --repository=http://dl-cdn.alpinelinux.org/alpine/edge/community kubectl

CMD ["/bin/sh"]
```

## operator-stepcd.yaml
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: stepcd
spec:
  schedule: "*/2 * * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      backoffLimit: 0
      template:
        spec:
          restartPolicy: Never
          serviceAccountName: gitops
          containers:
            - name: zad2gitops 
              image: kcaban99/lab10gitops
              command: [sh, -e, -c]
              args:
                - git clone https://github.com/KrzysztofCaban/lab10_config.git /tmp/lab10_config && find /tmp/lab10_config -name '*.yaml' -exec kubectl apply -f {} \;
```

Przed uruchomieniem cronjoba zostało założone gitops service account
```bash
kubectl create sa gitops
kubectl create clusterrolebinding gitops-admin --clusterrole=cluster-admin --serviceaccount default:gitops
```

# Task 4A
```bash
kubectl apply -f operator-stepcd.yaml
```
![apply_gitops](./screens/apply_gitops.png)

### Po uruchomieniu 
![first_run](./screens/first_run.png)

### Strona przed wykonaniem update
![site_before](./screens/site_before.png)

# Task 4B
### Zmiana wersji aplikacji 1.0.2 -> 1.0.3 i wykonanie commita
![commit](screens/commit.png)

### Wykonanie się github actions
![github_actions](screens/github_actions.png)

### Commit wykonany przez github workflow w repo [lab10_config](https://github.com/KrzysztofCaban/lab10_source)
![github_actions_commit](./screens/github_commit.png)

### Podczas wykonywania automatycznego update
![during_update](./screens/during_update.png)

### Po wykonaniu się update
![after_update](./screens/after_update.png)

### Strona po wykonaniu operacji update
![site_after](./screens/site_after.png)
