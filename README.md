# On-Premise Dockerized Attendance Tracking System with Monitoring and CI/CD for FCT College of Nursing Sciences, Gwagwalada

## 🧭 Introduction

The FCT College of Nursing Sciences, Gwagwalada, currently uses a **manual system for attendance tracking**. This causes **delays, errors**, and **lack of real-time insights** into staff and student attendance data.

As a DevOps engineer in transition, this project proposes a **modular, Docker-based solution for tracking attendance**, integrating monitoring and automation tools such as **Prometheus, Grafana, and Jenkins**, and hosted initially **on-premise** — but ready for **future migration to the cloud**.

This project is designed to be **scalable, automated, observable**, demonstrating real-world DevOps practices.

## 🎯 Project Objectives

- Replace manual attendance with a web-based system.
- Use Docker to deploy the entire stack on-premise.
- Use MongoDB for backend storage.
- Integrate Prometheus + Grafana for monitoring.
- Add Jenkins for CI/CD pipeline.
- Build the project with modular file/folder structure
- Ensure everything is extensible and cloud-ready.

## 🛠️ Technologies Used
- **Docker:**	Containerization of services
- **Docker Compose:**	Multi-container orchestration
- **MongoDB:**	Backend database for attendance records
- **Prometheus:**	Metrics collection and alerting
- **Grafana:**	Dashboard visualization of metrics
- **Jenkins:**	CI/CD automation
- **Bash:**	Automation scripts for setup
- **.env Files:**	Store environment variables securely

## ✅ Prerequisites

Ensure you have the following:

- A Linux server (Ubuntu 22.04+ recommended)
- User with sudo privileges
- Internet access on the server
- Basic knowledge of Linux command line

## 🗂️ Project Directory Structure

I’ll now create a clean, modular folder structure.

```bash
# Navigate to my github directory
cd fullstack-attendance-monitoring

# Create subfolders for configuration, images, scripts, etc.
mkdir -p \
  config/grafana/dashboards \
  config/grafana/datasources \
  config/prometheus \
  docs \
  images \
  jenkins \
  scripts

# Create top-level files
touch .env docker-compose.yml README.md

# Optional: View structure using tree (install tree with 'sudo apt install tree' if not available)
tree -L 3
```

```bash
fullstack-attendance-monitoring/
│
├── config/
│   ├── grafana/
│   │   ├── dashboards/          # Custom Grafana dashboards in JSON
│   │   └── datasources/         # Grafana data source configs
│   └── prometheus/              # Prometheus configuration (e.g., prometheus.yml)
│
├── docs/                        # Documentation files, diagrams, usage guides
│
├── images/                      # System architecture images, screenshots, diagrams
│
├── jenkins/                     # Jenkinsfiles, job configurations, pipeline scripts
│   └── Jenkinsfile
│
├── scripts/                     # Shell scripts for setup, deployment, backups, etc.
│   ├── install_docker.sh
│   └── setup.sh
│
├── .gitignore                   # Files and folders to ignore in Git
├── docker-compose.yml           # Main Docker Compose file
├── .env                         # Environment variables
└── README.md                    # Project overview and instructions
```

## 🧰 Step 1: Install Docker & Docker Compose

Let’s create a bash script to automate the installation of Docker and Docker Compose. Save this in the `scripts/` folder.

### Step 1.1: Create the Script

```bash
nano scripts/install_docker.sh
```

### Step 1.2: Add Script Contents

```bash
#!/bin/bash

# ---------------------------------------------------------------
# Script Name: install_docker.sh
# Purpose: Install Docker and Docker Compose on Ubuntu
# ---------------------------------------------------------------

# Exit the script on any error
set -e

# Update and upgrade system packages
echo "📦 Updating system packages..."
sudo apt update -y && sudo apt upgrade -y

# Install required packages for Docker installation
echo "📥 Installing dependencies..."
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker’s official GPG key
echo "🔐 Adding Docker GPG key..."
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up Docker repository
echo "📂 Setting up Docker APT repository..."
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
echo "🐳 Installing Docker Engine and CLI..."
sudo apt update -y
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Enable and start Docker service
echo "🚀 Enabling Docker service..."
sudo systemctl enable docker
sudo systemctl start docker

# Add current user to Docker group to avoid 'sudo' every time
echo "👤 Adding user to Docker group..."
sudo usermod -aG docker $USER

# Install Docker Compose manually (optional fallback)
echo "🔧 Installing Docker Compose standalone binary..."
DOCKER_COMPOSE_VERSION="2.24.0"
sudo curl -L "https://github.com/docker/compose/releases/download/v${DOCKER_COMPOSE_VERSION}/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose

# Make Docker Compose executable
sudo chmod +x /usr/local/bin/docker-compose

# Verify installations
echo "✅ Docker version:"
docker --version

echo "✅ Docker Compose version:"
docker-compose --version

echo "🎉 Docker & Docker Compose installation complete. Please log out and log back in to apply group changes."
```

### Step 1.3: Make It Executable & Run It

