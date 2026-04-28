# Docker Project: Dockerize a Full Application

## Task 1: Pick Our App
Make Project Directory:
```bash
mkdir docker-project
cd docker-project
```
Create Flask App (`docker-project/app.py`)
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

Create Requirements (`docker-project/requirements.txt`)
```Code
flask==3.0.2
psycopg2-binary==2.9.9
```

## Task 2: Write the Dockerfile
Create `docker-project/Dockerfile`
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
Create `docker-project/.dockerignore`
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

Build & Test Locally
```bash
# Build image
DOCKER_BUILDKIT=0 docker build -t flask-app:1.0 .app:1.0 .

# Run container
docker run -p 5000:5000 flask-app:1.0
```
Then visit:

http://localhost:5000/ → Hello message

http://localhost:5000/users → JSON list of users

## Task 3: Add Docker Compose
Create `docker-project/docker-compose.yml`
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

Create `docker-project/.env` File
```Code
POSTGRES_USER=admin
POSTGRES_PASSWORD=secret
POSTGRES_DB=flaskdb
```

Run & Verify
```bash
docker compose up --build
```
Visit http://localhost:5000/ ---> should show Hello from Flask + Postgres!

Visit http://localhost:5000/users ---> should return JSON list of users.

## Task 4: Ship It
Tag & Push Image
```bash
# Build image
docker build -t atul/flask-app:day36 .

# Login to Docker Hub
docker login

# Push image
docker push atul/flask-app:day36
```
After pushing, our image will be available at:<br>
Docker Hub Link: https://hub.docker.com/r/atul/flask-app

Create `docker-project/README.md`
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
## Task 5: Test the Whole Flow
Remove Local Containers & Images
```bash
# Stop and remove all containers
docker compose down

# Remove all containers (if any remain)
docker rm -f $(docker ps -aq)

# Remove all images (optional, but ensures clean test)
docker rmi -f $(docker images -q)
```
Pull Fresh Image from Docker Hub<br>
Update our `docker-compose.yml` so the `web` service uses the image you pushed instead of building locally:
```yaml
web:
  image: atul/flask-app:day36
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
This will pull the image from Docker Hub and start both services.

Verify Fresh Run

- Visit `http://localhost:5000/` - should display Hello from Flask + Postgres!
- Visit `http://localhost:5000/users` - should return JSON list of users.

Fix if Needed

- If Flask can’t connect to Postgres --->  check `.env` values match what you used when pushing.
- If Postgres isn’t ready ---> confirm healthcheck is working (`docker ps` shows `healthy`).
- If image doesn’t pull ---> ensure Docker Hub repo is public and tag matches (`atul/flask-app:day36`).