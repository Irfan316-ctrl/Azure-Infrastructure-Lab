Why mkdir -p ~/myapp:
- Organization: all project files in one place
- Build context: docker build . only copies from current directory (safety)
- Security: prevents accidentally copying secrets from home directory
- Professional structure: how real projects are organized

### 3. Understanding daemon off; (CRITICAL - You Asked Multiple Times)

You asked: "If daemon is on by default, why does nginx need daemon off?"

The Problem:
When nginx runs with daemon on (default), it backgrounds itself. The original process exits. Docker watches PID 1 (Process ID 1). When PID 1 exits, Docker thinks "main process finished, container must stop". Container stops immediately ❌

The Solution:
With daemon off;, nginx does NOT background itself. Stays in foreground. Original process (PID 1) KEEPS RUNNING. Docker sees "PID 1 still running! Container is alive". Container stays alive ✅

Why This Matters:
- Docker doesn't care if child processes are running
- Docker ONLY cares about PID 1
- If PID 1 is running = container is alive
- If PID 1 stops = container must stop
- Container lifetime = PID 1 lifetime

This is NOT "turning off" nginx. It's a MODE: "Don't run as daemon (background), stay in foreground (visible)"

On a regular Linux server: daemon on = good (you get prompt back, nginx keeps working)
In Docker: daemon off = required (keeps PID 1 running, container stays alive)

### 4. Dockerfile Line by Line Explanation

FROM nginx:latest
- Downloads nginx image from Docker Hub
- Creates Layer 1: nginx web server with all binaries
- Base for everything else

RUN echo "Hello..." > /usr/share/nginx/html/index.html
- Executes command DURING build (not at runtime)
- Creates Layer 2: adds your custom HTML file
- When container runs, nginx serves this file
- /usr/share/nginx/html/ is where nginx looks for web content

EXPOSE 80
- Documentation: "This container listens on port 80"
- Does NOT actually open the port
- Just tells people "this app uses port 80"
- Real opening happens with: -p 8081:80 flag when running

