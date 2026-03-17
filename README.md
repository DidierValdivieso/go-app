# go‑app

A minimal but fully‑featured Go API that demonstrates a CI/CD pipeline
with **GitHub Actions**, Docker Hub and an AWS deployment.

---

## 👀 Quick overview

*   Two public endpoints:
    * `GET /` – hello message
    * `GET /items` – static list of items
    * `GET /randomuser` – proxies `https://randomuser.me` and returns the first result
*   Runs in a single Alpine‑based Go 1.22 container
*   CI/CD rebuilds & pushes a new Docker image to Docker Hub on every push to `main`.
*   A subsequent deployment job pulls that image, replaces the container on a
    self‑hosted runner (or any ECS‑Fargate task).

---

## ⚙️ Prerequisites

| Tool | Minimum version | Install guide |
|------|------------------|----------------|
| Go | `1.22.x` | `brew install go` or `apt install golang` |
| Docker | `24.x` | Download from <https://docs.docker.com/get-docker/> |
| Git | `2.x` | `brew install git` |
| AWS CLI (optional for ECS) | `2.x` | `brew install awscli` |
| GitHub account | – | – |
| Docker Hub account | – | – |

> **Tip:** The Dockerfile is **single‑stage** – no need for `docker‑buildx` or multistage builds.

---

## 📦 Installation (Docker)

```bash
# 1️⃣ Pull the image
docker pull didiervald/go-app:latest

# 2️⃣ Run it (map host port 4040 → container port 4040)
docker run -d --name go-app-docker -p 4040:4040 didiervald/go-app:latest
```

If you want to run the service locally *without Docker*:

```bash
git clone https://github.com/DidierValdivieso/go-app.git
cd go-app
go run main.go
```

A `.env` file with `PORT=4040` is automatically read by the program.  
The server will print:

```
Server listening on port 4040
```

---

## 📡 API reference

| Method | Path | Description | Sample response |
|--------|------|-------------|-----------------|
| `GET` | `/` | Hello message | `{"text":"Hello from GoLang API!"}` |
| `GET` | `/items` | List of demo items | `[{ "ID":1, "Name":"Book", "Price":10.99 },...]` |
| `GET` | `/randomuser` | Proxy to randomuser.me | `{"gender":"female","name":{"title":"Miss","first":"Hilary","last":"Harrington"}, ...}` |

Responses are JSON, `Content‑Type: application/json`.

---

## 📦 Dockerfile

```Dockerfile
# Base image
FROM golang:1.22.2-alpine

# Build directory inside the container
WORKDIR /app

# Go modules
COPY go.mod go.sum ./
RUN go mod tidy

# Source files
COPY . .

# Compile -> binary named `main`
RUN go build -o main ./main.go

# Make binary executable
RUN chmod +x main

# Expose port (as defined in the container)
EXPOSE 4040

# Run the binary
CMD [ "./main" ]
```

---

## 🔁 GitHub Actions – CI/CD

Workflow file: `.github/workflows/cicd.yml`

```yaml
name: Deploy Go Application

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Create .env
        run: echo "PORT=4040" >> .env

      - name: Login to Docker Hub
        run: docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_PASSWORD }}

      - name: Build Docker image
        run: docker build -t didiervald/go-app .

      - name: Push image to Docker Hub
        run: docker push didiervald/go-app:latest

  deploy:
    needs: build
    runs-on: self-hosted   # change if you use AWS ECS workers
    steps:
      - name: Pull Docker image
        run: docker pull didiervald/go-app:latest

      - name: Delete old container
        run: docker rm -f go-app-container || true

      - name: Run Docker container
        run: docker run -d -p 4040:4040 --name go-app-container didiervald/go-app
```

**Secrets required**

| Secret | Value | How to set |
|--------|-------|------------|
| `DOCKER_USERNAME` | Your Docker Hub username | Repository Settings → Secrets → New repository secret |
| `DOCKER_PASSWORD` | Your Docker Hub password / PAT | Same place |
| `PORT` | `4040` | Optional – used only for creating a `.env` inside the job |

> If you prefer an ECS‑Fargate deployment, replace the `deploy` job with the Amazon ECS action (see the **AWS Deployment** section below).

---

## ⚙️ Deploying to AWS (ECS‑Fargate)

1. **Create an ECS cluster** – e.g., `go-app-cluster`.
2. **Create a task definition** (file `ecs-task-def.json`):

   ```json
   {
     "family": "go-app",
     "networkMode": "awsvpc",
     "requiresCompatibilities": ["FARGATE"],
     "cpu": "256",
     "memory": "512",
     "executionRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole",
     "containerDefinitions": [
       {
         "name": "go-app",
         "image": "didiervald/go-app:latest",
         "portMappings": [
           { "containerPort": 4040, "hostPort": 4040 }
         ],
         "essential": true
       }
     ]
   }
   ```

3. **Create a service** that uses the task definition.
4. **Add the GitHub Actions AWS credentials**:

   * `AWS_ACCESS_KEY_ID`
   * `AWS_SECRET_ACCESS_KEY`

5. **Modify the `deploy` job** in `cicd.yml`:

   ```yaml
   deploy:
     needs: build
     runs-on: ubuntu-latest
     steps:
       - name: Configure AWS credentials
         uses: aws-actions/configure-aws-credentials@v4
         with:
           aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
           aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
           aws-region: us-east-1

       - name: Deploy to ECS
         uses: aws-actions/amazon-ecs-deploy-task-definition@v1
         with:
           task-definition: ecs-task-def.json
           service: go-app-service
           cluster: go-app-cluster
           wait-for-service-stability: true
   ```

> The AWS role `ecsTaskExecutionRole` must have `AmazonECS_FullAccess` and the `AWSCloudTrailWrite` policy.

---

## 🚀 Running locally (no Docker)

```bash
# Clone repo
git clone https://github.com/DidierValdivieso/go-app.git
cd go-app

# Create a minimal .env (only needed if you want a custom port)
echo "PORT=4040" > .env

# Run the server
go run main.go
```

Open `http://localhost:4040/` in your browser – you should see the JSON greeting.

---