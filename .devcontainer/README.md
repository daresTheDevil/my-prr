# PRRIP Monorepo Dev Container

This directory contains the configuration for the VS Code Development Container, enabling a consistent and fully-featured local development environment for the PRRIP monorepo.

## Overview

The dev container setup uses Docker Compose (`../docker-compose.yml`) to orchestrate the `core-app` (the Remix/Epic Stack application) along with its backing services:
- PostgreSQL (for database)
- Redis (for caching)
- RabbitMQ (for messaging)

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running.
- [VS Code](https://code.visualstudio.com/) installed.
- [VS Code Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) installed in VS Code.

## How to Use

1.  Clone the PRRIP monorepo.
2.  Open the cloned repository folder in VS Code.
3.  VS Code should automatically detect the `.devcontainer/devcontainer.json` file and prompt you to "Reopen in Container". Click it.
    - If it doesn't prompt, open the Command Palette (Ctrl+Shift+P or Cmd+Shift+P) and type/select "Dev Containers: Reopen in Container".
4.  VS Code will build the necessary Docker images (if not already built) and start the services defined in `docker-compose.yml`. This might take a few minutes on the first run.
5.  Once the container is built and started, VS Code will connect to the `core-app` service within the container. Your workspace will be `/workspace`.

## What Happens on Start

- The `docker-compose.yml` file starts the `core-app`, `postgres_db`, `redis_db`, and `rabbitmq_broker` services.
- The `core-app` service:
    - Mounts the entire monorepo root into `/workspace`.
    - Runs `pnpm install --frozen-lockfile` to ensure all dependencies are installed.
    - Runs `pnpm run dev` to start the development server for `core-app` (via Turborepo). The application will be available at `http://localhost:3000`.
- The `postCreateCommand` in `devcontainer.json` runs `pnpm --filter core-app-2ece exec prisma migrate dev --name init_dev_db` and `pnpm --filter core-app-2ece exec prisma generate` after the container is created. This sets up the initial database schema in the PostgreSQL container and generates the Prisma client.

## Accessing Services

- **Core Application (Remix):** `http://localhost:3000`
- **PostgreSQL:** Connect via `localhost:5432` (User: `prripuser`, Pass: `prrippassword`, DB: `prripdb`)
- **Redis:** Connect via `localhost:6379`
- **RabbitMQ Management UI:** `http://localhost:15672` (User: `guest`, Pass: `guest`)

## Environment Variables

- The `core-app` service in `docker-compose.yml` is configured with `DATABASE_URL`, `REDIS_URL`, and `RABBITMQ_URL` pointing to the respective services.
- For other application-specific environment variables (e.g., `SESSION_SECRET`, API keys), you should:
    1. Create a `.env` file in the `packages/core-app/` directory by copying from `.env.example`.
    2. Populate the necessary values in this `.env` file. This file is gitignored by default.
    The Remix application (Epic Stack) will load these variables.

## Troubleshooting

- **Slow performance on macOS/Windows:** Ensure Docker Desktop has adequate resources allocated. The `:cached` option on volume mounts helps.
- **Port conflicts:** If any of the default ports (3000, 5432, 6379, 15672) are in use on your host machine, you can change the host-side port mapping in `docker-compose.yml` (e.g., change `"3000:3000"` to `"3001:3000"`).
- **`pnpm install` issues:** If you encounter issues after pulling changes, try rebuilding the dev container: Command Palette -> "Dev Containers: Rebuild Container".

This setup aims to provide a robust and consistent development experience, closely mirroring the planned production environment's use of containerized services.