CMD ["nginx", "-g", "daemon off;"]
- Default command when container starts
- Runs as PID 1 (critical for lifecycle)
- "nginx" = the command to run
- "-g" = global option flag
- "daemon off;" = the option (don't background yourself)

### 5. Docker Build Process (What Actually Happens)

Command: docker build -t myapp:1.0 .

What happened step by step:
1. Docker daemon received the command
2. Read Dockerfile from current directory (.)
3. Parsed each instruction
4. For each instruction:
   - Created temporary container
   - Executed instruction inside temp container
   - Committed output as immutable layer
   - Cached the layer
   - Deleted temporary container
5. Stacked all layers using union mount (overlay2)
6. Stored image at /var/lib/docker/overlay2/
7. Tagged as myapp:1.0
8. Success: "Successfully built bc98b891829"

Build context = the . (current directory)
Docker only has access to files in current directory
Security: if Dockerfile was in home directory and you did docker build ., it could copy everything (passwords, keys, etc.)
That's why we made ~/myapp/ - contains only app files

Layer caching:
Build 1: All instructions execute, all layers created
Build 2 (after editing line 2): Layer 1 (FROM) cached, Layer 2 rebuilt, Layers 3-4 rebuild
Benefit: Fast rebuilds

Each layer immutable: can't be modified, only rebuilt

### 6. Image Architecture & Storage

What is an image:
Template/blueprint for containers
Union of immutable layers stacked on top of each other
Each layer is read-only
Content-addressable by SHA256 digest

Where it's stored:
/var/lib/docker/overlay2/ on your VM
Contains layer directories (one per instruction)
Image metadata (config, history)
Registry (maps tags to images)

Image size: ~150MB (nginx base ~100MB + your layer ~50MB)

How layers work:
FROM nginx:latest = Layer 1 (241MB nginx)
RUN echo... = Layer 2 (small, just HTML file)
EXPOSE 80 = Layer 3 (metadata, tiny)
CMD... = Layer 4 (metadata, tiny)

When running: Docker stacks layers bottom to top, creates writable delta layer on top

Image naming:
myapp:1.0
- myapp = image name
- 1.0 = tag (version label)

Tags are just pointers:
- myapp:1.0 → points to digest sha256:abc123
- myapp:latest → points to digest sha256:def456
- myapp:prod → points to digest sha256:ghi789

Different tags can point to same image OR different images

### 7. Docker Run - Complete Process

Command: docker run -d -p 8081:80 --name myapp-container myapp:1.0

What happened:
1. Docker daemon received command
2. Resolved image reference (myapp:1.0)
3. Located image locally
4. Created container-specific writable layer (delta layer)
5. Mounted all image layers (read-only) + container layer (writable)
6. Created isolated namespaces:
   - pid namespace: isolated process tree
   - network namespace: isolated network stack
   - mount namespace: isolated filesystem view
   - ipc namespace: isolated inter-process communication
7. Set up cgroup limits (resource constraints): CPU, memory, I/O
8. Configured networking:
   - Created veth pair (virtual ethernet)
   - Added iptables rule: forward 0.0.0.0:8081 to container:80
   - Connected to docker0 bridge network
9. Executed PID 1: nginx -g "daemon off;"
10. Container now running and monitored

Port mapping mechanism:
User → 0.0.0.0:8081 (VM)
  → iptables rule (host networking)
  → veth pair (virtual ethernet cable)
  → docker0 bridge network
  → container network namespace
  → 127.0.0.1:80 (inside container)
  → nginx receives request

Flags explained:
-d = detached (run in background, return immediately)
-p 8081:80 = port mapping (VM port 8081 → container port 80)
--name myapp-container = human-readable identifier
myapp:1.0 = image to use

Result: Container running, nginx listening on port 80 inside container, accessible via port 8081 on VM

### 8. Image vs Container (Deep Understanding)

Image:
- Template/blueprint
- Immutable (read-only)
- Stored persistently on disk at /var/lib/docker/
- Can have multiple versions/tags
- Size: takes disk space
- No running processes
- Can be pushed to registries
- Can be deleted and recreated

Container:
- Running instance created from image
- Writable delta layer on top of image
- Running processes with isolated namespaces
- Resource-limited with cgroups
- Ephemeral (deleted when stopped unless explicitly preserved)
- Takes RAM + disk (delta layer)
- Has PID 1 that Docker watches
- Multiple containers from same image

Analogy:
Image = class definition in programming (the blueprint)
Container = object/instance (the actual running thing)

class Nginx {
  // this defines Nginx
}

Nginx obj1 = new Nginx();  // instance 1
Nginx obj2 = new Nginx();  // instance 2

Same way:
Image myapp:1.0 = definition
Container 1: running on 8081
Container 2: running on 8082

Both containers from same image but completely isolated

Isolation:
Container 1 can't see Container 2's files
Container 1 can't access Container 2's network
Container 1 crash doesn't affect Container 2
They can run simultaneously on same VM

### 9. Container Verification (What You Did)

docker ps | grep myapp
Output showed:
CONTAINER ID: b1929d6e1f1fc28...
IMAGE: myapp:1.0
COMMAND: "/docker-entrypoint..."
CREATED: 36 seconds ago
STATUS: Up 35 seconds ✅
PORTS: 0.0.0.0:8081->80/tcp
NAMES: myapp-container

This proved:
- Container is running ✅
- Port mapping working ✅
- Process is PID 1 and alive ✅

docker logs myapp-container
Output showed:
nginx/1.31.2
built by gcc 14.2.0 (Debian 14.2.0-19)
Configuration complete; ready for start up
start worker process 28
start worker process 29

This proved:
- nginx started successfully ✅
- Worker processes running ✅
- Container is healthy ✅
- Ready to serve requests ✅

### 10. Docker Hub Registry & Pushing

What is Docker Hub:
Public registry at hub.docker.com
Store images
Version them with tags
Share publicly or privately
Control who can access

Push workflow:

docker login
- Web-based authentication
- Authenticates with Docker Hub
- Allows you to push images
- Status: Login Succeeded

docker tag myapp:1.0 irfanay/myapp:1.0
- Adds username prefix to image name
- Required format for Docker Hub: username/imagename:tag
- Docker Hub will know where to push based on username
- Just creates a pointer, doesn't upload anything

docker push irfanay/myapp:1.0
- Connects to Docker Hub
- Splits image into layers
- Compresses each layer
- Uploads layers (only missing ones, reuses existing)
- Verifies checksums
- Stores manifest (JSON file referencing all layers)
- Updates tag pointer
- Image now publicly available

Image naming convention:
irfanay/myapp:1.0
- irfanay = your Docker Hub username
- myapp = image name
- 1.0 = tag (version label)

Full reference: docker.io/irfanay/myapp:1.0

Tag behavior:
- Tag is just a pointer
- myapp:1.0 → points to image digest sha256:abc123...
- myapp:latest → can point to different digest
- Same image can have multiple tags
- Different tags can point to different images

Public URL: https://hub.docker.com/r/irfanay/myapp
Anyone can: docker pull irfanay/myapp:1.0

### 11. Security - No Secrets in Images

❌ WRONG:
```dockerfile
FROM node:18
ENV DB_PASSWORD=secret123
ENV API_KEY=sk_live_abc123
ENV STRIPE_KEY=sk_stripe_123
COPY . /app
RUN npm install
CMD ["node", "server.js"]
```

Problems:
- Image contains all secrets
- Anyone who pulls image sees secrets
- Secrets visible in image history/layers (can't delete)
- Cannot rotate without rebuilding entire image
- If pushed to public registry, secrets exposed
- Auditing impossible
- Single point of failure

✅ CORRECT:
```dockerfile
FROM node:18
COPY . /app
RUN npm install
# NO secrets in Dockerfile
CMD ["node", "server.js"]
```

Provide secrets at runtime:
Option 1: Environment variables
docker run -e DB_PASSWORD=secret123 myapp:1.0

Option 2: Secrets Manager (Better)
HashiCorp Vault:
- App starts
- App requests: "Give me database password"
- Vault checks: "Is this app authorized?"
- Vault: "Yes, here's password: secret123"
- App stores in memory
- Connection established

Azure Key Vault:
- Same concept
- Azure-native
- Integrated with Azure services

AWS Secrets Manager:
- Same concept
- AWS-native
- Integrated with AWS services

Benefits of secrets manager:
- Secrets never in image
- Can rotate without rebuild
- Fine-grained access control per application
- Audit trails (who accessed what when)
- Secure in transit (encrypted)
- Centralized management

Principle:
Image = application code + dependencies ONLY
Secrets = managed externally, injected at runtime

### 12. Unfamiliar Job Posting Terms Explained

CI/CD Pipelines (Phase 3):
Problem: Manual deployment is slow and error-prone
Solution: Automated pipeline

Workflow:
1. Developer pushes code to GitHub
2. Tests run automatically
3. Code quality checked
4. Docker image built automatically
5. Image pushed to registry automatically
6. Deployment triggered automatically
7. Application updated in production

Tools: Jenkins (complex), GitHub Actions (easier), GitLab CI, AWS CodePipeline

Terraform (Phase 3):
Problem: Creating infrastructure by clicking cloud console is manual, error-prone, not reproducible

Solution: Write code
```hcl
resource "aws_instance" "web" {
  ami = "ami-123456"
  instance_type = "t2.micro"
}

resource "aws_db_instance" "db" {
  engine = "postgres"
  allocated_storage = 20
}
```

Benefits:
- Version controlled (git history)
- Reproducible (same code = same infrastructure)
- Automatable (deploy 100 servers with one command)
- Testable
- Documentable

Kubernetes (Phase 4+):
Problem: You have 1000 containers. Containers crash. Need to restart. Need to scale up/down. Need zero-downtime updates.

Solution: Kubernetes handles it
- Auto-restarts failed containers
- Scales based on load
- Updates with zero downtime
- Load balances traffic
- Manages resources across cluster
- Health checks
- Self-healing

Prometheus/Monitoring (Phase 4+):
Problem: Application is running. Is it healthy? How much CPU? How many requests/second? What's error rate?

Solution: Prometheus
- Scrapes metrics from applications
- Stores time-series data
- Creates dashboards
- Sets up alerts
- Metrics: request_latency_ms, error_rate, cpu_usage, memory_usage

Vault/Key Vault/Secrets Manager:
Problem: Where to safely store passwords? How to rotate them? Who can access? How to audit?

Solution: Centralized secret management
- Role-based access control
- Automatic rotation
- Audit trails
- API for apps to request secrets

AWS vs Azure:
You're learning: Azure (good choice!)
Jobs ask for: AWS (also good!)

Core concepts identical:
- Azure VM = AWS EC2 (virtual machines)
- Azure VNet = AWS VPC (networking)
- Azure SQL = AWS RDS (managed databases)
- Azure Storage = AWS S3 (object storage)
- Azure ACR = AWS ECR (container registry)

Transfer time: 2 weeks to learn AWS specifics once you master Azure

DevSecOps:
DevOps + Security
Additional concerns: secrets management, container scanning, security policies, compliance, access controls, audit logging

### 13. Docker Concepts - Container Lifecycle

Container states:
Created → Running → Paused → Stopped → Removed

PID 1 is critical:
- Container = PID 1 process
- If PID 1 exits → Container exits (must)
- If PID 1 paused → Container unresponsive
- Container lifetime = PID 1 lifetime

docker stop:
- Sends SIGTERM to PID 1
- Process has grace period (10s by default)
- If not exited, sends SIGKILL
- Container exits

docker start:
- Reuses same container layer
- Reruns CMD
- Attaches new network namespace

docker rm:
- Deletes container completely
- Writable delta layer deleted
- Image still exists (can recreate container)

### 14. Docker Architecture - The Full Picture

7 Layers on Your VM:

Layer 1: Hardware
- CPU, RAM, Disk, Network card from Azure

Layer 2: OS
- Ubuntu 22.04 kernel
- Filesystem (/home, /etc, /var, /docker)
- Drivers

Layer 3: Services
- systemd (manages everything)
- SSH daemon
- Docker daemon (background service)
- Cron jobs

Layer 4: Docker Engine
- Images stored locally
- Daemon manages containers
- Container processes

Layer 5: Networking
- IP address (20.5.184.90)
- Security groups (firewall rules)
- Port mappings (iptables)

Layer 6: Users & Permissions
- root user
- azureuser
- User permissions (chmod, chown)

Layer 7: Filesystem
- Everything stored on disk
- /var/lib/docker/ (Docker files)
- /home/azureuser/ (your files)

Docker sits INSIDE the OS, not replacing it:
Your app code
  ↓
Docker container (isolated Linux process)
  ↓
Docker daemon (Linux process)
  ↓
Ubuntu OS (kernel)
  ↓
Azure hardware

### 15. Commands Used Today

mkdir -p ~/myapp
cd ~/myapp

cat > Dockerfile << 'EOF'
FROM nginx:latest
RUN echo "Hello from my custom Docker image - Irfan's DevOps Portfolio!" > /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF

docker build -t myapp:1.0 .
docker images | grep myapp
docker run -d -p 8081:80 --name myapp-container myapp:1.0
docker ps | grep myapp
docker logs myapp-container
docker login
docker tag myapp:1.0 irfanay/myapp:1.0
docker push irfanay/myapp:1.0

## Phase 2 Progress

- Phase 2 Week 1: COMPLETE - Docker installation, basic containers, Docker Hub concepts
- Phase 2 Week 2: COMPLETE - Custom Dockerfile, building images, pushing to Docker Hub
- Phase 2 Week 3-4: PENDING - Docker Compose, volumes, networking

## Next Steps

Phase 3 (Aug-Sept): Terraform and CI/CD pipelines
Phase 4 (Oct-Nov): Multi-tier and microservices projects
Dec-Jan: Job applications with real portfolio pieces

## Portfolio Proof

GitHub: Dockerfile + documentation
Docker Hub: https://hub.docker.com/r/irfanay/myapp
Skills demonstrated: Image building, registry management, security awareness, architecture understanding
Interview ready: Can explain every concept and why it matters