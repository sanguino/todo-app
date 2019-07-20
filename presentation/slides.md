<!-- $theme: default -->
<!-- class: invert -->
<!-- $size: 16:9 1600px 1000px -->


# <!-- fit --> From Docker to Kubernetes 101
## <!-- fit --> Docker compose, multistage, healthchecks and todo-app included
https://github.com/sanguino/todo-app

---

## Contents

1. Another todo app :dog:
2. Docker multi stage
3. Healthchecks
4. Todo app running in docker-compose
5. Kubernetes
7. Minikube
9. Liveness and readyness
10. Todo app running in kubernetes

---

<!-- paginate: true -->
## Another todo app


![x% center](assets/todo.gif)

---


### Docker multi stage

- Multi-stage builds allows you to create multiple intermediate images from the same Dockerfile.

- You can use multiple FROM statements in your Dockerfile. Each FROM instruction begins a new stage of the build. 

- You can selectively copy artifacts from one stage to another, leaving behind everything you don't want in the final image.


---

#### Docker multi stage
###### old way

``` dockerfile
FROM node-chrome-sonnar-nginx:latest
WORKDIR /usr/src/app
COPY ./ .
RUN ["npm", "ci"]
RUN ["npm", "run", "lint"]
RUN ["sonar-scanner -Dsonar.projectBaseDir=/usr/src/app"]
RUN echo 'deb http://dl.google.com/linux/chrome/deb/ stable main' > /etc/apt/sources.list.d/chrome.list
RUN ["npm", "run", "test"]
RUN ["npm", "run", "build"]
WORKDIR /usr/src/app/dist
CMD ["nginx"]
```
> resutl is a node + chrome + sonnar + nginx 500mb image

---

#### Docker multi stage

###### new way
``` dockerfile
# Copies in our code and runs NPM Install and Lints Code
FROM node-chrome AS base
WORKDIR /usr/src/app
COPY ./ .
RUN ["npm", "ci"]
RUN ["npm", "run", "lint"]
RUN ["npm", "run", "test"]

# Gets Sonarqube Scanner from Dockerhub and runs it
FROM newtmitch/sonar-scanner:latest AS sonarqube
COPY --from=base /usr/src/app/src /usr/src
RUN ["sonar-scanner -Dsonar.projectBaseDir=/usr/src"]

# Runs Build
FROM base AS build
WORKDIR /usr/src/app
COPY --from=base /usr/src/app/ .
RUN ["npm", "run", "build"]

# Starts and Serves Web Page
FROM nginx:stable
COPY --from=build /usr/src/app/dist /usr/share/nginx/html
```
> resutl is a standard nginx 100mb image

---

#### Docker multi stage

``` dockerfile
FROM node:latest AS base
```

``` dockerfile
COPY --from=base /usr/src/app/ .
```

###### PROS
- From container about 500mb to only 100mb
- In production we will have ony a ngix standar image and `dist` files.

###### CONS
- Build time of each stage is the same, but starting multiple containers and copying files between them increases the build time.

---

#### Docker multi stage

#### Build of todo-front pipeline

![x% center](assets/docker-build.gif)


---

#### Docker multi stage

#### Target
- This is great for production use, but during development I dont want to have to build production image everytime I do a change. 

``` bash
$ docker build . --target=unit-test
```
``` yaml
...
build: 
  context: .
  target: build-env
...
```
- This will stop after execute unit-test. For example in a PR / MR you could execute it until unit test, and all the stages after merging it with master.

---

#### Docker multi stage

### look at docker files in this repos:

https://github.com/sanguino/todo-front
https://github.com/sanguino/todo-api
https://github.com/sanguino/todo-health

---

#### Docker healthcheck

``` yaml
version: '2.3'
services:
  todo-mongo:
    image: todo-mongo:latest
    volumes:
      - "mongodata:/data/db"
  todo-api:
    image: todo-api:latest
    depends_on: 
      - todo-mongo
volumes:
   mongodata:

```
> the problem with depends_on is that the container could be running, but the service not. So when mongo container is running, tha api and auth containers runs and fails conecting to mongo because the service is not running.

