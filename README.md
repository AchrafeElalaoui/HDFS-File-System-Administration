# Lab 2 : HDFS File System Administration

## Objectives

- Start and validate a mini HDFS + YARN cluster under Docker.
- Practice HDFS administration tasks: users, permissions, ACLs, snapshots, quotas, replication, and integrity.
- Discover basic monitoring through the NameNode and ResourceManager web interfaces.

## 1) Launching the Cluster

From `lab2-hdfs/`:

```bash
docker compose up -d
docker compose ps
```

Verify that the following services are running: `namenode`, `datanode`, `resourcemanager`, and `nodemanager`.

**Snapshots**

![docker compose up -d](snapshots/docker-compose-up-d.png)

![docker compose ps](snapshots/docker-compose-ps.png)

## 2) Quick Checks via Web UI

- NameNode UI (HDFS): http://localhost:9870  
  → Check: number of live DataNodes, capacity, and block status.
- YARN ResourceManager UI: http://localhost:8088  
  → Check: NodeManager is “HEALTHY.”

Take screenshots as evidence.

**Snapshots**

![NameNode UI](snapshots/localhost-9870.png)

![ResourceManager UI](snapshots/localhost-8088.png)

## 3) Admin Shell in the NameNode

All the following HDFS commands are executed from inside the NameNode container:

```bash
docker exec -it namenode bash
# Then inside the container:
hdfs dfs -ls /
hdfs dfsadmin -report
```

**Snapshots**

![hdfs dfsadmin -report](snapshots/hdfs-dfsadmin-report.png)

## 4) User Tree and Permissions

Create a “cours” directory and simulate two logical users (without OS accounts):

```bash
hdfs dfs -mkdir -p /cours/tp1
hdfs dfs -mkdir -p /user/student1 /user/student2
hdfs dfs -chmod 755 /user /cours
hdfs dfs -chmod 700 /user/student1 /user/student2
```

Create a small sample dataset:

```bash
cat > /tmp/sales.csv << 'EOF'
id_client,date,total
101,2025-09-01,120.50
102,2025-09-01,45.00
101,2025-09-02,33.20
103,2025-09-02,250.00
EOF

hdfs dfs -put -f /tmp/sales.csv /cours/tp1/
hdfs dfs -ls -h /cours/tp1
```

**Snapshots**

![User tree (mkdir/chmod)](snapshots/user-tree-4.1.png)

![User tree (permissions)](snapshots/user-tree-4.2.png)

![Create and put sales.csv](snapshots/creat-csv-4.3.png)

## 5) ACLs (Fine-Grained Access Control) on HDFS

Objective: give `student2` read-only access to `/cours/tp1` without changing standard permissions.

```bash
# Display existing ACLs
hdfs dfs -getfacl /cours/tp1

# Add read (r-x) ACL for “student2”
hdfs dfs -setfacl -m user:student2:r-x /cours/tp1

# Verify
hdfs dfs -getfacl /cours/tp1
```

If “student2” doesn’t exist on the host system, HDFS still stores it symbolically.  
For demonstration, focus on how ACLs work and the resulting access behavior.

**Snapshots**

![HDFS ACLs](snapshots/acls.png)

## 6) Snapshots (Logical Directory Backup)

Enable snapshots on `/cours` and create one before modification:

```bash
hdfs dfsadmin -allowSnapshot /cours
SNAP="pre_modif_$(date +%Y%m%d_%H%M%S)"
hdfs dfs -createSnapshot /cours "$SNAP"
hdfs dfs -ls /cours/.snapshot
```

Test accidental deletion and restoration:

```bash
hdfs dfs -rm /cours/tp1/sales.csv
hdfs dfs -ls /cours/tp1

# Restore from snapshot
hdfs dfs -cp /cours/.snapshot/$SNAP/tp1/sales.csv /cours/tp1/
hdfs dfs -ls /cours/tp1
```

**Snapshots**

![HDFS snapshots](snapshots/snapshots.png)

## 7) Quotas (Space & File Limits)

Set a quota on `/user/student1`:

```bash
hdfs dfsadmin -setSpaceQuota 1m /user/student1
hdfs dfsadmin -setQuota 100 /user/student1
hdfs dfs -count -q -h /user/student1
```

Test by copying a slightly large file:

