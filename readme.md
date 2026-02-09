# Dokumentasi

## Step 1: Run start-local script
Create a folder named elasticsearch in the root directory of the project:
```bash
mkdir elasticsearch
```

To set up Elasticsearch and Kibana locally, run the start-local script in the command line:
```bash
curl -fsSL https://elastic.co/start-local | sh
```
After running the script, you can access Elastic services at the following endpoints:
- Elasticsearch: `http://localhost:9200`
- Kibana: `http://localhost:5601`

expect output:
```bash
🎉 Congrats, Elasticsearch and Kibana are installed and running in Docker!

🌐 Open your browser at http://localhost:5601

   Username: elastic
   Password: xxxxxx

🔌 Elasticsearch API endpoint: http://localhost:9200
🔑 API key: NC1TMjQ1c0JsWVZVTjdUYW45bVU6cxxxxxxxxxxxxx

Learn more at https://github.com/elastic/start-local
```

## Step 2: Check Docker Containers
To check if the Elasticsearch and Kibana containers are running, run the following command in the command line:
```bash
docker ps
```
expect output:
```bash
CONTAINER ID   IMAGE                                                 COMMAND                  CREATED         STATUS                   PORTS                                NAMES
f3cc51c7d385   docker.elastic.co/kibana/kibana:9.2.4                 "/bin/tini -- /usr/l…"   7 minutes ago   Up 5 minutes (healthy)   127.0.0.1:5601->5601/tcp             kibana-local-dev
6abe2a1944d8   docker.elastic.co/elasticsearch/elasticsearch:9.2.4   "/bin/tini -- /usr/l…"   7 minutes ago   Up 7 minutes (healthy)   127.0.0.1:9200->9200/tcp, 9300/tcp   es-local-dev
```
## Step 3: Update Docker Compose File
Add the following content to the docker-compose.yml file:
```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:${ES_LOCAL_VERSION}
    container_name: ${ES_LOCAL_CONTAINER_NAME}
    volumes:
      - dev-elasticsearch:/usr/share/elasticsearch/data
    ports:
      - 127.0.0.1:${ES_LOCAL_PORT}:9200
    environment:
      - discovery.type=single-node
      - ELASTIC_PASSWORD=${ES_LOCAL_PASSWORD}
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=false
      - xpack.license.self_generated.type=trial
      - xpack.ml.use_auto_machine_memory_percent=true
      - ES_JAVA_OPTS=${ES_LOCAL_JAVA_OPTS}
      - cluster.routing.allocation.disk.watermark.low=${ES_LOCAL_DISK_SPACE_REQUIRED}
      - cluster.routing.allocation.disk.watermark.high=${ES_LOCAL_DISK_SPACE_REQUIRED}
      - cluster.routing.allocation.disk.watermark.flood_stage=${ES_LOCAL_DISK_SPACE_REQUIRED}
    ulimits:
      memlock:
        soft: -1
        hard: -1
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl --output /dev/null --silent --head --fail -u elastic:${ES_LOCAL_PASSWORD} http://elasticsearch:9200",
        ]
      interval: 10s
      timeout: 10s
      retries: 30

  kibana_settings:
    depends_on:
      elasticsearch:
        condition: service_healthy
    image: docker.elastic.co/elasticsearch/elasticsearch:${ES_LOCAL_VERSION}
    container_name: ${KIBANA_LOCAL_SETTINGS_CONTAINER_NAME}
    restart: 'no'
    command: >
      bash -c '
        echo "Setup the kibana_system password";
        start_time=$$(date +%s);
        timeout=60;
        until curl -s -u "elastic:${ES_LOCAL_PASSWORD}" -X POST http://elasticsearch:9200/_security/user/kibana_system/_password -d "{\"password\":\"${KIBANA_LOCAL_PASSWORD}\"}" -H "Content-Type: application/json" | grep -q "^{}"; do
          if [ $$(($$(date +%s) - $$start_time)) -ge $$timeout ]; then
            echo "Error: Elasticsearch timeout";
            exit 1;
          fi;
          sleep 2;
        done;
      '

  kibana:
    depends_on:
      kibana_settings:
        condition: service_completed_successfully
    image: docker.elastic.co/kibana/kibana:${ES_LOCAL_VERSION}
    container_name: ${KIBANA_LOCAL_CONTAINER_NAME}
    volumes:
      - dev-kibana:/usr/share/kibana/data
      - ./config/telemetry.yml:/usr/share/kibana/config/telemetry.yml
    ports:
      - 127.0.0.1:${KIBANA_LOCAL_PORT}:5601
    environment:
      - SERVER_NAME=kibana
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=${KIBANA_LOCAL_PASSWORD}
      - XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY=${KIBANA_ENCRYPTION_KEY}
      - ELASTICSEARCH_PUBLICBASEURL=http://localhost:${ES_LOCAL_PORT}
      - XPACK_SPACES_DEFAULTSOLUTION=es
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s -I http://kibana:5601 | grep -q 'HTTP/1.1 302 Found'",
        ]
      interval: 10s
      timeout: 10s
      retries: 30
  
  filebeat:
    image: docker.elastic.co/beats/filebeat:${ES_LOCAL_VERSION}
    container_name: ${FILEBEAT_LOCAL_CONTAINER_NAME}
    user: root
    depends_on:
      elasticsearch:
        condition: service_healthy
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /root/apps/logs:/var/log/laravel:ro
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=elastic
      - ELASTICSEARCH_PASSWORD=${ES_LOCAL_PASSWORD}
      - ALLOW_LOG_FROM_CONTAINER_1=${ALLOW_LOG_FROM_CONTAINER_1}
      - ALLOW_LOG_FROM_CONTAINER_2=${ALLOW_LOG_FROM_CONTAINER_2}
      - ALLOW_LOG_FROM_CONTAINER_3=${ALLOW_LOG_FROM_CONTAINER_3}
    command:
      [
        "filebeat",
        "-e",
        "-c",
        "/usr/share/filebeat/filebeat.yml",
        "--strict.perms=false",
      ]

volumes:
  dev-elasticsearch:
  dev-kibana:
```


