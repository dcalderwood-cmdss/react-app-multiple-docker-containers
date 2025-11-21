# Node.js App Docker Guide

This README explains how to build, run, and develop this Node.js application using Docker on Windows (PowerShell 5.1). It also covers best practices, optimizations, and troubleshooting tips.

## 1. Project Overview
The app is a simple Node.js server (entrypoint: `server.js`) with static assets under `public/`. Docker lets you package code + dependencies into a portable container image.

## 2. Prerequisites
- Installed Docker Desktop for Windows (ensure it is running)
- PowerShell terminal
- (Optional) Docker Hub account for pushing images

Verify Docker works:
```powershell
docker info
```

## 3. Key Files
- `Dockerfile` – Defines how the image is built
- `package.json` – Declares dependencies
- `server.js` – Application entrypoint
- `public/` – Static assets (served by the app)

## 4. Dockerfile (Annotated)
The included `Dockerfile` uses the official `node` base image, installs dependencies, copies source, exposes port 80, and starts the server:
```Dockerfile
FROM node
WORKDIR /app
COPY package.json /app
RUN npm install
COPY . /app
EXPOSE 80
CMD ["node", "server.js"]
```
Recommended production improvements:
- Pin version: `FROM node:20-alpine`
- Use `npm ci` if `package-lock.json` is present
- Add a non-root user
- Add `.dockerignore` to reduce build context

## 5. Quick Start
Build and run the container locally:
```powershell
docker build -t nodejs-app .
docker run -d -p 8080:80 --name nodejs-app nodejs-app
```
Open: http://localhost:8080

## 6. Building the Image
```powershell
# Basic build
docker build -t nodejs-app .

# Build with version tag
docker build -t nodejs-app:1.0.0 .

# See image list
docker images | Select-Object -First 10
```

### Caching Tip
Because `package.json` is copied before the rest of the source, dependency install layers are cached unless dependencies change.

## 7. Running the Container
```powershell
# Run detached, map host port 8080 -> container port 80
docker run -d -p 8080:80 --name nodejs-app nodejs-app:1.0.0

# View logs
docker logs nodejs-app --tail 100

# Exec into container shell (for debugging)
docker exec -it nodejs-app powershell
# or for Linux shell base images
docker exec -it nodejs-app sh
```

### Stop / Remove
```powershell
docker stop nodejs-app
docker rm nodejs-app
```

## 8. Development Workflow (Bind Mount)
Instead of rebuilding on every change, mount your local source:
```powershell
# Requires code watching (e.g., nodemon). First install nodemon locally.
npm install --save-dev nodemon

# Run with bind mount and override CMD to use nodemon
docker run -it --rm -p 8080:80 `
  -v ${PWD}:/app `
  -w /app `
  --name nodejs-app-dev nodejs-app `
  npx nodemon server.js
```
Changes on the host automatically reflect in the container.

## 9. Environment Variables
Inject configuration without baking into the image:
```powershell
# Example: pass PORT to override default
$env:PORT = 3000

docker run -d -p 3000:3000 --name nodejs-app `
  -e PORT=3000 nodejs-app
```
Inside `server.js`, you can read `process.env.PORT`.

Use an env file (`.env`):
```powershell
# Run with env file
docker run -d -p 8080:80 --env-file .env --name nodejs-app nodejs-app
```

## 10. Volumes & Persistence
For runtime-generated data, mount a volume:
```powershell
# Named volume for uploads
docker volume create nodejs_data

docker run -d -p 8080:80 --name nodejs-app `
  -v nodejs_data:/app/data nodejs-app
```
List volumes:
```powershell
docker volume ls
```
Remove volume (data lost):
```powershell
docker volume rm nodejs_data
```

## 11. Tagging & Pushing to Registry
Login and push to Docker Hub (example username: `myuser`):
```powershell
docker login

docker tag nodejs-app:1.0.0 myuser/nodejs-app:1.0.0

docker push myuser/nodejs-app:1.0.0
```
Pull later:
```powershell
docker pull myuser/nodejs-app:1.0.0
```

