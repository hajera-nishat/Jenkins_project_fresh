# Jenkins CI/CD: 3-tier Employee Registration app on EC2

A small three-tier app (Nginx frontend, Java backend, MySQL). A **Jenkins declarative pipeline** builds and tests it with Maven, builds Docker images, pushes them to Docker Hub, and deploys them to an AWS EC2 host over SSH with Docker Compose.

> **Cost warning:** the deploy target is an EC2 instance, and the Jenkins server is probably one too. Both are billed while running. Stop or terminate them when you finish.

---

## What the app does

The frontend is an "Employee Registration" form with name, email and designation fields. When you submit it, the browser sends `POST /api` with a JSON body. Nginx proxies the request to the backend, which inserts a row into the MySQL `employees` table and returns `Employee Saved Successfully`.

```
 Browser ──► frontend (nginx:alpine, :80)
               ├─ /      → serves index.html
               └─ /api   → proxy_pass http://backend:8080
                               │
                           backend (Java 17, JDK HttpServer, :8080, published on host :8081)
                               │  JDBC  jdbc:mysql://database:3306/mydb
                               ▼
                           database (mysql:8.0, :3306, volume db-data)

 All three containers share the user-defined bridge network "app-network",
 so they reach each other by service name (backend, database).
```

| Service | Image | Host port → container port |
|---|---|---|
| `frontend` | `hajeranishat11/cicd-frontend:latest` (built from `./frontend`) | `80 → 80` |
| `backend` | `hajeranishat11/cicd-backend:latest` (built from `./backend`) | `8081 → 8080` |
| `database` | `mysql:8.0` | `3306 → 3306` |

**Backend details** (`backend/src/main/java/App.java`):
- Uses the JDK's built-in `com.sun.net.httpserver.HttpServer`. There is no Spring framework.
- On startup it connects to MySQL and runs `CREATE TABLE IF NOT EXISTS employees (id, name, email, designation)`.
- It handles **POST only**. It pulls the three fields out of the JSON body with string splitting and inserts them with a `PreparedStatement`, which protects against SQL injection.
- It is packaged as a fat JAR with `maven-shade-plugin` so the MySQL JDBC driver is included. `maven-jar-plugin` sets `Main-Class: App`.

**Docker images:**
- `backend/Dockerfile` is a **multi-stage build**. Stage 1 (`maven:3.9-eclipse-temurin-17`) runs `mvn clean package`. Stage 2 (`eclipse-temurin:17-jre`) copies only `backend-app-1.0.jar`, so the final image has no Maven or JDK. The container runs `sleep 30` before starting Java to give MySQL time to start.
- `frontend/Dockerfile` uses `nginx:alpine`, replaces the default site with `index.html`, and installs a custom `nginx.conf` with the `/api` reverse proxy.

---

## The Jenkins pipeline (`Jenkinsfile`)

| Stage | What it does |
|---|---|
| Checkout Code | Prints a message only. The real checkout is done by Jenkins itself because the job is configured as *Pipeline script from SCM*. |
| Build | `mvn clean compile` in `backend/`, using the Jenkins tool named `MAVEN3`. |
| Test | `mvn test` runs the JUnit 5 tests. |
| Package | `mvn package` builds the shaded JAR. |
| Docker Build | `docker compose build` builds both images, tagged as defined in `docker-compose.yml`. |
| Push to Docker Hub | `withCredentials` loads the `docker-hub-creds` username/password and logs in with `--password-stdin`, so the password never appears in the process list. Then `docker compose push`. |
| Deploy to Remote EC2 | `sshagent` loads the `prod-ec2-key` private key. The stage creates `~/deployment` on the server, `scp`s `docker-compose.yml` there, then runs `docker compose pull && docker compose down && docker compose up -d`. |

---

## Repository structure

```
.
├── Jenkinsfile               # declarative CI/CD pipeline
├── docker-compose.yml        # frontend + backend + MySQL, network, volume
├── backend/
│   ├── Dockerfile            # multi-stage: Maven build → JRE runtime
│   ├── pom.xml               # Java 17, mysql-connector-java, JUnit 5, shade plugin
│   └── src/
│       ├── main/java/App.java    # HTTP server + JDBC insert
│       └── test/java/AppTest.java
└── frontend/
    ├── Dockerfile            # nginx:alpine with custom config
    ├── nginx.conf            # static files + /api reverse proxy to backend:8080
    └── index.html            # registration form (fetch POST /api)
```

---

## How to run

### 1. Locally with Docker Compose

```bash
docker compose up --build -d
docker compose ps
```

Wait about 30 seconds for the backend's startup delay, then open <http://localhost>. To test from the terminal:

```bash
curl -X POST http://localhost/api \
  -H "Content-Type: application/json" \
  -d '{"name":"Test User","email":"test@example.com","designation":"Engineer"}'
# → Employee Saved Successfully

docker exec -it database mysql -uroot -proot mydb -e "SELECT * FROM employees;"
```

To stop:

```bash
docker compose down        # keeps the db-data volume
docker compose down -v     # also deletes the database volume
```

### 2. Backend build and test only

```bash
cd backend
mvn clean package          # runs tests, produces target/backend-app-1.0.jar
```

### 3. Through Jenkins

