# Alfresco DR setup
## On DC create following files in ./volumes/config/postgres
####pg_hba.conf
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local socket connections
local   all             all                                     trust

# Localhost
host    all             all             127.0.0.1/32            trust
host    all             all             ::1/128                 trust

# --- REPLICATION (DR streaming) ---
# DR server (10.128.0.3) connects to DC primary as user "replicator"
host    replication     replicator      10.128.0.3/32           scram-sha-256

# Allow Alfresco app connections from docker network (host/VPC subnet)
host    alfresco        alfresco        10.128.0.0/16           scram-sha-256
host    all             alfresco        10.128.0.0/16           scram-sha-256

# --- DOCKER BRIDGE / COMPOSE NETWORK (current: 172.18.x.x seen in logs) ---
host    alfresco        alfresco        172.18.0.0/16           scram-sha-256
host    all             alfresco        172.18.0.0/16           scram-sha-256

# --- NORMAL DB ACCESS ---
# Allow app connections (tighten subnet if you know docker subnet)
host    alfresco        alfresco        172.31.0.0/24           scram-sha-256

# If you also want admin connections from the same subnet:
host    all             all             172.31.0.0/24           scram-sha-256

# (Optional but safe) Reject everything else explicitly
# host  all             all             0.0.0.0/0               reject
```
####postgresql.conf
```
listen_addresses = '*'
port = 5432

# Replication basics
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10

# Helpful to avoid WAL being removed too fast (tune as per storage)
wal_keep_size = 1024MB

# Standby can use this too (harmless on primary)
hot_standby = on

# Optional logging
log_min_messages = LOG
[root@dc1 postgres]#
```
create init directory and create file
####01-replication.sql
```
DO $$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname='replicator') THEN
    CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'StrongPass@123';
  END IF;
END $$;

DO $$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_replication_slots WHERE slot_name='dr_slot') THEN
    PERFORM pg_create_physical_replication_slot('dr_slot');
  END IF;
END $$;
```
In Config/postgres look as follow
```
├── init
│   └── 01-replication.sql
├── pg_hba.conf
└── postgresql.conf

