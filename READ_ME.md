# SOFTWARE ENGINEERING LAB — MASTER EXAM REVISION
## Docker Compose → Jenkins → Kubernetes + Nagios (Docker) → AWS EC2

**Purpose:** One practical recovery guide for lab exams. Follow the section matching the question paper. Commands use Windows CMD/PowerShell unless marked **Linux/EC2**.

> **Exam rule:** First identify the exact requirement in the question. Do not blindly run every section. Replace clearly marked placeholders such as `YOUR_PUBLIC_IP`, `YOUR_USERNAME`, or `POD_NAME`. Never commit passwords, private keys, or access tokens.

---

# CONTENTS

1. [Quick workflow map](#1-quick-workflow-map)
2. [Docker and Docker Compose — Sets 1 to 9 + OLTRS](#2-docker-and-docker-compose--sets-1-to-9--oltrs)
3. [Jenkins Scripted Pipeline — OLTRS](#3-jenkins-scripted-pipeline--oltrs)
4. [Kubernetes + Nagios using Docker — Set 7](#4-kubernetes--nagios-using-docker--set-7)
5. [AWS EC2 deployment — Apartment System](#5-aws-ec2-deployment--apartment-system)
6. [Last-minute viva answers](#6-last-minute-viva-answers)
7. [Troubleshooting cheat sheet](#7-troubleshooting-cheat-sheet)
8. [Final exam checklist](#8-final-exam-checklist)

---

# 1. QUICK WORKFLOW MAP

| Practical | Main flow |
|---|---|
| Docker Compose | Write `docker-compose.yml` → `docker compose config` → `docker compose up -d` → verify → test |
| Jenkins | GitHub checkout → Maven build/test → Docker build → Docker Hub push → webhook |
| Kubernetes | Start Minikube → create Deployment → 4 replicas → expose Service → delete Pod and verify recovery → scale |
| Nagios | Pull Nagios image → run container → open web UI → verify container/logs |
| AWS | Launch EC2 → configure security group → SSH → install Docker/Git → clone/build/run app → browser test |

**Important distinctions**
- Docker image = application/package template; container = running instance.
- Docker Compose manages multiple related containers.
- Jenkins automates build and delivery.
- Kubernetes manages containerized workloads and self-healing.
- Nagios monitors hosts/services when checks are configured.
- AWS EC2 is a cloud virtual machine.
- A Docker container running Nagios is not automatically monitoring Kubernetes. Configure actual Nagios checks if integration is required.

---

# 2. DOCKER AND DOCKER COMPOSE — SETS 1 TO 9 + OLTRS

## 2.1 Common Docker commands

Check Docker:

```bash
docker --version
docker info
```

List images and containers:

```bash
docker images
docker ps
docker ps -a
```

Run a test container:

```bash
docker run -d --name testweb -p 8080:80 nginx
```

Open `http://localhost:8080`.

Useful commands:

```bash
docker logs testweb
docker stop testweb
docker start testweb
docker restart testweb
docker rm -f testweb
docker rmi IMAGE_NAME
```

Build an image from the current folder and run it:

```bash
docker build -t myapp:latest .
docker run -d --name myapp -p 8080:8080 myapp:latest
```

## 2.2 Common Docker Compose workflow

The Compose file must be named exactly `docker-compose.yml` (or `compose.yaml`).

Run these commands in the folder containing the file:

```bash
docker compose config
docker compose up -d
docker compose ps
docker compose logs
docker compose logs -f
docker compose logs db
docker compose images
```

Lifecycle commands:

```bash
docker compose stop
docker compose start
docker compose restart
docker compose down
docker compose down -v
docker compose up -d --build
```

**Remember**
- `up -d` creates/starts services in the background.
- `stop` stops containers but keeps them.
- `start` starts stopped containers.
- `restart` restarts services.
- `down` removes containers and the default network.
- `down -v` also deletes named volumes and their stored data. Use only when the question asks for data removal/reset.
- `--build` rebuilds images before starting.
- `HOST_PORT:CONTAINER_PORT`; example `8081:80`.
- Containers communicate using service names, e.g. `db:5432`, not `localhost`.
- `depends_on` controls startup order; it does not guarantee that the database is ready.
- Do not use a fixed `container_name` when the question requires scaling a service.

## 2.3 Set identification table

| Set | Application / required stack | Main things to demonstrate |
|---|---|---|
| Set 1 | WordPress + MySQL 5.7 | WordPress at `localhost:8081`, database persistence |
| Set 2 | NGINX + PostgreSQL; Redis scenario | Compose services, dependency, Redis addition, scaling/rebuild |
| Set 3 | Flask application(s) | Change displayed message, build image, map host ports to container port 5000 |
| Set 4 | NGINX reverse proxy + Tomcat 9 | Proxy configuration, custom network, troubleshoot 502, host-port change |
| Set 5 | Frontend + PostgreSQL | Custom network, persistent volume, image export/import |
| Set 6 | Tomcat 9 + MySQL 5.7 | Application/database connection, persistent volume, troubleshoot connection refused |
| Set 7 | Apartment frontend + PostgreSQL 15 | Compose 3.9 format, network, depends_on, rebuild, volume recovery |
| Set 8 | WordPress + MySQL 5.7 | WordPress at `localhost:8081`, environment variables, volumes/lifecycle |
| Set 9 | Flask + PostgreSQL + Redis | Custom network, volumes, dependencies, rebuild/scaling |
| OLTRS | User's Docker Hub web image + MySQL | Pull/run web image, connect it to MySQL using the application's actual settings |

**Set correction:** Set 7 is Apartment System. Set 9 is Sports System.

> The examples below are practical templates. Where a paper supplies its own application image, source code, WAR file, database variables, or port, use those exact values. A plain NGINX container is only a frontend demonstration; it does not magically implement database connectivity.

## 2.4 Set 1 — WordPress + MySQL 5.7

Create `docker-compose.yml`:

```yaml
services:
  db:
    image: mysql:5.7
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
      MYSQL_ROOT_PASSWORD: rootpass
    volumes:
      - dbdata:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    ports:
      - "8081:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress
    depends_on:
      - db

volumes:
  dbdata:
```

Run:

```bash
docker compose up -d
docker compose ps
```

Open `http://localhost:8081`.

Persistence test:
1. Create a WordPress post.
2. Run `docker compose down`.
3. Run `docker compose up -d`.
4. Open WordPress and check the post remains.

The named volume `dbdata` stores MySQL data.

## 2.5 Set 2 — NGINX + PostgreSQL (+ Redis scenario)

Create `docker-compose.yml`:

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: sampledb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:alpine

volumes:
  pgdata:
```

Run:

```bash
docker compose up -d
docker compose ps
```

Open `http://localhost:8080`.

Add Redis by adding the `redis` service above, then run:

```bash
docker compose up -d
docker compose ps
```

Scale a service when the question asks:

```bash
docker compose up -d --scale web=2
```

If scaling fails, check for fixed `container_name` or a host-port mapping that every replica tries to claim. Remove the fixed container name and avoid publishing the same host port for every replica.

## 2.6 Set 3 — Flask application / two Flask instances

Create `app.py`:

```python
from flask import Flask
import os

app = Flask(__name__)

@app.route("/")
def home():
    return os.getenv("MESSAGE", "Hello from Flask")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

Create `requirements.txt`:

```text
flask
```

Create `Dockerfile`:

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```

Create `docker-compose.yml`:

```yaml
services:
  app1:
    build: .
    ports:
      - "5001:5000"
    environment:
      MESSAGE: "Hello from Flask App 1"

  app2:
    build: .
    ports:
      - "5002:5000"
    environment:
      MESSAGE: "Hello from Flask App 2"
```

Run:

```bash
docker compose up -d --build
docker compose ps
```

Open:
- `http://localhost:5001`
- `http://localhost:5002`

Change the message in the Compose file and rebuild:

```bash
docker compose up -d --build
```

## 2.7 Set 4 — NGINX reverse proxy + Tomcat 9

Create `nginx.conf`:

```nginx
events {}

http {
    server {
        listen 80;

        location / {
            proxy_pass http://tomcat:8080;
        }
    }
}
```

Create `docker-compose.yml`:

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "8081:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - tomcat

  tomcat:
    image: tomcat:9.0
    ports:
      - "8082:8080"
```

Run:

```bash
docker compose up -d
docker compose ps
docker compose logs nginx
docker compose logs tomcat
```

Open:
- NGINX proxy: `http://localhost:8081`
- Tomcat directly: `http://localhost:8082`

If the question supplies a WAR file, mount/copy that actual WAR into Tomcat's `webapps` folder. The stock Tomcat image may show its default page or an empty application list depending on image version.

**502 troubleshooting**
- Check `docker compose ps`.
- Check `docker compose logs nginx tomcat`.
- Confirm both services share the Compose network.
- Use `tomcat:8080` as the upstream hostname, not `localhost:8080`.
- Confirm Tomcat is running and the app is deployed.

If host port 80 is busy, change only the host side, e.g. `8081:80`. NGINX still listens on container port 80.

## 2.8 Set 5 — Frontend + PostgreSQL

Basic Compose template:

```yaml
services:
  frontend:
    image: nginx:alpine
    ports:
      - "8080:80"
    depends_on:
      - db
    networks:
      - appnet

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: bloodbank
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - appnet

networks:
  appnet:

volumes:
  pgdata:
```

Run:

```bash
docker compose up -d
docker compose ps
```

**Note:** This demonstrates the frontend and database containers, network, and persistent storage. Replace the NGINX frontend with the actual supplied application if it must communicate with PostgreSQL.

Export an image:

```bash
docker save -o frontend.tar IMAGE_NAME
```

Import an image:

```bash
docker load -i frontend.tar
docker images
```

Use the actual image name in place of `IMAGE_NAME`.

## 2.9 Set 6 — Tomcat 9 + MySQL 5.7

Create `docker-compose.yml`:

```yaml
services:
  tomcat:
    image: tomcat:9.0
    ports:
      - "8080:8080"
    depends_on:
      - db

  db:
    image: mysql:5.7
    environment:
      MYSQL_DATABASE: boutique
      MYSQL_USER: shopuser
      MYSQL_PASSWORD: shoppass
      MYSQL_ROOT_PASSWORD: rootpass
    volumes:
      - mysqldata:/var/lib/mysql

volumes:
  mysqldata:
```

Run:

```bash
docker compose up -d
docker compose ps
docker compose logs db
docker compose logs tomcat
```

Open `http://localhost:8080`.

If the actual application must connect to MySQL, its JDBC connection should use the Compose service hostname:

```text
jdbc:mysql://db:3306/boutique
```

The application also needs a compatible MySQL JDBC driver and the correct credentials. Compose environment variables create/configure the database; they do not automatically reconfigure an arbitrary WAR file.

**Connection refused troubleshooting**
- Confirm database container is running.
- Inspect `docker compose logs db`.
- Use hostname `db`, not `localhost`.
- Check port 3306 and credentials.
- Ensure the application waits/retries until the database is ready.

## 2.10 Set 7 — Apartment System + PostgreSQL 15

Template for the required frontend/database services:

```yaml
version: "3.9"

services:
  frontend:
    image: nginx:alpine
    ports:
      - "8080:80"
    depends_on:
      - db
    networks:
      - apartmentnet

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: apartment
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
    volumes:
      - apartmentdata:/var/lib/postgresql/data
    networks:
      - apartmentnet

networks:
  apartmentnet:

volumes:
  apartmentdata:
```

Run:

```bash
docker compose config
docker compose up -d
docker compose ps
```

Open `http://localhost:8080`.

Rebuild/recreate after changes:

```bash
docker compose up -d --build
```

Volume recovery demonstration:
1. Add data using the actual application/database.
2. Run `docker compose down`.
3. Run `docker compose up -d`.
4. Verify that data is still present.

Do not use `docker compose down -v` if the task is to demonstrate persistence.

**Important:** NGINX here is a placeholder frontend. Use the actual Apartment System image/source from the question if it is provided and configure its real database connection variables.

## 2.11 Set 8 — WordPress + MySQL 5.7

Use the Set 1 Compose file and workflow:
- MySQL image: `mysql:5.7`
- WordPress image: `wordpress:latest`
- Browser: `http://localhost:8081`
- Database volume: `dbdata`

```bash
docker compose up -d
docker compose ps
docker compose logs
```

To stop and restart without deleting data:

```bash
docker compose stop
docker compose start
```

To remove containers but preserve named volumes:

```bash
docker compose down
```

Only use `docker compose down -v` when the task explicitly asks to delete stored database data.

## 2.12 Set 9 — Flask + PostgreSQL + Redis

Use the Flask Dockerfile from Set 3. Create this `docker-compose.yml`:

```yaml
services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      DB_HOST: db
      DB_NAME: sports
      DB_USER: appuser
      DB_PASSWORD: apppass
      REDIS_HOST: redis
    depends_on:
      - db
      - redis
    networks:
      - appnet

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: sports
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - appnet

  redis:
    image: redis:alpine
    volumes:
      - redisdata:/data
    networks:
      - appnet

networks:
  appnet:

volumes:
  pgdata:
  redisdata:
```

Run:

```bash
docker compose up -d --build
docker compose ps
docker compose logs
```

Open `http://localhost:5000`.

**Important:** This Compose file provides the services and environment variables. The Flask application must actually read those variables and use PostgreSQL/Redis libraries and connection code to demonstrate real database/cache operations. `depends_on` alone does not create application integration.

## 2.13 OLTRS — Pull and run a Docker Hub application image

Use the exact image, ports, and database environment variable names specified by the OLTRS application/question.

Example template only:

```yaml
services:
  web:
    image: YOUR_USERNAME/oltrs:latest
    ports:
      - "8080:8080"
    environment:
      DB_HOST: db
      DB_NAME: oltrs
      DB_USER: appuser
      DB_PASSWORD: apppass
    depends_on:
      - db

  db:
    image: mysql:5.7
    environment:
      MYSQL_DATABASE: oltrs
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppass
      MYSQL_ROOT_PASSWORD: rootpass
    volumes:
      - oltrsdata:/var/lib/mysql

volumes:
  oltrsdata:
```

Pull and run:

```bash
docker login
docker compose pull
docker compose up -d
docker compose ps
docker compose logs
```

Open the actual mapped application port, e.g. `http://localhost:8080`.

Replace `YOUR_USERNAME/oltrs:latest` and the environment variables with the actual image name and application configuration. Do not assume every OLTRS image uses `DB_HOST` or port 8080.

## 2.14 Docker Compose troubleshooting and viva

| Problem | What to check |
|---|---|
| Port already allocated | Change host port, e.g. `8081:80` |
| YAML error | Correct indentation; use spaces, not tabs |
| File not found | Confirm exact filename `docker-compose.yml` and current folder |
| Container exits | `docker compose ps -a` and `docker compose logs SERVICE` |
| Database connection refused | Service name, port, credentials, readiness |
| Changes not visible | `docker compose up -d --build` |
| Data disappeared | Check whether `down -v` deleted the volume |
| NGINX 502 | Check upstream service name, network, port and logs |

Quick viva:
- Docker: platform for building and running containers.
- Image: read-only template used to create containers.
- Container: running or stopped instance of an image.
- Dockerfile: instructions to build an image.
- Compose: tool to define and run multi-container applications.
- Volume: persistent storage managed by Docker.
- Network: enables container communication.
- Port mapping: connects host port to container port.
- `depends_on`: startup dependency ordering, not full readiness.
- `docker compose up -d`: starts services in background.

---

# 3. JENKINS SCRIPTED PIPELINE — OLTRS

## 3.1 Actual values used

| Item | Value |
|---|---|
| Jenkins URL | `http://localhost:8081` |
| Job name | `selab` |
| GitHub repository | `https://github.com/archanareddyse/OLTRS.git` |
| Branch | `master` |
| Docker Hub username | `dockerseexercise` |
| Image | `dockerseexercise/oltrs:latest` |
| Jenkins credential ID | `dockerhub` |

Required flow:

**GitHub Checkout → Maven Build/Test → Dockerfile → Docker Build → Docker Hub Push**

## 3.2 Create/open the Pipeline job

1. Open `http://localhost:8081`.
2. Open job `selab`.
3. Click **Configure**.
4. Scroll to **Pipeline**.
5. Set Definition to **Pipeline script**.
6. Keep **Use Groovy Sandbox** checked.
7. Paste the pipeline code below.
8. Click **Save**.

For a new job: Dashboard → New Item → job name → Pipeline → OK.

## 3.3 Add Docker Hub credentials

Go to:

Manage Jenkins → Credentials → System → Global credentials (unrestricted) → Add Credentials.

Enter:
- Kind: Username with password
- Scope: Global
- Username: `dockerseexercise`
- Password: Docker Hub access token (not account password)
- ID: `dockerhub`
- Description: Docker Hub credentials

Click Create.

Never put the token in the Jenkinsfile or Git repository.

## 3.4 Scripted Pipeline code

```groovy
node {
    def imageName = "dockerseexercise/oltrs:latest"

    stage('Checkout') {
        git branch: 'master',
            url: 'https://github.com/archanareddyse/OLTRS.git'
    }

    stage('Build and Test') {
        bat 'mvn -B clean verify'
    }

    stage('Create Dockerfile') {
        writeFile file: 'Dockerfile', text: '''
FROM tomcat:9.0
COPY target/ticket-reservation.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
'''
    }

    stage('Docker Build') {
        bat "docker build -t ${imageName} ."
    }

    stage('Docker Push') {
        withCredentials([usernamePassword(
            credentialsId: 'dockerhub',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )]) {
            bat 'echo %DOCKER_PASS%| docker login -u %DOCKER_USER% --password-stdin'
            bat "docker push ${imageName}"
            bat 'docker logout'
        }
    }
}
```

**Important assumptions**
- This pipeline uses Windows CMD through Jenkins `bat`.
- Maven command: `mvn -B clean verify`.
- `clean verify` builds the Maven project and runs its configured tests.
- A failed command stops the sequential pipeline; later stages do not run.
- The Dockerfile is created in the Jenkins workspace.
- The Dockerfile assumes Maven creates `target/ticket-reservation.war`.
- Tomcat 9 must be compatible with the application's Java/Jakarta requirements.
- Docker must be installed and available to the Jenkins Windows service account.
- If the actual WAR filename differs, change the `COPY` source accordingly.

## 3.5 Run and verify

1. Open Jenkins → `selab`.
2. Click **Build Now**.
3. Open the latest build number.
4. Open **Pipeline Overview**.
5. Open **Console Output** for detailed logs.

Expected stages:
1. Checkout
2. Build and Test
3. Create Dockerfile
4. Docker Build
5. Docker Push

Expected final status: **SUCCESS**.

Verify in Docker Hub:
1. Open `https://hub.docker.com/`.
2. Sign in.
3. Open Repositories.
4. Find `oltrs` under `dockerseexercise`.

## 3.6 Actual error encountered: Maven not recognized

Build #2 checked out the GitHub repository and branch `master` successfully, but failed at Build and Test:

```text
'mvn' is not recognized as an internal or external command
```

Maven was installed on the laptop, but the Jenkins Windows service process did not have Maven available in its PATH.

Normal CMD check:

```bash
mvn --version
```

Previously observed:
- Apache Maven 3.9.16
- Maven home: `C:\Program Files\Apache maven\apache-maven-3.9.16`
- Java version 21.0.11

Fix:
1. Add this Maven `bin` directory to the **Windows System PATH**:

```text
C:\Program Files\Apache maven\apache-maven-3.9.16\bin
```

2. Ensure Java is available to the Jenkins service too.
3. Restart the Jenkins Windows service after changing system environment variables.
4. Run Build Now again.
5. Inspect the latest Console Output.

Do not reinstall Maven if it is already installed.

Useful checks:

```bash
java -version
mvn --version
git --version
docker --version
docker images
docker ps
```

## 3.7 GitHub webhook

In Jenkins:
1. Open `selab` → Configure.
2. Under Build Triggers, tick **GitHub hook trigger for GITScm polling**.
3. Save.

In GitHub:
1. Open `https://github.com/archanareddyse/OLTRS`.
2. Settings → Webhooks → Add webhook.
3. Payload URL: a publicly reachable Jenkins endpoint, for example:

```text
https://YOUR_PUBLIC_JENKINS_HOST/github-webhook/
```

4. Content type: `application/json`.
5. Select **Just the push event**.
6. Add webhook.
7. Check Recent Deliveries after pushing a commit.

**Critical:** `localhost` is reachable only from the same computer. GitHub servers cannot call `http://localhost:8081/github-webhook/`. Use a publicly reachable, securely configured endpoint or secure tunnel. Do not expose Jenkins directly to the internet without authentication, HTTPS and access controls.

Webhook verification requires a successful delivery and a corresponding automatically triggered Jenkins build. Selecting the Jenkins checkbox alone does not prove it works.

## 3.8 Five scenario answers

**Q1. A new commit is pushed but Jenkins does not start.**

Check webhook URL, recent delivery response, Jenkins trigger checkbox, public reachability, repository/branch and push event.

**Q2. Maven build fails. How do you skip Docker Build and Push?**

A failed `bat` command throws an error. The sequential pipeline stops, so later stages are skipped. Do not catch and suppress the error if later stages must not run.

**Q3. Docker Hub authentication fails.**

Store Docker Hub username and access token as a Jenkins Username with password credential. Use its ID through `withCredentials` and `usernamePassword`. Never hardcode the token.

**Q4. Build succeeds but push fails. How do you find the error?**

Open job → latest build → Pipeline Overview → failed stage → Console Output.

**Q5. Send email only when the pipeline fails.**

Use `try/catch`, send email in `catch`, then rethrow the error. Configure the Email Extension plugin and SMTP first.

Example pattern:

```groovy
node {
    try {
        stage('Build') {
            bat 'mvn -B clean verify'
        }
        // Put the remaining stages here.
    } catch (err) {
        emailext(
            subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: "Check console output: ${env.BUILD_URL}",
            to: "YOUR_EMAIL_ADDRESS"
        )
        throw err
    }
}
```

Replace the email placeholder with the intended address. This requires the Email Extension plugin and working SMTP configuration.

## 3.9 Jenkins quick viva

- Jenkins: automation server used for CI/CD.
- CI: Continuous Integration.
- CD: Continuous Delivery or Continuous Deployment.
- Pipeline: automated sequence of build/test/delivery steps.
- Scripted Pipeline: Groovy-based pipeline, commonly starts with `node {}`.
- `node`: allocates a Jenkins execution environment.
- `stage`: groups related steps.
- `git`: checks out source code.
- `bat`: runs Windows CMD commands.
- Maven: Java build and dependency management tool.
- WAR: Web Application Archive.
- `withCredentials`: securely exposes stored credentials to pipeline steps.
- Docker build: creates an image from a Dockerfile.
- Docker push: uploads an image to a registry.
- Webhook: sends an event notification when a GitHub action such as push occurs.

## 3.10 Actual Jenkins status

- Job `selab` configured.
- Docker Hub credential ID `dockerhub` created.
- Build #2 GitHub checkout succeeded.
- Build #2 failed because Jenkins could not find `mvn`.
- All five stages completing successfully: **not yet confirmed**.
- GitHub webhook triggering a build automatically: **not yet confirmed**.

Next practice: fix Maven PATH → restart Jenkins → Build Now → verify all stages → verify Docker Hub image → test webhook.

---

# 4. KUBERNETES + NAGIOS USING DOCKER — SET 7

## 4.1 Set 7 requirements

- Create a Kubernetes cluster using Minikube.
- Deploy the application.
- Run 4 replicas.
- Expose the application using a Service.
- Demonstrate scaling up/down.
- Delete a Pod and demonstrate automatic recovery.
- Monitor CPU, memory, disk and uptime using Nagios.

**Set 7 = Apartment System. Set 9 = Sports System.**

## 4.2 Start Minikube

Run in CMD/PowerShell:

```bash
minikube start
kubectl get nodes
kubectl get pods -A
```

If `kubectl` is not installed or configured, install/configure it as required by the lab machine.

## 4.3 Create Deployment with four replicas

Demonstration deployment using Tomcat 9:

```bash
kubectl create deployment apartment --image=tomcat:9
kubectl scale deployment apartment --replicas=4
kubectl get deployments
kubectl get pods
```

Expected: 4 Pods should be running after image download and startup.

**Application note:** `tomcat:9` is a demonstration image. If the question supplies an Apartment System Docker image or WAR file, deploy that actual application instead. Do not claim the real application was deployed if only the stock Tomcat image was used.

## 4.4 Expose the application

```bash
kubectl expose deployment apartment --type=NodePort --port=8080
kubectl get services
minikube service apartment
```

Minikube opens or displays the application URL.

## 4.5 Demonstrate automatic Pod recovery

List Pods:

```bash
kubectl get pods
```

Delete one Pod:

```bash
kubectl delete pod POD_NAME
```

Replace `POD_NAME` with an actual Pod name shown by `kubectl get pods`.

Check again:

```bash
kubectl get pods
```

A Deployment maintains the desired replica count. When one Pod is deleted, Kubernetes creates a replacement.

## 4.6 Demonstrate scaling

Scale up:

```bash
kubectl scale deployment apartment --replicas=4
kubectl get pods
```

Scale down:

```bash
kubectl scale deployment apartment --replicas=2
kubectl get pods
```

This is **manual scaling**. Do not call it CPU-based autoscaling/HPA unless an HPA is actually configured and verified.

## 4.7 Other useful Kubernetes commands

```bash
kubectl get nodes
kubectl get deployments
kubectl get pods
kubectl get services
kubectl describe pod POD_NAME
kubectl describe deployment apartment
kubectl logs POD_NAME
kubectl delete service apartment
kubectl delete deployment apartment
minikube status
minikube stop
```

## 4.8 Nagios using Docker

The lab approach discussed uses a Nagios Docker image, rather than a native Nagios installation with `systemctl`.

Pull the image:

```bash
docker pull jasonrivers/nagios:latest
```

Run it:

```bash
docker run -d --name nagios -p 8081:80 jasonrivers/nagios:latest
```

Check:

```bash
docker ps
docker logs nagios
```

Open:

```text
http://localhost:8081
```

Use the credentials provided by the lab/image documentation. Do not assume credentials if they were not supplied.

If port 8081 is already occupied, use another host port:

```bash
docker run -d --name nagios -p 8082:80 jasonrivers/nagios:latest
```

Then open `http://localhost:8082`.

If a container with the same name already exists:

```bash
docker rm -f nagios
```

Then run it again.

## 4.9 Resource monitoring commands

On a Linux host:

```bash
top
free -h
df -h
uptime
```

Meaning:
- `top`: CPU/process activity.
- `free -h`: memory usage.
- `df -h`: disk usage.
- `uptime`: uptime and load averages.

**Important monitoring distinction:** These Linux commands display system resource information. They do not automatically configure Nagios checks. For actual Nagios monitoring, configure the required host/service definitions and plugins, then verify the checks in the Nagios web interface.

Also, running Nagios in a Docker container does not automatically connect it to the Minikube cluster. Configure the required monitoring targets/checks if the examiner asks for integrated monitoring.

## 4.10 Kubernetes + Nagios viva

- Kubernetes: container orchestration platform.
- Minikube: runs a local Kubernetes cluster.
- Cluster: group of machines/nodes managed by Kubernetes.
- Node: machine that runs Pods.
- Pod: smallest deployable Kubernetes unit.
- Deployment: manages Pods and replica count.
- Replica: desired running copy of a Pod.
- Service: stable network access to Pods.
- NodePort: exposes a Service through a port on cluster nodes.
- Self-healing: Kubernetes replaces failed/deleted Pods to maintain desired state.
- Scaling: changing the number of replicas.
- HPA: Horizontal Pod Autoscaler; scales based on configured metrics.
- Nagios: infrastructure and service monitoring tool.
- Plugin: performs a monitoring check.
- CPU: processor usage.
- Memory: RAM usage.
- Disk: storage usage.
- Uptime: time a system has been running.

---

# 5. AWS EC2 DEPLOYMENT — APARTMENT SYSTEM

## 5.1 Launch an EC2 instance

Open AWS Console → EC2 → Launch instance.

Suggested lab settings:

| Setting | Value |
|---|---|
| Name | `Apartment-Server` |
| AMI | Ubuntu LTS (24.04 preferred for compatibility if available; use the lab's specified version) |
| Architecture | 64-bit x86 |
| Instance type | `t3.micro` if eligible/available |
| Key pair | Create `mykey`, RSA, PEM |
| VPC | Default VPC |
| Public IP | Enable |
| Storage | 8 GiB gp3 (or lab requirement) |

Download and safely store `mykey.pem`. AWS does not let you download the same private key again after creation.

## 5.2 Security group inbound rules

Configure only the ports required by the practical:

| Type | Port | Source |
|---|---:|---|
| SSH | 22 | My IP |
| HTTP | 80 | `0.0.0.0/0` for public web demo |
| HTTPS | 443 | `0.0.0.0/0` if required |
| Custom TCP | 8090 | Restrict to your IP where possible; public only for a temporary lab demonstration if necessary |

Do not expose SSH to everyone unless the lab explicitly requires it and you understand the risk.

Launch the instance. Wait until:
- Instance state = Running.
- Status checks = 2/2 passed.

Copy the Public IPv4 address.

## 5.3 Connect from Windows PowerShell

If the PEM file is in Downloads:

```powershell
cd Downloads
ssh -i "mykey.pem" ubuntu@YOUR_PUBLIC_IP
```

Replace `YOUR_PUBLIC_IP` with the EC2 public IPv4 address.

If asked to trust the host for the first time, type `yes`.

## 5.4 Install Docker and Git on EC2

Run these commands **inside the EC2 Ubuntu terminal**:

```bash
sudo apt update
sudo apt install docker.io git -y
sudo systemctl start docker
sudo systemctl enable docker
docker --version
git --version
```

If Docker commands require elevated permissions, use `sudo docker ...` as shown below.

## 5.5 Clone and deploy Apartment System

```bash
git clone https://github.com/sarasrija/ApartmentSystem.git
cd ApartmentSystem
sudo docker build -t apartment .
sudo docker run -d -p 8090:8080 --name apartment apartment
sudo docker ps
```

Open in your local browser:

```text
http://YOUR_PUBLIC_IP:8090
```

Replace `YOUR_PUBLIC_IP` with the EC2 public IPv4 address.

**Deployment assumptions:** The repository must contain a valid Dockerfile; the application must listen on container port 8080; and its build/runtime requirements must be satisfied. If the lab provides different repository instructions, application port or Dockerfile, follow those instead.

## 5.6 Troubleshooting on EC2

Check running containers:

```bash
sudo docker ps
```

Check all containers:

```bash
sudo docker ps -a
```

Check logs:

```bash
sudo docker logs apartment
```

List images:

```bash
sudo docker images
```

Stop/remove container:

```bash
sudo docker stop apartment
sudo docker rm apartment
```

If the browser cannot connect, check:
1. EC2 instance is Running.
2. Security group permits the required inbound port.
3. Public IP is correct.
4. Container is running.
5. Port mapping is `8090:8080`.
6. Application listens on `0.0.0.0` inside the container and on port 8080.

## 5.7 AWS viva — one-line answers

- AWS: Amazon cloud computing platform.
- EC2: virtual server in AWS.
- AMI: template used to launch an EC2 instance.
- Instance type: defines compute capacity.
- Key pair: public/private key used for secure login.
- PEM: private-key file format commonly used for SSH.
- SSH: secure remote terminal connection.
- Security Group: virtual firewall for an instance.
- Public IP: address used to reach a publicly accessible instance.
- EBS: block storage for EC2.
- S3: object storage service.
- EFS: shared elastic file storage.
- CloudWatch: monitoring and observability service.
- Auto Scaling: adjusts the number of instances based on configured policies.
- Load Balancer: distributes incoming traffic across targets.
- Docker on EC2: runs containerized applications on the virtual machine.

## 5.8 Avoid unnecessary AWS charges

After the practical:
- Stop the EC2 instance if you need to keep it for later.
- Terminate it if you no longer need it.
- Check attached EBS volumes and public IPv4 charges according to your account's current pricing.
- Keep the PEM key private; never commit it to GitHub.

---

# 6. LAST-MINUTE VIVA ANSWERS

## Docker
1. Docker is used to build and run containers.
2. An image is a template; a container is an instance of an image.
3. Dockerfile contains image build instructions.
4. Compose manages multi-container applications.
5. A volume persists data outside a container's writable layer.
6. A network enables container communication.
7. Port mapping connects host and container ports.
8. `docker compose down -v` removes named volumes as well as containers.

## Jenkins
1. Jenkins automates software build, test and delivery.
2. Scripted Pipeline is written in Groovy.
3. `node` allocates an execution environment.
4. `stage` groups pipeline steps.
5. Maven builds and tests Java projects.
6. `withCredentials` accesses stored secrets securely.
7. Webhooks notify Jenkins about GitHub events.
8. A failed stage normally stops later sequential stages.

## Kubernetes
1. Kubernetes orchestrates containers.
2. Minikube runs a local Kubernetes cluster.
3. A Pod is the smallest deployable unit.
4. A Deployment manages replicas.
5. A Service provides stable network access.
6. Self-healing replaces failed/deleted Pods.
7. Scaling changes the replica count.
8. HPA scales based on configured metrics.

## Nagios
1. Nagios monitors hosts and services.
2. Plugins perform checks.
3. CPU is processor utilization.
4. Memory is RAM utilization.
5. Disk monitoring checks storage usage.
6. Uptime is how long a system has been running.

## AWS
1. EC2 provides virtual machines.
2. AMI is an instance template.
3. Security Group is a virtual firewall.
4. SSH is used for secure remote access.
5. EBS provides block storage.
6. S3 stores objects.
7. CloudWatch monitors AWS resources.
8. A Load Balancer distributes incoming traffic.

---

# 7. TROUBLESHOOTING CHEAT SHEET

| Error / symptom | Quick action |
|---|---|
| Docker command not found | Check Docker Desktop/Engine is installed and running; reopen terminal |
| Compose YAML error | Fix indentation and validate with `docker compose config` |
| Port already in use | Change the host-side port |
| Container exits | `docker ps -a` and inspect `docker logs CONTAINER` |
| Data lost | Check whether `down -v` deleted the named volume |
| Flask not reachable | Listen on `0.0.0.0`, verify container port 5000 and port mapping |
| NGINX 502 | Check upstream service name, port, network and Tomcat logs |
| Database connection refused | Use Compose service name, correct port, credentials and readiness/retry |
| Jenkins cannot find Maven | Check System PATH and Jenkins service environment; restart Jenkins |
| Jenkins Docker build fails | Check Docker installation, daemon and Jenkins service-account permissions |
| Docker push denied | Check image name, repository permissions and Jenkins token credential |
| GitHub webhook not triggering | Check public URL, recent delivery, trigger checkbox and Jenkins reachability |
| Minikube not starting | Check virtualization/container driver and `minikube status` |
| Kubernetes Pod not ready | `kubectl describe pod POD_NAME` and `kubectl logs POD_NAME` |
| Pod does not recover | Check that it belongs to a Deployment/ReplicaSet and inspect events |
| Nagios page unavailable | Check `docker ps`, port mapping and `docker logs nagios` |
| EC2 browser timeout | Check public IP, security group inbound port, container and port mapping |
| EC2 SSH fails | Check PEM path, username `ubuntu`, public IP and inbound SSH rule |

---

# 8. FINAL EXAM CHECKLIST

## Docker Compose
- [ ] Identify the correct set and required services.
- [ ] Create the correct `docker-compose.yml`.
- [ ] Check indentation and run `docker compose config`.
- [ ] Start with `docker compose up -d`.
- [ ] Verify using `docker compose ps`.
- [ ] Open the correct browser URL.
- [ ] Demonstrate volume persistence or scaling if required.

## Jenkins
- [ ] Open `http://localhost:8081`.
- [ ] Open job `selab`.
- [ ] Verify Docker Hub credential ID `dockerhub`.
- [ ] Paste the Scripted Pipeline.
- [ ] Confirm Java, Maven, Git and Docker are available to the Jenkins agent.
- [ ] Click Build Now.
- [ ] Inspect Pipeline Overview and Console Output.
- [ ] Verify Docker Hub image.
- [ ] Configure/test webhook only with a publicly reachable secure endpoint.

## Kubernetes + Nagios
- [ ] `minikube start`.
- [ ] Create Deployment.
- [ ] Scale to 4 replicas.
- [ ] Expose Service.
- [ ] Delete one Pod and verify replacement.
- [ ] Scale down to 2.
- [ ] Pull and run Nagios Docker image.
- [ ] Open Nagios web interface.
- [ ] Demonstrate resource commands and explain actual Nagios checks.

## AWS
- [ ] Launch EC2 and wait for 2/2 checks.
- [ ] Save PEM key safely.
- [ ] Configure SSH and application inbound ports.
- [ ] SSH into Ubuntu.
- [ ] Install Docker and Git.
- [ ] Clone the correct repository.
- [ ] Build and run the container.
- [ ] Open the public IP and mapped port.
- [ ] Stop/terminate resources after the practical as appropriate.

---

## FINAL REMEMBER

**Docker:** Compose file → `up -d` → `ps` → browser.

**Jenkins:** Checkout → Maven → Docker Build → Docker Push.

**Kubernetes:** Minikube → Deployment → 4 Pods → Service → delete Pod → recovery → scale.

**Nagios:** Pull image → run container → browser → checks/logs.

**AWS:** Launch EC2 → SSH → install Docker/Git → clone → build → run → public browser URL.

**When something fails:** Read the exact error first. Do not reinstall everything or randomly change working settings.
