# Containers and Runtime Overview

This project intentionally runs only the backend API and the database in Docker containers. The React frontend runs in your browser (via the local dev server), not inside a container. This setup keeps frontend development fast with hot reloads while still giving you reproducible infrastructure for the API and DB.

## Architecture

- Backend (Node/Express): Runs inside a Docker container and listens on container port `80`. It exposes REST endpoints like `/goals`.
- Database (MongoDB): Runs inside its own Docker container. The backend connects to it via the hostname `mongodb` on port `27017` inside a shared Docker network.
- Frontend (React): Runs locally in your browser using the React dev server. The frontend calls the API on your host machine at `http://localhost` (not the Docker network), as seen in `frontend/src/App.js` where requests go to `http://localhost/goals`.

Why `localhost`? Because frontend code executes in the user's browser (your host), so all network requests are made from your host network context, not from inside a Docker container.

## Prerequisites

- Docker Desktop installed and running
- Node.js (LTS recommended) and npm for developing/running the frontend locally

## Start the Database and Backend in Containers

1) Create a dedicated Docker network so containers can resolve each other by name:

```powershell
docker network create goals-net
```

2) Start MongoDB (database container):

```powershell
# Persist DB data in a named volume; expose port only if you want host access
docker run -d --name mongodb --network goals-net -v mongodb_data:/data/db mongo:6

# Optional: expose to host as well
# docker run -d --name mongodb --network goals-net -p 27017:27017 -v mongodb_data:/data/db mongo:6
```

3) Build and run the backend (Node/Express) container:

```powershell
cd backend

docker build -t goals-backend .

# Bind container port 80 -> host port 80 so the frontend can reach http://localhost/goals
# If port 80 is in use, choose a different host port (e.g., 8080:80)
docker run -d --name goals-backend --network goals-net -p 80:80 goals-backend
```

Notes:
- The backend connects to MongoDB using the connection string `mongodb://mongodb:27017/course-goals` (see `backend/app.js`). The hostname `mongodb` works because both containers join `goals-net`.
- If you mapped the backend to a different host port (e.g., `-p 8080:80`), update the frontend API base URL accordingly (see next section).

## Run the Frontend in the Browser (not containerized)

The frontend is designed to run locally with the React dev server for a better developer experience.

```powershell
cd ../frontend
npm install
npm start
```

- The app will open in your browser (usually at `http://localhost:3000`).
- API calls go to `http://localhost/goals` as configured in `frontend/src/App.js`.

If you changed the backend host port mapping (e.g., `-p 8080:80`), adjust the fetch URLs in `frontend/src/App.js` to include the port:

```javascript
// Example if backend is on host port 8080
fetch('http://localhost:8080/goals')
```

## Troubleshooting

- CORS: The backend already sets permissive CORS headers. If you modify headers or routes, ensure CORS remains enabled for the dev server origin.
- Port conflicts: If host port 80 is in use, map another host port (e.g., `-p 8080:80`) and update the frontend URLs.
- MongoDB name resolution: Ensure both `mongodb` and `goals-backend` containers are on the same user-defined Docker network (e.g., `goals-net`).
- Logs: Backend access logs are written inside the container at `/app/logs/access.log` (mapped by the image). Use `docker logs goals-backend` for container stdout.

## Production Note

For production, you could containerize the built frontend (e.g., serve the React build with NGINX) or host it on a CDN. The key idea remains: the browser will call the API via a host URL, while the API and DB typically run as containers.
