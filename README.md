# MongoDB + Mongo Express

A minimal, developer-friendly setup for running MongoDB with the Mongo Express web UI. Use it to explore collections, run queries, and manage your MongoDB instance locally.

## What you get
- MongoDB database (default port 27017)
- Mongo Express admin UI (default port 8081)
- Easy start/stop with Docker Compose
- Persistent data via a named Docker volume

## Quick start

### Prerequisites
- Docker (20+ recommended)
- Docker Compose v2 (`docker compose` command)

### 1) Create `docker-compose.yml`
Copy the following into a new file named `docker-compose.yml` in the project root:

```yaml
services:
  mongo:
    image: mongo:7
    container_name: mongo
    restart: unless-stopped
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: admin
    volumes:
      - mongo_data:/data/db

  mongo-express:
    image: mongo-express:latest
    container_name: mongo-express
    restart: unless-stopped
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_SERVER: mongo
      ME_CONFIG_MONGODB_ADMINUSERNAME: admin
      ME_CONFIG_MONGODB_ADMINPASSWORD: admin
      ME_CONFIG_BASICAUTH: "true"
      ME_CONFIG_BASICAUTH_USERNAME: admin
      ME_CONFIG_BASICAUTH_PASSWORD: admin
    depends_on:
      - mongo

volumes:
  mongo_data:
```

Optionally, place credentials in a `.env` file and reference them in the compose file instead of hardcoding.

### 2) Start the stack
```bash
docker compose up -d
```

### 3) Open the UI
- Mongo Express: `http://localhost:8081`
- MongoDB connection string (example):
  ```
  mongodb://admin:admin@localhost:27017/?authSource=admin
  ```

## Common commands
```bash
# Start in the background
docker compose up -d

# View logs
docker compose logs -f mongo-express

# Stop containers
docker compose down

# Stop and remove data volume (DANGER: deletes data)
docker compose down -v
```

## Configuration
- Change default usernames/passwords via environment variables in `docker-compose.yml` or a `.env` file.
- Change exposed ports by editing the `ports` mappings.
- The MongoDB data volume is named `mongo_data` by default.

## Troubleshooting
- Port already in use: adjust `27017:27017` or `8081:8081` mappings.
- Authentication errors: ensure Mongo Express credentials match MongoDB admin credentials.
- Fresh start: run `docker compose down -v` to remove containers and volumes, then `docker compose up -d`.

## Security note
This setup is for local development. Do not use the default credentials in production and do not expose Mongo Express publicly without proper authentication and network controls.
