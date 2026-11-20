# Users Management System — Build & Deploy Guide

End-to-end commands for building the Spring Boot backend, containerizing it with Docker, running it against MySQL, deploying to Kubernetes (Minikube), and keeping the repo in sync.

> **Assumptions**
> - You are in the **backend/** directory (`pom.xml` present) for Maven/Docker commands.
> - Docker image names: `student-management:1.0` (backend), `student-frontend:1.0` (frontend).
> - Kubernetes namespace: `default` (all `kubectl` commands below assume this).
> - Minikube driver: `docker` (adjust if you use `kvm2`, `hyperkit`, etc.).

---

## Table of Contents

1. Build with Maven
2. Run Locally (without Docker)
3. Docker — Build & Push
4. MySQL Container
5. Docker Network Setup
6. Run the Backend Container
7. Kubernetes — Minikube
8. Kubernetes — Deploy & Test
9. Kubernetes — Cleanup
10. Git — Sync with Remote
11. Firewall (UFW)
12. Troubleshooting

---

## 1. Build with Maven

Build the Spring Boot jar. Output: `target/usersmanagementsystem-*.jar`.

    # Full build (runs tests)
    # WHY: produces a runnable jar in target/ — required before Docker build
    mvn clean package

    # Build without running tests (faster, for CI or quick iterations)
    # WHY: skips test phase to save ~30-60s during rapid iteration
    mvn clean package -DskipTests

Optional:

    # Compile only (no jar) — WHY: quick syntax check without packaging
    mvn clean compile

    # Run tests only — WHY: verify logic before building image
    mvn test

    # Install into local ~/.m2 — WHY: needed if other local modules depend on it
    mvn clean install

    # Show dependency tree — WHY: debug classpath / version conflicts
    mvn dependency:tree

---

## 2. Run Locally (without Docker)

Quickest way to smoke-test before containerizing.

    # Option A: Maven plugin (no jar needed)
    # WHY: fastest dev loop — no build/package step
    mvn spring-boot:run

    # Option B: Run the built jar
    # WHY: mimics production startup
    java -jar target/usersmanagementsystem-0.0.1-SNAPSHOT.jar

    # Option C: Override DB URL via CLI
    # WHY: point at a different MySQL without editing application.properties
    java -jar target/usersmanagementsystem-0.0.1-SNAPSHOT.jar \
      --spring.datasource.url=jdbc:mysql://localhost:3306/users_management

API will be available at `http://localhost:8080`.

---

## 3. Docker — Build & Push

### 3.1 Build the image

    # WHY: packages jar + JRE into a portable image
    docker build -t student-management:1.0 .

Run from the directory containing the `Dockerfile`. Verify the Dockerfile copies the jar from `target/` (multi-stage build) or expects it present (`COPY target/*.jar app.jar`). If you built with `-DskipTests`, the jar exists.

### 3.2 Tag for Docker Hub

    # WHY: gives the image a registry-qualified name so `push` knows where to send it
    docker tag student-management:1.0 callmee6/student-management:1.0

### 3.3 Log in and push

    # WHY: authenticates to Docker Hub (one-time per session, or after token expires)
    docker login

    # WHY: uploads the image so other machines / clusters can pull it
    docker push callmee6/student-management:1.0

Verify:

    # WHY: confirms the image exists locally with the expected tag
    docker images | grep student-management

    # WHY: sanity-check the container starts (skip if your app needs a DB to boot)
    docker run --rm student-management:1.0 --version

### 3.4 (Optional) Frontend image

    cd ../frontend

    # WHY: same build/tag/push flow for the React frontend
    docker build -t student-frontend:1.0 .
    docker tag student-frontend:1.0 callmee6/student-frontend:1.0
    docker push callmee6/student-frontend:1.0

---

## 4. MySQL Container

The backend expects MySQL on port `3306`. If you don't already have a container:

### 4.1 Create and run MySQL

    # WHY: persistent, named MySQL instance the backend can reach
    # -e MYSQL_DATABASE: auto-creates the schema on first boot
    # -v mysql-data: persists data across container restarts
    docker run -d \
      --name mysql \
      --restart unless-stopped \
      -e MYSQL_ROOT_PASSWORD=root \
      -e MYSQL_DATABASE=users_management \
      -p 3306:3306 \
      -v mysql-data:/var/lib/mysql \
      mysql:8

### 4.2 Open a MySQL shell

    # WHY: interactive SQL prompt for debugging / manual queries
    docker exec -it mysql mysql -uroot -p
    # then enter the root password (e.g. "root")

### 4.3 Start / stop / inspect

    # WHY: control container lifecycle without recreating it
    docker start mysql
    docker stop mysql

    # WHY: tail logs to see startup errors / connection issues
    docker logs mysql

    # WHY: full config incl. IP, volumes, env — useful when debugging networking
    docker inspect mysql

### 4.4 Create the database (if not created via env var)

    # WHY: idempotent — safe to run even if it exists
    CREATE DATABASE IF NOT EXISTS users_management;
    SHOW DATABASES;

---

## 5. Docker Network Setup

Containers on the **default bridge** can't resolve each other by name. Create a user-defined network so `student-management` can reach `mysql` by hostname.

### 5.1 Create the network

    # WHY: gives containers DNS — they can resolve each other by name, not just IP
    docker network create student-network

### 5.2 Connect MySQL

    # WHY: attaches the already-running MySQL container to the new network
    docker network connect student-network mysql

### 5.3 Verify

    # WHY: confirms `mysql` is listed under "Containers" for student-network
    docker network inspect student-network

If MySQL was created *after* the network, use `docker network connect` as above. If it was created *on* the network with `--network student-network`, this step is a no-op.

---

## 6. Run the Backend Container

### 6.1 Remove the old container (if any)

    # WHY: can't reuse a name; remove old container before recreating
    docker stop student-management
    docker rm student-management

### 6.2 Recreate on the new network

    # WHY: backend joins student-network so it can reach `mysql` by hostname
    # -e SPRING_DATASOURCE_*: overrides application.properties via env
    # --restart unless-stopped: survives host reboot
    docker run -d \
      --name student-management \
      --restart unless-stopped \
      --network student-network \
      -p 8082:8082 \
      -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/users_management \
      -e SPRING_DATASOURCE_USERNAME=root \
      -e SPRING_DATASOURCE_PASSWORD=root \
      student-management:1.0

**⚠️ Port mismatch:** the command maps `8082:8082`, but the backend defaults to `8080`. Either:
- change the mapping to `-p 8082:8080` (host 8082 → container 8080), **or**
- set `server.port=8082` in `application.properties` / via `-e SERVER_PORT=8082`.

The frontend's `UserService.BASE_URL` must match whichever host port you expose.

### 6.3 Verify

    # WHY: confirms container is running (STATUS column = Up)
    docker ps

    # WHY: check for startup exceptions (DB connection, port conflicts)
    docker logs student-management

    # WHY: end-to-end smoke test — if this returns a token, everything wired up
    curl http://localhost:8082/auth/login -X POST \
      -H "Content-Type: application/json" \
      -d '{"email":"admin@example.com","password":"admin123"}'

### 6.4 Stop / restart / remove

    # WHY: common lifecycle ops without losing the image
    docker stop student-management
    docker start student-management
    docker restart student-management

    # WHY: -f removes even if running (use with care)
    docker rm -f student-management

---

## 7. Kubernetes — Minikube

### 7.1 Start Minikube

    # WHY: spins up a single-node cluster in a VM/container
    minikube start
    minikube status

### 7.2 Point Docker at Minikube's daemon (only if building inside Minikube)

    # WHY: subsequent `docker build` goes into Minikube's image store,
    #      so K8s can find the image without `image load`
    eval $(minikube docker-env)

    # Undo with: eval $(minikube docker-env -u)

### 7.3 Load images into Minikube (recommended over `docker-env`)

If you built images with your **host** Docker, load them into Minikube:

    # WHY: K8s can only pull images that live in its node's image store
    minikube image load student-management:1.0
    minikube image load student-frontend:1.0

Verify images are visible:

    # WHY: confirms both images are present in Minikube's store
    minikube image ls | grep student

Remove an image from Minikube (when you rebuild):

    # WHY: frees space and forces K8s to use the freshly loaded image
    minikube image rm student-management:1.0

### 7.4 Apply manifests

    # WHY: creates/updates all resources defined in each YAML
    kubectl apply -f mysql-service.yaml
    kubectl apply -f mysql-deployment.yaml        # if separate
    kubectl apply -f student-management-deployment.yaml
    kubectl apply -f student-management-service.yaml
    kubectl apply -f student-frontend-deployment.yaml
    kubectl apply -f student-frontend-service.yaml

Or apply everything in a directory:

    # WHY: bulk apply — every *.yaml in k8s/ gets applied
    kubectl apply -f k8s/

Useful `kubectl` shortcuts:

    # WHY: quick visibility into cluster state
    kubectl get pods                    # list pods
    kubectl get svc                     # list services
    kubectl get deployments             # list deployments
    kubectl get all                     # everything

    # WHY: debug a misbehaving pod (events, status, container state)
    kubectl describe pod <pod-name>

    # WHY: see stdout/stderr from the app inside a pod
    kubectl logs <pod-name>
    kubectl logs -f <pod-name>          # follow logs

    # WHY: shell into a pod to poke around the container filesystem
    kubectl exec -it <pod-name> -- sh

---

## 8. Kubernetes — Deploy & Test

### 8.1 Get the Minikube IP

    # WHY: the IP you'll hit from the host to reach NodePort services
    minikube ip

Expected (default docker driver): `192.168.49.2`

### 8.2 Check the NodePort

    # WHY: reveals which host port maps to your service
    kubectl get svc student-management-service

Example output:

    NAME                          TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
    student-management-service    NodePort   10.96.123.45    <none>        8082:30082/TCP   5m

- `PORT(S)` = `8082:30082/TCP` → **NodePort `30082`**
- Full URL: `http://$(minikube ip):30082`

### 8.3 Test the endpoints

    # WHY: capture the IP once, reuse in commands below
    MINIKUBE_IP=$(minikube ip)

    # WHY: end-to-end auth check against the K8s-deployed backend
    curl -X POST http://$MINIKUBE_IP:30082/auth/login \
      -H "Content-Type: application/json" \
      -d '{"email":"admin@example.com","password":"admin123"}'

    # WHY: authenticated request — proves JWT filter + DB round-trip work in K8s
    curl http://$MINIKUBE_IP:30082/admin/get-all-users \
      -H "Authorization: Bearer <token>"

### 8.4 Port-forward (alternative to NodePort)

    # WHY: tunnels a local port directly to the service — no NodePort needed
    kubectl port-forward svc/student-management-service 8082:8082
    # Then: curl http://localhost:8082/...

### 8.5 Rebuild & redeploy after code change

    # WHY: K8s caches images; you must remove + reload + restart to pick up new build
    mvn clean package -DskipTests
    docker build -t student-management:1.0 .
    minikube image rm student-management:1.0
    minikube image load student-management:1.0
    kubectl rollout restart deployment student-management-deployment

    # WHY: block until rollout completes (helps catch CrashLoopBackOff quickly)
    kubectl rollout status deployment student-management-deployment

### 8.6 Scale / rollout

    # WHY: horizontal scale — adds/removes pods
    kubectl scale deployment student-management-deployment --replicas=2

    # WHY: see past revisions (for rollback decisions)
    kubectl rollout history deployment student-management-deployment

    # WHY: one-command rollback to previous revision
    kubectl rollout undo deployment student-management-deployment

---

## 9. Kubernetes — Cleanup

### 9.1 Delete individual resources

    # WHY: targeted deletion — useful when debugging one component
    kubectl delete pod <pod-name>
    kubectl delete service <service-name>
    kubectl delete deployment student-management-deployment
    kubectl delete service student-management-service

### 9.2 Delete everything at once

    # WHY: bulk teardown — one command instead of many
    kubectl delete -f k8s/
    # or
    kubectl delete all --all

### 9.3 Remove images from Minikube

    # WHY: free disk space; also forces re-load after a rebuild
    minikube image rm student-management:1.0
    minikube image rm student-frontend:1.0

### 9.4 Stop / delete Minikube cluster

    # WHY: stop keeps state (VM preserved); delete wipes everything
    minikube stop
    minikube delete

---

## 10. Git — Sync with Remote

If `git push` fails because the remote has commits you don't have locally:

    # WHY: rebase replays your local commits on top of remote — linear history
    git pull --rebase origin main

    # Resolve conflicts if any, then:
    git rebase --continue

    # WHY: publish local commits
    git push origin main

Other useful Git commands:

    # WHY: daily hygiene
    git status                         # what's changed
    git log --oneline --graph --all    # visual history
    git fetch origin                   # get remote refs without merging
    git branch -a                      # list all branches
    git stash                          # park uncommitted work
    git stash pop                      # restore it

**⚠️ Never** `git push --force` on `main` after a rebase unless you are sure no one else has pulled it.

---

## 11. Firewall (UFW)

If the NodePort is not reachable from another machine on the same LAN (e.g. Fedora host with UFW enabled), allow traffic from your LAN subnet:

    # WHY: opens the NodePort to the LAN so other devices can hit the app
    sudo ufw allow from 192.168.31.0/24 to any port 30080 proto tcp

    # WHY: confirm rule was added; note the rule number for removal
    sudo ufw status numbered

    # WHY: targeted removal (use the number from `status numbered`)
    sudo ufw delete <rule-number>

Adjust the subnet to your actual LAN (`ip a` → look at your `wlan0`/`eth0`). If using the default Docker driver on Minikube, the NodePort is bound to the **Minikube VM's IP** (`minikube ip`), not the host — use `kubectl port-forward` or `minikube service <name>` if the host firewall is in the way.

---

## 12. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `ClassNotFoundException: …Application` | Build didn't compile the class | `mvn clean package` (Section 1) |
| Backend exits with `Access denied for user` | Wrong DB creds in env | Check `SPRING_DATASOURCE_*` env vars (Section 6.2) |
| Backend exits with `Communications link failure` | MySQL unreachable by hostname | Both containers on `student-network` (Section 5) |
| `connection refused` from frontend | Port mapping mismatch (`8082:8082` vs `8082:8080`) | Fix `-p` mapping or set `server.port=8082` |
| Pod stuck in `ErrImagePull` / `ImagePullBackOff` | Image not loaded in Minikube | `minikube image load <name>:<tag>` (Section 7.3) |
| Pod in `CrashLoopBackOff` | App crash — check logs | `kubectl logs <pod> --previous` |
| `NodePort` not reachable from host | Minikube driver binds inside VM | `minikube service <name>` or `kubectl port-forward` |
| `git push` rejected (non-fast-forward) | Remote has commits you don't | `git pull --rebase origin main` (Section 10) |
| CORS error in browser | Frontend origin not allowed | Edit `CorsConfig.java`, rebuild, redeploy |
| `401 Unauthorized` on `/admin/**` | Token missing/invalid or role ≠ `ADMIN` | Re-login, ensure `role` in JWT is `ADMIN` |

General debugging checklist:

    # Docker — WHY: is the container even running? what do logs say?
    docker ps -a
    docker logs <container>
    docker inspect <container>
    docker network inspect student-network

    # Kubernetes — WHY: what's the cluster's view? what events happened?
    kubectl get all
    kubectl describe pod <pod>
    kubectl logs <pod> --previous
    kubectl get events --sort-by=.lastTimestamp

---

## Quick Reference Card

    # ── Build ──────────────────────────────────────────────
    # WHY: fastest path from source → runnable jar
    mvn clean package -DskipTests

    # ── Docker ─────────────────────────────────────────────
    # WHY: build → tag → push, the canonical publish flow
    docker build -t student-management:1.0 .
    docker tag student-management:1.0 callmee6/student-management:1.0
    docker push callmee6/student-management:1.0

    # ── Run (local Docker) ─────────────────────────────────
    # WHY: network + connect + run — the minimum to get backend talking to MySQL
    docker network create student-network
    docker network connect student-network mysql
    docker run -d --name student-management --restart unless-stopped \
      --network student-network -p 8082:8082 student-management:1.0

    # ── Kubernetes ─────────────────────────────────────────
    # WHY: start cluster → load image → apply manifests → restart to pick up new build
    minikube start
    minikube image load student-management:1.0
    kubectl apply -f k8s/
    minikube ip
    kubectl get svc
    kubectl rollout restart deployment student-management-deployment

    # ── Cleanup ────────────────────────────────────────────
    # WHY: full teardown — K8s resources, then Minikube image, then host container
    kubectl delete -f k8s/
    minikube image rm student-management:1.0
    docker rm -f student-management

---

## Related

- **Backend README:** `backend/README.md` (API reference, project structure)
- **Frontend README:** `frontend/README.md` (if present)
