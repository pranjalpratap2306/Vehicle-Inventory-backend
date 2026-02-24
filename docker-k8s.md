Docker & Kubernetes — Interview Prep + Project Containerization Guide
Table of Contents
Part 1 — Docker
What Is Docker?
Containers vs Virtual Machines
Docker Architecture
Key Docker Concepts
Docker Commands Cheat Sheet
Multi-Stage Builds
Docker Interview Q&A
Part 2 — Kubernetes
What Is Kubernetes?
Kubernetes Architecture
Core Kubernetes Objects
Key Kubernetes Concepts
kubectl Commands Cheat Sheet
Kubernetes Interview Q&A
Part 3 — Fleet Manager Containerization
Dockerfile Walkthrough
CI/CD Pipeline — buildspec.yaml
Deployment Architecture
Profile-Based Configuration
Part 1 — Docker
What Is Docker?
Docker is an open-source platform that packages applications and their dependencies into lightweight, portable units called containers. A container includes everything the application needs to run — code, runtime, libraries, environment variables, and config files — so it behaves identically on any machine.

Why Docker exists:

"It works on my machine" problem — Docker eliminates environment inconsistencies between development, staging, and production.
Dependency isolation — Each container has its own filesystem and libraries, so different apps can use different versions of Java, Node, Python, etc. without conflicts.
Fast startup — Containers start in seconds (vs minutes for VMs) because they share the host OS kernel.
Efficient resource usage — No guest OS overhead; containers are just isolated processes.
Containers vs Virtual Machines
graph TB
    subgraph VirtualMachines [Virtual Machines]
        VM_HW[Hardware] --> VM_HOS[Host OS]
        VM_HOS --> VM_HYP[Hypervisor]
        VM_HYP --> VM_G1[Guest OS 1]
        VM_HYP --> VM_G2[Guest OS 2]
        VM_G1 --> VM_A1[App A + Libs]
        VM_G2 --> VM_A2[App B + Libs]
    end

    subgraph DockerContainers [Docker Containers]
        DC_HW[Hardware] --> DC_HOS[Host OS]
        DC_HOS --> DC_ENG[Docker Engine]
        DC_ENG --> DC_C1[Container 1<br/>App A + Libs]
        DC_ENG --> DC_C2[Container 2<br/>App B + Libs]
    end
Feature	Container	Virtual Machine
OS	Shares host kernel	Full guest OS per VM
Startup time	Seconds	Minutes
Size	MBs (tens to hundreds)	GBs
Resource overhead	Minimal	Heavy (entire OS running)
Isolation	Process-level	Hardware-level
Portability	Extremely portable	Less portable (hypervisor-dependent)
Density	Hundreds per host	Tens per host
Use case	Microservices, CI/CD	Full OS isolation, legacy apps
Docker Architecture
graph LR
    subgraph ClientSide [Client]
        CLI["Docker CLI<br/>(docker build, run, push)"]
    end

    subgraph DockerHost [Docker Host]
        Daemon["Docker Daemon<br/>(dockerd)"]
        Images["Images"]
        Containers["Containers"]
        Daemon --> Images
        Daemon --> Containers
    end

    subgraph RegistrySide [Registry]
        Registry["Docker Registry<br/>(Docker Hub, ECR, GCR)"]
    end

    CLI -->|REST API| Daemon
    Daemon -->|"docker pull / push"| Registry
Component	Role
Docker CLI	Command-line tool you type commands into (docker build, docker run, etc.). Sends instructions to the daemon via REST API.
Docker Daemon (dockerd)	Background process that manages images, containers, networks, and volumes on the host. Does the actual work.
Docker Image	Read-only template (blueprint) for creating containers. Built from a Dockerfile. Composed of stacked layers.
Docker Container	A running instance of an image. It's an isolated process with its own filesystem, networking, and process space.
Docker Registry	Remote storage for images. Docker Hub is the default public registry. AWS ECR, Google GCR, and GitHub GHCR are private alternatives.
Key Docker Concepts
Dockerfile
A text file with instructions to build a Docker image. Each instruction creates a layer in the image.

