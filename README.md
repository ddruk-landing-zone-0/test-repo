No Vuln Image

https://storage.googleapis.com/cloud-ai-police-bucket-0/images/qdrant-qdrant-latest_23dec_2025.tar
https://storage.googleapis.com/cloud-ai-police-bucket-0/images/rabbitmq-management-alpine.tar
https://storage.googleapis.com/cloud-ai-police-bucket-0/images/redis-latest_23dec_2025.tar
https://storage.googleapis.com/cloud-ai-police-bucket-0/images/python_3_13-agent_gcloud.tar




----------
Dockerfile

```
# Python 3.13 slim base image
FROM python:3.13-slim

# Environment variables
ENV DEBIAN_FRONTEND=noninteractive
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Set working directory
WORKDIR /app

RUN apt-get update

RUN apt-get update && apt-get install -y --no-install-recommends \
    bash \
    # build-essential \
    # gcc \
    # g++ \
    make \
    curl \ 
    wget \
    git \
    ca-certificates \
    openssl \
    libssl-dev \
    libffi-dev \
    # libpq-dev \
    netcat-openbsd \
    iputils-ping \
    procps \
    tzdata \
    && rm -rf /var/lib/apt/lists/*

# Upgrade pip and build tools
RUN python -m pip install --upgrade pip setuptools wheel

# Copy requirements first (layer caching)
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt
```


Req.txt

```
# Server
fastapi
uvicorn
flask
gunicorn
flask-cors

# Database
psycopg2-binary
sqlalchemy
pymongo
dnspython

# Data Processing
pandas
numpy
scipy
scikit-learn


# gcloud
google-auth
google-auth-oauthlib
google-auth-httplib2


# Scalabilty
celery[redis]
redis
boto3
protobuf3
kombu

# Kubernetes
kubernetes
kopf

# Vector DB
# faiss-cpu
qdrant-client

# Langchain, langgraph etc
langchain
langchain-core
langgraph
langsmith
langchain-community
langchain-community[faiss]
langchain-community[qdrant]


# Others
python-dotenv
requests
pytest
pytest-cov
urllib3==2.6.0
colorlog
pydantic
pillow
python-multipart
aiofiles
asyncio
grpcio
psutil
typing
deprecated
datasketch      # MinHash + LSH for ≥90% similarity check
xxhash          # ultra-fast hashing for dedup keys
orjson          # very fast metadata serialization
uvloop          # faster asyncio event loop (Linux/Mac)
gensim          # streaming TF-IDF (better than sklearn for logs)
```
----------



https://0825-202-168-84-98.ngrok-free.app

What problem did it solve?
This Terraform refactoring reduced lines of code (LOC), removed redundancies, and increased reusability.

What solution is implemented?
Configuration lists were created in the locals. All types of roles and service accounts are mapped in locals and reused. Looping is used to allocate different instances of the same type of resource and module. Outputs are stored so they can be used as remote variables in other plugins.

How can it be adopted?
A README has been added in all plugins.