```bash
dd if=/dev/zero of=/tmp/bigfile.bin bs=64K count=40 # ~2.5 MB
hdfs dfs -put /tmp/bigfile.bin /user/student1/ || echo "Expected failure (quota exceeded)"
```

Reset if needed:

```bash
hdfs dfsadmin -clrSpaceQuota /user/student1
hdfs dfsadmin -clrQuota /user/student1
```

**Snapshots**

![HDFS quotas](snapshots/quotas.png)

## 8) Replication Factor & Data Integrity

Check replication settings:

```bash
# Default replication factor
hdfs getconf -confKey dfs.replication || true

# For a specific file
hdfs dfs -stat %r /cours/tp1/sales.csv
```

Change the replication factor (e.g., from 1 → 2) and optionally run the balancer:

```bash
hdfs dfs -setrep 2 /cours/tp1/sales.csv
hdfs dfs -stat %r /cours/tp1/sales.csv
hdfs balancer -threshold 10
```

Check integrity:

```bash
hdfs fsck / -files -blocks -locations | head -n 50
```

**Snapshots**

![Replication factor change](snapshots/replocation-factor-change.png)

![Balancer](snapshots/balancer-threshold-10.png)

![HDFS fsck](snapshots/check-integrity.png)

## 9) Safemode and Admin Operations

Show the “safemode” (read-only NameNode state):

```bash
hdfs dfsadmin -safemode get
```

**Snapshots**

![safemode get](snapshots/safemode.png)

## 10) Cluster Monitoring with Prometheus and Grafana

a monitoring system for Hadoop components (NameNode & DataNodes) using Prometheus and Grafana, to visualize HDFS metrics: capacity, blocks, files, and JVM health.

### 10.1 Monitoring Architecture

- JMX Exporter on each Hadoop service (NameNode, DataNode) exposes JVM and HDFS metrics.
- Prometheus collects the metrics.
- Grafana visualizes them as dashboards and charts.

```
NameNode (JMX) ─┐
                ├──► Prometheus ───► Grafana
DataNode (JMX) ─┘
```

### 10.2 JMX Configuration for HDFS

(File: `monitoring/jmx/hadoop.yml`)  
Contains metric mapping rules for HDFS and JVM.

**Snapshots**

![hadoop.yml](snapshots/hadoop-yml.png)

### 10.3 Prometheus & Grafana Integration

Launch monitoring:

```bash
docker compose up -d
```
![Grafana datasource (Prometheus)](snapshots/grafana-prometheus.png)

Access:

- Prometheus → http://localhost:9090
- Grafana → http://localhost:3000 (`admin`/`admin`)

**Snapshots**

![Prometheus UI](snapshots/Prometheus-localhost-9090.png)

![Grafana UI](snapshots/grafana-localhost-3000.png)


### 10.4 Grafana Dashboard

Import the dashboard JSON file `monitoring/jmx/hdfs_dashboard.json` (Grafana → Dashboards → Import) and visualize:

- HDFS Capacity Used
- HDFS Capacity Remaining
- HDFS Blocks Total
- HDFS Files Total
- JVM Heap Used
- JVM Threads Current

**Snapshots**


![Grafana dashboard](snapshots/Grafana-Dashboard.png)

### 10.5 Experimentation

```bash
hdfs dfs -mkdir /test
hdfs dfs -put /etc/passwd /test/
hdfs dfs -put /etc/hosts /test/
```

Then refresh Grafana to see metrics evolve.

**Snapshots**

![Metrics after data changes](snapshots/metrics1.png)

![Metrics after data changes](snapshots/metrics2.png)

## 11) Prometheus Alert System Integration

automatic alerts for real-time cluster issues: DataNode down, NameNode down, or low disk space.

Where to see alerts:

- Prometheus alerts page → http://localhost:9090/alerts
- Alertmanager UI → http://localhost:9093

![No alerts](snapshots/no-alert.png)

Simulate a failure:

```bash
docker stop datanode
```

![Prometheus targets](snapshots/targets-health2.png)


→ After 30 seconds, the `DataNodeDown` alert appears.

![Alert firing](snapshots/alert.png)


Restart:

```bash
docker start datanode
```

→ The alert automatically resolves.

![alt text](snapshots/no-alert.png)