## 12. Multi-Stage Build (Example)
Use multi-stage to create a small production image:
```Dockerfile
# Stage 1: build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

# Stage 2: runtime
FROM node:20-alpine
WORKDIR /app
COPY --from=build /app /app
EXPOSE 80
USER node
CMD ["node", "server.js"]
```
Benefits:
- Smaller final image
- Separates build tools from runtime
- Can add testing stage before runtime

## 13. Security & Best Practices
- Pin base image versions (predictable, secure updates)
- Run as non-root user (`USER node` or create custom user)
- Add a `.dockerignore` to exclude local artifacts (node_modules*, .git, logs)
- Use `npm ci` for reproducible installs
- Scan images: `docker scout quickview nodejs-app:1.0.0` (if Docker Scout available)
- Keep secrets out of images (use env vars / secret management)

### Sample `.dockerignore`
```
node_modules
npm-debug.log
Dockerfile*
.git
*.md
.env
```

## 14. Resource Management
List running containers:
```powershell
docker ps
```
List all containers:
```powershell
docker ps -a
```
Remove dangling images:
```powershell
docker image prune -f
```
Remove all stopped containers:
```powershell
docker container prune -f
```
Full cleanup (CAUTION):
```powershell
docker system prune -a
```

## 15. Troubleshooting
| Symptom | Possible Cause | Fix |
|---------|----------------|-----|
| Port already in use | Another service on host | Pick different host port (`-p 8081:80`) |
| Cannot connect | Container crashed | `docker logs <name>` for errors |
| Dependency not found | `npm install` failed | Rebuild, check network, lockfile |
| Slow rebuilds | Large context | Add `.dockerignore`, multi-stage, reduce COPY |
| Env var ignored | Not read in code | Confirm `process.env.VAR` usage |

Check container status:
```powershell
docker inspect nodejs-app | Select-String Status
```

## 16. Useful PowerShell Aliases (Optional)
Create quick functions in profile:
```powershell
function di { docker images }
function dps { docker ps }
function drm($name) { docker rm -f $name }
```

## 17. Cheat Sheet
```powershell
# Build image
docker build -t nodejs-app .
# Run mapping ports
docker run -d -p 8080:80 --name nodejs-app nodejs-app
# Tail logs
docker logs -f nodejs-app
# Exec into container
docker exec -it nodejs-app sh
# Stop & remove
docker stop nodejs-app; docker rm nodejs-app
# List images
docker images
# Remove image
docker rmi nodejs-app:latest
# Prune unused
docker system prune -f
```

## 18. Next Steps
- Convert to multi-stage Dockerfile
- Add non-root user
- Automate builds with GitHub Actions
- Publish to registry

---

## 19. Architecture Diagram: Node ↔ Docker Network ↔ MongoDB

                   ┌─────────────────────────────┐
                   │        Docker Host          │
                   │ (your Windows/macOS system) │
                   └───────────────┬─────────────┘
                                   │
                      (Created automatically by Docker Compose)
                                   │
                       ┌────────────────────────┐
                       │    Docker Network      │
                       │   "favorites-network"  │
                       └───────────┬────────────┘
                 internal DNS      │
        mongodb → 172.18.0.2       │       node → 172.18.0.3
                                   │
        ┌─────────────────────────────────────────────────────┐
        │                                                     │
        │   Containers inside the same Docker network         │
        │                                                     │
        │  ┌─────────────────────┐       ┌──────────────────┐ │
        │  │  Node.js Container  │       │ MongoDB Container│ │
        │  │  (favorites-node)   │ ←→    │   (mongodb)      │ │
        │  └─────────────────────┘       └──────────────────┘ │
        │   | mongoose.connect("mongodb://mongodb:27017/...") │
        └─────────────────────────────────────────────────────┘

        ✔ Both containers can reach each other by service name  
        ✔ No need for IP addresses  
        ✔ localhost inside the Node container = Node container, not host  