## Step 4: Make Filebeat Configuration File
Create a folder named filebeat in the root directory of the project:
```bash
mkdir filebeat
```
Create a filebeat.yml configuration file in the filebeat folder of the project.
Add the following content to the filebeat/filebeat.yml file:
```yaml
filebeat.inputs:
  - type: filestream
    id: filebeat-log
    paths:
      - "/var/lib/docker/containers/*/*.log"
    prospector.scanner.fingerprint.length: 128
    parsers:
      - container: ~

processors:
  - add_host_metadata: {}

  - add_docker_metadata:
      host: "unix:///var/run/docker.sock"

  - drop_event:
      when:
        not:
          or:
            - equals:
                container.labels.log_enabled: "true"
            - equals:
                container.name: "${ALLOW_LOG_FROM_CONTAINER_1}"
            - equals:
                container.name: "${ALLOW_LOG_FROM_CONTAINER_2}"
            - equals:
                container.name: "${ALLOW_LOG_FROM_CONTAINER_3}"

  - decode_json_fields:
      fields: ["message"]
      target: ""
      overwrite_keys: true
      add_error_key: true

  - add_fields:
      target: ""
      fields:
        service: "unknown-service"
      when:
        not:
          has_fields: ["service"]

  - drop_fields:
      fields: ["agent", "ecs", "input", "log", "host", "container.id"]
      ignore_missing: true

setup.template.name: "logs"
setup.template.pattern: "logs-*"
setup.template.enabled: false
setup.ilm.enabled: false

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  username: "elastic"
  password: "${ELASTICSEARCH_PASSWORD}"
  index: "logs-%{[service]}-%{+yyyy.MM.dd}"

logging.json: true
logging.metrics.enabled: false
```

## Step 5: Environment Configuration
Create a `.env` file in the root directory to manage configurations and secrets. This file is used by Docker Compose to set up the environment.

Example `.env` content:
```env
ES_LOCAL_VERSION=9.2.4
ES_LOCAL_CONTAINER_NAME=es-local-dev
ES_LOCAL_PASSWORD=ChangeMe123
ES_LOCAL_PORT=9200
ES_LOCAL_URL=http://localhost:${ES_LOCAL_PORT}
ES_LOCAL_DISK_SPACE_REQUIRED=1gb
ES_LOCAL_JAVA_OPTS="-Xms128m -Xmx2g"

KIBANA_LOCAL_CONTAINER_NAME=kibana-local-dev
KIBANA_LOCAL_SETTINGS_CONTAINER_NAME=kibana-local-settings
KIBANA_LOCAL_PORT=5601
KIBANA_LOCAL_PASSWORD=ChangeMe123
KIBANA_ENCRYPTION_KEY=YourEncryptionKeyHere...

FILEBEAT_LOCAL_CONTAINER_NAME=filebeat-local-dev

# Container Allowlist for Logging
ALLOW_LOG_FROM_CONTAINER_1=kibana-local-dev
ALLOW_LOG_FROM_CONTAINER_2=es-local-dev
ALLOW_LOG_FROM_CONTAINER_3=filebeat-local-dev
```

## Step 6: Usage Guide

### 1. Start the Stack
Run the following command to start all services:
```bash
docker-compose up -d
```

### 2. Verify Services
Check if containers are running:
```bash
docker ps
```
Ensure `es-local-dev`, `kibana-local-dev`, and `filebeat-local-dev` are up and healthy.

### 3. Access Kibana
1. Open [http://localhost:5601](http://localhost:5601).
2. Login with:
   - **Username**: `elastic`
   - **Password**: The value of `ES_LOCAL_PASSWORD` from your `.env` file.

### 4. Configure Data View in Kibana
To view logs:
1. Go to **Stack Management** > **Data Views**.
2. Create a new Data View.
3. Name: `logs-*` (matching the index pattern set in `filebeat.yml`).
4. Timestamp field: `timestamp`.
5. Go to **Discover** to see the logs.

### 5. Filter & Search Logs
Logs are indexed with the pattern `logs-%{[service]}-%{+yyyy.MM.dd}`.
- **Service Name Logic**:
  - If container has label `log_enabled=true`, it is processed.
  - If container name is in `ALLOW_LOG_FROM_CONTAINER_x`, it is processed.
  - If container name contains `laravel-frankenphp`, service name becomes `laravel-frankenphp`.
  - Default service name is `unknown-service` if not identified.

### 6. Troubleshooting
- **No logs in Kibana?**
  - Check Filebeat logs: `docker logs filebeat-local-dev`.
  - Verify indices in Elasticsearch:
    ```bash
    curl -u elastic:YOUR_PASSWORD http://localhost:9200/_cat/indices?v
    ```
  - Ensure your application container is running and printing to stdout/stderr.


