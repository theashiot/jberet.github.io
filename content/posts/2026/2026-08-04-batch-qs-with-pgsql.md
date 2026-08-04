---
layout: post
title:      "Migrating the WildFly Batch-Processing Quickstart to PostgreSQL with Production-Grade Security"
subtitle:   "A complete guide to database migration, credential management, TLS encryption, and authentication"
date:       2026-09-08
author:     Ashwin Mehendale
---

# Introduction

The WildFly [batch-processing quickstart](https://github.com/wildfly/quickstart/tree/main/batch-processing) provides an excellent introduction to Jakarta Batch applications. This post will guide you to:

1. Replace H2 with a containerized PostgreSQL database.
2. Store credentials securely using WildFly Elytron credential stores.
3. Encrypt database communication with TLS 1.3 (one-way TLS with server certificate).
4. Secure the web frontend with HTTPS.
5. Require user authentication to access the application.

The procedures work with the `provisioned-server` profile (default), defined in the `pom.xml` of the quickstart. Baremetal WildFly installations and deployment to OpenShift are not covered.

## Prerequisites

Before starting, ensure you have:

- **JDK 21** or later.
- **Maven 3.4** or later.
- **Podman** or **Docker** for running PostgreSQL.

## Part 1: PostgreSQL Container with TLS

Secure the communication between the PostgreSQL database and WildFly by enabling TLS.

### Step 1: Generate TLS Certificates

Create a self-signed certificate that the PostgreSQL database presents WildFly for authentication.

***NOTE***: For production environments, ensure you use only certificate authority (CA)-signed certificates.

Create a directory for certificates:

```bash
mkdir -p ~/postgres-certs
cd ~/postgres-certs
```
Generate the server private key and self-signed certificate:

```bash
# Generate private key
openssl genrsa -out server.key 2048

# Set restrictive permissions (PostgreSQL requirement)
chmod 600 server.key

# Generate self-signed certificate (valid for 365 days)
openssl req -new -x509 -key server.key -out server.crt -days 365 \
  -subj "/C=CA/ST=AB/L=Calgary/O=Organization/CN=localhost"

# Set permissions for certificate
chmod 644 server.crt
```

**Important**: The Common Name (CN) must match how WildFly connects to PostgreSQL. As we are using `localhost`, we set `CN=localhost`.

### Step 2: Map Certificate Ownership into the User Namespace 

Assign the ownership of the certificates to UID 999. This ownersip assignment is required when running rootless Podman.

```bash
# Run the chown inside the user namespace so 999 means "container postgres"
podman unshare chown 999:999 ~/postgres-certs/server.key ~/postgres-certs/server.crt
```

Confirm the result from inside the namespace:

```bash
podman unshare ls -n ~/postgres-certs/
```

Note that `server.key` and `server.crt` are now owned by the UID `999`.

Expected output:

```
-rw-r--r--. 1   0   0  352 ... pg_hba.conf
-rw-r--r--. 1   0   0  409 ... postgresql.conf
-rw-r--r--. 1 999 999 1294 ... server.crt
-rw-------. 1 999 999 1704 ... server.key
```

IMPORTANT:
- You can no longer read `server.key` as your normal user. Use `podman unshare cat ~/postgres-certs/server.key` if you need to inspect it.
- Re-run this step every time you regenerate the certificates. `openssl genrsa` creates a brand-new file owned by you, causes the earlier mentioned failure.


### Step 3: Create PostgreSQL Configuration Files

Create `postgresql.conf` to enable TLS:

```bash
cat > postgresql.conf << 'EOF'
# TLS/SSL Configuration
ssl = on
ssl_cert_file = '/var/lib/postgresql/server.crt'
ssl_key_file = '/var/lib/postgresql/server.key'
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'
ssl_prefer_server_ciphers = on
ssl_min_protocol_version = 'TLSv1.3'

# Connection Settings
listen_addresses = '*'
max_connections = 100

# Enable prepared transactions (required for XA if needed in future)
max_prepared_transactions = 100
EOF
```

Create `pg_hba.conf` to require SSL connections:

```bash
cat > pg_hba.conf << 'EOF'
# TYPE  DATABASE        USER            ADDRESS                 METHOD
# Require SSL for all connections
hostssl all             all             0.0.0.0/0               scram-sha-256
hostssl all             all             ::0/0                   scram-sha-256

# Local connections
local   all             all                                     trust
EOF
```

### Step 4: Start PostgreSQL Container with TLS

Pull the PostgreSQL image:

```bash
podman pull docker.io/library/postgres:latest
```

Start the container with TLS enabled:

```bash
podman run -d \
  --name postgres-jberet-secure \
  -e POSTGRES_USER=batch_user \
  -e POSTGRES_PASSWORD=SecurePass123! \
  -e POSTGRES_DB=batch_db \
  -v ~/postgres-certs/server.crt:/var/lib/postgresql/server.crt:z \
  -v ~/postgres-certs/server.key:/var/lib/postgresql/server.key:z \
  -v ~/postgres-certs/postgresql.conf:/var/lib/postgresql/postgresql.conf:z \
  -v ~/postgres-certs/pg_hba.conf:/var/lib/postgresql/pg_hba.conf:z \
  -p 5432:5432 \
  postgres:16 \
  -c config_file=/var/lib/postgresql/postgresql.conf \
  -c hba_file=/var/lib/postgresql/pg_hba.conf
```

### Step 5: Verify PostgreSQL TLS Configuration

Check that PostgreSQL is acceps SSL connections:

```bash
podman exec postgres-jberet-secure psql -U batch_user -d batch_db -c "SHOW ssl;"
```

Expected output:
```
 ssl 
-----
 on
(1 row)
```

Verify the TLS version:

```bash
podman exec -e PGPASSWORD='SecurePass123!' postgres-jberet-secure \
  psql "host=127.0.0.1 port=5432 user=batch_user dbname=batch_db sslmode=require" \
  -c "SELECT ssl, version AS tls_version, cipher
      FROM pg_stat_ssl JOIN pg_stat_activity USING (pid)
      WHERE pid = pg_backend_pid();"
```

Expected output:
```
 ssl | tls_version |         cipher         
-----+-------------+------------------------
 t   | TLSv1.3     | TLS_AES_256_GCM_SHA384
(1 row)
```

Confirm that unencrypted connections are refused:

```bash
podman exec -e PGPASSWORD='SecurePass123!' postgres-jberet-secure \
  psql "host=127.0.0.1 port=5432 user=batch_user dbname=batch_db sslmode=disable" -c "SELECT 1;"
```

Expected output:

```
psql: error: connection to server at "127.0.0.1", port 5432 failed: FATAL:  no pg_hba.conf entry
for host "127.0.0.1", user "batch_user", database "batch_db", no encryption
```

The connection failure confirms that unencrypted connections are refused.

## Part 2: WildFly Configuration for Provisioned Server

The `provisioned-server` profile packages a custom WildFly server using Maven.

<!-- What each step does should be described -->
### Step 1: Navigate to the Quickstart.

### Step 2: Update pom.xml Dependencies

**Change 1**: Update the manifest entry (line 201-202):

```xml
<manifestEntries>
    <!-- Replace H2 with PostgreSQL JDBC driver -->
    <Dependencies>org.postgresql.jdbc</Dependencies>
</manifestEntries>
```

**Change 2**: Update the `provisioned-server` profile add-ons (line 224-226) to provision the server with a PostgreSQL module and a management CLI:

```xml
<discover-provisioning-info>
    <version>${version.server}</version>
    <addOns>
        <addOn>wildfly-cli</addOn>
        <addOn>postgresql</addOn>
    </addOns>
</discover-provisioning-info>
```

**Change 3**: Add packaging scripts after `</discover-provisioning-info>` (after line 227) to configure WildFly to use PostgreSQL database for the `batch-jberet` subsystem:

```xml
<packaging-scripts>
    <packaging-script>
        <commands>
            <!-- Create credential store for database credentials -->
            <command>/subsystem=elytron/credential-store=batch-credentials:add(location=credentials/batch-store.jceks, relative-to=jboss.server.data.dir, credential-reference={clear-text=storePass123!}, create=true)</command>
            
            <!-- Add database password to credential store -->
            <command>/subsystem=elytron/credential-store=batch-credentials:add-alias(alias=db.password, secret-value=SecurePass123!)</command>
            
            <!-- Configure PostgreSQL datasource with credential reference and TLS -->
            <command>/subsystem=datasources/data-source=batch-processingDS:add( \
                jndi-name=java:jboss/datasources/batch-processingDS, \
                driver-name=postgresql, \
                connection-url="jdbc:postgresql://localhost:5432/batch_db?ssl=true&amp;sslmode=require", \
                user-name=batch_user, \
                credential-reference={store=batch-credentials, alias=db.password}, \
                enabled=true, \
                valid-connection-checker-class-name=org.jboss.jca.adapters.jdbc.extensions.postgres.PostgreSQLValidConnectionChecker, \
                exception-sorter-class-name=org.jboss.jca.adapters.jdbc.extensions.postgres.PostgreSQLExceptionSorter, \
                validate-on-match=true, \
                background-validation=false)</command>
            
            <!-- Configure JBeret JDBC job repository to use the same datasource -->
            <command>/subsystem=batch-jberet/jdbc-job-repository=JSR352_JobRepository:add(data-source=batch-processingDS)</command>
            
            <!-- Set as default job repository -->
            <command>/subsystem=batch-jberet/:write-attribute(name=default-job-repository,value=JSR352_JobRepository)</command>
        </commands>
    </packaging-script>
</packaging-scripts>
```

**Important Security Notes**:

1. **Credential Store**: The credential store itself is protected by a password (`storePass123!`). In production, this should come from an external source, such as HashiCorp vault, and should not be hardcoded.
2. **Credential Reference**: The datasource password is never written to `standalone.xml`. The `credential-reference={store=batch-credentials, alias=db.password}` attribute tells the datasource to fetch the password from the Elytron credential store at runtime. For information about Elytron credential store, see [Using Credential Stores to Replace Clear Text Passwords With WildFly](https://www.wildfly.org/guides/security-credential-store-for-passwords/).

### Step 2: Configure SSL Trust for PostgreSQL Certificate

To trust the PostgreSQL self-signed certificate, import it into a truststore that WildFly uses.

Create the truststore:

```bash
cd ~/postgres-certs

# Import PostgreSQL certificate into a JKS truststore
keytool -import -trustcacerts -alias postgres-server \
  -file server.crt \
  -keystore pg-truststore.jks \
  -storepass trustPass123! \
  -noprompt
```

Add an additional CLI command to the packaging scripts in `pom.xml`:

```xml
<packaging-scripts>
    <packaging-script>
        <commands>
            <!-- Previous commands... -->
            
            <!-- Configure SSL context for PostgreSQL certificate trust -->
            <command>/subsystem=elytron/key-store=postgresql-truststore:add(path=pg-truststore.jks, relative-to=jboss.server.config.dir, credential-reference={clear-text=trustPass123!}, type=JKS)</command>
            <command>/subsystem=elytron/trust-manager=postgresql-trust:add(key-store=postgresql-truststore)</command>
            <command>/subsystem=elytron/client-ssl-context=postgresql-ssl:add(trust-manager=postgresql-trust, protocols=["TLSv1.3"])</command>
        </commands>
    </packaging-script>
</packaging-scripts>
```

### Step 3: Remove Application-Level DataSource Definition

Open `<quickstart_home>/batch-processing/src/main/java/org/jboss/as/quickstarts/batch/model/Contact.java`.

**Remove lines 25-30** (the `@DataSourceDefinition` annotation) so that server-level datasource is used by both the application and the `batch-jberet` subsystem:

```java
@DataSourceDefinition(name="java:jboss/datasources/batch-processingDS",
        className="org.h2.jdbcx.JdbcDataSource",
        url="jdbc:h2:mem:batch-processing;DB_CLOSE_ON_EXIT=FALSE;DB_CLOSE_DELAY=-1",
        user="sa",
        password="sa"
)
```

### Step 4: Update Persistence Configuration

Open `<quickstart_home>/batch-processing/src/main/resources/META-INF/persistence.xml`.

Update the properties section to use PostgreSQL dialect:

```xml
<persistence-unit name="primary" transaction-type="JTA">
   <jta-data-source>java:jboss/datasources/batch-processingDS</jta-data-source>
   <properties>
      <!-- Properties for Hibernate with PostgreSQL -->
      <property name="hibernate.dialect" value="org.hibernate.dialect.PostgreSQLDialect" />
      <property name="hibernate.hbm2ddl.auto" value="create-drop" />
      <property name="hibernate.show_sql" value="false" />
   </properties>
</persistence-unit>

<!-- check for a better alternative to create-drop -->

### Step 5: Build and Test Provisioned Server

Build the provisioned server:

```bash
cd <quickstart_home>/batch-processing
mvn clean package
```

Copy the truststore so that WildFly tusts the certificate presented by PostgreSQL database:

```bash
cp ~/postgres-certs/pg-truststore.jks target/server/standalone/configuration/
```

Start the provisioned server:

```bash
target/server/bin/standalone.sh
```

Access the application:

```
http://localhost:8080/batch-processing/
```

Test the batch processing functionality:

1. Click "Generate a new file and start import job"
2. Click "Update jobs list" to verify the job completed
3. Check server logs for successful database operations

Verify TLS connection in PostgreSQL logs:

```bash
podman logs postgres-jberet-secure | grep SSL
```

You should see entries indicating SSL connections.

Run integration tests:

```bash
mvn verify -Pintegration-testing
```