FROM openjdk:17-jdk-slim        # Base image (layer 1)
WORKDIR /app                     # Set working directory (layer 2)
COPY target/*.jar app.jar        # Copy JAR into image (layer 3)
EXPOSE 8080                      # Document the port (metadata only)
ENTRYPOINT ["java", "-jar", "app.jar"]  # Default command to run
Image Layers and Build Cache
Each Dockerfile instruction creates an immutable layer. Docker caches these layers, so if a layer hasn't changed, it's reused from cache during subsequent builds.

graph TB
    L1["Layer 1: FROM openjdk:17<br/>(~200 MB, cached)"] --> L2["Layer 2: WORKDIR /app<br/>(tiny, cached)"]
    L2 --> L3["Layer 3: COPY pom.xml<br/>(cached if pom unchanged)"]
    L3 --> L4["Layer 4: RUN mvn dependency:go-offline<br/>(cached if deps unchanged)"]
    L4 --> L5["Layer 5: COPY src/<br/>(rebuilt when code changes)"]
    L5 --> L6["Layer 6: RUN mvn package<br/>(rebuilt)"]
Best practice: Put frequently changing instructions (like COPY src/) LATER in the Dockerfile so earlier layers (like dependency download) stay cached.

Volumes
Volumes persist data beyond the container's lifecycle. Without volumes, all data inside a container is lost when the container is removed.

# Named volume (managed by Docker)
docker run -v my-data:/app/data my-image

# Bind mount (maps host directory)
docker run -v /host/path:/container/path my-image
Docker Networking
Network Type	Description
bridge (default)	Containers on the same bridge can talk to each other. Isolated from the host network.
host	Container uses the host's network directly. No isolation.
none	No networking. Fully isolated.
overlay	For multi-host networking in Docker Swarm / Kubernetes.
Docker Commands Cheat Sheet
Image Commands
docker build -t my-app:1.0 .         # Build image from Dockerfile in current dir
docker images                          # List all local images
docker rmi my-app:1.0                  # Remove an image
docker pull nginx:latest               # Download image from registry
docker push myrepo/my-app:1.0         # Upload image to registry
docker tag my-app:1.0 myrepo/my-app:1.0  # Tag an image for a registry
docker image prune                     # Remove unused images
Container Commands
docker run -d -p 8080:8080 --name app my-app:1.0  # Run container in background
docker ps                              # List running containers
docker ps -a                           # List all containers (including stopped)
docker stop app                        # Gracefully stop a container
docker start app                       # Start a stopped container
docker restart app                     # Restart a container
docker rm app                          # Remove a stopped container
docker logs app                        # View container logs
docker logs -f app                     # Follow logs in real-time
docker exec -it app /bin/bash          # Open shell inside running container
docker inspect app                     # View detailed container metadata
System Commands
docker system df                       # Show disk usage
docker system prune -a                 # Remove ALL unused data (images, containers, volumes)
docker network ls                      # List networks
docker volume ls                       # List volumes
Multi-Stage Builds
A multi-stage build uses multiple FROM statements in a single Dockerfile. Each FROM starts a new build stage. Only the final stage becomes the output image — earlier stages are discarded.

Why it matters:

Build tools (Maven, npm, gcc) are needed to compile code but NOT needed to run it
Without multi-stage: final image includes build tools + source code = huge image (500MB+)
With multi-stage: final image has only the runtime + compiled artifact = small image (200MB)
graph LR
    subgraph Stage1 ["Stage 1: Build (discarded)"]
        S1A["maven:3.9.4 image<br/>(~800 MB)"]
        S1B["Download dependencies"]
        S1C["Compile source code"]
        S1D["Produce app.jar"]
        S1A --> S1B --> S1C --> S1D
    end

    subgraph Stage2 ["Stage 2: Runtime (final image)"]
        S2A["openjdk:17-slim<br/>(~200 MB)"]
        S2B["Copy app.jar from Stage 1"]
        S2C["Final image: ~220 MB"]
        S2A --> S2B --> S2C
    end

    S1D -->|"COPY --from=build"| S2B
Docker Interview Q&A
Q1: What is the difference between CMD and ENTRYPOINT?

ENTRYPOINT sets the main executable that always runs. CMD provides default arguments that can be overridden at runtime.

ENTRYPOINT ["java", "-jar"]   # Always runs java -jar
CMD ["app.jar"]                # Default arg, overridable: docker run my-app other.jar
If you use only CMD, the entire command can be replaced: docker run my-app /bin/bash replaces the CMD. If you use ENTRYPOINT, docker run arguments are appended to it.

Q2: What is the difference between COPY and ADD?

COPY simply copies files from host to image. ADD does the same but also supports URL downloads and auto-extracts tar archives. Best practice: always use COPY unless you specifically need ADD's extra features.

Q3: What happens when you run docker run?

Docker checks if the image exists locally; if not, pulls it from the registry
Creates a new container from the image (adds a writable layer on top)
Allocates a network interface and IP address
Starts the container and executes the ENTRYPOINT/CMD
Q4: How do you reduce Docker image size?

Use multi-stage builds (separate build and runtime stages)
Use slim/alpine base images (openjdk:17-jdk-slim instead of openjdk:17)
Combine RUN commands to reduce layers: RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
Use .dockerignore to exclude unnecessary files (.git, node_modules, target/)
Remove build artifacts and caches in the same layer they're created
Q5: What is a Docker layer?

Each instruction in a Dockerfile (FROM, RUN, COPY, etc.) creates an immutable filesystem layer. Layers are stacked on top of each other. Docker caches layers and reuses them when nothing has changed, making rebuilds faster.

Q6: What is the difference between docker stop and docker kill?

docker stop sends SIGTERM (graceful shutdown signal), waits 10 seconds, then sends SIGKILL. docker kill sends SIGKILL immediately (forceful termination). Always prefer docker stop to allow the application to clean up (close DB connections, flush logs, etc.).

Q7: What is .dockerignore?

Similar to .gitignore. It lists files/directories that should NOT be sent to the Docker daemon during build. This reduces build context size and prevents sensitive files from being included in the image.

Q8: How does container networking work?

By default, Docker creates a bridge network. Containers on the same bridge can communicate using container names as hostnames. To expose a container port to the host, use -p host:container (e.g., -p 8080:8080).

Q9: How do you persist data in Docker?

Using volumes or bind mounts. Volumes are managed by Docker and stored in /var/lib/docker/volumes/. Bind mounts map a host directory directly into the container. Volumes are preferred for production because they're managed by Docker and work across platforms.

Q10: What is Docker Compose?

A tool for defining and running multi-container applications using a YAML file (docker-compose.yml). You define services (e.g., app, database, cache), their images, ports, volumes, and networks in one file, then start everything with docker-compose up.

Q11: What is the difference between an image and a container?

An image is a read-only blueprint/template. A container is a running instance of an image. You can create many containers from one image. Analogy: image = class, container = object.

Q12: How do you pass environment variables to a container?

docker run -e DB_PASSWORD=secret my-app            # Single variable
docker run --env-file .env my-app                   # From a file
In Dockerfile: ENV DB_PASSWORD=default_value

Q13: What is the difference between EXPOSE and -p?

EXPOSE in the Dockerfile is documentation only — it tells others which port the app uses but doesn't actually publish it. -p 8080:8080 in docker run actually maps the container port to the host.

Q14: What is a dangling image?

An image that has no tag and is not referenced by any container. Usually created when you rebuild an image with the same tag — the old image loses its tag and becomes dangling. Clean up with docker image prune.

Q15: How do health checks work in Docker?

The HEALTHCHECK instruction tells Docker how to test if a container is still working. Docker runs the check command at intervals and marks the container as healthy, unhealthy, or starting.

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1
Q16: What is the Docker build context?

When you run docker build ., Docker sends the entire directory (the "build context") to the daemon. This is why .dockerignore is important — you don't want to send .git/, node_modules/, or target/ to the daemon.

Part 2 — Kubernetes
What Is Kubernetes?
Kubernetes (K8s) is an open-source container orchestration platform originally developed by Google. It automates the deployment, scaling, and management of containerized applications.

Problems Kubernetes solves:

Scaling — Automatically scale containers up/down based on load
Self-healing — Restart crashed containers, replace unhealthy ones, reschedule onto healthy nodes
Load balancing — Distribute traffic across container replicas
Rolling updates — Deploy new versions with zero downtime
Service discovery — Containers find each other by name, not IP
Secret management — Securely store and inject passwords, API keys, certificates
Resource management — Set CPU/memory limits per container, bin-pack efficiently
Kubernetes Architecture
graph TB
    subgraph ControlPlane [Control Plane - Master]
        API["API Server<br/>(kube-apiserver)"]
        ETCD["etcd<br/>(cluster state store)"]
        SCHED["Scheduler<br/>(kube-scheduler)"]
        CM["Controller Manager<br/>(kube-controller-manager)"]
        API --> ETCD
        SCHED --> API
        CM --> API
    end

    subgraph WorkerNode1 [Worker Node 1]
        KL1["kubelet"]
        KP1["kube-proxy"]
        CR1["Container Runtime<br/>(containerd)"]
        P1A["Pod A"]
        P1B["Pod B"]
        KL1 --> CR1
        CR1 --> P1A
        CR1 --> P1B
    end

    subgraph WorkerNode2 [Worker Node 2]
        KL2["kubelet"]
        KP2["kube-proxy"]
        CR2["Container Runtime<br/>(containerd)"]
        P2A["Pod C"]
        P2B["Pod D"]
        KL2 --> CR2
        CR2 --> P2A
        CR2 --> P2B
    end

    API --> KL1
    API --> KL2
    User["kubectl / CI/CD"] --> API
Control Plane Components
Component	Role
API Server (kube-apiserver)	The front door to Kubernetes. All communication goes through it — kubectl, dashboards, other components. Validates and processes REST requests.
etcd	Distributed key-value store that holds ALL cluster state (what pods exist, their status, configs, secrets). The single source of truth.
Scheduler (kube-scheduler)	Watches for new pods with no assigned node, picks the best node based on resource availability, affinity rules, and constraints.
Controller Manager	Runs controller loops that watch cluster state and make changes to reach desired state. E.g., if a Deployment says 3 replicas but only 2 exist, it creates one more.
Worker Node Components
Component	Role
kubelet	Agent running on each node. Receives pod specs from the API server, ensures containers are running and healthy. Reports status back.
kube-proxy	Handles networking rules on each node. Routes traffic to the correct pod, implements Service load balancing using iptables or IPVS.
Container Runtime	The software that actually runs containers. Kubernetes supports containerd (default), CRI-O, etc. Docker was removed as a runtime in K8s 1.24.
Core Kubernetes Objects
Pod
The smallest deployable unit. A pod wraps one or more containers that share the same network namespace (same IP, same localhost) and storage volumes.

apiVersion: v1
kind: Pod
metadata:
  name: fleet-manager-pod
spec:
  containers:
    - name: fleet-manager
      image: 053357668713.dkr.ecr.us-east-2.amazonaws.com/inventorybackend-repo:latest
      ports:
        - containerPort: 8080
Deployment
Manages a set of identical pod replicas. Handles rolling updates, rollbacks, and scaling. You almost never create pods directly — you create Deployments.

apiVersion: apps/v1
kind: Deployment
metadata:
  name: fleet-manager-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fleet-manager
  template:
    metadata:
      labels:
        app: fleet-manager
    spec:
      containers:
        - name: fleet-manager
          image: 053357668713.dkr.ecr.us-east-2.amazonaws.com/inventorybackend-repo:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
Service
Provides a stable network endpoint (IP + DNS name) to access a set of pods. Pods are ephemeral (they come and go), but a Service provides a constant address.

Service Type	Description
ClusterIP (default)	Internal IP, accessible only within the cluster
NodePort	Exposes on a static port on each node (30000-32767 range)
LoadBalancer	Provisions an external load balancer (cloud provider, e.g., AWS ELB)
ExternalName	Maps to a DNS name (e.g., an RDS endpoint)
apiVersion: v1
kind: Service
metadata:
  name: fleet-manager-service
spec:
  type: LoadBalancer
  selector:
    app: fleet-manager
  ports:
    - port: 80
      targetPort: 8080
ConfigMap
Stores non-sensitive configuration data as key-value pairs. Injected into pods as environment variables or mounted as files.

apiVersion: v1
kind: ConfigMap
metadata:
  name: fleet-manager-config
data:
  SPRING_PROFILES_ACTIVE: "prd"
  SERVER_PORT: "5000"
  CORS_ALLOWED_ORIGINS: "http://localhost:4200"
Secret
Like ConfigMap but for sensitive data (passwords, API keys, certificates). Values are Base64-encoded (NOT encrypted by default — use sealed secrets or external vaults for real encryption).

apiVersion: v1
kind: Secret
metadata:
  name: fleet-manager-secrets
type: Opaque
data:
  DB_PASSWORD: c2VjcmV0cGFzc3dvcmQ=      # base64 encoded
  JWT_SECRET_KEY: YVA5IXhAMyNMJHolN14=     # base64 encoded
Namespace
A virtual cluster within a cluster. Used to organize resources and apply access control.

kubectl create namespace production
kubectl create namespace staging
Ingress
Manages external HTTP/HTTPS access to services. Provides URL routing, SSL termination, and name-based virtual hosting.

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fleet-manager-ingress
spec:
  rules:
    - host: api.fleet-manager.in
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: fleet-manager-service
                port:
                  number: 80
Key Kubernetes Concepts
Scaling
# Manual scaling
kubectl scale deployment fleet-manager --replicas=5

# Auto-scaling based on CPU usage
kubectl autoscale deployment fleet-manager --min=2 --max=10 --cpu-percent=70
Horizontal Pod Autoscaler (HPA) watches CPU/memory metrics and adjusts replica count automatically.

Rolling Updates
When you update a Deployment's image, Kubernetes performs a rolling update by default:

graph LR
    subgraph Before [Before Update]
        V1A["Pod v1"]
        V1B["Pod v1"]
        V1C["Pod v1"]
    end

    subgraph During ["During Rolling Update"]
        V1D["Pod v1"]
        V1E["Pod v1"]
        V2A["Pod v2 (new)"]
    end

    subgraph After [After Update]
        V2B["Pod v2"]
        V2C["Pod v2"]
        V2D["Pod v2"]
    end

    Before --> During --> After
Creates a new pod with the updated image
Waits for it to become healthy
Terminates one old pod
Repeats until all pods are updated
Zero downtime throughout
kubectl set image deployment/fleet-manager fleet-manager=my-repo/fleet-manager:v2.0
kubectl rollout status deployment/fleet-manager    # Watch progress
kubectl rollout undo deployment/fleet-manager      # Rollback if needed
Self-Healing
Kubernetes continuously monitors pod health:

Crashed container — kubelet automatically restarts it (restart policy)
Failed health check — pod is killed and replaced
Node failure — scheduler reschedules pods to other healthy nodes
Fewer replicas than desired — controller creates new pods
Liveness and Readiness Probes
Probe	Purpose	What happens on failure
Liveness	"Is the container still running?"	Container is killed and restarted
Readiness	"Is the container ready to receive traffic?"	Pod is removed from Service load balancer (no traffic sent)
Startup	"Has the container finished starting?"	Delays liveness/readiness checks until startup is complete
livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
Resource Limits
resources:
  requests:           # Guaranteed minimum
    cpu: "250m"       # 250 millicores = 0.25 CPU
    memory: "512Mi"   # 512 MB
  limits:             # Maximum allowed
    cpu: "500m"       # 0.5 CPU
    memory: "1Gi"     # 1 GB
Requests — Used by the scheduler to find a node with enough resources. Guaranteed to the pod.
Limits — Maximum the container can use. If it exceeds memory limit, it's killed (OOMKilled). If it exceeds CPU limit, it's throttled.
kubectl Commands Cheat Sheet
Cluster Info
kubectl cluster-info                           # Display cluster endpoint info
kubectl get nodes                              # List all nodes
kubectl get namespaces                         # List all namespaces
kubectl config current-context                 # Show current cluster context
kubectl config get-contexts                    # List all configured contexts
Pod Operations
kubectl get pods                               # List pods in default namespace
kubectl get pods -n production                 # List pods in specific namespace
kubectl get pods -o wide                       # Show extra info (node, IP)
kubectl describe pod <pod-name>                # Detailed pod info + events
kubectl logs <pod-name>                        # View pod logs
kubectl logs -f <pod-name>                     # Follow logs in real-time
kubectl logs <pod-name> -c <container-name>    # Logs from specific container
kubectl exec -it <pod-name> -- /bin/bash       # Shell into a pod
kubectl delete pod <pod-name>                  # Delete a pod
kubectl port-forward <pod-name> 8080:8080      # Forward local port to pod
Deployment Operations
kubectl get deployments                        # List deployments
kubectl describe deployment <name>             # Detailed deployment info
kubectl apply -f deployment.yaml               # Create/update from YAML
kubectl delete deployment <name>               # Delete deployment
kubectl scale deployment <name> --replicas=5   # Scale replicas
kubectl rollout status deployment <name>       # Check rollout progress
kubectl rollout history deployment <name>      # View rollout history
kubectl rollout undo deployment <name>         # Rollback to previous version
Service Operations
kubectl get services                           # List services
kubectl describe service <name>                # Detailed service info
kubectl expose deployment <name> --port=80 --target-port=8080 --type=LoadBalancer
Config and Secrets
kubectl get configmaps                         # List ConfigMaps
kubectl get secrets                            # List Secrets
kubectl create secret generic my-secret --from-literal=password=abc123
kubectl create configmap my-config --from-file=app.properties
Debugging
kubectl get events --sort-by=.metadata.creationTimestamp  # Recent events
kubectl top pods                               # CPU/memory usage per pod
kubectl top nodes                              # CPU/memory usage per node
kubectl get pod <name> -o yaml                 # Full YAML definition
Kubernetes Interview Q&A
Q1: What is a Pod? Why not just run containers directly?

A Pod is the smallest deployable unit in Kubernetes — a wrapper around one or more containers that share the same network namespace (same IP, same localhost) and storage volumes. Kubernetes manages pods, not individual containers, because pods provide a higher-level abstraction for co-located, tightly-coupled containers (e.g., a main app + a sidecar logging agent).

Q2: What is the difference between a Deployment and a Pod?

A Pod is a single instance. A Deployment manages a set of identical pod replicas and handles rolling updates, rollbacks, and self-healing. You almost never create Pods directly — you create Deployments that create Pods for you.

Q3: What is the difference between a Deployment and a StatefulSet?

Deployments are for stateless applications — pods are interchangeable. StatefulSets are for stateful applications (databases, message queues) — each pod gets a stable hostname (e.g., mysql-0, mysql-1), persistent storage, and ordered startup/shutdown.

Q4: What happens when a node goes down?

The Control Plane detects the node failure (kubelet stops reporting). The controller manager marks its pods as Unknown. After a timeout (default 5 minutes), the scheduler reschedules those pods onto other healthy nodes. If you have a Deployment with 3 replicas and a node dies with 1 pod on it, Kubernetes creates a new pod on a healthy node to maintain 3 replicas.

Q5: How does a Service route traffic to Pods?

A Service uses label selectors to identify its target pods. kube-proxy on each node sets up iptables/IPVS rules that load-balance traffic to all pods matching the selector. When pods are created or destroyed, the Service's endpoint list is automatically updated.

Q6: What is the difference between ClusterIP, NodePort, and LoadBalancer?

ClusterIP: Internal-only IP. Only accessible within the cluster. Used for inter-service communication.
NodePort: Opens a static port (30000-32767) on every node. Accessible from outside via <NodeIP>:<NodePort>.
LoadBalancer: Provisions a cloud provider's load balancer (e.g., AWS ELB). Accessible via a public IP/DNS.
Each type builds on the previous: LoadBalancer creates a NodePort which creates a ClusterIP.

Q7: What is an Ingress?

An Ingress is an API object that manages external HTTP/HTTPS access to Services. It provides URL-based routing (/api goes to service A, /web goes to service B), SSL termination, and name-based virtual hosting — all through a single external IP/load balancer.

Q8: How do rolling updates work?

Kubernetes gradually replaces old pods with new ones. It creates a new pod with the updated image, waits for it to pass readiness checks, then terminates an old pod. This continues until all pods are updated. At no point are all pods down — this ensures zero-downtime deployments.

Q9: What is the difference between liveness and readiness probes?

Liveness probe: "Is the container alive?" Failure = container is killed and restarted. Catches deadlocks, infinite loops.
Readiness probe: "Can the container serve traffic?" Failure = pod is removed from the Service's load balancer. Catches slow startups, dependency unavailability.
Q10: What is a ConfigMap vs a Secret?

Both store key-value configuration data. ConfigMaps are for non-sensitive data (URLs, feature flags). Secrets are for sensitive data (passwords, tokens) and are Base64-encoded. Secrets can be encrypted at rest with additional configuration.

Q11: What is a Namespace?

A virtual partition within a cluster. Used to organize resources (e.g., production, staging, testing), apply resource quotas, and control access via RBAC. Default namespaces: default, kube-system, kube-public.

Q12: What is Helm?

Helm is the "package manager for Kubernetes." It bundles related Kubernetes YAML manifests into reusable charts with templating support. Instead of managing dozens of YAML files, you install/upgrade/rollback an entire application with a single command: helm install my-app ./chart.

Q13: What is the difference between a ReplicaSet and a Deployment?

A ReplicaSet ensures a specified number of pod replicas are running. A Deployment manages ReplicaSets and adds rolling update/rollback capabilities. You never create ReplicaSets directly — Deployments create them under the hood.

Q14: How does Kubernetes achieve self-healing?

Through control loops in the Controller Manager. The Deployment controller continuously compares the desired state (e.g., 3 replicas) with the actual state. If a pod crashes, the controller detects the difference and creates a new pod. If a node dies, the scheduler places replacement pods on healthy nodes.

Q15: What is a DaemonSet?

A DaemonSet ensures that a copy of a pod runs on every node (or every matching node). Used for infrastructure tasks: log collectors (Fluentd), monitoring agents (Prometheus node-exporter), network plugins (Calico).

Q16: What is a PersistentVolume (PV) and PersistentVolumeClaim (PVC)?

PV is a piece of storage provisioned by an admin (or dynamically). PVC is a user's request for storage. The binding works like this: PVC says "I need 10GB of SSD storage," Kubernetes finds a matching PV and binds them. The pod then mounts the PVC.

Q17: How do you debug a pod that won't start?

kubectl describe pod <name> — Check Events section for scheduling errors, image pull failures
kubectl logs <name> — Check application logs
kubectl get events — Cluster-wide events
Check if image exists: docker pull <image>
Check resource limits: node might not have enough CPU/memory
kubectl exec -it <name> -- /bin/bash — Shell in if container is running but misbehaving
Part 3 — Fleet Manager Containerization
Dockerfile Walkthrough
The project uses a multi-stage Docker build to create a lightweight production image.

File: Dockerfile

# ============================================================
# STAGE 1: BUILD — Uses Maven to compile the application
# ============================================================
FROM maven:3.9.4-eclipse-temurin-17 AS build
WORKDIR /app

# Copy the Maven project files
COPY pom.xml ./
RUN mvn dependency:go-offline

# Copy the source code and build the application
COPY src ./src
RUN mvn package -DskipTests

# ============================================================
# STAGE 2: RUNTIME — Uses a slim JDK image to run the JAR
# ============================================================
FROM openjdk:17-jdk-slim
WORKDIR /app

# Copy the built JAR file from the build stage
COPY --from=build /app/target/*.jar app.jar

# Expose the port your Spring Boot application runs on
EXPOSE 8080

# Add a health check (optional)
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]
Line-by-Line Explanation
Stage 1: Build
Line	What It Does
FROM maven:3.9.4-eclipse-temurin-17 AS build	Uses the official Maven image (which includes Java 17 via Eclipse Temurin JDK) as the build environment. The AS build names this stage so it can be referenced later. This image is ~800 MB but won't appear in the final image.
WORKDIR /app	Sets /app as the working directory inside the container. All subsequent commands run from here.
COPY pom.xml ./	Copies ONLY pom.xml first (not source code). This is a build cache optimization — dependencies change rarely, so this layer stays cached across builds.
RUN mvn dependency:go-offline	Downloads all Maven dependencies. Since this layer depends only on pom.xml, it's cached as long as dependencies don't change. Saves minutes on subsequent builds.
COPY src ./src	Copies the Java source code. This layer changes frequently (every code change), so it's placed AFTER the dependency layer.
RUN mvn package -DskipTests	Compiles the code, runs the build, and creates the JAR file in target/. Tests are skipped (-DskipTests) because they should run in CI before the Docker build, not during it.
Stage 2: Runtime
Line	What It Does
FROM openjdk:17-jdk-slim	Starts a NEW stage with a lightweight JDK image (~200 MB vs ~800 MB for the Maven image). Everything from Stage 1 is discarded except what we explicitly copy.
WORKDIR /app	Sets working directory in the runtime image.
COPY --from=build /app/target/*.jar app.jar	Copies ONLY the compiled JAR from Stage 1's /app/target/ into the runtime image as app.jar. The Maven installation, source code, and build artifacts are left behind.
EXPOSE 8080	Documents that the application listens on port 8080. This is metadata only — you still need -p 8080:8080 when running the container.
HEALTHCHECK ...	Tells Docker to check container health every 30 seconds by hitting the Spring Boot Actuator endpoint. Waits 10 seconds for initial startup. If 3 consecutive checks fail, the container is marked unhealthy.
ENTRYPOINT ["java", "-jar", "app.jar"]	The command that runs when the container starts. Uses exec form (JSON array) which runs Java directly as PID 1, ensuring it receives shutdown signals (SIGTERM) properly.
Build and Run Commands
# Build the image
docker build -t fleet-manager:latest .

# Run locally
docker run -d -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=local \
  --name fleet-manager \
  fleet-manager:latest

# Run with production profile
docker run -d -p 5000:5000 \
  -e SPRING_PROFILES_ACTIVE=prd \
  --name fleet-manager-prd \
  fleet-manager:latest
CI/CD Pipeline — buildspec.yaml
The project uses AWS CodeBuild for CI/CD. The buildspec.yaml defines the automated build pipeline.

File: buildspec.yaml

version: 0.2

phases:
  pre_build:
    commands:
      - mvn clean install                          # Build and test the application
      - echo Logging in to Amazon ECR...
      - aws --version
      - REPOSITORY_URI=053357668713.dkr.ecr.us-east-2.amazonaws.com/inventorybackend-repo
      - aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin $REPOSITORY_URI
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=build-$(echo $CODEBUILD_BUILD_ID | awk -F":" '{print $2}')
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
  post_build:
    commands:
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - DOCKER_CONTAINER_NAME=inventorybackend-repo
      - printf '[{"name":"%s","imageUri":"%s"}]' $DOCKER_CONTAINER_NAME $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
    - target/inventory-service.jar
Phase-by-Phase Explanation
graph TB
    subgraph PreBuild [Phase 1: pre_build]
        PB1["mvn clean install<br/>Compile + run tests"] --> PB2["Login to Amazon ECR<br/>(container registry)"]
        PB2 --> PB3["Generate IMAGE_TAG<br/>from build ID"]
    end

    subgraph Build [Phase 2: build]
        B1["docker build -t repo:latest .<br/>Build image using Dockerfile"] --> B2["docker tag repo:latest repo:IMAGE_TAG<br/>Tag with unique build ID"]
    end

    subgraph PostBuild [Phase 3: post_build]
        PO1["docker push repo:latest<br/>Push latest tag to ECR"] --> PO2["docker push repo:IMAGE_TAG<br/>Push versioned tag to ECR"]
        PO2 --> PO3["Generate imagedefinitions.json<br/>for ECS deployment"]
    end

    PreBuild --> Build --> PostBuild
Phase 1: pre_build
Step	What It Does
mvn clean install	Compiles the Java code, runs unit tests, and packages the JAR. If tests fail, the build stops here.
REPOSITORY_URI=053357668713.dkr.ecr...	Sets the AWS ECR (Elastic Container Registry) repository URL where Docker images will be stored.
aws ecr get-login-password ... | docker login	Authenticates Docker CLI with AWS ECR using IAM credentials. Required before pushing images.
IMAGE_TAG=build-$(...)	Creates a unique tag for this build using the CodeBuild build ID, ensuring every image is traceable to a specific build.
Phase 2: build
Step	What It Does
docker build -t $REPOSITORY_URI:latest .	Builds the Docker image using the Dockerfile, tags it as latest.
docker tag ... :$IMAGE_TAG	Creates a second tag with the unique build ID (e.g., build-abc1234). This ensures you can always roll back to a specific version.
Phase 3: post_build
Step	What It Does
docker push ... :latest	Pushes the latest tagged image to ECR.
docker push ... :$IMAGE_TAG	Pushes the version-specific tagged image to ECR.
printf ... > imagedefinitions.json	Creates a JSON file that tells AWS ECS (Elastic Container Service) which container image to deploy. ECS reads this file to update the running service.
Artifacts
The pipeline outputs two artifacts:

imagedefinitions.json — Consumed by AWS CodeDeploy/ECS to trigger a deployment
target/inventory-service.jar — The compiled JAR file (backup/archive)
Deployment Architecture
graph TB
    subgraph Developer [Developer]
        DEV["Git Push"]
    end

    subgraph AWSCodePipeline [AWS CodePipeline]
        SRC["Source Stage<br/>(GitHub / CodeCommit)"]
        CB["CodeBuild<br/>(buildspec.yaml)"]
        CD["Deploy Stage"]
    end

    subgraph AWSECR [AWS ECR]
        REG["inventorybackend-repo<br/>Docker Image Registry"]
    end

    subgraph AWSECS [AWS ECS / Elastic Beanstalk]
        SVC["Fleet Manager Service"]
        T1["Task 1<br/>(Container)"]
        T2["Task 2<br/>(Container)"]
        ALB["Application Load Balancer<br/>api.fleet-manager.in"]
    end

    subgraph AWSRDS [AWS RDS]
        DB["MySQL Database<br/>fleetmanager.cduuc8a4e2kv.us-east-2.rds.amazonaws.com"]
    end

    subgraph AWSSecretsManager [AWS Secrets Manager]
        SEC["DB credentials<br/>API keys"]
    end

    DEV -->|"git push"| SRC
    SRC --> CB
    CB -->|"docker push"| REG
    CB -->|"imagedefinitions.json"| CD
    CD -->|"Deploy new image"| SVC
    REG -->|"pull image"| SVC
    SVC --> T1 & T2
    ALB --> T1 & T2
    T1 & T2 --> DB
    T1 & T2 --> SEC
Components in the Deployment
AWS Service	Role in This Project
CodePipeline	Orchestrates the CI/CD pipeline (Source -> Build -> Deploy)
CodeBuild	Runs buildspec.yaml — compiles code, builds Docker image, pushes to ECR
ECR	Private Docker image registry storing fleet-manager images
ECS / Elastic Beanstalk	Runs the containerized application. Pulls images from ECR.
Application Load Balancer	Routes traffic to container instances. Exposes api.fleet-manager.in
RDS (MySQL)	Managed MySQL database at fleetmanager.cduuc8a4e2kv.us-east-2.rds.amazonaws.com
Secrets Manager	Stores DB credentials, loaded by SecretsManagerConfig.java at runtime
Profile-Based Configuration
The application uses Spring Boot profiles to switch between local development and production configurations. The active profile determines which application-*.properties file is loaded.

graph LR
    subgraph ProfileSelection [Profile Selection]
        MAIN["application.properties<br/>spring.profiles.active=prd"]
        MAIN -->|"local"| LOCAL["application-local.properties"]
        MAIN -->|"prd"| PRD["application-prd.properties"]
    end
Local Profile (application-local.properties)
Property	Value	Purpose
spring.datasource.url	jdbc:mysql://localhost:3306/fleet_manager	Connects to local MySQL
server.port	8080	Standard dev port
cors.allowed-origins	http://localhost:4200,...	Allows local Angular dev server
logging.level	DEBUG	Verbose logging for development
Production Profile (application-prd.properties)
Property	Value	Purpose
spring.datasource.url	Loaded from AWS Secrets Manager	Connects to RDS MySQL
server.port	5000	ECS/Beanstalk convention
cors.allowed-origins	https://api.fleet-manager.in,...	Only production domains
logging.level	INFO	Reduced logging for production
How It Works Inside the Container
When the container starts with java -jar app.jar, the profile is set via:

# Option 1: Environment variable (recommended for containers)
docker run -e SPRING_PROFILES_ACTIVE=prd fleet-manager:latest

# Option 2: JVM argument
docker run fleet-manager:latest --spring.profiles.active=prd

# Option 3: Default in application.properties (current setup)
# spring.profiles.active=prd (already set in the file)
In the current project, application.properties has spring.profiles.active=prd hardcoded, so the container always starts with the production profile unless overridden.
