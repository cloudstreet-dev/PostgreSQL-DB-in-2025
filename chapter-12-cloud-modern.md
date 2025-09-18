# Chapter 12: Cloud and Modern Deployments

*By Claude Code Opus 4.1*

PostgreSQL in the cloud isn't just about lifting and shifting—it's about leveraging cloud-native capabilities while avoiding vendor lock-in. This chapter navigates the modern deployment landscape from containers to serverless.

## Container Deployments

### Docker Best Practices

```dockerfile
# Optimized PostgreSQL Dockerfile
FROM postgres:16-alpine AS base

# Install additional extensions
RUN apk add --no-cache \
    postgresql16-contrib \
    postgresql16-pg_stat_statements \
    postgresql16-pgaudit

# Custom configuration
COPY postgresql.conf /etc/postgresql/postgresql.conf
COPY pg_hba.conf /etc/postgresql/pg_hba.conf

# Initialization scripts
COPY init-scripts/ /docker-entrypoint-initdb.d/

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD pg_isready -U postgres || exit 1

# Non-root user for security
USER postgres

CMD ["postgres", "-c", "config_file=/etc/postgresql/postgresql.conf"]
```

Docker Compose for development:

```yaml
version: '3.8'

services:
  postgres:
    build: .
    container_name: postgres_dev
    environment:
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-secret}
      POSTGRES_DB: ${DB_NAME:-myapp}
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=C"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./backups:/backups
    ports:
      - "${DB_PORT:-5432}:5432"
    networks:
      - app_network
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 512M

  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "8080:80"
    networks:
      - app_network
    depends_on:
      - postgres

volumes:
  postgres_data:
    driver: local

networks:
  app_network:
    driver: bridge
```

## Kubernetes Deployments

### Using CloudNativePG Operator

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: database

---
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-cluster
  namespace: database
spec:
  instances: 3

  postgresql:
    parameters:
      max_connections: "200"
      shared_buffers: "1GB"
      effective_cache_size: "3GB"
      maintenance_work_mem: "256MB"
      checkpoint_completion_target: "0.9"
      wal_buffers: "16MB"
      default_statistics_target: "100"
      random_page_cost: "1.1"
      effective_io_concurrency: "200"
      work_mem: "4MB"
      min_wal_size: "1GB"
      max_wal_size: "4GB"

  bootstrap:
    initdb:
      database: app
      owner: app
      secret:
        name: app-user
      dataChecksums: true
      encoding: UTF8
      localeCollate: C
      localeCType: C

  primaryUpdateStrategy: unsupervised

  nodeMaintenanceWindow:
    inProgress: false
    reusePVC: true

  monitoring:
    enabled: true
    customQueriesConfigMap:
      - name: custom-queries
        key: queries.yaml

  backup:
    retentionPolicy: "30d"
    barmanObjectStore:
      destinationPath: "s3://postgres-backups/prod"
      s3Credentials:
        accessKeyId:
          name: backup-credentials
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: backup-credentials
          key: SECRET_ACCESS_KEY
      wal:
        retention: "7d"
        maxParallel: 8
      data:
        compression: gzip
        encryption: AES256
        immediateCheckpoint: false
        jobs: 2

  resources:
    requests:
      memory: "4Gi"
      cpu: "2"
    limits:
      memory: "8Gi"
      cpu: "4"

  affinity:
    enablePodAntiAffinity: true
    topologyKey: kubernetes.io/hostname
    podAntiAffinityType: preferred

  storage:
    size: 100Gi
    storageClass: fast-ssd
```

### Stateful Sets (Manual Approach)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
  namespace: database
spec:
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: postgres
    role: primary

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: database
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        role: primary
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: POSTGRES_REPLICATION_MODE
          value: master
        - name: POSTGRES_REPLICATION_USER
          value: replicator
        - name: POSTGRES_REPLICATION_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: replication-password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        - name: postgres-config
          mountPath: /etc/postgresql
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 5
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
```

## AWS RDS PostgreSQL

### Terraform Configuration

