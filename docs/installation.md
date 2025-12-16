# 🛠️ Installation Guide

Get Booklore up and running in minutes with Docker Compose. This comprehensive guide walks you through every step of the installation process, from prerequisites to your first login.

:::tip[Quick Start]
If you're experienced with Docker, skip to [Step 3](#step-3-create-the-docker-composeyml-file) to grab the configuration and start deploying.
:::

---

## 🌟 What You'll Achieve

By following this guide, you will:

- **Deploy Booklore** using Docker containers
- **Set up a MariaDB database** for persistent storage
- **Configure storage paths** for books, data, and automatic imports
- **Access the web interface** and start building your library
- **Understand the architecture** and how components work together

---

## 📦 Prerequisites

Before you start, ensure you have the following installed on your system:

### Required Software

- **[Docker](https://docs.docker.com/get-docker/)** - Container runtime (v20.10 or later recommended)
- **[Docker Compose](https://docs.docker.com/compose/install/)** - Multi-container orchestration (v2.0 or later recommended)

### Optional Tools

- **git** - For cloning repositories and version control
- **A reverse proxy** (Nginx, Traefik, Caddy) - For HTTPS and custom domain support
- **Basic terminal knowledge** - Familiarity with command-line operations

### System Requirements

- **Minimum:** 2GB RAM, 10GB disk space
- **Recommended:** 4GB+ RAM, 50GB+ disk space (varies based on library size)
- **OS:** Linux, macOS, Windows (with WSL2 recommended)

---

## 🐳 Understanding the Docker Setup

### Architecture Overview

Booklore uses a two-container architecture:

1. **booklore** - The main application container
   - Runs the web interface and API
   - Handles book processing and metadata enrichment
   - Manages file operations and library organization

2. **mariadb** - The database container
   - Stores user accounts, library metadata, and settings
   - Provides persistent data storage
   - Uses LinuxServer.io's optimized MariaDB image

### Official Docker Images

You can pull Booklore from two registries:

- **Docker Hub:** [booklore/booklore-app](https://hub.docker.com/r/booklore/booklore-app)
- **GitHub Container Registry:** [ghcr.io/booklore-app/booklore](https://github.com/booklore-app/booklore/pkgs/container/booklore)

Both registries host identical images. Choose based on your preference or network accessibility.

---

## 🚀 Installation Steps

### Step 1: Create the Directory Structure

Booklore needs several directories to store data, books, and database files. Create them with proper permissions:

```bash
# Create the main Booklore directory structure
mkdir -p ~/booklore/config/mariadb
mkdir -p ~/booklore/data
mkdir -p ~/booklore/books
mkdir -p ~/booklore/bookdrop
```

#### 📁 Directory Breakdown

| Directory | Purpose | Important Notes |
|-----------|---------|-----------------|
| `config/mariadb` | Database configuration and data files | **Critical:** Never delete - contains all metadata |
| `data` | Booklore application data, cache, logs | Stores settings and temporary processing files |
| `books` | Your main library storage | Add books here or import via Bookdrop |
| `bookdrop` | Automatic import folder | Drop files here for hands-free importing |

:::warning[Backup Important Directories]
Always backup `config/mariadb` and `books` to prevent data loss.
:::

---

### Step 2: Install Docker & Docker Compose

If you haven't installed Docker yet, follow the official installation guides:

#### Installation by Platform

**Linux:**
```bash
# Ubuntu/Debian
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
# Log out and back in for group changes to take effect
```

**macOS:**
- Download and install [Docker Desktop for Mac](https://docs.docker.com/desktop/install/mac-install/)

**Windows:**
- Download and install [Docker Desktop for Windows](https://docs.docker.com/desktop/install/windows-install/)
- Enable WSL2 backend for better performance

#### Verify Installation

```bash
docker --version
docker compose version
```

You should see version numbers for both commands.

:::warning[Docker Daemon Required]
Ensure the Docker daemon is running before proceeding. On Linux, use:
```bash
sudo systemctl start docker
```
:::

---

### Step 3: Create the `docker-compose.yml` File

Navigate to your Booklore directory:

```bash
cd ~/booklore
```

#### Create Environment File

Create a `.env` file with your configuration:

```bash
nano .env
```

```ini
# BookLore Application Settings
TZ=Etc/UTC
BOOKLORE_PORT=6060

# Database Connection (BookLore)
DATABASE_URL=jdbc:mariadb://mariadb:3306/booklore
DB_USER=booklore
DB_PASSWORD=ChangeMe_BookLoreApp_2025!

# MariaDB Container Settings
DB_USER_ID=1000
DB_GROUP_ID=1000
MYSQL_ROOT_PASSWORD=ChangeMe_MariaDBRoot_2025!
MYSQL_DATABASE=booklore
```

:::warning[Security]
Change default passwords before deploying!
:::

#### Create Docker Compose File

Create `docker-compose.yml`:

```bash
nano docker-compose.yml
```

````yaml
services:
  booklore:
    image: booklore/booklore:latest
    container_name: booklore
    environment:
      - TZ=${TZ}
      - DATABASE_URL=${DATABASE_URL}
      - DATABASE_USERNAME=${DB_USER}
      - DATABASE_PASSWORD=${DB_PASSWORD}
      - BOOKLORE_PORT=${BOOKLORE_PORT}
    depends_on:
      mariadb:
        condition: service_healthy
    ports:
      - "${BOOKLORE_PORT}:${BOOKLORE_PORT}"
    volumes:
      - ./data:/app/data
      - ./books:/books
      - ./bookdrop:/bookdrop
    restart: unless-stopped

  mariadb:
    image: lscr.io/linuxserver/mariadb:11.4.5
    container_name: mariadb
    environment:
      - PUID=${DB_USER_ID}
      - PGID=${DB_GROUP_ID}
      - TZ=${TZ}
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${DB_USER}
      - MYSQL_PASSWORD=${DB_PASSWORD}
    volumes:
      - ./mariadb/config:/config
    restart: unless-stopped
    healthcheck:
      test: [ "CMD", "mariadb-admin", "ping", "-h", "localhost" ]
      interval: 5s
      timeout: 5s
      retries: 10
````

#### Key Configuration Notes

**User IDs:**
- `DB_USER_ID/GROUP_ID`: Typically `1000` or match your user IDs

**Passwords:**
- `DB_PASSWORD` must match in both `DATABASE_PASSWORD` and `MYSQL_PASSWORD`
- Use strong, unique passwords for production

**Timezone:**
- Set `TZ` to your timezone (e.g., `America/New_York`)
- List timezones: `timedatectl list-timezones`

**Volumes:**
- Use absolute paths for production: `/home/username/booklore/data:/app/data`
- Relative paths work for simple setups

**Image Registry:**
Alternatively, use GitHub Container Registry:
```yaml
image: ghcr.io/booklore-app/booklore:latest
```

:::tip[Version Pinning]
For production, pin to specific versions (e.g., `booklore/booklore:v1.2.3`). Check [releases](https://github.com/adityachandelgit/booklore/releases).
:::

---

### Step 4: Start the Containers

#### Initial Startup

From the directory containing your `docker-compose.yml`:

```bash
docker compose up -d
```

This command will:
1. Pull the required Docker images (first time only)
2. Create and start both containers
3. Initialize the database with proper schema
4. Run in detached mode (background)

#### Monitor Startup Progress

```bash
# View logs from both containers
docker compose logs -f

# View logs from specific container
docker compose logs -f booklore
docker compose logs -f mariadb
```

:::info[First Start]
Initial startup may take 1-2 minutes as the database initializes. Subsequent starts are much faster.
:::

#### Verify Containers are Running

```bash
docker compose ps
```

You should see both containers with "Up" status:

```
NAME       IMAGE                                  STATUS
booklore   booklore/booklore-app:latest          Up 2 minutes
mariadb    lscr.io/linuxserver/mariadb:11.4.5    Up 2 minutes (healthy)
```

:::success[Healthy Status]
Wait until MariaDB shows "(healthy)" before accessing Booklore.
:::

---

### Step 5: Access Booklore

#### Open the Web Interface

Once containers are running, open your web browser and navigate to:

```
http://localhost:6060
```

Or from another device on your network:

```
http://YOUR_SERVER_IP:6060
```

:::success[Installation Complete]
You're now ready to start building your digital library.
:::

---

## 🐛 Troubleshooting

### Container Won't Start

**Check logs for errors:**
```bash
docker compose logs booklore
docker compose logs mariadb
```

**Common issues:**
- Incorrect volume paths
- Port 6060 already in use
- Database password mismatch
- Insufficient permissions on mounted directories

### Database Connection Errors

**Verify MariaDB is healthy:**
```bash
docker compose ps mariadb
```

**Check database credentials:**
- Ensure `DATABASE_PASSWORD` matches `MYSQL_PASSWORD`
- Verify `DATABASE_USERNAME` matches `MYSQL_USER`

**Reset database (last resort):**
```bash
docker compose down
rm -rf ~/booklore/config/mariadb/*
docker compose up -d
```

:::danger[Database Reset Warning]
This deletes all library metadata. Books remain but need re-importing.
:::

### Permission Errors

**Fix directory permissions:**
```bash
# Find your user/group IDs
id -u  # PUID
id -g  # PGID

# Update docker-compose.yml with correct PUID/PGID

# Fix directory ownership
sudo chown -R $USER:$USER ~/booklore
```

### Port Already in Use

**Check what's using port 6060:**
```bash
# Linux/macOS
sudo lsof -i :6060

# Windows
netstat -ano | findstr :6060
```

**Use different port in docker-compose.yml:**
```yaml
ports:
  - "8080:6060"  # Access via http://localhost:8080
```

### Can't Access from Network

**Check firewall rules:**
```bash
# Linux (UFW)
sudo ufw allow 6060/tcp

# Linux (firewalld)
sudo firewall-cmd --permanent --add-port=6060/tcp
sudo firewall-cmd --reload
```

**Verify Docker network:**
```bash
docker network inspect booklore_default
```

---

## 📚 Next Steps

Now that Booklore is installed, explore these essential features to get started:

1. **[Setup Admin User](initial-setup)** - Create your administrator account and secure your instance
2. **[Setup First Library](library/setup-first-library)** - Configure your first library and organize your collection
3. **[Configure Bookdrop](bookdrop)** - Enable automatic imports by dropping files into a folder
4. **[Metadata Fetch Configuration](metadata/metadata-fetch-configuration)** - Set up metadata sources for enriching book information
5. **[Magic Shelf](magic-shelf)** - Automatically organize books with smart shelving rules

### Optional Advanced Setup

- **[Email Setup](email-setup)** - Configure email notifications and password resets
- **[Kobo Integration](integration/kobo)** - Connect your Kobo e-reader
- **[OPDS Integration](integration/opds)** - Access your library from compatible reading apps
- **[System Task Manager](tools/task-manager)** - Monitor background tasks and system performance

---

## 💡 Additional Resources

- **[GitHub Repository](https://github.com/adityachandelgit/booklore)** - Source code and issue tracking
- **[GitHub Releases](https://github.com/adityachandelgit/booklore/releases)** - Version history and changelogs
- **[Docker Hub](https://hub.docker.com/r/booklore/booklore-app)** - Official container images
- **[Community Discord](#)** - Get help and share tips

---

Happy reading! 📚✨ If you encounter any issues not covered here, please check the GitHub issues or reach out to the community for support.