```bash
chmod +x scripts/install_docker.sh
./scripts/install_docker.sh
```

### ✅ At this point, Docker and Docker Compose should be successfully installed.
Please log out and log back in to make the Docker group permissions apply.

![](./images/1.confirm-docker-installation.png)

## 🧾 Step 2: Create the `.env` File

🔐 The `.env` file stores **environment variables** like service ports, database names, and credentials. This makes your `docker-compose.yml` file cleaner, easier to maintain, and more secure.

![](./images/2.env-file.png)

### 💡 Why use a .env file?

- **Modularity:**	Reuse variables across services
- **Security:**	Prevent hardcoding sensitive info in Compose file
- **Portability:**	Easier to adjust ports/settings on new systems
- **CI/CD Ready:**	Integrates easily with Jenkins/DevOps workflows

## 🔜 Step 3: Write the `docker-compose.yml` file

📦 This file defines and connects all the containers (`MongoDB, attendance app, Jenkins, Prometheus, Grafana`) into a **single stack** using environment variables from the `.env` file we created earlier.

```bash
version: '3.8'  # Docker Compose version

services:
  # ----------------------------------------------------
  # MongoDB Database for Attendance App
  # ----------------------------------------------------
  mongo:
    image: mongo:latest
    container_name: mongo
    restart: always
    ports:
      - "${MONGO_PORT}:27017"  # Map container port 27017 to host's configured port
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_INITDB_ROOT_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_INITDB_ROOT_PASSWORD}
    volumes:
      - mongo-data:/data/db  # Persist database data
    networks:
      - attendance-net

  # ----------------------------------------------------
  # Very Simple Attendance App (rakshitbharat image)
  # ----------------------------------------------------
  attendance-app:
    image: rakshitbharat/very-simple-attendance:latest
    container_name: attendance-app
    restart: always
    depends_on:
      - mongo  # Ensure MongoDB starts first
    ports:
      - "${APP_PORT}:3000"  # Expose the app on host port defined in .env
    environment:
      DB_HOST: mongo
      DB_PORT: 27017
      DB_USERNAME: ${MONGO_INITDB_ROOT_USERNAME}
      DB_PASSWORD: ${MONGO_INITDB_ROOT_PASSWORD}
      DB_NAME: ${MONGO_DB_NAME}
    networks:
      - attendance-net

  # ----------------------------------------------------
  # Jenkins - CI/CD Tool
  # ----------------------------------------------------
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: unless-stopped
    ports:
      - "${JENKINS_PORT}:8080"
    volumes:
      - jenkins_home:/var/jenkins_home  # Persist Jenkins configurations
    networks:
      - attendance-net

  # ----------------------------------------------------
  # Prometheus - Metrics Collection
  # ----------------------------------------------------
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "${PROMETHEUS_PORT}:9090"
    volumes:
      - ./config/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml  # Custom config
    networks:
      - attendance-net

  # ----------------------------------------------------
  # Grafana - Dashboard Visualization
  # ----------------------------------------------------
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "${GRAFANA_PORT}:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./config/grafana/datasources:/etc/grafana/provisioning/datasources
      - ./config/grafana/dashboards:/etc/grafana/provisioning/dashboards
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    networks:
      - attendance-net

# ----------------------------------------------------
# Docker Volumes - Persist important data
# ----------------------------------------------------
volumes:
  mongo-data:
  grafana_data:
  jenkins_home:

# ----------------------------------------------------
# Docker Network - Isolate and connect containers
# ----------------------------------------------------
networks:
  attendance-net:
    driver: bridge
```

### ✅ Let me Recap the Services:

- **mongo:**	Stores attendance records
- **attendance-app:**	Web UI for tracking attendance
- **jenkins:**	Automates builds & deployments
- **prometheus:**	Collects performance metrics
- **grafana:**	Visualizes metrics and dashboards

### 🚀 To Launch All Services

After saving the file, I would launch my full stack:

```bash
docker-compose up -d
```

## Error

![](./images/3.error1.png)

```bash
Error response from daemon: pull access denied for rakshitbharat/very-simple-attendance, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
```

means Docker is trying to pull an image from Docker Hub named:

```
rakshitbharat/very-simple-attendance
```

But it's failing because:

1. The image does not exist on Docker Hub (it may have been deleted or never pushed).

2. The image is private, and you're not logged in to an authorized Docker Hub account.

3. The image name is misspelled or incorrect in your `docker-compose.yml`.

## ✅ Fix

### Use build Instead of image

1. Create the `attendance-app` folder inside the root directory
2. Clone the app inside `attendance-app/`
   
```bash
git clone https://github.com/rakshitbharat/very-simple-attendance.git .
```

Notice the `.` at the end — it means “clone into the current folder” (so you don’t end up with a nested folder).

![](./images/4.git-clone.png)

3. Replace `image`: with `build`: in `docker-compose.yml`

Inside my existing `docker-compose.yml`, update the `attendance-app/source` service:

![](./images/5.uppppdddaattee-docker-compose-file.png)

