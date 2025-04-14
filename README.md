
```
ARG BASE_IMAGE=ubuntu:22.04
FROM $BASE_IMAGE

USER 0

ENV DEBIAN_FRONTEND=noninteractive

# Copy the local MongoDB 8.0 GPG key
COPY server-8.0.asc /tmp/server-8.0.asc

RUN apt-get update && apt-get install -y wget gnupg sudo curl && \
    gpg --dearmor /tmp/server-8.0.asc > /etc/apt/trusted.gpg.d/mongodb.gpg && \
    echo "deb [ arch=amd64,arm64 signed-by=/etc/apt/trusted.gpg.d/mongodb.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/8.0 multiverse" | tee /etc/apt/sources.list.d/mongodb-org-8.0.list && \
    apt-get update && \
    apt-get install -y mongodb-org && \
    apt-get clean

COPY start-mongo.sh ./start-mongo.sh
RUN chmod +x ./start-mongo.sh

USER 1001
EXPOSE 27017

ENTRYPOINT ["./start-mongo.sh"]


```


Here is a technical-style README draft tailored for your Cloud Run restart solution:

🔁 Manual Restart for Google Cloud Run Services
This repository provides a mechanism to manually trigger a restart for a Cloud Run service by updating a dummy environment variable, useful when traditional configuration changes or deployments aren't performed via a service.yaml.

🔍 Problem Statement
In Google Cloud Run (GCR), services are serverless and do not maintain active compute resources when idle. This architecture doesn't support an explicit "stop" or "restart" operation like GKE. The concept of restart is simulated by updating the service configuration — which triggers a redeployment.
In typical deployments, a service.yaml file is used. Any change to this file (e.g., image tag, environment variable) triggers a redeployment of the specified service.
However, in manual GitHub UI-based workflows, the service.yaml file is not always available, and there’s a need to restart the service without modifying core configurations.

✅ Solution: Dummy Environment Variable Update
We trigger a redeployment by patching the Cloud Run service to update a dummy environment variable (LAST_RESTART) with the current UNIX timestamp. This introduces a configuration change, prompting GCR to redeploy the service.

📁 Old Deployment Workflow Overview
Workflow triggers a reusable deploy job.
Authenticated using a Deployer Service Account, credentials retrieved from Google Secret Manager.
Uses: bash CopyEdit   gcloud run services replace service.yaml
  
The service.yaml includes the service's metadata.name. If any config changes (env var, image tag, etc.), GCR redeploys the service.

🚀 New Restart Mechanism
No service.yaml involved. Works even with GitHub UI-triggered manual deployments.
🔧 CLI Command to Restart a Service
bash
CopyEdit
SERVICE_NAME=my-cloud-run-service
REGION=us-central1
PROJECT_ID=my-gcp-project

gcloud run services update $SERVICE_NAME \
  --region=$REGION \
  --project=$PROJECT_ID \
  --update-env-vars "LAST_RESTART=$(date +%s)"
This command does not affect other configuration and only updates the dummy variable LAST_RESTART.
GCR detects the change and triggers a zero-downtime redeployment.

🛡️ Permissions Required
Make sure the Deployer Service Account used has the following roles:
roles/run.admin — to update Cloud Run services.
roles/iam.serviceAccountUser — to impersonate service accounts (if applicable).
Access to Google Secret Manager (if secrets are used for credentials).

🔁 Automation via GitHub Actions
If you're triggering this from a GitHub Actions workflow:
yaml
CopyEdit
jobs:
  restart-cloud-run:
    runs-on: ubuntu-latest
    steps:
      - name: Authenticate with GCP
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_DEPLOYER_SA_KEY }}

      - name: Restart Cloud Run service
        run: |
          gcloud run services update my-cloud-run-service \
            --region=us-central1 \
            --project=my-gcp-project \
            --update-env-vars LAST_RESTART=$(date +%s)

📌 Notes
This method introduces no downtime for live traffic.
The variable LAST_RESTART can be used for debugging or logging restart times.
Avoid modifying real configuration parameters unless required.

📄 References
GCP: Updating Cloud Run Services
Google GitHub Actions Auth

Let me know if you’d like to add a shell script or GitHub reusable workflow for this!






# test-repo

