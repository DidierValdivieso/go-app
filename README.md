# go‑app

A compact demonstration project that shows how a Go API can be built,
containerised, and automatically deployed using GitHub Actions and AWS.

---  

## What I built

| Layer | Description |
|-------|------------|
| **API** | A single‑package Go server exposing three JSON endpoints: <br>• `GET /` – greets the user. <br>• `GET /items` – returns a static slice of items. <br>• `GET /randomuser` – proxies the public randomuser API and streams the first result. |
| **Container** | A single‑stage Docker image built with Go 1.22.2‑alpine. |
| **CI** | GitHub Actions that: <br>1. Builds the Go binary.<br>2. Builds a Docker image.<br>3. Pushes the image to Docker Hub. |
| **CD** | A self‑hosted (or ECS‑Fargate) runner that: <br>• Pulls the newly pushed image.<br>• Stops any existing container.<br>• Starts the new container on port 4040. |

The goal was to keep the example **small and focused** – it does not
contain extensive production‑level features such as authentication,
logging frameworks, or database access – but it is fully functional
and works out‑of‑the‑box.

---  

## How it works

 1. `main.go` loads a `PORT` value from a `.env` file (defaults to `4040`).
 2. Handlers are defined under `handlers/` and use model structs from
    `models/`.
 3. The Dockerfile compiles the binary and exposes port 4040.
 4. The GitHub workflow (`cicd.yml`) automatically rebuilds the image
    whenever a commit lands on `main` and pushes it to Docker‐Hub.
 5. The CD step pulls that image, removes any running container
    named `go-app-container`, and starts a fresh instance.

---  

## What I Can Do Next

* Extend the API with more complex endpoints and persistence.
* Add unit tests and code coverage reports to the CI pipeline.
* Replace the self‑hosted deploy with a fully managed ECS‑Fargate
  deployment—the workflow already contains the necessary
  configuration.
