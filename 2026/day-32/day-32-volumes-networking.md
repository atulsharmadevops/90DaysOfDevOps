# DOCKER VOLUMES AND NETWORKING.

## Task 1: The Problem – Ephemeral Containers
1. Run a Postgres Container
    ```bash
    docker run -d --name pg1 -e POSTGRES_PASSWORD=secret postgres
    ```
    - `-d` - run in detached mode
    - `--name pg1` - name the container
    - `-e POSTGRES_PASSWORD=secret` - set the database password
    - This starts a Postgres server inside a container.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(126).png)

2. Connect to the Database
    ```bash
    docker exec -it pg1 psql -U postgres
    ```
    - This opens the Postgres shell inside the container.

3. Create Some Data<br>
    Inside the Postgres shell:
    ```sql
    CREATE DATABASE testdb;
    \c testdb
    CREATE TABLE users (id SERIAL PRIMARY KEY, name VARCHAR(50));
    INSERT INTO users (name) VALUES ('Alice'), ('Bob');
    SELECT * FROM users;
    ```
    - We should see rows returned.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(128).png)

4. Stop and Remove the Container
    ```bash
    docker stop pg1
    docker rm pg1
    ```
    - This deletes the container (and its writable layer).

5. Run a New Container
    ```bash
    docker run -d --name pg2 -e POSTGRES_PASSWORD=secret postgres
    docker exec -it pg2 psql -U postgres
    ```
6. Check for Your Data
    ```sql
    \c testdb
    SELECT * FROM users;
    ```
    - We’ll get an error: database "testdb" does not exist.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(131).png)

    What Happened and Why?<br>
    **Ans.**
    The data is gone.

    **Reason:** Containers are ephemeral. Their writable filesystem layer is destroyed when the container is removed. Since you didn’t use a volume, the database files were tied to the container’s lifecycle.

    
    To persist data, we need volumes (named or bind mounts).

## Task 2: Named Volumes
1. Create a Named Volume
    ```bash
    docker volume create mydbdata
    ```
    - This creates a managed storage location that Docker controls.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(134).png)

2. Run a Database Container with the Volume
    ```bash
    docker run -d --name pg1 -e POSTGRES_PASSWORD=secret -v mydbdata:/var/lib/postgresql -p 5432:5432 postgres:18
    ```
    - `-v mydbdata:/var/lib/postgresql/data` - attaches the named volume to Postgres’ data directory.
    - Now, all database files are stored in the volume, not inside the container’s ephemeral layer.

3. Add Some Data
    - Connect to Postgres:
        ```bash
        docker logs pg1

        docker exec -it pg1 psql -U postgres
        ```
    - Inside Postgres:
        ```sql
        CREATE DATABASE testdb;
        \c testdb
        CREATE TABLE users (id SERIAL PRIMARY KEY, name VARCHAR(50));
        INSERT INTO users (name) VALUES ('Alice'), ('Bob');
        SELECT * FROM users;
        ```
        - We’ll see our rows.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(136).png)

4. Stop and Remove the Container
    ```bash
    docker stop pg1
    docker rm pg1
    ```
    - The container is gone, but the volume remains.

5. Run a New Container with the Same Volume
    ```bash
    docker run -d --name pg2 -e POSTGRES_PASSWORD=secret -v mydbdata:/var/lib/postgresql -p 5432:5432 postgres:18

    docker exec -it pg2 psql -U postgres
    ```
- Inside Postgres:
    ```sql
    \c testdb
    SELECT * FROM users;
    ```
    - Our data is still there!

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(138).png)

6. Verify the Volume
    - List volumes:
        ```bash
        docker volume ls
        ```
    - Inspect the volume:
        ```bash
        docker volume inspect mydbdata
        ```
        - We’ll see the mount path on your host where Docker stores the data.
    
    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(140).png)

    Is the data still there?<br>
    **Ans.** Data persisted across containers.

    Reason: Named volumes are independent of container lifecycle. Even if the container is deleted, the volume remains until you explicitly remove it.

    Volumes are the solution for persistence in Docker.

## Task 3: Bind Mounts
1. Create a Folder on Your Host<br>
    On our host machine:
    ```bash
    mkdir site
    cd site
    echo '<h1>Hello from Bind Mount!</h1>' > index.html
    ```
    - This creates a folder `site/` with a simple `index.html`.

2. Run an Nginx Container with Bind Mount
    ```bash
    docker run -d --name web1 -p 8081:80 -v $(pwd)/site:/usr/share/nginx/html nginx
    ```
    - `-p 8081:80` - maps container port 80 to host port 8081.
    - `-v $(pwd)/site:/usr/share/nginx/html` - bind mounts your local folder into Nginx’s web root.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(142).png)