```
name: Progressive Blue-Green Deployment to Cloud Run

on:
  workflow_dispatch:
    inputs:
      SERVICE_NAME:
        description: 'Cloud Run service name'
        required: true
        type: string
      PROJECT_ID:
        description: 'GCP Project ID'
        required: true
        type: string
      REGION:
        description: 'GCP Region'
        required: true
        type: string
      TRAFFIC_SHIFT:
        description: 'Traffic shift percentage (positive integer, e.g., 20)'
        required: true
        type: number

jobs:
  shift-traffic:
    name: Progressive Traffic Shift
    runs-on: ubuntu-latest
    environment: production

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v2
      with:
        credentials_json: '${{ secrets.GCP_SA_KEY }}'

    - name: Set up gcloud CLI
      uses: google-github-actions/setup-gcloud@v2
      with:
        project_id: '${{ inputs.PROJECT_ID }}'
        install_components: 'beta'

    - name: Fetch Current Traffic Split
      id: traffic
      run: |
        echo "Fetching current traffic split..."

        SERVICE_JSON=$(gcloud run services describe ${{ inputs.SERVICE_NAME }} \
          --platform=managed \
          --region=${{ inputs.REGION }} \
          --format=json)

        BLUE_REVISION=$(echo "$SERVICE_JSON" | jq -r '.status.traffic[] | select(.percent >= 50) | .revisionName')
        GREEN_REVISION=$(echo "$SERVICE_JSON" | jq -r '.status.traffic[] | select(.revisionName != "'$BLUE_REVISION'") | .revisionName')

        BLUE_PERCENT=$(echo "$SERVICE_JSON" | jq -r '.status.traffic[] | select(.revisionName == "'$BLUE_REVISION'") | .percent')
        GREEN_PERCENT=$(echo "$SERVICE_JSON" | jq -r '.status.traffic[] | select(.revisionName == "'$GREEN_REVISION'") | .percent')

        echo "BLUE_REVISION=$BLUE_REVISION" >> $GITHUB_ENV
        echo "GREEN_REVISION=$GREEN_REVISION" >> $GITHUB_ENV
        echo "BLUE_PERCENT=$BLUE_PERCENT" >> $GITHUB_ENV
        echo "GREEN_PERCENT=$GREEN_PERCENT" >> $GITHUB_ENV

        echo "Blue Revision: $BLUE_REVISION at $BLUE_PERCENT%"
        echo "Green Revision: $GREEN_REVISION at $GREEN_PERCENT%"

    - name: Perform Traffic Shift
      if: ${{ env.BLUE_PERCENT != '0' && env.GREEN_PERCENT != '100' }}
      run: |
        echo "Calculating new traffic split with user-specified shift..."

        SHIFT=${{ inputs.TRAFFIC_SHIFT }}

        BLUE_CURRENT=${BLUE_PERCENT}
        GREEN_CURRENT=${GREEN_PERCENT}

        # New calculations with boundary checks
        if [ "$BLUE_CURRENT" -le "$SHIFT" ]; then
          NEW_BLUE=0
        else
          NEW_BLUE=$((BLUE_CURRENT - SHIFT))
        fi

        if [ "$((GREEN_CURRENT + SHIFT))" -ge 100 ]; then
          NEW_GREEN=100
        else
          NEW_GREEN=$((GREEN_CURRENT + SHIFT))
        fi

        echo "New traffic allocation: $NEW_BLUE% to BLUE, $NEW_GREEN% to GREEN."

        gcloud run services update-traffic ${{ inputs.SERVICE_NAME }} \
          --region=${{ inputs.REGION }} \
          --to-revisions ${BLUE_REVISION}=${NEW_BLUE},${GREEN_REVISION}=${NEW_GREEN}

    - name: Skip Traffic Shift (Already completed)
      if: ${{ env.BLUE_PERCENT == '0' || env.GREEN_PERCENT == '100' }}
      run: |
        echo "✅ Traffic shift not needed. Deployment already complete (Blue at 0% or Green at 100%)."










name: Blue Green Deployment to Cloud Run

on:
  workflow_dispatch:
    inputs:
      SERVICE_NAME:
        description: 'Cloud Run service name'
        required: true
        type: string
      PROJECT_ID:
        description: 'GCP Project ID'
        required: true
        type: string
      REGION:
        description: 'GCP Region (e.g. us-central1)'
        required: true
        type: string

jobs:
  deploy:
    name: Blue Green Deployment
    runs-on: ubuntu-latest
    environment: production

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v2
      with:
        credentials_json: '${{ secrets.GCP_SA_KEY }}'  # Service Account JSON stored as GitHub secret

    - name: Set up gcloud CLI
      uses: google-github-actions/setup-gcloud@v2
      with:
        project_id: '${{ inputs.PROJECT_ID }}'
        install_components: 'beta' # Needed for traffic splitting commands

    - name: Get current service revisions
      id: get-revisions
      run: |
        BLUE_REVISION=$(gcloud run services describe ${{ inputs.SERVICE_NAME }} \
          --platform=managed \
          --region=${{ inputs.REGION }} \
          --format="value(status.traffic[0].revisionName)")
        
        GREEN_REVISION=$(gcloud run revisions list \
          --service=${{ inputs.SERVICE_NAME }} \
          --region=${{ inputs.REGION }} \
          --sort-by="~createTime" \
          --format="value(metadata.name)" \
          --limit=1)

        echo "BLUE_REVISION=${BLUE_REVISION}" >> $GITHUB_ENV
        echo "GREEN_REVISION=${GREEN_REVISION}" >> $GITHUB_ENV

    - name: Initial Split (Blue 90% - Green 10%)
      run: |
        echo "Splitting traffic 90% BLUE and 10% GREEN"
        gcloud run services update-traffic ${{ inputs.SERVICE_NAME }} \
          --region=${{ inputs.REGION }} \
          --to-revisions ${BLUE_REVISION}=90,${GREEN_REVISION}=10

    - name: Sleep between shifts
      run: sleep 60  # Wait a bit for traffic to settle

    - name: Shift to 50% - 50%
      run: |
        echo "Splitting traffic 50% BLUE and 50% GREEN"
        gcloud run services update-traffic ${{ inputs.SERVICE_NAME }} \
          --region=${{ inputs.REGION }} \
          --to-revisions ${BLUE_REVISION}=50,${GREEN_REVISION}=50

    - name: Sleep between shifts
      run: sleep 60

    - name: Shift to 20% Blue - 80% Green
      run: |
        echo "Splitting traffic 20% BLUE and 80% GREEN"
        gcloud run services update-traffic ${{ inputs.SERVICE_NAME }} \
          --region=${{ inputs.REGION }} \
          --to-revisions ${BLUE_REVISION}=20,${GREEN_REVISION}=80

    - name: Sleep between shifts
      run: sleep 60

    - name: Shift to 0% Blue - 100% Green (Finalize Green)
      run: |
        echo "Finalizing deployment, 100% GREEN"
        gcloud run services update-traffic ${{ inputs.SERVICE_NAME }} \
          --region=${{ inputs.REGION }} \
          --to-revisions ${GREEN_REVISION}=100

    - name: Done
      run: echo "✅ Blue-Green deployment complete!"














from flask import Flask, jsonify, render_template_string, request
import random

app = Flask(__name__)

# Version config (set by environment variable)
import os
COLOR = os.environ.get('COLOR', 'blue')

# Stats
stats = {
    'blue_200': 0,
    'green_200': 0,
    'error': 0
}

@app.route('/')
def index():
    if COLOR == 'blue':
        stats['blue_200'] += 1
        return "blue,200", 200
    else:
        stats['green_200'] += 1
        return "green,200", 200

@app.route('/test')
def test():
    html = '''
    <!DOCTYPE html>
    <html>
    <head>
        <title>Test Page</title>
        <script>
            async function hit() {
                try {
                    const response = await fetch('/');
                    const text = await response.text();
                    console.log(text);
                } catch (error) {
                    console.error('Error:', error);
                }
                updateStats();
            }

            async function updateStats() {
                const res = await fetch('/stats');
                const data = await res.json();

                const total = data.blue_200 + data.green_200 + data.error;
                let html = `<p>Blue, 200: ${data.blue_200} (${(data.blue_200/total*100).toFixed(2)}%)</p>`;
                html += `<p>Green, 200: ${data.green_200} (${(data.green_200/total*100).toFixed(2)}%)</p>`;
                html += `<p>Error (4xx/5xx): ${data.error} (${(data.error/total*100).toFixed(2)}%)</p>`;

                document.getElementById('distribution').innerHTML = html;
            }

            window.onload = updateStats;
        </script>
    </head>
    <body>
        <h1>Test Page - {{ color.capitalize() }}</h1>
        <button onclick="hit()">Hit</button>
        <div id="distribution"></div>
    </body>
    </html>
    '''
    return render_template_string(html, color=COLOR)

@app.route('/stats')
def get_stats():
    return jsonify(stats)

# Simulate random errors (optional)
@app.errorhandler(Exception)
def handle_error(e):
    # stats['error'] += 1
    return "Internal Server Error", 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001)













mkdir -p /gcs/data/db /gcs/mongo/data /gcs/var/lib/mysql
mkdir -p /data/db /mongo/data /var/lib/mysql

mount --bind /gcs/data/db /data/db
mount --bind /gcs/mongo/data /mongo/data
mount --bind /gcs/var/lib/mysql /var/lib/mysql





mkdir -p /gcs/data/db /gcs/mongo/data /gcs/var/lib/mysql

ln -s /gcs/data/db /data/db
ln -s /gcs/mongo/data /mongo/data
ln -s /gcs/var/lib/mysql /var/lib/mysql






FILE_NAME="report-open-only.csv"
OUTFILE="/tmp/mail.txt"
ENCODED_MAIL="/tmp/final_mail.b64"

# Step 1: Generate beautified mail body
awk -F',' '
  function pad_right(str, len) {
    pad = ""
    while (length(str) + length(pad) < len) pad = pad " "
    return str pad
  }

  NR==1 {
    for (i = 1; i <= NF; i++) {
      header[i] = $i
      width[i] = length($i)
    }
    next
  }
  {
    for (i = 1; i <= NF; i++) {
      width[i] = (length($i) > width[i]) ? length($i) : width[i]
      data[NR-1, i] = $i
    }
    maxrow = NR - 1
  }
  END {
    print "Hi,\n\nHere is the trace for Hardcoded Credential:\n"
    for (i = 1; i <= length(header); i++) {
      printf "%s%s", pad_right(header[i], width[i]), (i == length(header) ? ORS : " | ")
    }
    for (i = 1; i <= length(header); i++) {
      sep = ""
      for (j = 1; j <= width[i]; j++) sep = sep "-"
      printf "%s%s", sep, (i == length(header) ? ORS : "-+-")
    }
    for (r = 1; r <= maxrow; r++) {
      for (i = 1; i <= length(header); i++) {
        printf "%s%s", pad_right(data[r, i], width[i]), (i == length(header) ? ORS : " | ")
      }
    }
  }
' "$FILE_NAME" > "$OUTFILE"

# Step 2: Encode the whole content
base64 -w 0 "$OUTFILE" > "$ENCODED_MAIL"

# Step 3: Replace placeholder in Helm YAML (base64-safe)
ESCAPED=$(cat "$ENCODED_MAIL")
sed -i "s#__EMAIL_B64_CONTENT__#$ESCAPED#" ./utils/mta-job/helm-chart/templates/job.yaml










containers:
  - name: mail-job
    image: your-image
    command: ["/bin/sh", "-c"]
    args:
      - |
        echo "__EMAIL_B64_CONTENT__" | base64 -d > /tmp/mail.txt && \
        sendmail < /tmp/mail.txt












awk -F',' '
  function pad_right(str, len) {
    pad = ""
    while (length(str) + length(pad) < len) pad = pad " "
    return str pad
  }

  NR==1 {
    for (i = 1; i <= NF; i++) {
      header[i] = $i
      width[i] = length($i)
    }
    next
  }
  {
    for (i = 1; i <= NF; i++) {
      width[i] = (length($i) > width[i]) ? length($i) : width[i]
      data[NR-1, i] = $i
    }
    maxrow = NR - 1
  }
  END {
    # Header row
    for (i = 1; i <= length(header); i++) {
      printf "%s%s", pad_right(header[i], width[i]), (i == length(header) ? ORS : " | ")
    }

    # Separator
    for (i = 1; i <= length(header); i++) {
      sep = ""
      for (j = 1; j <= width[i]; j++) sep = sep "-"
      printf "%s%s", sep, (i == length(header) ? ORS : "-+-")
    }

    # Data rows
    for (r = 1; r <= maxrow; r++) {
      for (i = 1; i <= length(header); i++) {
        printf "%s%s", pad_right(data[r, i], width[i]), (i == length(header) ? ORS : " | ")
      }
    }
  }
' "$FILE_NAME" > "$OUTFILE"




- name: Inject formatted email body into job.yaml safely
  run: |
    FILE_NAME="report-open-only.csv"
    FINAL_MAIL="/tmp/final_mail.txt"

    # Format the CSV content into a pretty table
    awk -F',' '
      NR==1 {
        for(i=1;i<=NF;i++) {
          header[i]=$i;
          width[i]=length($i);
        }
        next;
      }
      {
        for(i=1;i<=NF;i++) {
          width[i]=(length($i) > width[i]) ? length($i) : width[i];
          data[NR-1,i]=$i;
        }
        maxrow=NR-1;
      }
      END {
        for(i=1;i<=length(header);i++) {
          printf "%-*s%s", width[i], header[i], (i==length(header) ? ORS : " | ");
        }
        for(i=1;i<=length(header);i++) {
          printf "%-*s%s", width[i], gensub(/./,"-","g", sprintf("%*s", width[i], "")), (i==length(header) ? ORS : "-+-");
        }
        for(r=1;r<=maxrow;r++) {
          for(i=1;i<=length(header);i++) {
            printf "%-*s%s", width[i], data[r,i], (i==length(header) ? ORS : " | ");
          }
        }
      }
    ' "$FILE_NAME" > /tmp/formatted_table.txt

    {
      echo "From: pnl_sankalp@list.db.com"
      echo "To: debasmit-a.roy@db.com"
      echo "Subject: GCP CSV Report"
      echo
      echo "Hello from GCP,"
      echo
      echo "Here is the beautified CSV report:"
      echo
      cat /tmp/formatted_table.txt
    } > "$FINAL_MAIL"

    # Now safely escape it line-by-line for YAML
    ESCAPED=$(awk '{gsub(/["\\]/, "\\\\&"); gsub(/\n/, ""); print}' "$FINAL_MAIL" | sed ':a;N;$!ba;s/\n/@/g')

    # Replace placeholder using safer method (delimiter @)
    sed -i.bak "s@__EMAIL_CONTENT__@$ESCAPED@" ./utils/mta-job/helm-chart/templates/job.yaml





command: ["sh", "-c"]
args:
  - |
    echo "__EMAIL_CONTENT__" | tr '@' '\n' > /tmp/mail.txt

    curl -vk --url 'smtp://mta-service.mail-transfer-agent.svc.cluster.local:25' \
      --mail-from 'pnl_sankalp@list.db.com' \
      --mail-rcpt 'debasmit-a.roy@db.com' \
      --upload-file /tmp/mail.txt






- name: Generate beautified report body
  run: |
    FILE_NAME="report-open-only.csv"
    OUTFILE="/tmp/mail.txt"

    # Extract header and data
    HEADER=$(head -n 1 $FILE_NAME)
    BODY=$(tail -n +2 $FILE_NAME)

    # Convert commas to pipes, pad spaces
    HEADER_FMT=$(echo "$HEADER" | awk -F',' '{for(i=1;i<=NF;i++) printf "%-10s%s", $i, (i==NF?"":"| ")}')
    SEPARATOR=$(echo "$HEADER" | awk -F',' '{for(i=1;i<=NF;i++) printf "%-10s%s", "----------", (i==NF?"":"| ")}')
    DATA_FMT=$(echo "$BODY" | awk -F',' '{printf "%-10s| %-10s| %-10s\n", $1, $2, $3}')

    {
      echo "From: pnl_sankalp@list.db.com"
      echo "To: debasmit-a.roy@db.com"
      echo "Subject: Beautified CSV Report"
      echo
      echo "Hi,"
      echo
      echo "Here is the CSV content in a table-like format:"
      echo
      echo "$HEADER_FMT"
      echo "$SEPARATOR"
      echo "$DATA_FMT"
    } > "$OUTFILE"

    # Replace placeholder in job.yaml (escape for Helm template)
    ESCAPED=$(sed -e 's/[\/&]/\\&/g' "$OUTFILE" | tr '\n' '@')
    sed -i "s|__EMAIL_CONTENT__|$ESCAPED|" ./utils/mta-job/helm-chart/templates/job.yaml



command: ["sh", "-c"]
args:
  - |
    echo "__EMAIL_CONTENT__" | tr '@' '\n' > /tmp/mail.txt

    curl -vk --url 'smtp://mta-service.mail-transfer-agent.svc.cluster.local:25' \
      --mail-from 'pnl_sankalp@list.db.com' \
      --mail-rcpt 'debasmit-a.roy@db.com' \
      --upload-file /tmp/mail.txt



- name: Prepare MIME email with CSV attachment
  run: |
    FILE_NAME="report-open-only.csv"
    MIME_FILE="/tmp/email_mime.txt"

    # Base64 encode the file into a temporary
    base64 "$FILE_NAME" > /tmp/encoded.txt

    # Write the full MIME message
    {
      echo "From: pnl_sankalp@list.db.com"
      echo "To: debasmit-a.roy@db.com"
      echo "Subject: GCP CSV Report"
      echo "MIME-Version: 1.0"
      echo "Content-Type: multipart/mixed; boundary=\"boundary42\""
      echo
      echo "--boundary42"
      echo "Content-Type: text/plain; charset=\"UTF-8\""
      echo "Content-Transfer-Encoding: 7bit"
      echo
      echo "Hello from GCP,"
      echo "Please find the attached CSV report."
      echo
      echo "--boundary42"
      echo "Content-Type: text/csv; name=\"$FILE_NAME\""
      echo "Content-Transfer-Encoding: base64"
      echo "Content-Disposition: attachment; filename=\"$FILE_NAME\""
      echo
      cat /tmp/encoded.txt
      echo
      echo "--boundary42--"
    } > "$MIME_FILE"

    # Escape for sed (use `@` to preserve line breaks in Helm template)
    ESCAPED=$(sed -e 's/[\/&]/\\&/g' "$MIME_FILE" | tr '\n' '@')

    # Replace __EMAIL_CONTENT__ in job.yaml
    sed -i "s|__EMAIL_CONTENT__|$ESCAPED|" ./utils/mta-job/helm-chart/templates/job.yaml




command: ["sh", "-c"]
args:
  - |
    echo "__EMAIL_CONTENT__" | tr '@' '\n' > /tmp/mail.txt

    curl -vk --url 'smtp://mta-service.mail-transfer-agent.svc.cluster.local:25' \
      --mail-from 'pnl_sankalp@list.db.com' \
      --mail-rcpt 'debasmit-a.roy@db.com' \
      --upload-file /tmp/mail.txt



- name: Prepare MIME email with CSV attachment
  run: |
    FILE_NAME="report-open-only.csv"
    ENCODED_CONTENT=$(base64 -w 0 "$FILE_NAME")

    # Create MIME email directly as a file
    {
      echo "From: pnl_sankalp@list.db.com"
      echo "To: debasmit-a.roy@db.com"
      echo "Subject: GCP CSV Report"
      echo "MIME-Version: 1.0"
      echo "Content-Type: multipart/mixed; boundary=\"boundary42\""
      echo
      echo "--boundary42"
      echo "Content-Type: text/plain; charset=\"UTF-8\""
      echo
      echo "Hello from GCP,"
      echo "Please find the attached CSV report."
      echo
      echo "--boundary42"
      echo "Content-Type: text/csv; name=\"$FILE_NAME\""
      echo "Content-Transfer-Encoding: base64"
      echo "Content-Disposition: attachment; filename=\"$FILE_NAME\""
      echo
      echo "$ENCODED_CONTENT"
      echo "--boundary42--"
    } > /tmp/email_mime.txt

    # Escape for sed
    ESCAPED=$(sed -e 's/[\/&]/\\&/g' /tmp/email_mime.txt | tr '\n' '@')

    # Replace placeholder with escaped content in job.yaml
    sed -i "s|__EMAIL_CONTENT__|$ESCAPED|" ./utils/mta-job/helm-chart/templates/job.yaml




command: ["sh", "-c"]
args:
  - |
    echo "__EMAIL_CONTENT__" | tr '@' '\n' > /tmp/mail.txt

    curl -vk --url 'smtp://mta-service.mail-transfer-agent.svc.cluster.local:25' \
      --mail-from 'pnl_sankalp@list.db.com' \
      --mail-rcpt 'debasmit-a.roy@db.com' \
      --upload-file /tmp/mail.txt
















- name: Prepare MIME email with CSV attachment
  run: |
    FILE_NAME=report-open-only.csv
    ENCODED_CONTENT=$(base64 -w 0 $FILE_NAME)
    MIME_CONTENT=$(cat <<EOF
From: xyz@list.xyz.com
To: debasmit-a.roy@xyz.com
Subject: GCP CSV Report
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="boundary42"

--boundary42
Content-Type: text/plain; charset="UTF-8"

Hello from GCP,
Please find the attached CSV report.

--boundary42
Content-Type: text/csv; name="$FILE_NAME"
Content-Transfer-Encoding: base64
Content-Disposition: attachment; filename="$FILE_NAME"

$ENCODED_CONTENT
--boundary42--
EOF
    )
    
    # Escape special characters for sed
    ESCAPED=$(printf '%s\n' "$MIME_CONTENT" | sed -e 's/[\/&]/\\&/g')
    
    # Replace __EMAIL_CONTENT__ placeholder in your Helm job.yaml
    sed -i "s|__EMAIL_CONTENT__|$ESCAPED|" ./utils/mta-job/helm-chart/templates/job.yaml










command: ["sh", "-c"]
args:
  - |
    cat <<EOF > /tmp/mail.txt
    __EMAIL_CONTENT__
    EOF

    curl -vk --url 'smtp://mta-service.mail-transfer-agent.svc.cluster.local:25' \
      --mail-from 'xyz@list.xyz.com' \
      --mail-rcpt 'debasmit-a.roy@xyz.com' \
      --upload-file /tmp/mail.txt




























IMAGE_NAME_TAG=$(docker images --format '{{.Repository}}:{{.Tag}}' | head -n 1)


#!/usr/bin/env bash
set -Eeuo pipefail

java -version

: "${GITHUB_REF?Expected env var GITHUB_REF not set}"
: "${GITHUB_SHA?Expected env var GITHUB_SHA not set}"

VERSION=""
if [[ "$GITHUB_REF" == refs/heads/* ]]; then
  GIT_BRANCH="${GITHUB_REF#refs/heads/}"
  GIT_BRANCH_SLUG=$(echo "$GIT_BRANCH" | tr '[:upper:]' '[:lower:]' | tr -c '[:alnum:]' '-' | sed 's/^-*//' | sed 's/-*$//')
  VERSION="${GIT_BRANCH_SLUG}-${GITHUB_SHA}"
elif [[ "$GITHUB_REF" == refs/tags/* ]]; then
  GIT_TAG="${GITHUB_REF#refs/tags/}"
  VERSION="$GIT_TAG"
else
  VERSION="local-${GITHUB_SHA}"
fi

export VERSION

# Determine Maven command
MVN=$( [ -f "./mvnw" ] && echo "./mvnw" || echo "mvn" )

# Build image locally (to Docker daemon)
$MVN compile jib:dockerBuild \
  -Djib.to.image="my-app:${VERSION}" \
  -DskipTests \
  --batch-mode

echo "✅ Image built: my-app:${VERSION}"
echo "📦 Listing Docker images..."
docker images | grep "my-app"









FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# Install dependencies
RUN apt-get update && apt-get install -y \
    curl gnupg lsb-release wget software-properties-common \
    mysql-server postgresql postgresql-contrib \
    supervisor

# --------------------
# Install MongoDB
# --------------------
RUN curl -fsSL https://www.mongodb.org/static/pgp/server-6.0.asc | gpg --dearmor -o /usr/share/keyrings/mongodb.gpg && \
    echo "deb [signed-by=/usr/share/keyrings/mongodb.gpg] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse" | tee /etc/apt/sources.list.d/mongodb-org-6.0.list && \
    apt-get update && apt-get install -y mongodb-org && \
    mkdir -p /data/db && chown -R mongodb:mongodb /data/db

# Setup MySQL
RUN mkdir -p /var/lib/mysql && chown -R mysql:mysql /var/lib/mysql
RUN echo "ALTER USER 'root'@'localhost' IDENTIFIED WITH 'mysql_native_password' BY 'root'; FLUSH PRIVILEGES;" > /tmp/init.sql

# Setup PostgreSQL
RUN mkdir -p /var/lib/postgresql/data && chown -R postgres:postgres /var/lib/postgresql/data

# ---------------------
# Supervisor config
# ---------------------
RUN mkdir -p /var/log/supervisor

COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

EXPOSE 27017 3306 5432

VOLUME ["/data/db", "/var/lib/mysql", "/var/lib/postgresql/data"]

CMD ["/usr/bin/supervisord"]





[supervisord]
nodaemon=true

[program:mongodb]
command=/usr/bin/mongod --dbpath /data/db
user=mongodb
autostart=true
autorestart=true
stderr_logfile=/var/log/supervisor/mongo.err.log
stdout_logfile=/var/log/supervisor/mongo.out.log

[program:mysql]
command=/usr/sbin/mysqld
user=mysql
autostart=true
autorestart=true
stderr_logfile=/var/log/supervisor/mysql.err.log
stdout_logfile=/var/log/supervisor/mysql.out.log

[program:postgres]
command=/usr/lib/postgresql/14/bin/postgres -D /var/lib/postgresql/data
user=postgres
autostart=true
autorestart=true
stderr_logfile=/var/log/supervisor/postgres.err.log
stdout_logfile=/var/log/supervisor/postgres.out.log













# Copy the local GPG key into the container
COPY server-8.0.asc /tmp/server-8.0.asc

RUN apt-get update && apt-get install -y wget gnupg curl sudo && \
    # Convert to gpg binary format and move to trusted location
    gpg --dearmor -o /etc/apt/trusted.gpg.d/mongodb-8.gpg /tmp/server-8.0.asc && \
    # Add MongoDB 8.0 repo
    echo "deb [arch=amd64,arm64 signed-by=/etc/apt/trusted.gpg.d/mongodb-8.gpg] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/8.0 multiverse" > /etc/apt/sources.list.d/mongodb-org-8.0.list && \
    apt-get update && \
    apt-get install -y mongodb-org && \
    apt-get clean






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


