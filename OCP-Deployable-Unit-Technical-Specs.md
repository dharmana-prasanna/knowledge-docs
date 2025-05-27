# OpenShift Deployable Unit - Technical Specifications

## Table of Contents
1. [Component Specifications](#component-specifications)
2. [Configuration Requirements](#configuration-requirements)
3. [Network Architecture](#network-architecture)
4. [Security Implementation](#security-implementation)
5. [Deployment Procedures](#deployment-procedures)
6. [Monitoring and Observability](#monitoring-and-observability)
7. [Troubleshooting Guide](#troubleshooting-guide)

## Component Specifications

### Enterprise Service Dependencies

#### Okta Integration
- **Protocol**: SAML 2.0 / OAuth 2.0 / OpenID Connect
- **Required Scopes**: `openid`, `profile`, `email`, `groups`
- **Token Lifetime**: 8 hours (configurable)
- **Connection Requirements**: HTTPS, port 443

#### Protegrity Data Protection
- **API Version**: v2.x
- **Supported Protocols**: REST API over HTTPS
- **Data Classification**: PII, PHI, Financial
- **Encryption Standards**: AES-256, Format Preserving Encryption (FPE)

#### FutureX HSM
- **Model Support**: VirtuCrypt cloud, Vectera series
- **API Interface**: PKCS#11, REST API
- **Key Types**: RSA 2048/4096, ECC P-256/P-384/P-521
- **High Availability**: Active-Active clustering

#### F5 GTM Configuration
- **DNS Resolution**: Intelligent DNS with health checks
- **Load Balancing**: Global Server Load Balancing (GSLB)
- **Health Monitoring**: HTTP/HTTPS, TCP, custom scripts
- **Failover**: Automatic with configurable thresholds

#### Panorama Network Security
- **Management Interface**: REST API v10.x
- **Policy Types**: Security, NAT, QoS, Decryption
- **Log Forwarding**: Syslog, SNMP, REST API
- **Template Management**: Device groups and templates

#### JFrog Artifactory
- **Repository Types**: Docker, Helm, Maven, NPM, PyPI
- **API Version**: REST API v7.x
- **Authentication**: API Keys, Access Tokens, LDAP
- **Replication**: Multi-site, real-time sync

#### Active Directory
- **Protocol**: LDAP/LDAPS (port 389/636)
- **Schema**: RFC 2307bis for POSIX attributes
- **Group Mapping**: Security groups to OpenShift RBAC
- **Certificate Requirements**: CA-signed certificates for LDAPS

### System Component Brokers

#### PostgreSQL Operator
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-cluster
spec:
  instances: 3
  postgresql:
    parameters:
      max_connections: "100"
      shared_preload_libraries: "pg_stat_statements"
  storage:
    size: 100Gi
    storageClass: fast-ssd
  monitoring:
    enabled: true
```

#### RabbitMQ Operator
```yaml
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rabbitmq-cluster
spec:
  replicas: 3
  rabbitmq:
    additionalConfig: |
      cluster_formation.peer_discovery_backend = rabbit_peer_discovery_k8s
      cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
```

#### Redis Operator
```yaml
apiVersion: databases.spotahome.com/v1
kind: RedisFailover
metadata:
  name: redis-cluster
spec:
  sentinel:
    replicas: 3
  redis:
    replicas: 3
    storage:
      persistentVolumeClaim:
        metadata:
          name: redis-storage
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 50Gi
```

#### Object Storage Operator
```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: app-bucket
spec:
  generateBucketName: "app-data"
  storageClassName: "noobaa-default-bucket-class"
```

### Observability Integrators

#### Splunk Universal Forwarder
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: splunk-forwarder
spec:
  selector:
    matchLabels:
      app: splunk-forwarder
  template:
    spec:
      containers:
      - name: splunk-forwarder
        image: splunk/universalforwarder:latest
        env:
        - name: SPLUNK_START_ARGS
          value: "--accept-license --answer-yes"
        - name: SPLUNK_FORWARD_SERVER
          value: "splunk-indexer:9997"
```

#### Turbonomic Agent
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: turbonomic-agent
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: turbonomic-agent
        image: turbonomic/kubeturbo:latest
        env:
        - name: TURBO_SERVER_URL
          value: "https://turbonomic.company.com"
        - name: TURBO_SERVER_VERSION
          value: "8.x"
```

#### DataDog Agent
```yaml
apiVersion: datadoghq.com/v2alpha1
kind: DatadogAgent
metadata:
  name: datadog
spec:
  global:
    site: "datadoghq.com"
    apiKey:
      secretName: datadog-secret
      keyName: api-key
  features:
    apm:
      enabled: true
    npm:
      enabled: true
    usm:
      enabled: true
```

#### Sysdig Agent
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: sysdig-agent
spec:
  template:
    spec:
      containers:
      - name: sysdig-agent
        image: sysdig/agent:latest
        env:
        - name: ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: sysdig-agent
              key: access-key
        - name: TAGS
          value: "environment:production,cluster:ocp-main"
```

## Configuration Requirements

### Resource Requirements

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit | Storage |
|-----------|-------------|-----------|----------------|--------------|---------|
| Postgres Operator | 100m | 500m | 128Mi | 512Mi | 100Gi |
| RabbitMQ Operator | 100m | 300m | 128Mi | 256Mi | 50Gi |
| Redis Operator | 50m | 200m | 64Mi | 128Mi | 50Gi |
| Splunk Forwarder | 100m | 200m | 128Mi | 256Mi | 10Gi |
| DataDog Agent | 200m | 400m | 256Mi | 512Mi | - |
| Sysdig Agent | 150m | 300m | 512Mi | 1Gi | - |
| Turbonomic Agent | 100m | 200m | 128Mi | 256Mi | - |

### Network Configuration

#### Port Requirements
```yaml
# Enterprise Services
okta: 443/tcp
protegrity: 443/tcp, 8443/tcp
futurex: 443/tcp, 22/tcp
f5-gtm: 443/tcp, 53/udp
panorama: 443/tcp
artifactory: 443/tcp, 80/tcp
active-directory: 389/tcp, 636/tcp, 88/tcp, 464/tcp

# Internal Services
postgres: 5432/tcp
rabbitmq: 5672/tcp, 15672/tcp, 25672/tcp
redis: 6379/tcp, 26379/tcp
kafka: 9092/tcp, 9093/tcp
jenkins: 8080/tcp, 50000/tcp
```

#### Network Policies
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deployable-unit-netpol
spec:
  podSelector:
    matchLabels:
      app: deployable-unit
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: system
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to: []
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 5432
```

## Security Implementation

### RBAC Configuration
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployable-unit-operator
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### Pod Security Standards
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: deployable-unit
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Service Mesh Configuration
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deployable-unit-authz
spec:
  selector:
    matchLabels:
      app: deployable-unit
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"]
```

## Deployment Procedures

### Role-Based Deployment Workflow

#### Phase 1: AWS Infrastructure Setup (AWS Engineer)
**Timeline**: 1-2 weeks  
**Prerequisites**: AWS account access, network design approval

```bash
# 1. AWS Account Registration and Setup
aws organizations create-account --account-name "ocp-production" \
  --email "aws-admin@company.com"

# 2. IAM Configuration
aws iam create-role --role-name OpenShiftMachineRole \
  --assume-role-policy-document file://trust-policy.json
aws iam attach-role-policy --role-name OpenShiftMachineRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess

# 3. VPC Configuration
aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=ocp-vpc}]'

# 4. Security Groups and Network ACLs
aws ec2 create-security-group --group-name ocp-cluster-sg \
  --description "OpenShift Cluster Security Group" --vpc-id vpc-xxxxxxxx

# 5. AWS Service Integration
aws rds create-db-cluster --db-cluster-identifier aurora-cluster \
  --engine aurora-postgresql --master-username postgres
```

**AWS Engineer Deliverables Checklist**:
- ✅ AWS Account registered and configured
- ✅ IAM roles and policies created
- ✅ VPC with subnets and routing configured
- ✅ Security groups and NACLs implemented
- ✅ AWS services (RDS, S3, EFS) provisioned
- ✅ Network connectivity to enterprise services verified

#### Phase 2: OpenShift Platform Deployment (OCP Engineer)
**Timeline**: 2-3 weeks  
**Prerequisites**: AWS infrastructure ready, enterprise service access

```bash
# 1. OpenShift Installation
openshift-install create install-config --dir=./cluster-config
openshift-install create cluster --dir=./cluster-config --log-level=info

# 2. HashiCorp Vault Setup
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault --namespace vault-system --create-namespace \
  --set "server.dev.enabled=true"

# 3. Configure OpenShift
oc create namespace deployable-unit
oc apply -f security/rbac.yaml
oc apply -f security/pod-security-standards.yaml

# 4. Deploy System Component Brokers
oc apply -f operators/postgres-operator.yaml
oc apply -f operators/rabbitmq-operator.yaml
oc apply -f operators/redis-operator.yaml
oc apply -f operators/object-storage-operator.yaml

# 5. Setup Observability Stack
oc apply -f observability/splunk-forwarder.yaml
oc apply -f observability/datadog-agent.yaml
oc apply -f observability/turbonomic-agent.yaml
oc apply -f observability/sysdig-agent.yaml

# 6. Configure Enterprise Service Integration
oc apply -f integrations/okta-config.yaml
oc apply -f integrations/active-directory-config.yaml
oc apply -f integrations/f5-gtm-config.yaml
```

**OCP Engineer Deliverables Checklist**:
- ✅ OpenShift cluster deployed and configured
- ✅ HashiCorp Vault installed and integrated
- ✅ System operators deployed and operational
- ✅ Observability agents configured
- ✅ Enterprise service integrations established
- ✅ Security policies and RBAC implemented
- ✅ Platform services configured

#### Phase 3: Application Deployment (Product Team Engineer)
**Timeline**: 1-2 weeks  
**Prerequisites**: OpenShift platform ready, access to application source code

```bash
# 1. Jenkins Configuration and Pipeline Setup
oc new-app jenkins-persistent -n cicd
oc create configmap jenkins-config --from-file=jenkins/config/ -n cicd

# 2. Create Application Container Images
docker build -t registry.company.com/app:v1.0.0 .
docker push registry.company.com/app:v1.0.0

# 3. Configure CI/CD Pipeline
oc apply -f cicd/jenkins-pipeline.yaml
oc apply -f cicd/build-config.yaml

# 4. Deploy Application Components
oc apply -f applications/deployment.yaml
oc apply -f applications/service.yaml
oc apply -f applications/route.yaml

# 5. Setup Application Monitoring
oc apply -f monitoring/application-metrics.yaml
oc apply -f monitoring/alerting-rules.yaml

# 6. Execute CI/CD Pipeline
oc start-build app-pipeline -n cicd
```

**Product Engineer Deliverables Checklist**:
- ✅ Jenkins pipelines configured and operational
- ✅ Application container images built and stored
- ✅ CI/CD automation implemented
- ✅ Application deployed and accessible
- ✅ Application monitoring configured
- ✅ Artifact management setup in JFrog

### Pre-deployment Checklist by Role

#### AWS Engineer Prerequisites
1. ✅ AWS account access and billing setup
2. ✅ Network architecture design approved
3. ✅ Security compliance requirements reviewed
4. ✅ Cost management policies defined

#### OCP Engineer Prerequisites
1. ✅ OpenShift installer and CLI tools
2. ✅ Enterprise service credentials and certificates
3. ✅ Operator installation permissions
4. ✅ Storage and networking requirements validated

#### Product Engineer Prerequisites
1. ✅ Application source code and build requirements
2. ✅ Container registry access
3. ✅ Jenkins and CI/CD tool access
4. ✅ Application deployment configurations

### Cross-Team Coordination Points

| Milestone | AWS Engineer | OCP Engineer | Product Engineer |
|-----------|--------------|--------------|------------------|
| Infrastructure Ready | ✅ Complete | 🏁 Start | ⏳ Prepare |
| Platform Deployed | - | ✅ Complete | 🏁 Start |
| Application Live | - | 🤝 Support | ✅ Complete |

### Deployment Verification Script
```bash
#!/bin/bash
# Multi-role deployment verification

echo "=== AWS Infrastructure Verification ==="
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=ocp-vpc"
aws iam get-role --role-name OpenShiftMachineRole

echo "=== OpenShift Platform Verification ==="
oc get nodes
oc get operators
oc get pods -n vault-system

echo "=== Application Verification ==="
oc get deployments -n deployable-unit
oc get routes -n deployable-unit
curl -s http://$(oc get route app-route -o jsonpath='{.spec.host}')/health
```

### Health Checks
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

## Monitoring and Observability

### Metrics Collection
- **Application Metrics**: Prometheus format via `/metrics` endpoint
- **Infrastructure Metrics**: Node Exporter, cAdvisor
- **Custom Metrics**: Business KPIs, SLA metrics

### Alerting Rules
```yaml
groups:
- name: deployable-unit
  rules:
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High error rate detected"
```

### Dashboards
- Application Performance Dashboard
- Infrastructure Overview Dashboard
- Security Monitoring Dashboard
- Business Metrics Dashboard

## Troubleshooting Guide

### Common Issues

#### 1. Pod Startup Failures
```bash
# Check pod status
oc get pods -l app=deployable-unit

# View pod logs
oc logs -f pod-name

# Describe pod for events
oc describe pod pod-name
```

#### 2. Connectivity Issues
```bash
# Test service connectivity
oc exec -it pod-name -- curl -v service-name:port

# Check network policies
oc get networkpolicy
```

#### 3. Resource Constraints
```bash
# Check resource usage
oc top pods
oc top nodes

# View resource quotas
oc get resourcequota
```

### Diagnostics Scripts
```bash
#!/bin/bash
# Health check script
./scripts/health-check.sh

# Connectivity test
./scripts/connectivity-test.sh

# Performance baseline
./scripts/performance-test.sh
```

---

**Document Version**: 1.0  
**Last Updated**: $(date)  
**Maintained By**: Platform Engineering Team 