3. Access the Page in Your Browser<br>
    Open:
    ```Code
    http://localhost:8081
    ```
    We should see “Hello from Bind Mount!” rendered.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(143).png)

4. Edit the File on our Host
    ```bash
    echo '<h1>Updated content!</h1>' > index.html
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(144).png)

    Refresh our browser:
    - The page updates immediately, because the container is serving files directly from your host folder.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(145).png)

    What is the difference between a **Named Volume** and a **Bind Mount**?<br>
        **Ans.**<br>
       **Named Volume:** Managed by Docker, stored in Docker’s internal volume directory. Best for persistence in production.

    **Bind Mount:** Directly maps a host folder. Great for development because changes on the host reflect instantly inside the container.

## Task 4: Docker Networking Basics
1. List All Docker Networks
    ```bash
    docker network ls
    ```
    We’ll see something like:

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(147).png)

    - By default, Docker creates bridge, host, and none networks.

2. Inspect the Default Bridge Network
    ```bash
    docker network inspect bridge
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(148).png)

    This shows details like:
    - Subnet (e.g., 172.17.0.0/16)
    - Gateway (e.g., 172.17.0.1)
    - Connected containers (if any)

3. Run Two Containers on the Default Bridge
    ```bash
    docker run -dit --name c1 alpine sh
    docker run -dit --name c2 alpine sh
    ```
    - `alpine` is a lightweight Linux image.
    - `-dit` runs interactively in detached mode.

4. Test Ping by Name
    ```bash
    docker exec c1 ping -c 3 c2
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(149).png)

    - This will fail as containers on the default bridge **cannot resolve names**.

5. Test Ping by IP<br>
    First, find container IPs:
    ```bash
    docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' c2
    ```
    Now ping from c1:
    ```bash
    docker exec c1 ping -c 3 172.17.0.3
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(152).png)

    - This works as containers can reach each other by IP on the default bridge.
    - **Default bridge** allows communication by IP, but **not by name**.
    - This is why custom networks are needed — they provide built-in DNS so containers can resolve each other by name.

## Task 5: Custom Networks
1. Create a Custom Bridge Network
    ```bash
    docker network create my-app-net
    ```
    - This creates a new bridge network with built-in DNS support.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(153).png)

2. Run Two Containers on my-app-net
    ```bash
    docker run -dit --name c1 --network my-app-net alpine sh
    docker run -dit --name c2 --network my-app-net alpine sh
    ```
    - Both containers are now attached to the custom network.

3. Ping Each Other by Name<br>
    From container c1:
    ```bash
    docker exec c1 ping -c 3 c2
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(154).png)

    This works as containers on a custom bridge network can resolve each other by name.

    Why does custom networking allow name-based communication but the default bridge doesn't?<br>
    **Ans.** The **default bridge network** does not provide automatic DNS resolution.

    **Custom bridge networks** include an internal DNS service, so containers can resolve each other by name. This makes communication much easier for multi-container applications.


## Task 6: Put It Together
1. Create a Custom Network
    ```bash
    docker network create app-net
    ```
    - This ensures containers can resolve each other by **name**.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(155).png)

2. Run a Database Container with a <br>
    For Postgres:
    ```bash
    docker volume create mydbdata
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(156).png)

    ```bash
    docker run -d --name mydb \
    --network app-net \
    -e POSTGRES_PASSWORD=secret \
    -v mydbdata:/var/lib/postgresql/data \
    postgres
    ```
    - `--network app-net` - attaches DB to the custom network.
    - `-v mydbdata:/var/lib/postgresql/data` - persists data in a named volume.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(157).png)

3. Run an App Container on the Same Network<br>
    We can use a lightweight container like **Alpine** to test connectivity:
    ```bash
    docker run -it --rm --network app-net alpine sh
    ```
    Inside the Alpine shell, install `ping` and `psql` if needed:
    ```bash
    apk add --no-cache postgresql-client
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(158).png)

4. Verify Connectivity by Name<br>
    From inside the app container:
    ```bash
    psql -h mydb -U postgres -W
    ```
    - Enter the password (`secret`).
    - We’ll connect to Postgres using the container name (`mydb`) instead of an IP.

    Or simply test with ping:
    ```bash
    ping -c 3 mydb
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/940d6ec4dcabe0a32c9d427eac3d6e5bd73189bc/2026/day-32/Screenshots/Screenshot%20(159).png)
    
    - Works because the custom network provides DNS resolution.