---

#### Docker healthcheck
- Example of how to implement healthcheck using compose
``` yaml
version: '2.3'
services:
  todo-mongo:
    image: todo-mongo:latest
    volumes:
      - "mongodata:/data/db"
    healthcheck:
      test: ["CMD", "/usr/local/bin/mongo-healthcheck"]
  todo-api:
    image: todo-api:latest
    depends_on: 
      todo-mongo:
        condition: service_healthy
```
- Example of how to implement healthcheck in a docker file
``` dockerfile
HEALTHCHECK [OPTIONS] CMD command
```
---

#### Docker healthcheck

- HEALTHCHECK let you implement a command that expose if the container is healthy or not. 
- The command should exit with 1 when non healthy and 0 when healthy
- Using condition `service_healthy` on `dependes_on` will make api container waits until mongo is up to starts.
- But docker swarm, and docker compose v3.x, does not respect depends_on, so you need to stay in version upper than 2.1 and less than 3.

---

#### Docker healthcheck


* Anyway you use it in compose or dockerfile, you could set some options:

``` bash
interval (default: 30s) first wait and interval between executons
timeout (default: 30s) timeout of each single run
start-period (default: 0s) initial wait to start checking
retries (default: 3) consecutive failures to be considered unhealthy.
```

* You could disable a inherit healthcheck too
``` dockerfile
HEALTHCHECK NONE
```
``` yaml
healthcheck:
  disable: true
```

---

#### Docker healthcheck

``` yaml
version: '2.3'
services:
  ...
  todo-mongo:
    image: todo-mongo:latest
    volumes:
      - "mongodata:/data/db"
    healthcheck:
      test: ["CMD", "/usr/local/bin/mongo-healthcheck"]
      interval: 5s
      timeout: 1s
      retries: 5
  todo-api:
    image: todo-api:latest
    depends_on: 
      todo-mongo:
        condition: service_healthy
 ...

```

---

#### Todo app in docker compose

![x% center](assets/demogods.jpg)

---


#### Todo app in docker compose if demo gods doesn't listen

![x% center](assets/docker-compose.gif)

---

#### Kubernetes

> Kubernetes is a portable, extensible, open-source platform for managing containerized workloads and services, that facilitates both declarative configuration and automation. It has a large, rapidly growing ecosystem. Kubernetes services, support, and tools are widely available.

- Kubernetes is a system for managing containerized applications across a cluster of nodes

- You could use k8s in a imperative way, like docker, but the real power of k8s is that you can use it in a declarative way. 

- Describe your desired state, and k8s will try to keep it.

---

#### Kubernetes

### Kubernetes is a system for managing containerized applications across a cluster of nodes

- Each node runs kubernetes it self and are where your containerized apps run

- Master node is the controlling node and operate as the main management contact point for users. 

- You run your containerized apps in nodes and you control them through the master.
---

### Kubernetes

![x% center](assets/cluster.jpg)

---

### Kubernetes

![x% center](assets/node.jpg)



---

#### Kubernetes Work Units

- **Pod** is the most basic unit in Kubernetes. Could consist of any number (1-n) of containers that share resources like storage, and has a unique network IP. Its like your local machine running N containers with volumes. In general best option is 1 pod 1 container.

![x% center](assets/pods.jpg)

---

#### Kubernetes Work Units

- **DNS** Every Service defined in the cluster (including the DNS server itself) is assigned a DNS name.

- **Service** is the way you present a group of pods to other pods (services). It acts as a basic load balancer between pods. Could be exposed outside k8s with nodeport or with an ingress.

---

#### Kubernetes Work Units


- **Deployment** is your desired state of pods. It's a declarative syntax to create/update pods.

- **Label** is an arbitrary tag to mark work units. Is the way to config witch service will be able to forward traffic to those pods.

