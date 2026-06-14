# Docker Project: Dockerize a Full Application
> End-to-end Dockerization of a Python Flask application with a PostgreSQL backend - environment setup, multi-stage Dockerfile, Docker Compose with health checks, environment variable management, and a full publish-and-pull cycle through Docker Hub.
 
## Table of Contents 
1. [Environment Setup](#1-environment-setup)
2. [Create the Application](#2-create-the-application)
3. [Write the Dockerfile](#3-write-the-dockerfile)
4. [Add Docker Compose](#4-add-docker-compose)
5. [Publish to Docker Hub](#5-publish-to-docker-hub)
6. [Test the Full Flow](#6-test-the-full-flow)

## 1. Environment Setup
### Update packages
```bash
sudo apt-get update && sudo apt-get upgrade -y
```
### Install Docker
```bash
sudo apt-get install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
```
### Add your user to docker group
```bash
sudo usermod -aG docker $USER
newgrp docker
```
### Install Docker Compose plugin
```bash
# Set up Docker’s repo
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker + Compose plugin
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# Verify
docker compose version
```
### Install Git
```bash
sudo apt-get install git -y
```
### Install Python + pip
```bash
sudo apt-get install python3 python3-pip -y
```
### (Optional) Install Postgres client
```bash
sudo apt-get install postgresql-client -y
```

## 2. Create the Application
Make Project Directory:
```bash
mkdir docker-project && cd docker-project
```
Create Flask App<br>
`docker-project/app.py`
```python
from flask import Flask, jsonify
import psycopg2
import os

app = Flask(__name__)

def get_db_connection():
    conn = psycopg2.connect(
        host="db",
        database=os.getenv("POSTGRES_DB"),
        user=os.getenv("POSTGRES_USER"),
        password=os.getenv("POSTGRES_PASSWORD")
    )
    return conn

@app.route("/")
def home():
    return "Hello from Flask + Postgres!"

@app.route("/users")
def users():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("CREATE TABLE IF NOT EXISTS users (id SERIAL PRIMARY KEY, name TEXT);")
    cur.execute("INSERT INTO users (name) VALUES ('Atul') ON CONFLICT DO NOTHING;")
    cur.execute("SELECT * FROM users;")
    rows = cur.fetchall()
    cur.close()
    conn.close()
    return jsonify(rows)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

Create Requirements<br>
`docker-project/requirements.txt`
```Code
flask==3.0.2
psycopg2-binary==2.9.9
```

## 3. Write the Dockerfile
### Create Multi-Stage Dockerfile<br>
`docker-project/Dockerfile`
```dockerfile
# Stage 1: Builder
FROM python:3.11-slim AS builder

# Set working directory
WORKDIR /app

# Install build dependencies (for psycopg2 compilation)
RUN apt-get update && apt-get install -y gcc libpq-dev && rm -rf /var/lib/apt/lists/*

# Copy requirements and install
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Stage 2: Runtime
FROM python:3.11-slim

# Create non-root user
RUN useradd -m flaskuser

WORKDIR /app

# Copy installed packages from builder
COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages 
COPY --from=builder /usr/local/bin /usr/local/bin

# Copy application code
COPY . .

RUN chown -R flaskuser:flaskuser /app

# Switch to non-root user
USER flaskuser

# Expose Flask port
EXPOSE 5000

# Run the app
CMD ["python", "app.py"]
```
**Why multi-stage?** The builder stage installs `gcc` and `libpq-dev` to compile psycopg2. The runtime stage copies only the installed Python packages — the compiler toolchain is discarded, keeping the final image lean and reducing the attack surface.

`docker-project/.dockerignore`
```Code
__pycache__/
*.pyc
*.pyo
*.pyd
*.db
*.log
.env
.git
.gitignore
README.md
```
- **Non-root user**: improves container security.
- **Slim base image**: keeps image size low.
- **.dockerignore**: prevents unnecessary files from bloating the image.

### Build & Test Locally
```bash
# Build image
docker build -t flask-app:1.0 .
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/869c3dae484e6e566cae9f55a48cc9a31581bfef/2026/day-36/Screenshots/Screenshot%20(1334).png)

```bash
# Run container
docker run -p 5000:5000 flask-app:1.0
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/869c3dae484e6e566cae9f55a48cc9a31581bfef/2026/day-36/Screenshots/Screenshot%20(1339).png)

### Verify
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/869c3dae484e6e566cae9f55a48cc9a31581bfef/2026/day-36/Screenshots/Screenshot%20(1337).png)

## 4. Add Docker Compose
`docker-project/docker-compose.yml`
```yaml
services:
  web:
    build: .
    container_name: flask_app
    ports:
      - "5000:5000"
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app_net
    restart: always

  db:
    image: postgres:15-alpine
    container_name: postgres_db
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - app_net
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  db_data:

networks:
  app_net:
    driver: bridge
```

`docker-project/.env`
```Code
POSTGRES_USER=admin
POSTGRES_PASSWORD=secret
POSTGRES_DB=flaskdb
```
> Never commit `.env` to Git. Add it to `.gitignore`.

### Run & Verify
```bash
docker compose up --build
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/869c3dae484e6e566cae9f55a48cc9a31581bfef/2026/day-36/Screenshots/Screenshot%20(1340).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/869c3dae484e6e566cae9f55a48cc9a31581bfef/2026/day-36/Screenshots/Screenshot%20(1341).png)

Visit http://localhost:5000/ ---> should show Hello from Flask + Postgres!

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/869c3dae484e6e566cae9f55a48cc9a31581bfef/2026/day-36/Screenshots/Screenshot%20(1342).png)

Visit http://localhost:5000/users ---> should return JSON list of users.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/869c3dae484e6e566cae9f55a48cc9a31581bfef/2026/day-36/Screenshots/Screenshot%20(1343).png)

## 5. Publish to Docker Hub
Tag & Push Image
```bash
# Build image
docker build -t atul/flask-app:day36 .
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/869c3dae484e6e566cae9f55a48cc9a31581bfef/2026/day-36/Screenshots/Screenshot%20(1345).png)

```bash
# Login to Docker Hub
docker login
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/869c3dae484e6e566cae9f55a48cc9a31581bfef/2026/day-36/Screenshots/Screenshot%20(1350).png)

```bash
# Push image
docker push atulsharmadochub/flask-app:day36
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/869c3dae484e6e566cae9f55a48cc9a31581bfef/2026/day-36/Screenshots/Screenshot%20(1355).png)

After pushing, our image will be available at:<br>
Docker Hub Link: https://hub.docker.com/repository/docker/atulsharmadochub/flask-app/general

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/869c3dae484e6e566cae9f55a48cc9a31581bfef/2026/day-36/Screenshots/Screenshot%20(1362).png)

`docker-project/README.md`
```markdown
# Flask + Postgres Dockerized App

## 📌 What the App Does
A simple Python Flask application connected to a PostgreSQL database.  
- `/` → Returns a welcome message.  
- `/users` → Creates a `users` table if it doesn’t exist, inserts a sample user, and returns all users in JSON format.  

This project demonstrates **end-to-end Dockerization** with Dockerfile, Docker Compose, environment variables, persistent volumes, and healthchecks.

---

## 🚀 How to Run with Docker Compose

1. Clone the repository:
   ```bash
   git clone https://github.com/yourusername/day36-docker-project.git
   cd day36-docker-project
2. Create a `.env` file:
    ```bash
    POSTGRES_USER=admin
    POSTGRES_PASSWORD=secret
    POSTGRES_DB=flaskdb
    ```
3. Build and start containers:
    ```bash
    docker compose up --build
    ```
4. Access the app:
    - Home: `http://localhost:5000/` (localhost in Bing)
    - Users: `http://localhost:5000/users` (localhost in Bing)

5. Environment Variables
The app requires the following variables (defined in `.env`):
- `POSTGRES_USER` - Database username
- `POSTGRES_PASSWORD` - Database password
- `POSTGRES_DB` - Database name

6. Docker Hub
Image available at:
Docker Hub - atul/flask-app (hub.docker.com in Bing)
```
## 6. Test the Full Flow
This step validates the published image by pulling it fresh - simulating what anyone else would do to run your application.

### Remove Local Containers & Images
```bash
# Stop and remove all containers
docker compose down

# Remove all containers (if any remain)
docker rm -f $(docker ps -aq)

# Remove all images (optional, but ensures clean test)
docker rmi -f $(docker images -q)
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/869c3dae484e6e566cae9f55a48cc9a31581bfef/2026/day-36/Screenshots/Screenshot%20(1369).png)

### Update Compose to Use the Published Image
Edit the `web` service in `docker-compose.yml` to pull from Docker Hub instead of building locally:
```yaml
web:
  image: atulsharmadochub/flask-app:day36
  ports:
    - "5000:5000"
  env_file:
    - .env
  depends_on:
    db:
      condition: service_healthy
  networks:
    - app_net
  restart: always
```
Now run:
```bash
docker compose up -d
```
Docker pulls `atulsharmadochub/flask-app:day36` from Hub and starts both services.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/869c3dae484e6e566cae9f55a48cc9a31581bfef/2026/day-36/Screenshots/Screenshot%20(1372).png)

### Verify Fresh Run

- Visit `http://localhost:5000/` - should display Hello from Flask + Postgres!

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/869c3dae484e6e566cae9f55a48cc9a31581bfef/2026/day-36/Screenshots/Screenshot%20(1373).png)

- Visit `http://localhost:5000/users` - should return JSON list of users.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/869c3dae484e6e566cae9f55a48cc9a31581bfef/2026/day-36/Screenshots/Screenshot%20(1374).png)

### Troubleshooting
| Symptom | Likely Cause | Fix |
|---|---|---|
| Flask cannot connect to Postgres | `.env` values don't match the Compose environment variables | Verify all three variables match exactly |
| Postgres container not healthy | Health check failing during startup | Run `docker logs postgres_db` to inspect startup errors |
| Image fails to pull | Repo is private or tag is wrong | Confirm the Docker Hub repo is public and the tag is `atulsharmadochub/flask-app:day36` |
| `/users` returns 500 | Postgres not ready when Flask started | Confirm `depends_on: condition: service_healthy` is present in the Compose file |