**Jenkins server setup:**
1. Plugins: Pipeline, Git, **SSH Agent**, Credentials Binding.
2. *Manage Jenkins → Tools*: add a Maven installation named exactly **`MAVEN3`**.
3. Docker and the Docker Compose plugin must be installed on the Jenkins agent. The `jenkins` user must be in the `docker` group.
4. Credentials:
   - `docker-hub-creds`: *Username with password* (Docker Hub username and access token).
   - `prod-ec2-key`: *SSH Username with private key* (user `ubuntu` and the EC2 `.pem` key).
5. Create a **Pipeline** job, choose *Pipeline script from SCM*, point it at this repo, and set the script path to `Jenkinsfile`.

**EC2 deploy target setup:**
- Install Docker and the Compose plugin, and add `ubuntu` to the `docker` group.
- The security group must allow port 22 from the Jenkins server and port 80 from users.
- Update `EC2_IP` in the `Jenkinsfile` if the instance's public IP changes. The IP changes after every stop/start unless you use an Elastic IP.

---

## What I learned

- **Declarative pipeline structure:** `agent`, `tools`, `environment` and `stages`, and how a failing stage stops the pipeline.
- **Secrets handling in Jenkins:** credentials are stored in Jenkins and injected with `withCredentials` / `sshagent` instead of being written in the Jenkinsfile.
- **Multi-stage Docker builds:** the runtime image ships only a JRE and one JAR, so it is much smaller than the Maven build image.
- **Docker networking:** containers on the same user-defined bridge network resolve each other by service name. That is why the backend uses `database:3306` and Nginx uses `backend:8080`.
- **Reverse proxy:** Nginx serves static files and forwards `/api` to the backend, so the browser only talks to one origin and needs no CORS configuration.
- **`depends_on` controls start order, not readiness.** MySQL takes time to accept connections, which is why the backend has a `sleep 30`.
- **Fat JARs:** without the shade plugin, the JDBC driver is not on the classpath and the app fails with `ClassNotFoundException: com.mysql.cj.jdbc.Driver`.
- **A full CI/CD flow:** build, test, package, image, registry, remote deploy.

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Backend container exits with `Communications link failure` | MySQL was not ready within 30 s. Restart it with `docker compose restart backend`. The proper fix is a MySQL healthcheck plus `depends_on: condition: service_healthy`. |
| Form shows a 502 Bad Gateway | The backend is not up yet or has crashed. Check `docker compose logs backend`. |
| `curl http://localhost/api` (GET) hangs | Expected. The handler only answers `POST` and sends no response for other methods. |
| Jenkins: `mvn: not found` / `Tool type "maven" does not have an install of "MAVEN3"` | Configure the Maven tool with the exact name `MAVEN3`. |
| Jenkins: `permission denied ... /var/run/docker.sock` | `sudo usermod -aG docker jenkins`, then restart Jenkins. |
| Push fails with `denied: requested access to the resource is denied` | The image prefix (`hajeranishat11/`) must match the Docker Hub account in `docker-hub-creds`. |
| Deploy: `Permission denied (publickey)` | `prod-ec2-key` has the wrong key or username. It must be `ubuntu` plus the instance's `.pem`. |
| Deploy: `ssh: connect to host ... Connection timed out` | The EC2 public IP changed or the SG blocks port 22. Update `EC2_IP` and the SG. |
| `bind: address already in use` on 80/3306/8081 | Another service is using the port. Stop it or change the host port in `docker-compose.yml`. |
| `the attribute 'version' is obsolete` warning | Harmless with Compose v2. You can remove the `version:` line. |

---

## Known limitations

This is a learning project. Here is what I would change for production:

1. **Hardcoded DB credentials** (`root`/`root`) in both `App.java` and `docker-compose.yml`. They should come from environment variables, a `.env` file kept out of git, or a secrets manager. The app should also use a non-root DB user.
2. **MySQL port 3306 is published on the host.** On EC2 this exposes the database if the security group allows the port. Only the backend needs to reach it, over the Docker network.
3. **The unit test is a placeholder** (`assertTrue(true)`). It shows the Test stage working but does not test any app logic.
4. **JSON is parsed by string splitting.** Input with escaped quotes or reordered fields breaks it. A JSON library such as Jackson fixes this.
5. **Only `:latest` tags** are used, so you cannot roll back to a specific build.
6. **`StrictHostKeyChecking=no`** skips SSH host verification.
7. **The JAR is built twice:** once by Maven on Jenkins, and again inside the Docker build.
8. **`sleep 30`** is a timing workaround, not a readiness check.
9. **The EC2 IP is hardcoded** in the Jenkinsfile.

## Next steps

- [ ] Add a MySQL `healthcheck` and use `depends_on: condition: service_healthy`, then remove `sleep 30`.
- [ ] Move DB credentials to environment variables and a git-ignored `.env`, and stop publishing port 3306.
- [ ] Tag images with `${BUILD_NUMBER}` or the git SHA as well as `latest`.
- [ ] Write real tests, for example for the request-parsing logic, and publish JUnit reports in Jenkins.
- [ ] Add a `GET /health` endpoint and a post-deploy smoke test (`curl -f`) in the pipeline.
- [ ] Add a `post { always { sh 'docker logout' } }` block and a failure notification.
- [ ] Add `target/` to `.gitignore`.
- [ ] Rebuild the same pipeline in GitHub Actions to compare.
