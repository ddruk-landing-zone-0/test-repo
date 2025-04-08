# test-repo

```
#!/bin/bash

set -e

# Start the PostgreSQL service
pg_ctlcluster 12 main start

# Check if user exists
USER_EXISTS=$(su - postgres -c "psql -tAc \"SELECT 1 FROM pg_roles WHERE rolname='${POSTGRES_USER}'\"")

if [ "$USER_EXISTS" = "1" ]; then
  echo "User ${POSTGRES_USER} exists. Altering password..."
  su - postgres -c "psql -c \"ALTER USER ${POSTGRES_USER} WITH PASSWORD '${POSTGRES_PASSWORD}';\""
else
  echo "User ${POSTGRES_USER} does not exist. Creating user..."
  su - postgres -c "psql -c \"CREATE USER ${POSTGRES_USER} WITH PASSWORD '${POSTGRES_PASSWORD}';\""
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


