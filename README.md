# T02: Containerize and Run Locally

## Project Overview

This project demonstrates multi-stage Docker builds, container orchestration with Docker Compose, and local execution of an AWS SAM Lambda function. The application is a static HTML page served by Nginx, with Redis running as a separate service.

## Prerequisites

- Docker Desktop with WSL2 integration enabled
- Git
- AWS SAM CLI
- Internet access for downloading container images

An AWS account is not required to invoke the Lambda function locally.

## 1. Build and Compare Docker Images

Build the multi-stage image:

```bash
docker build -f Dockerfile -t demo:multi .
```

Build the single-stage comparison image:

```bash
docker build -f Dockerfile.single -t demo:single .
```

Compare image sizes:

```bash
docker images demo:multi demo:single
```

The recorded image sizes were 93.6 MB for the multi-stage image and 1.65 GB for the single-stage image. Results can vary with image versions and rebuilds.

## 2. Start the Application with Docker Compose

From the project root, run:

```bash
docker compose up -d --build
docker compose ps
```

The Compose configuration defines two services:

- `web`: builds the application image and maps host port 8080 to container port 80.
- `cache`: runs Redis using `redis:7-alpine`.

The web service depends on Redis passing its health check before startup.

## 3. Verify the Services

Check the HTTP response:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080
```

Expected result:

```text
200
```

Check the Redis health response:

```bash
docker compose exec cache redis-cli ping
```

Expected result:

```text
PONG
```

Check the Nginx version:

```bash
docker compose exec web sh -c 'nginx -v'
```

Check the web container state and restart count:

```bash
docker inspect $(docker compose ps -q web) --format '{{.State.Status}} started={{.State.StartedAt}} restarts={{.RestartCount}}'
```

## 4. Stop and Remove the Containers

Run:

```bash
docker compose down
```

This removes the Compose containers and network. The built images remain available locally.

## 5. Invoke the Lambda Function Locally

From the project root, run:

```bash
cd local-fn
sam local invoke HelloWorldFunction
```

The first invocation may download a Lambda runtime image. A successful invocation returns a JSON response containing `statusCode: 200` and a Hello World message.

Return to the project root with:

```bash
cd ..
```

## Design Rationale

### Multi-stage Docker build

The main Dockerfile uses a Node.js build stage to install dependencies and generate the application output. A second stage uses Nginx Alpine to serve only the generated static files. This separates build-time tooling from the runtime image and avoids including Node.js and development dependencies in the final web image.

The single-stage comparison image includes Node.js, build dependencies, and an HTTP server. In the recorded comparison, the multi-stage image was 93.6 MB, compared with 1.65 GB for the single-stage image. This represents an approximate 94% reduction in image size. The smaller runtime image reduces image storage and transfer requirements.

### Service separation and Redis

Docker Compose runs the web server and Redis as separate services. Each service has its own image, process, and responsibilities. Redis has a health check using `redis-cli ping`, and the web service uses `depends_on` with the `service_healthy` condition so Redis becomes healthy before the web service starts.

This arrangement demonstrates containerization as a practical approach between traditional IaaS and managed PaaS. Like IaaS, containers provide control over application packaging and runtime configuration. Like PaaS, they simplify deployment and reduce the need to manage the underlying application environment directly. The operator still manages the container platform, images, configuration, and service availability.

The current web application is static and does not query Redis. Therefore, the web page can continue returning HTTP 200 when Redis is stopped. The startup dependency does not provide automatic recovery or guarantee Redis availability after startup.

### Local serverless execution

The `local-fn` directory contains an AWS SAM Hello World function. Running `sam local invoke HelloWorldFunction` executes the function locally through Docker, allowing its response to be tested without deploying it to AWS or requiring an AWS account.
