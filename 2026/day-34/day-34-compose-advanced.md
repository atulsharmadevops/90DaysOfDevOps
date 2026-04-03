# DOCKER COMPOSE: REAL-WORLD MULTI-CONTAINER APPS.

## Task 1: Build Your Own App Stack
1. Create Project Structure
    ```bash
    mkdir docker-compose-advanced
    cd docker-compose-advanced
    mkdir app
    ```
    - Inside `app/` we'll put our web app code and Dockerfile.
2. Write a Simple Flask Web App<br>
    Create `app/app.py`:
    ```python
    from flask import Flask #Imports Flask Class from Flask library, used to create our web application object.
    import psycopg2 #Python library for connecting to PostgreSQL database.
    import redis #Pytho client for connecting to Redis cache.
    #Even though we're not using psycopg2 & redis yet, importing shows the app is intended to connect to DB + cache.

    app = Flask(__name__) #Creates new Flask application instance.
    #__name__ tells Flask where to look for resources (templates, static files) and helps with debugging.

    @app.route("/") #decorator that maps URL path / (homepage) to the function hello(). When visited http://localhost:5000/, Flask runs hello() and returns string - string becomes HTTP response body.
    def hello():
        return "Hello World! Connected to DB + Cache."

    if __name__ == "__main__": #It ensures app only runs when we execute script directly and not when imported.
        app.run(host="0.0.0.0", port=5000)
    #app.run(...) starts the Flask development server.
    #host="0.0.0.0" makes the app accessible from any network interface (important inside Docker).
    #port=5000 - the server listens on port 5000.
    ```
    Create `app/requirements.txt`:
    ```Code
    flask
    psycopg2-binary
    redis
    ```
    - It is a list of Python dependencies our project needs.
    - It’s used by tools like `pip` to install all those libraries into our environment or container.
    - This makes our project reproducible - anyone can set up the same environment by using our `requirements.txt`.

    Create `app/Dockerfile`:
    ```dockerfile
    FROM python:3.12-slim

    WORKDIR /app
    COPY requirements.txt .
    RUN pip install -r requirements.txt

    COPY . .
    CMD ["python", "app.py"]
    ```
3. Create `docker-compose.yml`<br>
    In the project root, create `docker-compose.yml`:
    ```yaml
    services:
      web:
        build: ./app    
        ports:
          - "5000:5000"
        networks:
          - app_net

      db:
        image: postgres:15
        environment:
          POSTGRES_USER: user
          POSTGRES_PASSWORD: pass
          POSTGRES_DB: mydb
        volumes:
          - db_data:/var/lib/postgresql/data
        networks:
          - app_net

      cache:
        image: redis:7
        networks:
          - app_net

    networks:
      app_net:
        driver: bridge

    volumes:
      db_data:
    ```
4. Build and Run
    ```bash
    docker compose up --build
    ```
    - Compose builds the web app image from our Dockerfile.
    - Starts Postgres, Redis, and Flask.
    - Web waits until DB passes healthcheck before starting.

5. Test<br>
Open browser at http://localhost:5000 (localhost in Bing).

    We should see:

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/de6d4151eaab51b53bcfd2ad316cf06a67aa667e/2026/day-34/Screenshots/Screenshot%20(240).png)

## Task 2: depends_on & Healthchecks
1. Add a Healthcheck to the Database<br>
    In our `docker-compose.yml`, update the **db service**:
    ```yaml
    db:
      image: postgres:15
      environment:
        POSTGRES_USER: user
        POSTGRES_PASSWORD: pass
        POSTGRES_DB: mydb
      volumes:
        - db_data:/var/lib/postgresql/data
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
        interval: 10s
        timeout: 5s
        retries: 5
      networks:
        - app_net
    ```
    - `pg_isready` checks if Postgres is accepting connections.
    - `interval`, `timeout`, `retries` control how often and how long Docker waits before marking the service healthy.

2. Update **depends_on** for Web App<br>
    In the **web service**, change `depends_on`:
    ```yaml
    web:
      build: ./app
      ports:
        - "5000:5000"
      depends_on:
        db:
          condition: service_healthy
      networks:
        - app_net
    ```
    - This ensures the **web app waits until the DB passes its healthcheck** before starting.

3. Bring Everything Down<br>
    Stop and remove containers:
    ```bash
    docker compose down -v
    ```
    - This clears out all running containers and networks.

4. Bring Everything Up Again<br>
    Start fresh:
    ```bash
    docker compose up --build
    ```
    - Compose starts **db** first.
    - Healthcheck runs until Postgres is ready.
    - Only then does **web** start.
    - Redis starts independently since it doesn’t have a healthcheck dependency.

5. Verify Behavior<br>
    Watch logs in our terminal:
    - We’ll see Postgres starting, then healthcheck retries.
    - Web won’t start until DB is marked healthy.

    Open browser: http://localhost:5000

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/de6d4151eaab51b53bcfd2ad316cf06a67aa667e/2026/day-34/Screenshots/Screenshot%20(241).png)

    - We should see our Flask app response.