---

#### Kubernetes Work Units

![x% center](assets/service1.jpg)

---

#### Kubernetes Work Units

- **Ingress** recomended way to manage external access to the services. Provides load balancing, SSL termination. It's a ngix that expose a 80/443 port outside kubernetes.

``` yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: todo-gateway
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: todo-front
          servicePort: 80
      - path: /api/task
        backend:
          serviceName: todo-api
          servicePort: 3000
      - path: /auth
        backend:
          serviceName: todo-auth
          servicePort: 3000
```
---

#### Kubernetes Work Units

![x% center](assets/ingress.jpg)

---

#### Kubernetes Work Units

- **Nodeport** Exposes the Service on each Node’s IP at a static port (the NodePort).

---

#### Kubernetes Work Units

![x% center](assets/nodeport.jpg)

---

#### Minikube

- Minikube is a cli tool that runs a single-node Kubernetes cluster in a virtual machine on your personal computer.

``` bash
$ minikube start
$ kubectl config use-context minikube
$ eval $(minikube docker-env)
$ bashminikube dashboard
```

---

#### Kubernetes declarative yaml

``` yaml
apiVersion: v1
kind: Service/Deployment/PersistentVolumeClaim/Ingress
metadata:
  name: appname
spec:
  selector:
    app: appname
  ports:
    - port: 3000
      targetPort: 3000
```
> `metadata` and `selector`  is what kubernetes uses to link a service, ingress, volumes and deployment between them.

---

#### Kubernetes declarative yaml

``` bash
$ kubectl apply -f fileordirectory
```
> use `apply` instead of `create`, execution of apply multiple time makes kubernetes updates the config. 
---
#### Services

``` yaml
kind: Service
apiVersion: v1
metadata:
  name: todo-api
spec:
  selector:
    app: todo-api
  ports:
    - port: 3000
      targetPort: 3000
```

---
#### Deployments

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-api-deployment
  labels:
    app: todo-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: todo-api
  template:
    metadata:
      labels:
        app: todo-api
    spec:
      containers:
      - name: todo-api
        image: todo-api:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 3000
```
---

#### Deployments, liveness and readiness probes

- **livenessProbe** Many applications running for long periods of time eventually transition to broken states, and cannot recover except by being restarted. Kubernetes provides liveness probes to detect and remedy such situations.

- **readinessProbe** Sometimes, applications are temporarily unable to serve traffic. Kubernetes provides readiness probes to detect and mitigate these situations. A pod with containers reporting that they are not ready does not receive traffic through Kubernetes Services.

> probes could be a command, http or tcp.

---

#### Deployments, liveness and readiness probes


``` yaml
        livenessProbe:
          httpGet:
            path: /api/health/task
            port: 3000
          initialDelaySeconds: 30
          timeoutSeconds: 10
          periodSeconds: 30
          failureThreshold: 3
          successThreshold: 1
```
``` yaml
        readinessProbe:
          httpGet:
            path: /api/health/task
            port: 3000
          initialDelaySeconds: 20
          timeoutSeconds: 10
          periodSeconds: 30
          failureThreshold: 3
          successThreshold: 1
```

--- 

#### Secrets

``` yaml
apiVersion: v1
kind: Secret
metadata:
  name: jwtconfig
type: Opaque
data:
  SUPER_SECRET: cGFzcw==  ("pass" base64 encoded)
  TOKEN_EXPIRATION: ODY0MDAwMDA= ("86400000" base64 encoded)
```

--- 

#### Deployments, environment with secrets

``` yaml
        env:
          - name: MONGO_HOST
            value: "todo-mongo"
          - name: MONGO_PORT
            value: "27017"
          - name: MONGO_DB
            value: "tasksDB"
          - name: SUPER_SECRET
            valueFrom:
              secretKeyRef:
                name: jwtconfig
                key: SUPER_SECRET
```

--- 


# The End

Thank you very much

