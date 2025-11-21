# Docker Images Overview

Docker containers run your application, but Docker images define what a container *is made from*.  
An image is the blueprint, while a container is the running instance of that blueprint.

This guide explains how Docker images work in the context of your multi-service project (backend, database, Docker network), and how to build, tag, and manage images efficiently.

---

# What Is a Docker Image?

A Docker image is a **read-only filesystem snapshot** that contains:

- Your application code
- Installed dependencies
- The Node.js runtime (or MongoDB binary, Python interpreter, etc.)
- Any OS packages your app requires
- Metadata such as environment variables and default commands

When you start a container, Docker adds a thin **read/write layer** on top of the image. This keeps images immutable and reproducible.

---

# How Images Fit Into This Project

Your project uses multiple images:

| Service | Image Source | Purpose |
|--------|--------------|---------|
| **Backend (Node/Express)** | Built locally from your `backend/Dockerfile` | Runs the API logic |
| **MongoDB database** | Official `mongo:6` image | Provides a database server |
| **Frontend** | Runs locally (not containerized) | No image needed during development |

Only the backend and database need images.  
Each image fully defines what's inside its containers.

---

# Backend Image Architecture

Your backend image contains:

- The Node.js runtime (from a `node:<version>` base image)
- Your backend source code
- Installed dependencies (`npm install`)
- A default command to start your API (`node app.js`)
- Logging and filesystem structure

Example Dockerfile structure:

```dockerfile
FROM node:20

WORKDIR /app

COPY package.json .
RUN npm install

COPY . .

EXPOSE 80
CMD ["node", "app.js"]