```hcl
provider "aws" {
  region = var.aws_region
}

# RDS Subnet Group
resource "aws_db_subnet_group" "postgres" {
  name       = "${var.environment}-postgres-subnet-group"
  subnet_ids = var.private_subnet_ids

  tags = {
    Name        = "${var.environment}-postgres-subnet-group"
    Environment = var.environment
  }
}

# Security Group
resource "aws_security_group" "postgres" {
  name_prefix = "${var.environment}-postgres-"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = var.app_security_groups
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  lifecycle {
    create_before_destroy = true
  }
}

# Parameter Group
resource "aws_db_parameter_group" "postgres" {
  family = "postgres16"
  name   = "${var.environment}-postgres-params"

  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements,pgaudit,pg_cron"
  }

  parameter {
    name  = "log_statement"
    value = "all"
  }

  parameter {
    name  = "log_min_duration_statement"
    value = "1000"  # Log queries over 1 second
  }

  parameter {
    name  = "autovacuum_max_workers"
    value = "4"
  }

  parameter {
    name  = "max_connections"
    value = "200"
  }
}

# RDS Instance
resource "aws_db_instance" "postgres" {
  identifier = "${var.environment}-postgres"

  engine         = "postgres"
  engine_version = "16.1"
  instance_class = var.instance_class

  allocated_storage     = var.allocated_storage
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id           = var.kms_key_id

  db_name  = var.database_name
  username = var.master_username
  password = var.master_password

  vpc_security_group_ids = [aws_security_group.postgres.id]
  db_subnet_group_name   = aws_db_subnet_group.postgres.name
  parameter_group_name   = aws_db_parameter_group.postgres.name

  backup_retention_period = 30
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"

  enabled_cloudwatch_logs_exports = [
    "postgresql"
  ]

  performance_insights_enabled    = true
  performance_insights_retention_period = 7

  multi_az               = var.multi_az
  publicly_accessible    = false
  deletion_protection    = var.environment == "production"
  skip_final_snapshot    = var.environment != "production"
  final_snapshot_identifier = "${var.environment}-postgres-final-${timestamp()}"

  tags = {
    Name        = "${var.environment}-postgres"
    Environment = var.environment
  }
}

# Read Replica
resource "aws_db_instance" "postgres_replica" {
  count = var.create_read_replica ? 1 : 0

  identifier             = "${var.environment}-postgres-replica"
  replicate_source_db    = aws_db_instance.postgres.identifier
  instance_class         = var.replica_instance_class

  publicly_accessible = false
  auto_minor_version_upgrade = false

  performance_insights_enabled = true

  tags = {
    Name        = "${var.environment}-postgres-replica"
    Environment = var.environment
  }
}

# Outputs
output "endpoint" {
  value = aws_db_instance.postgres.endpoint
}

output "replica_endpoint" {
  value = var.create_read_replica ? aws_db_instance.postgres_replica[0].endpoint : null
}
```

### RDS Proxy for Connection Pooling

```hcl
resource "aws_db_proxy" "postgres" {
  name                   = "${var.environment}-postgres-proxy"
  engine_family          = "POSTGRESQL"
  auth {
    auth_scheme = "SECRETS"
    secret_arn  = aws_secretsmanager_secret.db_credentials.arn
  }

  role_arn               = aws_iam_role.proxy.arn
  vpc_security_group_ids = [aws_security_group.postgres.id]
  vpc_subnet_ids         = var.private_subnet_ids

  max_connections_percent        = 100
  max_idle_connections_percent   = 50
  connection_borrow_timeout      = 120
  session_pinning_filters        = ["EXCLUDE_VARIABLE_SETS"]

  target {
    db_instance_identifier = aws_db_instance.postgres.identifier
  }

  tags = {
    Name        = "${var.environment}-postgres-proxy"
    Environment = var.environment
  }
}
```

## Google Cloud SQL

```hcl
provider "google" {
  project = var.gcp_project_id
  region  = var.gcp_region
}

resource "google_sql_database_instance" "postgres" {
  name             = "${var.environment}-postgres"
  database_version = "POSTGRES_16"
  region           = var.gcp_region

  settings {
    tier              = var.instance_tier
    availability_type = var.environment == "production" ? "REGIONAL" : "ZONAL"

    disk_size         = var.disk_size
    disk_type         = "PD_SSD"
    disk_autoresize   = true

    backup_configuration {
      enabled                        = true
      start_time                     = "03:00"
      point_in_time_recovery_enabled = true
      transaction_log_retention_days = 7
      backup_retention_settings {
        retained_backups = 30
      }
    }

    ip_configuration {
      ipv4_enabled    = false
      private_network = var.vpc_network
      require_ssl     = true
    }

    database_flags {
      name  = "max_connections"
      value = "200"
    }

    database_flags {
      name  = "shared_buffers"
      value = "1073741824"  # 1GB in bytes
    }

    insights_config {
      query_insights_enabled  = true
      query_string_length     = 1024
      record_application_tags = true
      record_client_address   = true
    }

    maintenance_window {
      day          = 7
      hour         = 4
      update_track = "stable"
    }
  }

  deletion_protection = var.environment == "production"
}

# Read Replica
resource "google_sql_database_instance" "read_replica" {
  count = var.create_read_replica ? 1 : 0

  name                 = "${var.environment}-postgres-replica"
  master_instance_name = google_sql_database_instance.postgres.name
  region               = var.gcp_replica_region
  database_version     = "POSTGRES_16"

  settings {
    tier = var.replica_tier

    ip_configuration {
      ipv4_enabled    = false
      private_network = var.vpc_network
      require_ssl     = true
    }
  }
}
```

