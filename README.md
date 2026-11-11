# Infrastructure

Part of the e-commerce microservice application (Innowise Java Advanced course).
This repository holds everything that runs the whole system: the master Docker Compose file and,
later, the Kubernetes manifests. The applications live in their own repositories; nothing is built
here, every application is an image that its CI pipeline publishes to GHCR.

[![CI](https://github.com/ArsShark/Infrastructure/actions/workflows/ci.yml/badge.svg)](https://github.com/ArsShark/Infrastructure/actions/workflows/ci.yml)

## Contents

| Path | What it is |
|---|---|
| `docker-compose.yml` | The whole system: the API Gateway, the Authentication Service, the User Service, the Order Service, their three PostgreSQL databases and Redis |
| `.env.example` | The variables of the Compose file with placeholders; copy it to `.env` |
| `.github/workflows/ci.yml` | Validates the Compose file on every push and pull request |
| `k8s/` | The Kubernetes manifests (not there yet: the Kubernetes task) |

The Payment Service, MongoDB and Kafka are added with the tasks of those services.

## Services

| Service | Compose name | Port | Networks | Published |
|---|---|---|---|---|
| API Gateway | `api-gateway` | 8080 | `backend` | yes |
| Authentication Service | `authentication-service` | 8081 | `backend`, `data` | no |
| User Service | `user-service` | 8082 | `backend`, `data` | no |
| Order Service | `order-service` | 8083 | `backend`, `data` | no |
| PostgreSQL of the Authentication Service | `auth-postgres` | 5432 | `data` | no |
| PostgreSQL of the User Service | `user-postgres` | 5432 | `data` | no |
| PostgreSQL of the Order Service | `order-postgres` | 5432 | `data` | no |
| Redis (cache of the User Service) | `redis` | 6379 | `data` | no |

Images (all from GHCR, tag `latest` by default):

| Variable | Default |
|---|---|
| `API_GATEWAY_IMAGE` | `ghcr.io/arsshark/apigateway:latest` |
| `AUTH_SERVICE_IMAGE` | `ghcr.io/arsshark/authenticationservice:latest` |
| `USER_SERVICE_IMAGE` | `ghcr.io/arsshark/userservice:latest` |
| `ORDER_SERVICE_IMAGE` | `ghcr.io/arsshark/orderservice:latest` |

## Networks

| Network | Type | Who is in it |
|---|---|---|
| `backend` | bridge | the applications; the Gateway is the only container with a published port |
| `data` | bridge, `internal: true` | the databases and Redis, and the applications that own them |

- Every request from outside enters through the Gateway (`localhost:8080`) and goes to a service by
  its Compose name over `backend`: the routes, the key set of the Authentication Service (read by the
  Gateway, the User Service and the Order Service), the call of the Authentication Service to the
  User Service during a registration, and the call of the Order Service to the User Service for
  the owner of an order.
- A service reaches its database or Redis over `data`. The network is internal: it has no route to
  the outside, and a port of a container that sits only in it cannot be published. The Gateway is
  not in it.
- The Payment Service will call an external API, so it stays in `backend`, which has a route out.
  MongoDB and Kafka will join `data`.

To look into a database from the host, add a temporary `ports` entry in a local override file; do
not commit it.

## Run the system

Requirements: Docker with Docker Compose v2.

1. Copy `.env.example` to `.env` and set the values: the three database passwords, the internal key,
   the signing key of the tokens and the first admin (`ADMIN_LOGIN` is an email). The comments in
   the file say what each value is. `.env` is ignored by Git: never commit it.

   ```powershell
   Copy-Item .env.example .env
   ```

   The signing key (PKCS#8 DER, base64, one line). With OpenSSL on Linux or macOS:

   ```bash
   openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 | openssl pkcs8 -topk8 -nocrypt -outform DER | base64 -w0
   ```

   On Windows, `openssl` of Git for Windows is often not on the PATH of PowerShell; call it with the
   full path, then delete the two files:

   ```powershell
   & "C:\Program Files\Git\usr\bin\openssl.exe" genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out key.pem
   & "C:\Program Files\Git\usr\bin\openssl.exe" pkcs8 -topk8 -nocrypt -in key.pem -outform DER -out key.der
   [Convert]::ToBase64String([IO.File]::ReadAllBytes("key.der"))
   Remove-Item key.pem, key.der
   ```

   A value with `$` or `#` goes in single quotes in `.env`.

2. Pull the images and start the system:

   ```powershell
   docker compose pull
   docker compose up
   ```

3. Check it. The first request after the start may need a few seconds, because the Gateway and the
   User Service read the key set of the Authentication Service with the first token:

   ```powershell
   curl.exe --noproxy "*" -i -X POST http://localhost:8080/api/auth/login `
     -H "Content-Type: application/json" `
     -d '{\"login\":\"<ADMIN_LOGIN>\",\"password\":\"<ADMIN_PASSWORD>\"}'
   ```

4. Stop it with `docker compose down`. The data of the databases is kept in named volumes;
   `docker compose down -v` deletes it.

A password of a database is read by PostgreSQL only when its volume is empty: after changing it, run
`docker compose down -v`. The same `INTERNAL_API_KEY` must reach the Authentication Service, the
User Service and the Order Service (it sends the key, the User Service checks it), and the same
`JWT_ISSUER` all four applications; the Compose file passes one value to each.

### Run images that you built yourself

Build an image in the repository of its service and set its name in `.env`:

| Repository | Build | Variable |
|---|---|---|
| ApiGateway | `docker build -t api-gateway:dev .` | `API_GATEWAY_IMAGE=api-gateway:dev` |
| AuthenticationService | `docker build -t authentication-service:dev .` | `AUTH_SERVICE_IMAGE=authentication-service:dev` |
| UserService | `docker build -t user-service:dev .` | `USER_SERVICE_IMAGE=user-service:dev` |
| OrderService | `docker build -t order-service:dev .` | `ORDER_SERVICE_IMAGE=order-service:dev` |

Then run `docker compose up` and do not run `docker compose pull`: the pull fails for an image that
is not in a registry. Compose pulls an image only when it is missing locally, so an old `latest`
that is already on the machine is used without a message: run `docker compose pull` (with the
default names) to refresh it.

The images on GHCR are built from `main` of each repository after a merge. A change that is merged
but not published yet (see the tab Actions of that repository) is missing from `latest`: build the
image from the branch, as above, until the publication is done.

## Postman

The collection lives in the ApiGateway repository
(`postman/ApiGateway_postman_collection.json`, only the Gateway at `http://localhost:8080`). Start the
system as above, then follow the section "Postman" of the README of that repository.

## Troubleshooting

| Symptom | Usual cause |
|---|---|
| `docker compose up` says `denied` or `unauthorized` for a `ghcr.io` image | the package of the service is private or has not been published yet: make it public (Package settings) or use your own image |
| `required variable ... is missing a value` | a value of `.env` is missing; variables set in the shell win over `.env`, and `--env-file` replaces it |
| the Authentication Service stops with `JWT_PRIVATE_KEY is not valid base64 on one line` | the placeholder of `.env.example` is still there: generate a key |
| login says 401 for the admin | `ADMIN_LOGIN` and `ADMIN_PASSWORD` were changed after the first start: the admin is created only when the login does not exist yet |
| every token is 401 and the log has `A token could not be verified` | the key set of the Authentication Service cannot be read: the service is not running or is not in the `backend` network |
| an order has `user: null` and `userInfoAvailable: false` | the Order Service cannot ask the User Service: it is not running, or `INTERNAL_API_KEY` differs between them (the User Service answers 401), or the User Service image has no `GET /internal/users/by-email` yet |
| a service cannot reach its database | the service is not in the `data` network, or the database is still starting (`docker compose ps` shows its health) |
| `curl` cannot connect to `127.0.0.1` on a proxy port | an `http_proxy` variable is set: use `curl.exe --noproxy "*"` |

Never paste the output of `docker compose config` anywhere: it prints the values of the variables,
secrets included. To check the networks:

```powershell
docker network inspect e-commerce_data --format "{{.Internal}}"
```

The answer must be `true`.

## Add a service

Each service task adds its own part here, in its own pull request (`feature/JACAG-<n>-...`):

1. A service block with the image from GHCR (`<SERVICE>_IMAGE` with a default), `SPRING_PROFILES_ACTIVE:
   docker`, and its variables; required values with `${VAR:?message}`.
2. Networks: `backend` for every application; `data` too when it owns a database, a cache or a
   broker. A database or a broker is in `data` only.
3. The variables in `.env.example` with placeholders and a comment.
4. The Gateway needs the address of the new service (for example `PAYMENT_SERVICE_URI`) only when the
   `docker` profile default of the Gateway is not right. It also needs the route and the security
   rule of the new paths: that is a pull request of the ApiGateway repository.
5. The tables of this README.

## Known limits

- `data` is one network: the Authentication Service can reach the database of the User Service and
  the other way round. The network separates the data from the outside, not the services from each
  other; a network per service would fix that.
- The passwords and keys are in `.env` and are visible to `docker inspect`: accepted for
  development, Kubernetes Secrets replace them in the Kubernetes task.
- The services have no health checks yet: probes belong to the Kubernetes task, so the Gateway
  waits only for the start of the services, not for their readiness.

## CI pipeline

`.github/workflows/ci.yml` runs on every push, on pull requests into `main` and manually. It runs
`docker compose --env-file .env.example config --quiet`: the file must stay valid, and the example
environment must have a value for every required variable. It needs no secret and starts nothing.
