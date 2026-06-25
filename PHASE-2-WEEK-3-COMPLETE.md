# Phase 2 – Week 3 Notes
### Docker Compose, Volumes, Networking and Environment Variables

**Dates:** 21–24 June 2026
**VM:** devops-vm (Ubuntu 22.04, 20.5.184.90)

---

## What I did this week

Up until now I was only running one container at a time. This week I learned how to run a whole app made of more than one container, using a single file and a single command. I built a real WordPress website backed by a MySQL database on my Azure VM, and then I deleted the database container on purpose to check if my data would survive. It did.

---

## The technical concepts we covered (the proper terms)

These are the actual words I need to know for interviews, with what each one really means.

- **Container** – A running instance of an image. A program packaged with everything it needs, running in its own isolated space. It is *ephemeral*, meaning if you delete it, anything stored inside it is gone.
- **Image** – The read-only template a container is built from. The software is already frozen inside it. One image can create many containers.
- **Multi-container application** – An app made of more than one container working together, e.g. a web container plus a database container.
- **Docker Compose** – A tool that defines and runs a multi-container app from a single file (`docker-compose.yml`) using one command (`docker compose up`). It is the new v2 plugin, run as `docker compose` with a space.
- **Service** – Each container defined inside the compose file. My two services were `db` and `wordpress`.
- **Volume** – Persistent storage that lives on the host disk, OUTSIDE the container. The database writes its data here so it survives even when the container is deleted. I used a *named volume* (`db-data`), where Docker picks the location. The other type is a *bind mount*, where you choose an exact folder.
- **Environment variable (env var)** – A setting or password passed into a container at startup, under `environment:`. Not baked into the image, which keeps secrets out and lets the same image be reused with different settings.
- **Network** – Compose automatically creates a private network between the containers and gives each a name, so one container reaches another by its service name instead of an IP. Example: `WORDPRESS_DB_HOST: db`.
- **Port mapping** – The `ports` line, written `outside:inside` (e.g. `"8082:80"`), connects a VM port to a port inside the container.
- **depends_on** – Controls start order. `depends_on: db` starts the database before WordPress.
- **Restart policy** – `restart: always` auto-restarts the container if it crashes or the VM reboots.
- **Persistence / Ephemeral** – Ephemeral = temporary, lost on deletion (the running container). Persistent = survives (the volume on disk). This is the whole reason data goes in a volume.
- **Load balancer** – Sits in front of many identical containers and spreads traffic across them so none gets overwhelmed.
- **Kernel** – The core of the OS. All containers share the one host kernel, which is why they are light and fast, and why a container is not a full virtual machine.

---

## The same concepts in my own words

A container is just a program running inside its own little box.

A real app needs more than one box. My WordPress site needed two: one for the website and one for the database that remembers everything. That is multi-container.

Docker Compose is basically a recipe file plus one command. Instead of starting each box by hand, I write them all into one file and start everything with one command.

A volume is a safe folder that lives outside the box. Containers are throwaway, so if you delete the box everything inside disappears. The database keeps its data in a volume on the disk, outside the box, so it stays safe.

The network is like a road between the boxes plus a name tag, so one container finds another by its name. The environment variables are like sticky notes of settings and passwords handed to a box when it starts.

---

## Some things that finally clicked

An image is like a cookie cutter with the software already frozen inside. A container is the cookie you stamp from it. One image makes many identical containers. For different jobs, like a web server and a database, you need different images because each image only has one program inside it.

The rule is one container, one job. Keep them separate so that if one crashes the others keep going, and so each part can grow or update on its own.

All containers share one kernel. That is why they are light and fast, and that is the difference between a container and a full virtual machine.

For lots of users you do not make one box bigger. You make many copies of the busy box from the same image and put a load balancer in front to spread the traffic. The database usually stays one shared box.

When I write my own custom image at a job later, the connection works the same way. The image holds the code that knows HOW to connect. The environment variable tells it WHAT to connect to. So the same image can connect to a test database today and a production one tomorrow without rebuilding.

---