## Task 3: Restart Policies
1. Add `restart: always` to Database Service<br>
    Update our `docker-compose.yml`:
    ```yaml
    db:
      image: postgres:15
      environment:
        POSTGRES_USER: user
        POSTGRES_PASSWORD: pass
        POSTGRES_DB: mydb
      volumes:
        - db_data:/var/lib/postgresql/data
      restart: always
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
        interval: 10s
        timeout: 5s
        retries: 5
      networks:
        - app_net
    ```
2. Test `restart: always`<br>
    Start our stack:
    ```bash
    docker compose up -d
    ```
    Find the DB container ID:
    ```bash
    docker ps
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/de6d4151eaab51b53bcfd2ad316cf06a67aa667e/2026/day-34/Screenshots/Screenshot%20(249).png)

    Kill the container manually:
    ```bash
    docker kill <container_id>
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/de6d4151eaab51b53bcfd2ad316cf06a67aa667e/2026/day-34/Screenshots/Screenshot%20(250).png)

    Run `docker ps` again - we’ll see Docker automatically starts a new Postgres container because of `restart: always`.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/de6d4151eaab51b53bcfd2ad316cf06a67aa667e/2026/day-34/Screenshots/Screenshot%20(251).png)

3. Try `restart: on-failure`<br>
    Change the policy:
    ```yaml
    db:
      image: postgres:15
      environment:
        POSTGRES_USER: user
        POSTGRES_PASSWORD: pass
        POSTGRES_DB: mydb
      volumes:
        - db_data:/var/lib/postgresql/data
      restart: on-failure
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
        interval: 10s
        timeout: 5s
        retries: 5
      networks:
        - app_net
    ```
    Now repeat the test:
    - If we kill the container manually (docker kill), it will **not** restart - because that’s considered a clean stop, not a failure.
    - If the **container exits with a non-zero code** (e.g., Postgres crashes due to misconfiguration), Docker will restart it.

4. When would we use each restart policy?<br>
**Ans.**

    `restart: always` - Use for critical services like databases or caches that must stay running no matter what, even if stopped manually.

    `restart: on-failure` - Use for apps where you only want automatic recovery from crashes, but not when you intentionally stop them.



## Task 4: Custom Dockerfiles in Compose
1. Use `build:` in Compose<br>
    Update our `docker-compose.yml` so the **web service** builds from our Dockerfile instead of pulling an image:
    ```yaml
    web:
        build: ./app
        ports:
        - "5000:5000"
        networks:
        - app_net
    ```
    - This tells Compose to look in the `./app` directory for a Dockerfile.

2. Confirm our Dockerfile<br>
    Our `app/Dockerfile` should already look like this:
    ```dockerfile
    FROM python:3.12-slim

    WORKDIR /app
    COPY requirements.txt .
    RUN pip install -r requirements.txt

    COPY . .
    CMD ["python", "app.py"]
    ```
3. Make a Code Change<br>
    Edit `app/app.py` - for example, change the route message:
    ```python
    @app.route("/")
    def hello():
        return "Hello World! Updated version with DB + Cache."
    ```
4. Rebuild and Restart in One Command
    ```bash
    docker compose up --build
    ```
    - `--build` forces Compose to rebuild the web app image from our Dockerfile.
    - Containers restart automatically with the new code.

5. Verify<br>
    Open http://localhost:5000 again

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/de6d4151eaab51b53bcfd2ad316cf06a67aa667e/2026/day-34/Screenshots/Screenshot%20(254).png)

    - We should see our updated message.

## Task 5: Named Networks & Volumes
1. Define Explicit Networks<br>
Instead of relying on the default network, we declared our own:
    ```yaml
    networks:
    app_net:
        driver: bridge
    ```
    - This makes networking clearer and easier to manage in larger stacks.
2. Define Named Volumes<br>
Give our database data volume a name so it persists across container restarts:
    ```yaml
    volumes:
      db_data:
    ```
    - This ensures Postgres data isn’t lost when we bring containers down.
3. Add Labels to Services<br>
Labels help organize and identify services, especially in monitoring or orchestration tools:
    ```yaml
    labels:
      com.example.service: "web"

    labels:
      com.example.service: "database"
    ```
    - We can add different labels for each service.

## Task 6: Scaling (Bonus)
1. Scale the Web Service
  ```bash
  docker compose up --scale web=3 -d
  ```
  - This tells Compose to start **3 replicas** of our `web` service.

2. What Happens?<br>
**Ans.**
    - Docker creates 3 containers for the web app.
    - Each tries to bind to port `5000` on the host.
    - Only the first succeeds while the others fail because a single host port can’t be shared by multiple containers directly.

3. What Breaks?<br>
**Ans.** We'll see errors like `Bind for 0.0.0.0:5000 failed: port is already allocated`.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/de6d4151eaab51b53bcfd2ad316cf06a67aa667e/2026/day-34/Screenshots/Screenshot%20(255).png)

    - Scaling works internally (containers run and can talk to DB/Redis), but external access via the host port breaks.

4. Why doesn't simple scaling work with port mapping?<br>
**Ans.** Because host ports are unique  we can’t map multiple containers to the same host port.

    **Solution in production:** use a reverse proxy or load balancer (e.g., Nginx, Traefik) to distribute traffic across replicas instead of direct port mapping.