```
apache-superset
psycopg2-binary



#!/bin/bash
set -e

# Initialize the database (only runs if not already initialized)
superset db upgrade

# Create an admin user if it doesn't exist
superset fab create-admin \
    --username admin \
    --firstname Admin \
    --lastname User \
    --email admin@example.com \
    --password admin || true

# Load examples (optional)
# superset load_examples

# Setup roles and permissions
superset init

# Start the server
superset run -h 0.0.0.0 -p 8088






chmod +x entrypoint.sh



# Base image
FROM python:3.9-slim

# Set environment variables
ENV LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    SUPERSET_HOME=/app/superset

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    libssl-dev \
    libffi-dev \
    libpq-dev \
    libsasl2-dev \
    libldap2-dev \
    curl \
    default-libmysqlclient-dev \
    git \
    && rm -rf /var/lib/apt/lists/*

# Create app directory
WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy entrypoint
COPY entrypoint.sh .

# Expose Superset port
EXPOSE 8088

# Run Superset
CMD ["./entrypoint.sh"]









import psycopg2
from psycopg2 import sql, OperationalError

def connect_to_postgres_db(host='localhost', port=5432, db_name='', user='', password=''):
    try:
        conn = psycopg2.connect(
            dbname=db_name,
            user=user,
            password=password,
            host=host,
            port=port
        )
        print(f"✅ Connected to database '{db_name}' as user '{user}'")
        return conn
    except OperationalError as e:
        print("❌ Connection failed:", e)
        return None

if __name__ == '__main__':
    conn = connect_to_postgres_db(
        db_name='mydb',
        user='myuser',
        password='mypassword',
        host='localhost',
        port=5432
    )

    if conn:
        with conn.cursor() as cur:
            cur.execute("SELECT current_database(), current_user;")
            print("📦 Result:", cur.fetchone())

        conn.close()


echo "host all all 172.20.0.1/32 md5" >> /etc/postgresql/12/main/pg_hba.conf
















#!/bin/bash

set -e

# Start PostgreSQL service
pg_ctlcluster 12 main start

# Create user if env vars are set
if [[ -n "$POSTGRES_USER" && -n "$POSTGRES_PASSWORD" ]]; then
  echo "Attempting to create user: $POSTGRES_USER"

  if su - postgres -c "psql -c \"CREATE USER ${POSTGRES_USER} WITH PASSWORD '${POSTGRES_PASSWORD}';\""; then
    echo "User ${POSTGRES_USER} created successfully."
  else
    echo "User creation failed. The user '${POSTGRES_USER}' may already exist."
    echo "Please choose a different username and password."
  fi
fi

# Create database if env var is set
if [[ -n "$POSTGRES_DATABASE" ]]; then
  echo "Attempting to create database: $POSTGRES_DATABASE"

  if su - postgres -c "psql -tc \"SELECT 1 FROM pg_database WHERE datname='${POSTGRES_DATABASE}'\" | grep -q 1"; then
    echo "Database '${POSTGRES_DATABASE}' already exists."
  else
    su - postgres -c "psql -c \"CREATE DATABASE ${POSTGRES_DATABASE} OWNER ${POSTGRES_USER};\""
    echo "Database '${POSTGRES_DATABASE}' created and owned by '${POSTGRES_USER}'."
  fi
fi

# Keep the container running
tail -f /dev/null










#!/bin/bash

set -e

# Start the PostgreSQL service
pg_ctlcluster 12 main start

# Only try to create the user if env vars are set
if [[ -n "$POSTGRES_USER" && -n "$POSTGRES_PASSWORD" ]]; then
  echo "Attempting to create user: $POSTGRES_USER"

  if su - postgres -c "psql -c \"CREATE USER ${POSTGRES_USER} WITH PASSWORD '${POSTGRES_PASSWORD}';\""; then
    echo "User ${POSTGRES_USER} created successfully."
  else
    echo "User creation failed. The user '${POSTGRES_USER}' may already exist."
    echo "Please choose a different username and password."
  fi
else
  echo "POSTGRES_USER and POSTGRES_PASSWORD must be set to create a user."
fi

# Keep the container running
tail -f /dev/null







FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive

# Install PostgreSQL
RUN apt-get update && apt-get install -y postgresql postgresql-contrib

# Copy entrypoint script
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# Expose the default Postgres port
EXPOSE 5432

# Run the script
ENTRYPOINT ["/entrypoint.sh"]







version: "3.8"

services:
  custom-postgres:
    build: .
    container_name: custom_postgres
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypass
    volumes:
      - ./pgdata:/var/lib/postgresql
    ports:
      - "5432:5432"





metadata:
  annotations:
    run.googleapis.com/revision-suffix: "rev-{{timestamp}}"

sed -i "s/{{timestamp}}/$(date +%s)/" service.yaml
gcloud run services replace service.yaml

```