This tells **Docker Compose**: "Build an image from the `Dockerfile` in the `attendance-app/` folder."


**4. 🚀 Launch the App**

Then run:

```bash
sudo docker-compose up -d --build
```

![](./images/6.building.png)

## Error

![](./images/7.error.png)

The error you're encountering suggests that there's an issue with the mounting of the prometheus.yml configuration file. Specifically, Docker is trying to mount the host file /home/cloudit/Documents/darey-learning-ubuntu/projects/fullstack-attendance-monitoring/config/prometheus/prometheus.yml to the container's directory /etc/prometheus/prometheus.yml, but it appears the destination path in the container is not a directory as expected.

**Here’s how to fix the issue:**

Ensure that the `prometheus.yml` file exists at the specified path on the host system:

![](./images/8.prometheus-file.png)

✅ Next Steps:

Re-run:

```bash
sudo docker-compose up -d --build
```

This will attempt to:

- Rebuild the services if necessary.

- Restart containers using your current configuration.

- Mount the corrected `prometheus.yml` file properly.

![](./images/9.prometheus-started.png)


### Error

My `CPU` doesn't support `AVX` (needed for newer MongoDB).

### ✅ Step-by-Step Fix Summary

- Replace `MongoDB` block in `docker-compose.yml` with `PostgreSQL`

- Update environment variables in `.env` to reflect `PostgreSQL` settings

- Update `attendance-app` **service** to use `PostgreSQL`

- Ensure `Prisma` is already set up for `PostgreSQL` ✅ (which I confirmed)

### ✅ Updated .env File (PostgreSQL Version)

![](./images/10.env-update.png)

### ✅ Updated `docker-compose.yml` File

![](./images/docker-compose-update.png)

### ✅ Update attendance-app service to use PostgreSQL

![](./images/attendance-app-service-update.png)

### Rebuild the app with PostgreSQL

```bash
docker-compose down -v   # Stops and removes all containers, volumes, and network
docker-compose up --build
```

### Enter the `attendance-app/source` directory

```bash
cd attendance-app/source
```

### Generate the `Prisma`client

```bash
npx prisma generate
```

- ✅ This uses the DATABASE_URL in schema.prisma to prepare the client code

    ![](./images/11.prima-generate.png)

### Push schema to `PostgreSQL`

```bash
npx prisma db push
```

- ✅ This creates the tables in your attendance_db inside the PostgreSQL container

- ✅ Based on the models defined in your schema.prisma

### Error

![](./images/12.error.png)

Prisma is working fine, but it's throwing this error:

```
Error: Environment variable not found: DATABASE_URL.
```

This means Prisma can't find the DATABASE_URL variable it needs to connect to your PostgreSQL DB.

Prisma does not automatically read your root .env file unless it’s in the same folder where schema.prisma is located (in your case, attendance-app/source/prisma/ or attendance-app/source/ depending on where schema.prisma lives).

✅ Update `.env` file with `DATABASE_URL` added:

![](./images/13.env-updated-with-url.png)

✅ Next Steps (after updating the `.env` file):

1. Copy the `.env` file to the Prisma folder so Prisma can access it:

If your schema.prisma is in attendance-app/source/prisma/, then:

```bash
cp ../../.env prisma/.env    # Adjust path based on where prisma/ is
```

### Bonus: Prisma Best Practice ✅

To avoid surprises later (and warnings like this):

```bash
Warning: You did not specify an output path for your `generator`
```

Edit your `schema.prisma` like so:

```bash
generator client {
  provider = "prisma-client-js"
  output   = "./prisma/client"
}
```

![](./images/14.edittt---prima.png)

Then run:

```bash
npx prisma generate
npx prisma db push
```

![](./images/15.prima-generated-successfully.png)

**Boom! 🎉** That worked perfectly — Prisma Client **generated successfully!**

Now you're all set to push your schema to the PostgreSQL database.

**🚀 Next Step:**

Run:

```bash
npx prisma db push
```
### Error

![](./images/16.error.png)

✅ 1. Confirm Your PostgreSQL Container Is Running

Run:

```bash
sudo docker ps -a
```

![](./images/17.db-running.png)

**Perfect!** From your `sudo docker ps -a` output, we can see this:

- ✅ The PostgreSQL container is named postgres, which matches what Prisma expects.

- ✅ It’s running and exposing port 5432.

But Prisma still says:

`Can't reach database server at postgres:5432`

So the issue is most likely network-related—because Prisma is trying to resolve postgres as a hostname from the host machine, instead of from within a containerized network.

**✅ Solution:** Run `prisma db push` Inside the `attendance-app` Container

To connect via service name (`postgres`), Prisma **must run inside the Docker network**. So instead of running Prisma on your host (Ubuntu), run it inside the `attendance-app` container.

Here’s how:

```bash
docker exec -it attendance-app npx prisma db push
```

![](./images/18.prima-db-push.png)

This ensures `Prisma` can resolve the `hostname postgres` within the `Docker network`.

To avoid use of sudo at every run - run:

```bash
sudo usermod -aG docker $USER
newgrp docker
```