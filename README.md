# test-repo

```
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


