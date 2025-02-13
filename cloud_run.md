# Cloud Run Configuration Settings Explained

## 1. Top-Level Metadata Configuration
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: spring-boot-app
  namespace: default
  labels:
    environment: production
    app: spring-boot
```

- `apiVersion`: Specifies the Knative Serving API version used by Cloud Run
- `kind`: Defines this as a Knative Service resource
- `name`: The service identifier used in Cloud Run console and CLI commands
- `namespace`: Logical grouping of resources (default is recommended for most cases)
- `labels`: Key-value pairs for resource organization and filtering

## 2. Service Annotations
```yaml
annotations:
  run.googleapis.com/client-name: "spring-boot-app"
  run.googleapis.com/ingress: "all"
  run.googleapis.com/launch-stage: "BETA"
```

- `client-name`: Custom identifier for the client application
- `ingress`: Controls access to the service:
  - `all`: Allows all traffic (public and internal)
  - `internal-only`: Only allows internal VPC traffic
  - `internal-and-cloud-load-balancing`: Allows internal VPC and Cloud Load Balancer traffic
- `launch-stage`: Indicates feature maturity level (BETA, GA, etc.)

## 3. Template Metadata Annotations
```yaml
template:
  metadata:
    annotations:
      autoscaling.knative.dev/maxScale: "100"
      autoscaling.knative.dev/minScale: "1"
      run.googleapis.com/cpu-throttling: "true"
      run.googleapis.com/execution-environment: "gen2"
      run.googleapis.com/startup-cpu-boost: "true"
```

- `maxScale`: Maximum number of container instances for autoscaling (1-1000)
- `minScale`: Minimum number of container instances to maintain (0-1000)
- `cpu-throttling`: Enables CPU throttling to reduce costs during idle periods
- `execution-environment`: Specifies runtime environment version:
  - `gen1`: First generation (legacy)
  - `gen2`: Second generation (recommended, better performance)
- `startup-cpu-boost`: Allocates additional CPU during container startup

## 4. VPC and Cloud SQL Configuration
```yaml
run.googleapis.com/vpc-access-connector: "projects/${PROJECT_ID}/locations/${REGION}/connectors/my-vpc-connector"
run.googleapis.com/vpc-access-egress: "private-ranges-only"
run.googleapis.com/cloudsql-instances: "${PROJECT_ID}:${REGION}:${INSTANCE_NAME}"
run.googleapis.com/sandbox: "gvisor"
```

- `vpc-access-connector`: Connects service to a VPC network
- `vpc-access-egress`: Controls outbound traffic routing:
  - `private-ranges-only`: Only private IP ranges use VPC
  - `all`: All traffic routes through VPC
- `cloudsql-instances`: Connects to Cloud SQL instances
- `sandbox`: Specifies container runtime sandbox (gvisor recommended)

## 5. Container Specification
```yaml
spec:
  serviceAccountName: "service-account@${PROJECT_ID}.iam.gserviceaccount.com"
  containerConcurrency: 80
  timeoutSeconds: 300
```

- `serviceAccountName`: Service account for container workload identity
- `containerConcurrency`: Maximum concurrent requests per container (0-1000)
  - `0`: Unlimited concurrency
  - `1`: Single request at a time
  - `80`: Default value, recommended for most applications
- `timeoutSeconds`: Maximum request duration (0-3600)

## 6. Container Resources
```yaml
resources:
  limits:
    cpu: "2000m"      # 2 vCPU
    memory: "2Gi"     # 2 GB RAM
  requests:
    cpu: "1000m"      # 1 vCPU
    memory: "1Gi"     # 1 GB RAM
```

- `limits`: Maximum resources allocated to container
  - `cpu`: Maximum CPU (in millicores, 1000m = 1 vCPU)
  - `memory`: Maximum memory (supports Mi, Gi units)
- `requests`: Minimum resources guaranteed to container
  - `cpu`: Minimum CPU guarantee
  - `memory`: Minimum memory guarantee

## 7. Health Checks
```yaml
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

- `startupProbe`: Checks if application has started successfully
  - `initialDelaySeconds`: Wait before first check
  - `periodSeconds`: Time between checks
  - `failureThreshold`: Consecutive failures before declaring failure

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  periodSeconds: 30
  timeoutSeconds: 4
```

- `livenessProbe`: Checks if application is running properly
  - `periodSeconds`: Frequency of checks
  - `timeoutSeconds`: Time before probe times out

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  periodSeconds: 30
  timeoutSeconds: 4
```

- `readinessProbe`: Checks if application is ready to handle traffic
  - Similar settings to livenessProbe
  - Used for traffic management

## 8. Environment Configuration
```yaml
env:
  - name: SPRING_PROFILES_ACTIVE
    value: "prod"
  - name: JAVA_TOOL_OPTIONS
    value: "-XX:MaxRAMPercentage=75 -XX:InitialRAMPercentage=50"
```

- `env`: Direct environment variable definitions
  - Name-value pairs for configuration
  - Can reference secrets and config maps

```yaml
envFrom:
  - configMapRef:
      name: app-config
  - secretRef:
      name: app-secrets
```

- `envFrom`: Bulk environment variable loading
  - `configMapRef`: Load from ConfigMap
  - `secretRef`: Load from Secret

## 9. Volume Configuration
```yaml
volumeMounts:
  - name: secrets
    mountPath: /secrets
    readOnly: true
  - name: config
    mountPath: /config
    readOnly: true
volumes:
  - name: secrets
    secret:
      secretName: app-secrets
  - name: config
    configMap:
      name: app-config
```

- `volumeMounts`: Where volumes are mounted in container
  - `name`: Volume identifier
  - `mountPath`: Container filesystem path
  - `readOnly`: Prevent writes to volume
- `volumes`: Volume definitions
  - Can reference Secrets and ConfigMaps
  - Supports various volume types

## 10. Traffic Configuration
```yaml
traffic:
  - percent: 100
    latestRevision: true
```

- `traffic`: Controls request routing
  - `percent`: Percentage of traffic to this revision
  - `latestRevision`: Whether to use latest deployment
  - Supports gradual rollouts and A/B testing

## Best Practices

1. Resource Configuration:
   - Start with lower limits and adjust based on monitoring
   - Set appropriate memory-to-CPU ratios
   - Use startup CPU boost for faster initialization

2. Scaling Configuration:
   - Set minScale=1 for improved cold start performance
   - Configure maxScale based on expected traffic patterns
   - Adjust containerConcurrency based on application characteristics

3. Health Checks:
   - Implement all three probe types
   - Set appropriate timeouts and thresholds
   - Use separate endpoints for different health checks

4. Security:
   - Use minimum required IAM roles
   - Enable VPC connector when needed
   - Store sensitive data in Secret Manager

5. Performance:
   - Enable CPU throttling for cost optimization
   - Use gen2 execution environment
   - Configure proper JVM options for containerized environments