1 directory, 3 files
```

The Docker-compose.yaml looks as follow

```
services:
  postgres:
    image: postgres:16.5
    restart: always
    environment:
      POSTGRES_PASSWORD: alfresco
      POSTGRES_USER: alfresco
      POSTGRES_DB: alfresco

    command: >
      postgres
      -c config_file=/etc/postgresql/postgresql.conf
      -c hba_file=/etc/postgresql/pg_hba.conf
      -c max_connections=300
      -c log_min_messages=LOG

    ports:
      - "5432:5432"

    volumes:
      - ./volumes/data/postgres-data:/var/lib/postgresql/data:Z
      - ./volumes/logs/postgres:/var/log/postgresql:Z
      - ./volumes/config/postgres/postgresql.conf:/etc/postgresql/postgresql.conf:Z,                                                                                                                        ro
      - ./volumes/config/postgres/pg_hba.conf:/etc/postgresql/pg_hba.conf:Z,ro
      - ./volumes/config/postgres/init:/docker-entrypoint-initdb.d:Z,ro

    networks:
      - internal

    healthcheck:
     # test: ["CMD-SHELL", "pg_isready -d $POSTGRES_DB -U $POSTGRES_USER"]
      test: ["CMD-SHELL", "pg_isready -h 127.0.0.1 -p 5432 -U alfresco -d postgres"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 10s

####################################################################################                                                                                                                        #####################
  activemq:
    image: alfresco/alfresco-activemq:5.18-jre17-rockylinux8
    restart: always
    ports:
      - "8161:8161"   # Web Console
      - "5672:5672"   # AMQP
      - "61616:61616" # OpenWire
      - "61613:61613" # STOMP
    networks:
      - internal
    healthcheck:
      test: ["CMD-SHELL", "/opt/activemq/bin/activemq query --objname 'type=Broker,b                                                                                                                        rokerName=*,service=Health' | grep Good"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 20s
####################################################################################                                                                                                                        #####################

  solr6:
    image: docker.io/alfresco/alfresco-search-services:2.0.16
    environment:
      SOLR_ALFRESCO_HOST: "alfresco"
      SOLR_ALFRESCO_PORT: "8080"
      SOLR_SOLR_HOST: "solr6"
      SOLR_SOLR_PORT: "8983"
      SOLR_CREATE_ALFRESCO_DEFAULTS: "alfresco,archive"
      ALFRESCO_SECURE_COMMS: "secret"
      JAVA_TOOL_OPTIONS: "-Dalfresco.secureComms.secret=secret"
      NO_PROXY: "alfresco,solr6,postgres,activemq,localhost,127.0.0.1"
      no_proxy: "alfresco,solr6,postgres,activemq,localhost,127.0.0.1"
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "awk 'NR>1{split($2,a,\":\"); p=toupper(a[2]); s=toupper($4); if(p==\"2317                                                                                                                        \" && s==\"0A\"){f=1}} END{exit(f?0:1)}' /proc/net/tcp /proc/net/tcp6"
        ]
      interval: 20s
      timeout: 5s
      retries: 30
      start_period: 60s
    networks:
      - internal
####################################################################################                                                                                                                        #########################

  alfresco:
    image: docker.io/alfresco/alfresco-content-repository-community:25.2.0
    restart: always
    ports:
      - "8080:8080"
    environment:
      JAVA_TOOL_OPTIONS: >-
        -Dencryption.keystore.type=JCEKS
        -Dencryption.cipherAlgorithm=DESede/CBC/PKCS5Padding
        -Dencryption.keyAlgorithm=DESede
        -Dencryption.keystore.location=/usr/local/tomcat/shared/classes/alfresco/ext                                                                                                                        ension/keystore/keystore
        -Dmetadata-keystore.password=mp6yc0UD9e
        -Dmetadata-keystore.aliases=metadata
        -Dmetadata-keystore.metadata.password=oKIWzVdEdA
        -Dmetadata-keystore.metadata.algorithm=DESede
      JAVA_OPTS: >-
        -Ddb.driver=org.postgresql.Driver
        -Ddb.username=alfresco
        -Ddb.password=alfresco
        -Ddb.url=jdbc:postgresql://postgres:5432/alfresco
        -Dsolr.host=solr6
        -Dsolr.port=8983
        -Dsolr.http.connection.timeout=1000
        -Dsolr.secureComms=secret
        -Dsolr.sharedSecret=secret
        -Dsolr.base.url=/solr
        -Dindex.subsystem.name=solr6
        -Dshare.host=localhost
        -Dshare.port=8080
        -Dalfresco.host=localhost
        -Dalfresco.port=8080
        -Dcsrf.filter.enabled=false
        -Daos.baseUrlOverwrite=http://localhost:8080/alfresco/aos
        -Dmessaging.broker.url="failover:(nio://activemq:61616)?timeout=3000&jms.use                                                                                                                        Compression=true"
        -Ddeployment.method=DOCKER_COMPOSE
        -DlocalTransform.core-aio.url=http://transform-core-aio:8090/
        -XX:MinRAMPercentage=50
        -XX:MaxRAMPercentage=80
    volumes:
      - ./volumes/data/alf-repo-data:/usr/local/tomcat/alf_data:Z
      - ./volumes/logs/alfresco:/usr/local/tomcat/logs:Z
      - ./volumes/config/alfresco-global.properties:/usr/local/tomcat/shared/classes                                                                                                                        /alfresco-global.properties:Z
      # - ./volumes/config/keystore:/usr/local/tomcat/shared/classes/alfresco/extens                                                                                                                        ion/keystore:Z
    networks:
      - internal
    depends_on:
      postgres:
        condition: service_healthy
      solr6:
        condition: service_healthy
      activemq:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/alfresco/api/-default-/publ                                                                                                                        ic/alfresco/versions/1/probes/-ready-"]
      interval: 30s
      timeout: 5s
      retries: 30
      start_period: 120s
####################################################################################                                                                                                                        #################################################

  transform-core-aio:
    image: alfresco/alfresco-transform-core-aio:5.2.1
    restart: always
    environment:
      JAVA_OPTS: "-XX:MinRAMPercentage=50 -XX:MaxRAMPercentage=80"
    ports:
      - "8090:8090"
    networks:
      - internal
    depends_on:
      activemq:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8090/ready"]
      interval: 20s
      timeout: 5s
      retries: 20
      start_period: 20s
####################################################################################                                                                                                                        ##################################################

  # ---- NEW SERVICE ----
  ocr-transformer:
    image: ketan32/karnataka-bank:ocr-app
    restart: always
    mem_limit: "512m"
    environment:
      JAVA_OPTS: >-
        -XX:MinRAMPercentage=50
        -XX:MaxRAMPercentage=80
    ports:
      - "8091:8090"
    networks:
      - internal
    healthcheck:
      test:
        [
        "CMD-SHELL",
        "awk 'NR>1{split($2,a,\":\"); p=toupper(a[2]); s=toupper($4); if(p==\"1F9A\"                                                                                                                         && s==\"0A\"){f=1}} END{exit(f?0:1)}' /proc/net/tcp /proc/net/tcp6"
        ]
      interval: 20s
      timeout: 5s
      retries: 10
      start_period: 20s
  # ---------------------###########################################################                                                                                                                        ####################################################

  share:
    image: docker.io/alfresco/alfresco-share:25.2.0
    restart: always
    ports:
      - "8081:8080"
    environment:
      CSRF_FILTER_ORIGIN: "http://localhost:8080"
      CSRF_FILTER_REFERER: "http://localhost:8080/share/.*"
      REPO_HOST: "alfresco"
      REPO_PORT: "8080"
      JAVA_OPTS: "-XX:MinRAMPercentage=50 -XX:MaxRAMPercentage=80 -Dalfresco.host=lo                                                                                                                        calhost -Dalfresco.port=8080 -Dalfresco.context=alfresco -Dalfresco.protocol=http"
    volumes:
      - ./volumes/logs/share:/usr/local/tomcat/logs:Z
      - ./volumes/config/ext-share-config-custom.xml:/usr/local/tomcat/shared/classe                                                                                                                        s/alfresco/web-extension/ext-share-config-custom.xml:Z
    networks:
      - internal
    depends_on:
      alfresco:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/share"]
      interval: 20s
      timeout: 5s
      retries: 20
      start_period: 60s
####################################################################################                                                                                                                        #####################################################

  content-app:
    image: alfresco/alfresco-content-app:7.1.0
    restart: always
    environment:
      APP_BASE_SHARE_URL: "http://alfresco:8080/share/#/preview/s"
    ports:
      - "8070:8080"
    networks:
      - internal
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/"]
      interval: 20s
      timeout: 5s
      retries: 10
      start_period: 10s
####################################################################################                                                                                                                        #################################################

  proxy:
    image: nginx:stable-alpine
    restart: always
    depends_on:
      content-app:
        condition: service_healthy
      share:
        condition: service_healthy
    volumes:
      - ./volumes/config/nginx:/etc/nginx/conf.d:Z
      - ./volumes/logs/nginx:/var/log/nginx:Z
    ports:
      - "9090:80"
    networks:
      - internal
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1/health >/dev/null 2>&1 || exit                                                                                                                         1"]
      interval: 20s
      timeout: 5s
      retries: 10
      start_period: 10s

volumes:
  solr-data:

networks:
  internal:
```