## RAM versus disk

When a container is running, the live part sits in RAM, which is fast but temporary. Stop the container and that RAM is freed, so the running part is gone. The files and the volume data live on disk, which is permanent. Every start loads the program fresh into RAM from the same disk files and reconnects the same volume. So the running part is temporary but the saved data survives.

I also asked what happens in a live environment if a server crashes while the database is working in RAM. My worry was real, and databases handle it by writing important data to disk straight away, keeping a recovery diary called a write-ahead log (WAL) so they can recover after a crash, and in big setups running extra copies on other servers (replication) plus backups. The point is that data spends almost no time living only in RAM.

---

## The install problem I hit

Docker the engine and Docker Compose are two separate things. Docker was already installed but Compose was not. The apt method could not find the package, because that package lives in Docker's own repo, not Ubuntu's. So I downloaded the Compose binary straight from GitHub.

```bash
sudo curl -SL "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64" \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
docker compose version    # => Docker Compose version v5.1.4
```

Two lessons: use `docker compose` with a space (new version), not `docker-compose` with a hyphen (old dead one). And run multi-line commands one at a time, because a pasted block got squished together and broke.

---

## The docker-compose.yml I wrote

```yaml
services:
  db:
    image: mysql:8.0
    container_name: wordpress-db
    restart: always
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass123
      MYSQL_ROOT_PASSWORD: rootpass123
    volumes:
      - db-data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    container_name: wordpress-site
    restart: always
    ports:
      - "8082:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass123
      WORDPRESS_DB_NAME: wordpress
    depends_on:
      - db

volumes:
  db-data:
```

What each part means:

- `services` is the list of my boxes (the two services are `db` and `wordpress`).
- `image` is which cutter to use, written as `name:version`.
- `environment` is the settings and passwords handed in at startup (env vars).
- `volumes: - db-data:/var/lib/mysql` connects the outside volume `db-data` to where MySQL keeps its data.
- `ports: - "8082:80"` means outside door 8082 goes to inside door 80 (port mapping).
- `WORDPRESS_DB_HOST: db` lets WordPress find the database by its name (networking).
- `depends_on: db` starts the database before WordPress.
- `restart: always` restarts the container automatically if it crashes.
- the `volumes` block at the bottom officially registers the named volume.

YAML rules that kept tripping me up: space after every colon, space after every dash, and indent each thing under its parent (two more spaces each step).

---

## Commands I ran

```bash
mkdir -p ~/wordpress-app
cd ~/wordpress-app

cat > docker-compose.yml << 'EOF'
... (the file above) ...
EOF
cat docker-compose.yml
docker compose config

docker compose up -d
docker compose ps

docker volume ls
docker volume inspect wordpress-app_db-data
```

After that I opened port 8082 in the Azure NSG firewall and went to `http://20.5.184.90:8082`, and the WordPress setup screen showed up.

---

## The delete and survive test

This was the part that proved volumes actually work.

First I published a blog post in WordPress so there was real data in the database. Then I destroyed the database container on purpose:

```bash
docker compose stop db
docker compose rm -f db
docker compose ps      # only wordpress-site left; site showed a DB connection error
```

Then I brought it back:

```bash
docker compose up -d
docker compose ps      # both Up again
```

Both containers were up again, and when I refreshed the site my blog post was still there.

It worked because deleting a container does not touch the `docker-compose.yml` file, the image, or the volume. It only removes the running box. So `docker compose up` read the recipe, rebuilt the box, and reconnected the same volume that still had my data.

---

## Where Compose fits and where it stops

Compose is used on a developer laptop, in CI/CD pipelines, and for small production setups – all single machine. Once you need to run across many machines with auto scaling and self healing, that is Kubernetes, which is Phase 4 for me.

---

## Where I am now

Done this week:

- Installed Docker Compose v2 (v5.1.4)
- Built a live WordPress and MySQL stack on the VM
- Covered and understood volumes, networking, environment variables, port mapping, depends_on and restart policy
- Passed the delete and survive test
- Practiced writing compose YAML by hand