## Azure Database for PostgreSQL

```hcl
provider "azurerm" {
  features {}
}

resource "azurerm_postgresql_flexible_server" "postgres" {
  name                = "${var.environment}-postgres"
  resource_group_name = var.resource_group_name
  location            = var.location

  administrator_login    = var.admin_username
  administrator_password = var.admin_password

  sku_name   = var.sku_name
  version    = "16"
  storage_mb = var.storage_mb

  backup_retention_days        = 30
  geo_redundant_backup_enabled = var.environment == "production"
  auto_grow_enabled            = true

  high_availability {
    mode                      = "ZoneRedundant"
    standby_availability_zone = "2"
  }

  maintenance_window {
    day_of_week  = 0
    start_hour   = 4
    start_minute = 0
  }

  tags = {
    Environment = var.environment
  }
}

resource "azurerm_postgresql_flexible_server_configuration" "example" {
  name      = "shared_preload_libraries"
  server_id = azurerm_postgresql_flexible_server.postgres.id
  value     = "pg_stat_statements,pgaudit"
}
```

## Serverless PostgreSQL

### AWS Aurora Serverless v2

```hcl
resource "aws_rds_cluster" "aurora_postgresql" {
  cluster_identifier = "${var.environment}-aurora-postgres"
  engine             = "aurora-postgresql"
  engine_version     = "15.4"
  engine_mode        = "provisioned"

  database_name   = var.database_name
  master_username = var.master_username
  master_password = var.master_password

  serverlessv2_scaling_configuration {
    max_capacity = var.max_acu
    min_capacity = var.min_acu
  }

  backup_retention_period = 30
  preferred_backup_window = "03:00-04:00"

  enabled_cloudwatch_logs_exports = ["postgresql"]

  vpc_security_group_ids = [aws_security_group.postgres.id]
  db_subnet_group_name   = aws_db_subnet_group.postgres.name

  skip_final_snapshot = var.environment != "production"
  deletion_protection = var.environment == "production"

  tags = {
    Name        = "${var.environment}-aurora-postgres"
    Environment = var.environment
  }
}

resource "aws_rds_cluster_instance" "aurora_instance" {
  count              = var.instance_count
  identifier         = "${var.environment}-aurora-instance-${count.index}"
  cluster_identifier = aws_rds_cluster.aurora_postgresql.id
  instance_class     = "db.serverless"
  engine             = aws_rds_cluster.aurora_postgresql.engine
  engine_version     = aws_rds_cluster.aurora_postgresql.engine_version

  performance_insights_enabled = true

  monitoring_role_arn = aws_iam_role.enhanced_monitoring.arn
  monitoring_interval = 60
}
```

### Neon Serverless PostgreSQL

```typescript
// Using Neon with Vercel Edge Functions
import { neon } from '@neondatabase/serverless';
import { drizzle } from 'drizzle-orm/neon-http';

const sql = neon(process.env.DATABASE_URL!);
const db = drizzle(sql);

export async function GET(request: Request) {
  // Connection pooling handled automatically
  const users = await db.select().from(usersTable);

  return new Response(JSON.stringify(users), {
    headers: { 'Content-Type': 'application/json' },
  });
}

// Branch for preview deployments
export async function createPreviewBranch(pullRequestId: string) {
  const response = await fetch('https://console.neon.tech/api/v2/projects/YOUR_PROJECT/branches', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.NEON_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      branch: {
        name: `pr-${pullRequestId}`,
        parent_id: 'main',
      },
    }),
  });

  return response.json();
}
```

## Edge Deployments

