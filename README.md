# NexTalk Dockerization

> Dockerizing a real-time MERN chat application with separate Frontend, Backend, and MongoDB containers.

**Original project:** [hemjaygfx/NexTalk](https://github.com/hemjaygfx/NexTalk)

**Original repository owner:** `hemjaygfx`

> **Note:** GitHub currently identifies `hemjaygfx/NexTalk` as the repository URL provided for this project. The public repository page does not show a separate "Forked from" parent, so `hemjaygfx` is the verified owner of the referenced repository.

---

## 1. Project Overview

![](https://github.com/Workwithaditya01/NexTalk-Docker/blob/ce302d71ed00de90f7fa4e4a4a93e7120d0661f9/Images/Applicaition%20images/Final%20Output.png)

NexTalk is a real-time MERN-stack chat application.

The original application provides:

- User registration and authentication
- JWT-based sessions
- Real-time messaging using Socket.IO
- Online presence
- Image sharing
- Chat history
- MongoDB persistence
- Cloudinary integration
- Resend email integration
- Arcjet protection

This project focuses on **containerizing the application** and understanding how the individual services communicate.


### Application architecture

![Systemdesign](https://github.com/Workwithaditya01/NexTalk-Docker/blob/ce302d71ed00de90f7fa4e4a4a93e7120d0661f9/system%20design.png)

---

## 2. Repository Structure

```text
NexTalk-Docker/
│
├── backend/
│   ├── src/
│   ├── package.json
│   ├── package-lock.json
│   └── Dockerfile
│
├── frontend/
│   ├── src/
│   ├── package.json
│   ├── package-lock.json
│   ├── Dockerfile
│   └── nginx.conf
│
├── .env
├── docker-compose.yml
└── README.md
```

---

## 3. Backend Dockerfile

The backend uses Node.js 24.

```dockerfile
FROM node:24

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

EXPOSE 3000

CMD ["npm","start"]
```

### What this Dockerfile does

**`FROM node:24`**
Uses Node.js 24 as the base image.

**`WORKDIR /app`**
Creates `/app` as the working directory inside the container.

**`COPY package*.json ./`**
Copies the package files first. This allows Docker to reuse the dependency-installation layer when application source code changes.

**`RUN npm ci`**
Installs the exact dependencies defined by `package-lock.json`.

**`COPY . .`**
Copies the backend source code into the image.

**`EXPOSE 3000`**
Documents that the Node.js backend listens on port `3000`.

**`CMD ["npm","start"]`**
Starts the backend using the `start` script from `package.json`.

---

## 4. Running the Backend Container Individually

Before using Docker Compose, the backend can be tested as an individual container.

Because the backend needs MongoDB, create a Docker network first:

```bash
docker network create nextalk-network
```

Run MongoDB:

```bash
docker run -d --name nextalk-mongodb --network nextalk-network mongo:7
```

Build the backend image:

```bash
docker build -t nextalk-backend ./backend
```

Run the backend:

```bash
docker run -d --name nextalk-backend --network nextalk-network -p 3001:3000 -e PORT=3000 -e MONGO_URI=mongodb://nextalk-mongodb:27017/nextalk nextalk-backend
```

Check the container:

```bash
docker ps
```

Check backend logs:

```bash
docker logs nextalk-backend
```

The important Docker networking concept here is:

```text
Backend container
       |
       | MongoDB connection
       v
nextalk-mongodb:27017
```

Inside a Docker network, the container name can be used as the hostname.

Do **not** use:

```text
localhost:27017
```

from inside the backend container because `localhost` refers to the backend container itself.

---

## 5. Frontend Dockerfile

The frontend uses a **multi-stage Docker build**:

```dockerfile
FROM node:24 AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build


FROM nginx:alpine

WORKDIR /app

RUN rm -rf /usr/share/nginx/html/*

COPY --from=builder /app/dist /usr/share/nginx/html

COPY nginx.conf /etc/nginx/conf.d/default.conf

CMD ["nginx", "-g", "daemon off;"]
```

### Why use two stages?

The first stage:

```text
Node.js
   |
   +-- install dependencies
   |
   +-- npm run build
   |
   v
/dist
```

creates the production frontend files.

The second stage:

```text
Nginx Alpine
      |
      +-- receives /dist files
      |
      +-- serves them on port 80
```

means the final image does not need Node.js or the frontend development dependencies.

This produces a cleaner production-style frontend container.

---

## 6. Running the Frontend Container Individually

Build the frontend image:

```bash
docker build -t nextalk-frontend ./frontend
```

Because the frontend's Nginx configuration proxies requests to the backend service name, run the frontend on the same Docker network:

```bash
docker run -d --name nextalk-frontend --network nextalk-network 4173:80 nextalk-frontend
```

Check:

```bash
docker ps
```

Open:

```text
http://localhost:4173
```

At this stage the React application should be served by Nginx.

---

## 7. Why Do We Need `nginx.conf`?

A React/Vite application is normally developed using a development server.

After:

```bash
npm run build
```

the frontend becomes a collection of static files such as:

```text
index.html
assets/
    *.js
    *.css
    images/
```

Nginx is excellent at serving these static files.

However, NexTalk has another requirement:

- REST API requests
- Socket.IO real-time connections

The browser needs to communicate with the Node.js backend.

Instead of exposing the backend directly to the browser for every request, Nginx can act as a **reverse proxy**.

The flow becomes:

```text
Browser
   |
   | /api/*
   | /socket.io/*
   v
 Nginx
   |
   | Docker network
   v
Backend:3000
```

This is why `nginx.conf` contains separate locations for:

```text
/
/api/
/socket.io/
```

---

## 8. Nginx Configuration

Current configuration:

```nginx
server {
    listen 80;

    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://backend:3000/api/;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /socket.io/ {
        proxy_pass http://backend:3000/socket.io/;

        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### `/` — React application

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

This is important for React client-side routing.

For example:

```text
/login
/register
/chat
```

may not exist as physical files on disk.

Nginx therefore falls back to:

```text
/index.html
```

and React Router handles the route.

### `/api/` — Backend REST API

```nginx
location /api/ {
    proxy_pass http://backend:3000/api/;
}
```

A browser request such as:

```text
http://localhost:4173/api/auth/login
```

is forwarded internally to:

```text
http://backend:3000/api/auth/login
```

The important point is that `backend` is the **Docker Compose service name**.

Docker's internal DNS resolves `backend` to the backend container.

### `/socket.io/` — Real-Time Chat

NexTalk uses Socket.IO for real-time messaging.

Socket.IO needs WebSocket upgrade headers:

```nginx
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

Without the appropriate WebSocket proxy configuration, real-time communication can fail when Nginx sits between the browser and Node.js.

The flow is:

```text
User 1 Browser
       |
       | WebSocket
       v
     Nginx
       |
       | WebSocket
       v
Backend / Socket.IO
       |
       v
     MongoDB
```

This allows multiple users to communicate in real time.

---

## 9. Important Docker Networking Concept

The hostname `backend` works **inside the Docker network**.

It does not mean the user's browser can resolve:

```text
http://backend:3000
```

The browser runs outside Docker.

That is one of the reasons the Nginx reverse proxy is useful.

The browser communicates with:

```text
http://localhost:4173
```

and Nginx internally communicates with:

```text
http://backend:3000
```

So the browser does not need to know the backend container hostname.

---

## 10. Docker Compose

After understanding the individual containers, Docker Compose can be used to manage the complete application.

Current Compose configuration:

```yaml
services:
  mongodb:
    image: mongo:7
    volumes:
      - mongo_data:/data/db
      - mongo_config:/data/configdb

  backend:
    build: ./backend
    ports:
      - "${BACKEND_HOST_PORT}:${BACKEND_PORT}"
    depends_on:
      - mongodb

    environment:
      MONGO_URI: mongodb://mongodb:27017/nextalk
      JWT_SECRET: ${JWT_SECRET}

  frontend:
    build: ./frontend
    depends_on:
      - backend
    ports:
      - "${FRONTEND_HOST_PORT}:80"

volumes:
  mongo_data:
  mongo_config:
```

---

## 11. What Docker Compose Provides

Compose creates a shared Docker network for the services.

Conceptually:

```text
                  Docker Compose Network
                         |
       +-----------------+-----------------+
       |                 |                 |
       v                 v                 v
   frontend           backend           mongodb
     :80               :3000             :27017
       |                 |
       |                 |
       +-------> backend |
                         |
                         +-------> mongodb
```

The services can communicate using their Compose service names:

```text
frontend
backend
mongodb
```

The backend uses `mongodb:27017` instead of `localhost:27017`.

---

## 12. Environment Variables

Example root `.env`:

```env
MONGO_INITDB_ROOT_USERNAME=admin
MONGO_INITDB_ROOT_PASSWORD=secret

BACKEND_PORT=3000

FRONTEND_HOST_PORT=4173
BACKEND_HOST_PORT=3001
MONGODB_HOST_PORT=27017
```

If the backend requires JWT configuration, provide the required secret as well:

```env
JWT_SECRET=change_this_to_a_long_random_secret
```

Do not commit real secrets to GitHub.

Add `.env` to `.gitignore`:

```gitignore
.env
```

---

## 13. Start the Complete Stack

Build and start all services:

```bash
docker compose up --build
```

Run in detached mode:

```bash
docker compose up --build -d
```

Check running services:

```bash
docker compose ps
```

Check all logs:

```bash
docker compose logs
```

Check only the backend:

```bash
docker compose logs backend
```

Check MongoDB:

```bash
docker compose logs mongodb
```

Check frontend:

```bash
docker compose logs frontend
```

---

## 14. Access the Application

With the current port mapping:

```text
Frontend:
http://localhost:4173

Backend:
http://localhost:3001

MongoDB:
internal Docker network
```

The recommended browser entry point is:

```text
http://localhost:4545
```

Nginx handles `/api/*` and `/socket.io/*` and forwards those requests to the backend.

---

## 15. Docker Volumes

MongoDB uses named volumes:

```yaml
volumes:
  - mongo_data:/data/db
  - mongo_config:/data/configdb
```

The important volume is `mongo_data`, because MongoDB stores its database files there.

Without persistent storage, deleting the MongoDB container could remove the database data stored inside that container.

With the volume:

```text
MongoDB container
       |
       v
mongo_data volume
       |
       v
Database survives container recreation
```

List volumes:

```bash
docker volume ls
```

Inspect the volume:

```bash
docker volume inspect nextalk-docker_mongo_data
```

The exact Compose-generated volume name may differ depending on the project directory.

---

## 16. Useful Docker Commands

| Purpose | Command |
|---|---|
| List images | `docker images` |
| List containers | `docker ps` |
| Include stopped containers | `docker ps -a` |
| View logs | `docker logs <container_name>` |
| Follow logs | `docker logs -f <container_name>` |
| Stop Compose services | `docker compose stop` |
| Stop and remove containers | `docker compose down` |
| Stop containers and remove volumes | `docker compose down -v` |

> **Warning:** `docker compose down -v` removes the Compose volumes and therefore deletes the MongoDB data stored in those volumes.

---

## 17. Containerization Learning Flow

This project is intentionally approached in stages.

### Stage 1 — Backend Container

Learn:

- Dockerfile
- Node.js image
- Working directory
- Dependency installation
- Port exposure
- Environment variables
- Container logs

```text
Node.js application
        |
        v
Backend Docker image
        |
        v
Backend container
```

### Stage 2 — Frontend Container

Learn:

- Multi-stage Docker builds
- React production build
- Nginx
- Static file serving
- Smaller production-style image

```text
React source
     |
     | npm run build
     v
/dist
     |
     v
Nginx image
     |
     v
Frontend container
```

### Stage 3 — Nginx Reverse Proxy

Learn:

- Reverse proxy
- REST API forwarding
- WebSocket forwarding
- React SPA routing
- Internal Docker DNS

```text
Browser
   |
   v
Nginx
   |
   +---- /api/ ------> Backend
   |
   +---- /socket.io/ -> Backend
```

### Stage 4 — MongoDB Container

Learn:

- Database containers
- Docker volumes
- Internal networking
- Persistent data

### Stage 5 — Docker Compose

Learn:

- Multi-container orchestration
- Service discovery
- Dependencies
- Volumes
- Environment variables
- Port mapping

Final architecture:

```text
                       Browser
                          |
                          | :4173
                          v
                +-------------------+
                | Frontend + Nginx  |
                |       :80         |
                +---------+---------+
                          |
                  Docker network
                          |
                          v
                +-------------------+
                | Node + Express    |
                | Socket.IO         |
                |       :3000       |
                +---------+---------+
                          |
                          | mongodb:27017
                          v
                +-------------------+
                |     MongoDB       |
                |      :27017       |
                +---------+---------+
                          |
                          v
                    Docker Volume
```

---

## 18. Common Problems

### Backend says MongoDB connection failed

Check:

```bash
docker compose ps
```

Then:

```bash
docker compose logs mongodb
```

Remember that inside Docker the backend should use `mongodb:27017`, not `localhost:27017`.

### Resend API key error

NexTalk's original application expects a Resend API key for its email functionality.

If the backend initializes Resend without a key, it can fail during startup.

For learning/testing, make the Resend initialization optional or provide the required Resend credentials.

Do not put real API keys directly inside a Dockerfile.

### React route returns 404 after refreshing

Check that Nginx contains:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

This allows React Router routes to work after refreshing the browser.

### Real-time chat doesn't work

Check:

```bash
docker compose logs backend
```

and verify that the Nginx Socket.IO location exists:

```nginx
location /socket.io/ {
    proxy_pass http://backend:3000/socket.io/;
    proxy_http_version 1.1;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

Also verify that the frontend Socket.IO client is using the same-origin/proxied endpoint expected by this Nginx configuration.

---

## 19. Security Notes

For a learning environment:

- Never commit `.env`
- Never commit API keys
- Never expose MongoDB publicly unless there is a specific reason
- Use strong JWT secrets
- Use strong database credentials when MongoDB authentication is enabled
- Restrict EC2 Security Group ports
- Prefer HTTPS in production
- Use a reverse proxy rather than exposing internal services unnecessarily

---

## 20. Future EC2 Deployment

The same Compose architecture can be deployed to an Ubuntu EC2 instance.

High-level flow:

```text
                         Internet
                            |
                            v
                    AWS EC2 Instance
                            |
                    Docker Compose
                            |
          +-----------------+-----------------+
          |                 |                 |
          v                 v                 v
      Frontend           Backend           MongoDB
       Nginx             Node.js            Database
       :80                :3000              :27017
          |                 |
          +------ Docker Network ------------+
                            |
                            v
                       Mongo Volume
```

For an EC2 deployment, the AWS Security Group should expose only the ports that are actually required by the application. MongoDB should normally remain accessible only through the internal Docker network.

---

## 21. Technology Stack

| Layer | Technology |
|---|---|
| Frontend | React 19 + Vite |
| Styling | Tailwind CSS + daisyUI |
| State Management | Zustand |
| API Client | Axios |
| Real-time Communication | Socket.IO |
| Backend | Node.js + Express |
| Database | MongoDB |
| ODM | Mongoose |
| Authentication | JWT + bcryptjs |
| Web Server | Nginx |
| Containers | Docker |
| Orchestration | Docker Compose |
| Cloud Target | AWS EC2 |

---

## 22. What I Learned From This Project

This project demonstrates practical Docker concepts:

- Writing a backend Dockerfile
- Writing a frontend multi-stage Dockerfile
- Building React applications for production
- Serving React with Nginx
- Configuring Nginx as a reverse proxy
- Proxying REST APIs
- Proxying WebSocket/Socket.IO traffic
- Creating Docker networks
- Container-to-container communication
- Docker service discovery
- Environment variables
- MongoDB containers
- Persistent Docker volumes
- Docker Compose
- Multi-container application architecture
- Preparing a full-stack application for EC2 deployment

---

## Original Project

The application was built from the public repository:

**NexTalk — `hemjaygfx/NexTalk`**

Repository: https://github.com/hemjaygfx/NexTalk

This Dockerization work focuses on running the existing MERN application as separate containers rather than changing the application's core architecture.
