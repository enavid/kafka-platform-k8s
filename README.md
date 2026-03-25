# kafka-platform-k8s

A production-ready, fully open-source streaming data platform deployed on Kubernetes. This project provisions Apache Kafka (via Strimzi Operator), Schema Registry, Kafka Connect, REST Proxy, Apache Flink, Apache NiFi, and Kafka UI as a unified platform — all secured with mutual TLS and exposed through an nginx Ingress controller.

---

## Table of Contents

* [Overview](https://claude.ai/chat/f0bbc13b-ab45-4736-a71d-14f81f3efdf1#overview)
* [Kafka Operators](https://claude.ai/chat/f0bbc13b-ab45-4736-a71d-14f81f3efdf1#kafka-operators--a-comparison)
* [Why This Project](https://claude.ai/chat/f0bbc13b-ab45-4736-a71d-14f81f3efdf1#why-this-project)
* [Component Capabilities](https://claude.ai/chat/f0bbc13b-ab45-4736-a71d-14f81f3efdf1#component-capabilities)
* [Architecture](https://claude.ai/chat/f0bbc13b-ab45-4736-a71d-14f81f3efdf1#architecture)
* [Hardware Requirements](https://claude.ai/chat/f0bbc13b-ab45-4736-a71d-14f81f3efdf1#hardware-requirements)
* [Project Structure](https://claude.ai/chat/f0bbc13b-ab45-4736-a71d-14f81f3efdf1#project-structure)
* [Prerequisites](https://claude.ai/chat/f0bbc13b-ab45-4736-a71d-14f81f3efdf1#prerequisites)
* [Secrets Configuration](https://claude.ai/chat/f0bbc13b-ab45-4736-a71d-14f81f3efdf1#secrets-configuration)
* [Domain Configuration](https://claude.ai/chat/f0bbc13b-ab45-4736-a71d-14f81f3efdf1#domain-configuration)
* [Step-by-Step Deployment](https://claude.ai/chat/f0bbc13b-ab45-4736-a71d-14f81f3efdf1#step-by-step-deployment)
* [Scaling the Platform](https://claude.ai/chat/f0bbc13b-ab45-4736-a71d-14f81f3efdf1#scaling-the-platform)
* [Useful Commands](https://claude.ai/chat/f0bbc13b-ab45-4736-a71d-14f81f3efdf1#useful-commands)

---

## Overview

This platform brings together the most widely used open-source tools in the modern data streaming ecosystem under a single Kubernetes namespace. Each component is independently scalable, secured with mTLS internally, and accessible externally via HTTPS through a centralized Ingress layer.

The platform is designed for teams and individuals who want the power of the Confluent ecosystem without the licensing costs. Every component used here is either Apache-licensed or community-licensed with no production restrictions.

---

## Kafka Operators

When running Kafka on Kubernetes, an operator automates the operational complexity that Kubernetes itself does not understand: broker identity, partition rebalancing, certificate rotation, rolling upgrades, and storage management. Without an operator, these tasks require manual intervention and significant operational expertise.

There are two primary operators in the ecosystem:

**Confluent for Kubernetes (CFK)** is the official operator developed and maintained by Confluent. It manages the full Confluent Platform stack including Kafka brokers, Schema Registry, ksqlDB, Kafka Connect, and Control Center  through custom Kubernetes resources. CFK provides deep integration with Confluent's enterprise features such as RBAC, audit logging, schema linking, tiered storage, and multi-datacenter replication. However, most of these capabilities require a paid Confluent Platform license. CFK offers a Developer License that allows a single-broker deployment for non-production use, but any multi-broker or production configuration requires a commercial subscription.

**Strimzi** is a CNCF incubating project that brings Kafka to Kubernetes through a fully open-source operator. It manages Kafka brokers, topics, users, and Connect clusters through native Kubernetes CRDs. Strimzi handles all operational concerns automatically: TLS certificate generation and rotation, rolling upgrades without downtime, KRaft mode (ZooKeeper-free), rack awareness, and partition rebalancing via Cruise Control integration. Strimzi is free with no license requirements for any cluster size or environment.

---

## Why This Project

Confluent's operator and managed platform provide significant value, particularly for large enterprises that require RBAC, audit logging, and multi-region replication. However, for teams building data pipelines, event-driven architectures, or streaming analytics platforms, the core Kafka ecosystem is entirely sufficient and entirely free.

This project takes the following approach: Strimzi manages the Kafka broker with full automation, while the remaining Confluent Community Edition images (Schema Registry, Kafka Connect, REST Proxy) are deployed as standard Kubernetes workloads. Apache Flink handles stream processing, Apache NiFi handles data flow orchestration, and Kafka UI provides a modern management interface. The result is a platform that is functionally equivalent to a licensed Confluent deployment for the vast majority of use cases, at zero licensing cost.

The key difference from Confluent's enterprise offering is the absence of Control Center (replaced by Kafka UI), RBAC (replaced by Kafka ACLs), and tiered storage. For production workloads that do not require these specific features, this platform is a complete solution.

---

## Component Capabilities

**Apache Kafka (via Strimzi)** is the core message broker. It enables durable, high-throughput publish-subscribe messaging and event streaming. With Kafka, you can build event-driven microservices, real-time data pipelines, activity tracking systems, and log aggregation platforms. Strimzi manages the broker lifecycle, certificates, and topic configuration declaratively.

**Schema Registry** enforces data contracts between producers and consumers. It stores Avro, Protobuf, and JSON Schema definitions and validates messages against registered schemas before they are written to Kafka topics. This prevents incompatible schema changes from breaking downstream consumers.

**Kafka Connect** is a framework for streaming data between Kafka and external systems without writing application code. Using connectors, you can ingest data from databases, object storage, APIs, and files into Kafka topics, or sink Kafka data into external systems such as Elasticsearch, PostgreSQL, S3, and others. The Datagen connector included in this deployment generates synthetic data for testing.

**REST Proxy** provides an HTTP interface for producing and consuming Kafka messages without a native Kafka client. This is useful for languages or environments where a Kafka client library is unavailable, or for simple integrations that do not justify a full Kafka client.

**Apache Flink** is a distributed stream processing engine. With Flink, you can perform stateful computations over data streams, execute SQL queries against live Kafka topics, build complex event processing pipelines, and implement exactly-once processing guarantees. The Flink SQL Client included in this deployment provides an interactive SQL interface for querying Kafka data in real time.

**Apache NiFi** is a visual data flow orchestration platform. It provides a browser-based interface for designing, monitoring, and managing data pipelines that move data between systems. NiFi supports hundreds of processors for reading from and writing to APIs, files, databases, message queues, and cloud storage. In this platform, NiFi connects directly to Kafka using the mTLS certificates provided by Strimzi.

**Kafka UI** is a modern web interface for managing and monitoring the Kafka cluster. It provides visibility into topics, consumer groups, messages, connectors, Schema Registry entries, and Flink jobs in a single dashboard.

---

## Architecture

```
Internet
    |
    v
nginx Ingress Controller (HTTPS / TLS termination)
    |
    |-- kafka-ui.yourdomain.com        --> kafka-ui:8080
    |-- flink.yourdomain.com           --> flink-jobmanager:9081
    |-- schema-registry.yourdomain.com --> kafka-schema-registry:8081
    |-- kafka-connect.yourdomain.com   --> kafka-connect-svc:8083
    |-- rest-proxy.yourdomain.com      --> kafka-rest-proxy:8082
    |-- nifi.yourdomain.com            --> nifi:8443

Internal (kafka-platform namespace, mTLS)
    |-- Kafka Broker (Strimzi KRaft)   port 9092
    |-- Schema Registry                port 8081
    |-- Kafka Connect + Datagen        port 8083
    |-- REST Proxy                     port 8082
    |-- Flink JobManager               port 9081
    |-- Flink TaskManager
    |-- Flink SQL Client
    |-- Apache NiFi                    port 8443
    |-- Kafka UI                       port 8080
```

All internal communication between components and the Kafka broker uses mutual TLS. Certificates are generated automatically by Strimzi and converted to JKS format for use by Confluent components and NiFi.

---

## Hardware Requirements

The following table lists the minimum recommended resources per component for a functional single-node deployment. For production use, each component should be scheduled on dedicated nodes with headroom for peak load.

| Component              | Min CPU Request | Min Memory Request | Persistent Storage                  |
| ---------------------- | --------------- | ------------------ | ----------------------------------- |
| Kafka Broker (Strimzi) | 1 core          | 1 GB               | 50 GB                               |
| Schema Registry        | 0.25 core       | 256 MB             | None                                |
| Kafka Connect          | 0.5 core        | 1 GB               | None                                |
| REST Proxy             | 0.25 core       | 256 MB             | None                                |
| Flink JobManager       | 0.1 core        | 512 MB             | None                                |
| Flink TaskManager      | 0.1 core        | 512 MB             | None                                |
| Flink SQL Client       | 0.25 core       | 256 MB             | None                                |
| Apache NiFi            | 0.5 core        | 2 GB               | 46 GB total across all repositories |
| Kafka UI               | 0.25 core       | 256 MB             | None                                |

The minimum total for a single-node cluster running all components is approximately 4 CPU cores and 8 GB of RAM, with at least 100 GB of available storage for Longhorn or equivalent.

---

## Project Structure

```
kafka-platform-k8s/
├── kafka-platform-namespace.yml
├── strimzi/
│   └── kafka-cluster.yml          # KafkaNodePool, Kafka, KafkaUser, KafkaTopic, ConfigMap
├── secrets/
│   ├── jks-password.yml           # JKS keystore password secret
│   └── tls-jks-converter.yml      # Job: converts Strimzi certs to JKS for Confluent components
├── schema-registry/
│   ├── deployment.yml
│   └── service.yml
├── kafka-connect/
│   ├── deployment.yml
│   └── service.yml
├── rest-proxy/
│   ├── deployment.yml
│   └── service.yml
├── flink/
│   └── deployment.yml             # JobManager, TaskManager, SQL Client
├── nifi/
│   ├── deployment.yml             # StatefulSet with PVCs
│   ├── secret.yml
│   └── service.yml
├── kafka-ui/
│   ├── deployment.yml
│   ├── secret.yml
│   └── service.yml
└── ingress/
    ├── ingress.yml
    └── secret.yml                 # Basic auth htpasswd secret
```

---

## Prerequisites

The following tools must be available before deployment:

* A running Kubernetes cluster (version 1.25 or later)
* `kubectl` configured and pointing to the target cluster
* `helm` v3 installed
* A StorageClass available in the cluster (this project was tested with Longhorn)
* TLS certificates for your domain (Let's Encrypt or equivalent)
* DNS A records pointing to your cluster's Ingress external IP for all subdomains

---

## Secrets Configuration

Before deploying any component, the following secrets must be configured. None of the default values in this repository should be used in production.

**JKS Keystore Password** (`secrets/jks-password.yml`)

This password protects the JKS keystores generated from Strimzi's TLS certificates. It is used by Schema Registry, Kafka Connect, REST Proxy, Kafka UI, and NiFi to authenticate with the Kafka broker.

```yaml
stringData:
  password: "changeit"    # Replace with a strong password
```

**Kafka UI Credentials** (`kafka-ui/secret.yml`)

Used for the LOGIN_FORM authentication built into Kafka UI.

```yaml
stringData:
  username: "admin"
  password: "admin123"    # Replace with a strong password
```

**NiFi Credentials** (`nifi/secret.yml`)

Used for NiFi's single-user authentication. The password must be at least 12 characters.

```yaml
stringData:
  username: "admin"
  password: "adminpassword123"    # Must be at least 12 characters
```

**Ingress Basic Auth** (`ingress/secret.yml`)

Protects Flink, Schema Registry, Kafka Connect, and REST Proxy. Generate the value with:

```bash
htpasswd -nb yourusername yourpassword | base64
```

Replace the `auth` field in `ingress/secret.yml` with the output.

**TLS Certificate Secret**

Create this secret from your domain's TLS certificate before applying the Ingress:

```bash
kubectl create secret tls kafka-platform-tls \
  --cert=./certs/fullchain.pem \
  --key=./certs/privkey.pem \
  --namespace kafka-platform
```

The certificate must be valid for all subdomains used in the Ingress configuration.

---

## Domain Configuration

All services are exposed through subdomains. Replace `yourdomain.com` with your actual domain throughout the configuration files.

The following DNS A records must point to your cluster's Ingress external IP:

| Subdomain                      | Service                | Auth Method     |
| ------------------------------ | ---------------------- | --------------- |
| kafka-ui.yourdomain.com        | Kafka UI               | Login form      |
| flink.yourdomain.com           | Flink Web UI           | HTTP Basic Auth |
| schema-registry.yourdomain.com | Schema Registry API    | HTTP Basic Auth |
| kafka-connect.yourdomain.com   | Kafka Connect REST API | HTTP Basic Auth |
| rest-proxy.yourdomain.com      | Kafka REST Proxy       | HTTP Basic Auth |
| nifi.yourdomain.com            | Apache NiFi            | Login form      |

---

## Step-by-Step Deployment

### Step 1  Create Namespace

```bash
kubectl apply -f kafka-platform-namespace.yml
kubectl get namespace kafka-platform
```

### Step 2  Install nginx Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=1
```

### Step 3 Install Strimzi Operator

```bash
helm repo add strimzi https://strimzi.io/charts/
helm repo update

helm install strimzi-operator strimzi/strimzi-kafka-operator \
  --namespace kafka-platform \
  --set watchNamespaces="{kafka-platform}"

kubectl rollout status deployment/strimzi-cluster-operator -n kafka-platform
```

### Step 4 Deploy Kafka Cluster

```bash
kubectl apply -f strimzi/kafka-cluster.yml

kubectl wait kafka/kafka-platform \
  --for=condition=Ready \
  --timeout=300s \
  -n kafka-platform
```

### Step 5 Create TLS Secret

Before running this step, you must already have valid TLS certificate files for your domain. These are typically obtained via Let's Encrypt (using Certbot or a similar ACME client), a commercial certificate authority, or an internal PKI. The required files are `fullchain.pem` (the certificate chain) and `privkey.pem` (the private key).

Note that Kubernetes also supports automatic certificate provisioning through cert-manager, which can request and renew Let's Encrypt certificates entirely within the cluster without manual file management. This project does not use cert-manager in order to keep the setup straightforward, but it is a viable alternative for teams who prefer fully automated certificate lifecycle management.

```bash
kubectl create secret tls kafka-platform-tls \
  --cert=./certs/fullchain.pem \
  --key=./certs/privkey.pem \
  --namespace kafka-platform
```

### Step 6  Convert Strimzi Certificates to JKS

Strimzi generates its own TLS certificates automatically. This step converts them into JKS format for use by Confluent components and NiFi.

```bash
kubectl apply -f secrets/jks-password.yml

kubectl wait secret/admin \
  --for=jsonpath='{.data.user\.crt}' \
  --timeout=120s \
  -n kafka-platform

kubectl apply -f secrets/tls-jks-converter.yml

kubectl wait job/tls-jks-converter \
  --for=condition=Complete \
  --timeout=120s \
  -n kafka-platform

kubectl get secret kafka-client-jks -n kafka-platform
```

### Step 7  Deploy All Services

```bash
kubectl apply -f schema-registry/
kubectl apply -f kafka-connect/
kubectl apply -f rest-proxy/
kubectl apply -f flink/
kubectl apply -f nifi/
kubectl apply -f kafka-ui/
kubectl apply -f ingress/
```

### Step 8  Verify

```bash
kubectl get pods -n kafka-platform
```

Expected state:

```
NAME                                         READY   STATUS    RESTARTS
kafka-platform-kafka-0                       1/1     Running   0
kafka-schema-registry-xxxxxxxxx-xxxxx        1/1     Running   0
kafka-connect-svc-xxxxxxxxx-xxxxx            1/1     Running   0
kafka-rest-proxy-xxxxxxxxx-xxxxx             1/1     Running   0
flink-jobmanager-xxxxxxxxx-xxxxx             1/1     Running   0
flink-taskmanager-xxxxxxxxx-xxxxx            1/1     Running   0
flink-sql-client-xxxxxxxxx-xxxxx             1/1     Running   0
nifi-0                                       1/1     Running   0
kafka-ui-xxxxxxxxx-xxxxx                     1/1     Running   0
strimzi-cluster-operator-xxxxxxxxx-xxxxx     1/1     Running   0
```

---

## Scaling the Platform

One of the primary advantages of running this platform on Kubernetes with Strimzi is that scaling is a declarative, operator-managed operation rather than a manual process. As your workload grows, you can add worker nodes to your cluster and expand each component independently.

**Scaling Kafka Brokers**

In a single-node setup, the `KafkaNodePool` in `strimzi/kafka-cluster.yml` runs with `replicas: 1`. When you add worker nodes to your Kubernetes cluster, Strimzi can distribute additional brokers across those nodes automatically. To scale to three brokers, update the replica count:

```bash
kubectl patch kafkanodepool kafka -n kafka-platform   --type='merge'   -p '{"spec":{"replicas":3}}'
```

Strimzi handles everything that follows: it provisions new broker pods, assigns unique broker IDs, updates the cluster metadata, and begins rebalancing partitions across all brokers. You do not need to touch any configuration file or restart any service manually. If you have also configured rack awareness in the KafkaNodePool, Strimzi will schedule each broker on a different node or availability zone to ensure fault isolation.

When you scale down, Strimzi performs the same process in reverse: it migrates all partition leadership and replicas away from the brokers being removed before terminating them, ensuring no data loss.

**Scaling Flink TaskManagers**

Flink's processing capacity is determined by the number of TaskManagers and the task slots configured per manager. To increase parallelism, scale the TaskManager deployment (or update deployment file):

```bash
kubectl scale deployment/flink-taskmanager --replicas=3 -n kafka-platform
```

Alternatively, set `replicas: 3` in `flink/deployment.yml` and apply:

```bash
kubectl apply -f flink/deployment.yml
```

Flink's JobManager detects the new TaskManagers automatically and makes their slots available for job execution. Running jobs are not interrupted by this operation.

**Scaling Kafka Connect**

Kafka Connect workers in a distributed group coordinate through Kafka topics. Scaling the Connect deployment adds workers to the existing group, and the connector tasks are automatically rebalanced across all available workers (or update deployment file):

```bash
kubectl scale deployment/kafka-connect-svc --replicas=3 -n kafka-platform
```

Alternatively, set `replicas: 3` in `kafka-connect/deployment.yml` and apply:

```bash
kubectl apply -f kafka-connect/deployment.yml
```

**Scaling Schema Registry and REST Proxy**

Both Schema Registry and REST Proxy are stateless services that store no local state. They can be scaled horizontally at any time without coordination (or update deployment file):

```bash
kubectl scale deployment/kafka-schema-registry --replicas=2 -n kafka-platform
kubectl scale deployment/kafka-rest-proxy --replicas=2 -n kafka-platform
```

Alternatively, set `replicas: 2` in `schema-registry/deployment.yml` or `rest-proxy/deployment.yml` and apply:

```bash
kubectl apply -f schema-registry/deployment.yml
kubectl apply -f rest-proxy/deployment.yml
```

**What the Platform Manages For You**

When operating a multi-broker, multi-node Kafka cluster manually, the following tasks require careful manual intervention: assigning broker IDs consistently after restarts, ensuring partition replicas are distributed evenly, rotating TLS certificates without downtime, and performing rolling upgrades one broker at a time. Strimzi automates all of these. The operator continuously reconciles the actual cluster state against the desired state declared in the `Kafka` and `KafkaNodePool` resources. If a broker pod is evicted, rescheduled to a new node, or loses its storage, Strimzi detects the divergence and corrects it without manual input.

---

## Useful Commands

Check Kafka cluster readiness:

```bash
kubectl get kafka kafka-platform -n kafka-platform
```

List all Kafka topics managed by Strimzi:

```bash
kubectl get kafkatopics -n kafka-platform
```

Create a new topic declaratively:

```bash
kubectl apply -f - <<EOF
apiVersion: kafka.strimzi.io/v1
kind: KafkaTopic
metadata:
  name: my-topic
  namespace: kafka-platform
  labels:
    strimzi.io/cluster: kafka-platform
spec:
  partitions: 3
  replicas: 1
EOF
```

Access the Flink SQL Client interactively:

```bash
kubectl exec -it deploy/flink-sql-client -n kafka-platform -- ./bin/sql-client.sh
```

View logs for any component:

```bash
kubectl logs -f deploy/kafka-ui -n kafka-platform
kubectl logs -f deploy/kafka-connect-svc -n kafka-platform
kubectl logs -f deploy/kafka-schema-registry -n kafka-platform
kubectl logs -f nifi-0 -n kafka-platform
kubectl logs -n kafka-platform kafka-platform-kafka-0
```

Restart a deployment after a configuration change:

```bash
kubectl rollout restart deployment/kafka-ui -n kafka-platform
```

Scale Flink TaskManagers:

```bash
kubectl scale deployment/flink-taskmanager --replicas=3 -n kafka-platform
```