### Cloudflare D1 (SQLite-compatible)

```typescript
// wrangler.toml
// [[d1_databases]]
// binding = "DB"
// database_name = "my-database"
// database_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

export interface Env {
  DB: D1Database;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // D1 provides PostgreSQL-like syntax
    const { results } = await env.DB.prepare(
      "SELECT * FROM users WHERE email = ?1"
    )
    .bind(email)
    .all();

    return Response.json(results);
  },
};
```

### Fly.io PostgreSQL

```toml
# fly.toml
app = "my-postgres-app"

[env]
  PRIMARY_REGION = "iad"

[[services]]
  internal_port = 5432
  protocol = "tcp"

  [[services.ports]]
    port = 5432

[mounts]
  source = "postgres_data"
  destination = "/var/lib/postgresql/data"

# Create and deploy
# fly postgres create
# fly postgres attach my-postgres-app
```

## Observability and Monitoring

### Prometheus + Grafana

```yaml
# PostgreSQL Exporter
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-exporter-config
data:
  queries.yaml: |
    pg_replication:
      query: |
        SELECT
          client_addr::text as client,
          state,
          sync_state,
          pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) as lag_bytes
        FROM pg_stat_replication
      metrics:
        - client:
            usage: "LABEL"
            description: "Replica IP"
        - state:
            usage: "LABEL"
        - sync_state:
            usage: "LABEL"
        - lag_bytes:
            usage: "GAUGE"
            description: "Replication lag in bytes"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres-exporter
  template:
    metadata:
      labels:
        app: postgres-exporter
    spec:
      containers:
      - name: postgres-exporter
        image: prometheuscommunity/postgres-exporter:latest
        env:
        - name: DATA_SOURCE_NAME
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: exporter-dsn
        - name: PG_EXPORTER_EXTEND_QUERY_PATH
          value: /etc/config/queries.yaml
        volumeMounts:
        - name: config
          mountPath: /etc/config
        ports:
        - containerPort: 9187
      volumes:
      - name: config
        configMap:
          name: postgres-exporter-config
```

### Application Performance Monitoring

```python
# Using OpenTelemetry with PostgreSQL
from opentelemetry import trace
from opentelemetry.instrumentation.psycopg2 import Psycopg2Instrumentor
import psycopg2

# Auto-instrument psycopg2
Psycopg2Instrumentor().instrument()

# Traces will automatically include:
# - Query execution time
# - Query text (sanitized)
# - Connection info
# - Error tracking

tracer = trace.get_tracer(__name__)

def get_user(user_id):
    with tracer.start_as_current_span("get_user") as span:
        span.set_attribute("user.id", user_id)

        conn = psycopg2.connect(DATABASE_URL)
        with conn.cursor() as cur:
            cur.execute("SELECT * FROM users WHERE id = %s", (user_id,))
            return cur.fetchone()
```

## Cost Optimization

### Right-Sizing Instances

```sql
-- Analyze actual resource usage
CREATE OR REPLACE FUNCTION analyze_instance_sizing()
RETURNS TABLE(
    metric text,
    current_value numeric,
    recommended_action text
) AS $$
BEGIN
    RETURN QUERY
    -- Check CPU usage
    SELECT
        'CPU Cores Used',
        (SELECT count(*) FROM pg_stat_activity WHERE state = 'active')::numeric,
        CASE
            WHEN (SELECT count(*) FROM pg_stat_activity WHERE state = 'active') < 2 THEN
                'Consider smaller instance'
            WHEN (SELECT count(*) FROM pg_stat_activity WHERE state = 'active') >
                 current_setting('max_connections')::int * 0.8 THEN
                'Consider larger instance'
            ELSE 'Size appropriate'
        END

    UNION ALL

    -- Check memory usage
    SELECT
        'Memory Used (GB)',
        (pg_database_size(current_database()) / 1024 / 1024 / 1024.0)::numeric,
        CASE
            WHEN pg_database_size(current_database()) < 1073741824 THEN -- < 1GB
                'Consider smaller instance'
            ELSE 'Check shared_buffers setting'
        END

    UNION ALL

    -- Check storage IOPS
    SELECT
        'Storage IOPS',
        (SELECT sum(blks_read + blks_hit) FROM pg_stat_database)::numeric,
        CASE
            WHEN (SELECT sum(blks_read) FROM pg_stat_database) >
                 (SELECT sum(blks_hit) FROM pg_stat_database) THEN
                'Consider faster storage or more memory'
            ELSE 'IOPS acceptable'
        END;
END;
$$ LANGUAGE plpgsql;
```

### Reserved Instances and Savings Plans

```python
# Calculate potential savings
def calculate_ri_savings(
    instance_type: str,
    region: str,
    hours_per_month: int = 730
):
    on_demand_rates = {
        "db.r6g.large": 0.258,
        "db.r6g.xlarge": 0.516,
        "db.r6g.2xlarge": 1.032,
    }

    ri_rates = {
        "db.r6g.large": 0.164,
        "db.r6g.xlarge": 0.328,
        "db.r6g.2xlarge": 0.656,
    }

    on_demand_cost = on_demand_rates[instance_type] * hours_per_month
    ri_cost = ri_rates[instance_type] * hours_per_month
    savings = on_demand_cost - ri_cost
    savings_percent = (savings / on_demand_cost) * 100

    return {
        "on_demand_monthly": on_demand_cost,
        "ri_monthly": ri_cost,
        "monthly_savings": savings,
        "savings_percent": savings_percent,
        "annual_savings": savings * 12
    }
```

## Disaster Recovery in the Cloud

```yaml
# Multi-region DR setup with GitOps
apiVersion: v1
kind: ConfigMap
metadata:
  name: dr-config
data:
  dr-runbook.sh: |
    #!/bin/bash
    set -e

    # Function to promote DR region
    promote_dr_region() {
      echo "Promoting DR region to primary..."

      # Update DNS
      aws route53 change-resource-record-sets \
        --hosted-zone-id $ZONE_ID \
        --change-batch file://dns-failover.json

      # Promote read replica
      aws rds promote-read-replica \
        --db-instance-identifier $DR_INSTANCE_ID

      # Update application configuration
      kubectl set env deployment/app \
        DATABASE_URL=$DR_DATABASE_URL \
        -n production

      # Verify health
      check_health $DR_ENDPOINT
    }

    # Function to check endpoint health
    check_health() {
      local endpoint=$1
      for i in {1..10}; do
        if pg_isready -h $endpoint; then
          echo "Endpoint $endpoint is healthy"
          return 0
        fi
        sleep 10
      done
      echo "Endpoint $endpoint is not responding"
      return 1
    }

    # Main DR procedure
    main() {
      echo "Starting DR procedure at $(date)"

      # Check primary health
      if ! check_health $PRIMARY_ENDPOINT; then
        echo "Primary is down, initiating failover"
        promote_dr_region

        # Send notifications
        send_notification "DR Activated" \
          "Primary region failed. DR region is now active."
      else
        echo "Primary is healthy, no action needed"
      fi
    }

    main "$@"
```

## GitOps for Database Management

```yaml
# ArgoCD Application for database migrations
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: database-migrations
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourorg/database
    targetRevision: main
    path: migrations
  destination:
    server: https://kubernetes.default.svc
    namespace: database
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

## Exercise: Cloud Migration

1. **Containerize**: Package your PostgreSQL setup in Docker
2. **Orchestrate**: Deploy to Kubernetes with high availability
3. **Cloudify**: Migrate to a managed cloud service
4. **Monitor**: Set up comprehensive observability
5. **Optimize**: Implement cost optimization strategies
6. **DR Planning**: Create and test disaster recovery procedures

## The Cloud Trade-offs

Cloud PostgreSQL isn't always better:

**Pros:**
- Managed maintenance and backups
- Automatic failover and HA
- Elastic scaling
- Pay-as-you-go pricing
- Global distribution

**Cons:**
- Vendor lock-in risk
- Less control over configuration
- Potential for surprise costs
- Network latency
- Compliance complexities

Choose cloud when the operational benefits outweigh the loss of control. Choose self-managed when you need absolute control or have specific compliance requirements.

## Summary

We've covered the modern PostgreSQL deployment landscape:
- Container deployments with Docker and Kubernetes
- Major cloud providers (AWS, GCP, Azure)
- Serverless and edge deployments
- Comprehensive monitoring and observability
- Cost optimization strategies
- Disaster recovery planning
- GitOps for database management

The cloud has transformed PostgreSQL from a database you run to a service you consume. Whether that's good or bad depends entirely on your needs, skills, and budget.

Congratulations! You've completed the main journey through PostgreSQL. The appendices that follow provide quick references, exercise solutions, and troubleshooting guides for your ongoing PostgreSQL adventures.