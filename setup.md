# Building and Deploying a Spring Boot Application to Cloud Run with GitLab CI/CD

## 1. Project Setup

### Prerequisites
- Java 17 or later
- Maven or Gradle
- Google Cloud account
- GitLab account
- Docker installed locally

### Initial Spring Boot Setup

1. Create a new Spring Boot project using Spring Initializer:
   ```bash
   curl https://start.spring.io/#!type=maven-project&language=java&platformVersion=3.2.0&packaging=jar&jvmVersion=17&groupId=com.example&artifactId=demo&name=demo&description=Demo%20project%20for%20Spring%20Boot&packageName=com.example.demo&dependencies=web,actuator
   ```# Complete Spring Boot Application Setup for Cloud Run

## 1. Required Dependencies
Add to `pom.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>cloud-run-demo</name>

    <properties>
        <java.version>17</java.version>
        <spring-cloud-gcp.version>4.8.0</spring-cloud-gcp.version>
    </properties>

    <dependencies>
        <!-- Core -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <!-- Actuator for health checks -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Database -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
        </dependency>

        <!-- Security -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <!-- Cloud -->
        <dependency>
            <groupId>com.google.cloud</groupId>
            <artifactId>spring-cloud-gcp-starter</artifactId>
            <version>${spring-cloud-gcp.version}</version>
        </dependency>
        <dependency>
            <groupId>com.google.cloud</groupId>
            <artifactId>spring-cloud-gcp-starter-secretmanager</artifactId>
            <version>${spring-cloud-gcp.version}</version>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>com.google.cloud.tools</groupId>
                <artifactId>jib-maven-plugin</artifactId>
                <version>3.4.0</version>
            </plugin>
        </plugins>
    </build>
</project>
```

## 2. Application Configuration
Create `src/main/resources/application.yml`:
```yaml
spring:
  application:
    name: spring-boot-cloud-run
  
  # Database Configuration
  datasource:
    url: jdbc:postgresql:///${DB_NAME}
    username: ${DB_USER}
    password: ${DB_PASS}
    hikari:
      maximum-pool-size: 5
      minimum-idle: 2
  
  # JPA Configuration
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
    
  # Security Configuration
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://accounts.google.com
          jwk-set-uri: https://www.googleapis.com/oauth2/v3/certs

# Actuator Configuration
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      probes:
        enabled: true
      show-details: always
      group:
        readiness:
          include: db,diskSpace
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true

# Server Configuration
server:
  port: 8080
  shutdown: graceful
  tomcat:
    threads:
      max: 50
      min-spare: 5

# Logging Configuration
logging:
  level:
    root: INFO
    com.example: DEBUG
  pattern:
    console: "%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
```

## 3. Security Configuration
Create `src/main/java/com/example/demo/config/SecurityConfig.java`:
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt());
        return http.build();
    }
}
```

## 4. Health Checks
Create `src/main/java/com/example/demo/health/CustomHealthIndicator.java`:
```java
@Component
public class CustomHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        return Health.up()
                    .withDetail("app", "running")
                    .withDetail("error", "none")
                    .build();
    }
}
```

## 5. Database Scripts
Create `src/main/resources/db/migration/V1__init.sql`:
```sql
CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
```

## 6. Update Cloud Run Service Configuration
Add these environment variables to your `service.yaml`:
```yaml
spec:
  template:
    spec:
      containers:
        - env:
            # Database
            - name: DB_NAME
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: database
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: username
            - name: DB_PASS
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: password
            
            # Application
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            - name: SERVER_PORT
              value: "8080"
            
            # JVM Options
            - name: JAVA_TOOL_OPTIONS
              value: >-
                -XX:MaxRAMPercentage=75
                -XX:InitialRAMPercentage=50
                -XX:+UseContainerSupport
                -Djava.security.egd=file:/dev/./urandom
                -Duser.timezone=UTC
```

## 7. Create Required Secrets in Google Cloud

```bash
# Create secrets
gcloud secrets create db-secrets --replication-policy="automatic"

# Add secret versions
echo -n "your-database-name" | \
  gcloud secrets versions add db-secrets --data-file=-

echo -n "your-username" | \
  gcloud secrets versions add db-secrets --data-file=-

echo -n "your-password" | \
  gcloud secrets versions add db-secrets --data-file=-

# Grant access to the service account
gcloud secrets add-iam-policy-binding db-secrets \
    --member="serviceAccount:${SERVICE_ACCOUNT}" \
    --role="roles/secretmanager.secretAccessor"
```

## 8. Create Cloud SQL Instance
```bash
# Create Cloud SQL instance
gcloud sql instances create my-postgresql \
    --database-version=POSTGRES_15 \
    --cpu=2 \
    --memory=4GB \
    --region=${REGION} \
    --root-password=${DB_PASSWORD}

# Create database
gcloud sql databases create ${DB_NAME} --instance=my-postgresql

# Create user
gcloud sql users create ${DB_USER} \
    --instance=my-postgresql \
    --password=${DB_PASSWORD}
```

## 9. Additional GitLab CI/CD Updates
Add these steps to `.gitlab-ci.yml`:
```yaml
.setup_cloud_sql_proxy: &setup_cloud_sql_proxy
  - wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -O cloud_sql_proxy
  - chmod +x cloud_sql_proxy
  - ./cloud_sql_proxy -instances=${PROJECT_ID}:${REGION}:${INSTANCE_NAME}=tcp:5432 &

deploy:
  script:
    - *setup_cloud_sql_proxy
    - |
      gcloud run deploy $SERVICE_NAME \
        --image gcr.io/$PROJECT_ID/$APP_NAME:$GOOGLE_TAG \
        --platform managed \
        --region $REGION \
        --allow-unauthenticated \
        --set-secrets=DB_USER=db-secrets:username:latest,DB_PASS=db-secrets:password:latest \
        --set-env-vars=DB_NAME=${DB_NAME} \
        --service-account=${SERVICE_ACCOUNT} \
        --cpu=2 \
        --memory=2Gi \
        --min-instances=1 \
        --max-instances=10
```

## 10. Required IAM Roles
Ensure your service account has these roles:
```bash
# Add required roles
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${SERVICE_ACCOUNT}" \
    --role="roles/cloudsql.client"

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${SERVICE_ACCOUNT}" \
    --role="roles/secretmanager.secretAccessor"
```

Now the application includes:
1. All required dependencies
2. Proper database configuration
3. Security setup
4. Health checks
5. Secrets management
6. Cloud SQL setup
7. Required IAM roles
8. Updated deployment configuration

Would you like me to explain any specific component in more detail?```

## 2. Containerization

### Dockerfile
```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Cloud Run Configuration
Create a `service.yaml`:
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: spring-boot-app
spec:
  template:
    spec:
      containers:
        - image: gcr.io/${PROJECT_ID}/${APP_NAME}
          ports:
            - containerPort: 8080
          resources:
            limits:
              memory: "1Gi"
              cpu: "1000m"
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
```

## 3. GitLab CI/CD Pipeline

Create `.gitlab-ci.yml`:
```yaml
image: eclipse-temurin:17-jdk-alpine

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"
  PROJECT_ID: $PROJECT_ID
  APP_NAME: spring-boot-app
  REGION: us-central1
  GOOGLE_TAG: $CI_COMMIT_SHA

cache:
  paths:
    - .m2/repository

stages:
  - test
  - build
  - deploy

test:
  stage: test
  script:
    - ./mvnw test

build:
  stage: build
  script:
    - ./mvnw clean package -DskipTests
    - echo $GOOGLE_CLOUD_KEY > /tmp/google_cloud_key.json
    - gcloud auth activate-service-account --key-file /tmp/google_cloud_key.json
    - gcloud config set project $PROJECT_ID
    - gcloud auth configure-docker
    - docker build -t gcr.io/$PROJECT_ID/$APP_NAME:$GOOGLE_TAG .
    - docker push gcr.io/$PROJECT_ID/$APP_NAME:$GOOGLE_TAG
  artifacts:
    paths:
      - target/*.jar

deploy:
  stage: deploy
  script:
    - echo $GOOGLE_CLOUD_KEY > /tmp/google_cloud_key.json
    - gcloud auth activate-service-account --key-file /tmp/google_cloud_key.json
    - gcloud config set project $PROJECT_ID
    - |
      gcloud run deploy $APP_NAME \
        --image gcr.io/$PROJECT_ID/$APP_NAME:$GOOGLE_TAG \
        --platform managed \
        --region $REGION \
        --allow-unauthenticated
  only:
    - main
```

## 4. Google Cloud Setup Steps

1. Create a new Google Cloud Project:
   ```bash
   gcloud projects create YOUR_PROJECT_ID
   gcloud config set project YOUR_PROJECT_ID
   ```

2. Enable required APIs:
   ```bash
   gcloud services enable \
     cloudbuild.googleapis.com \
     run.googleapis.com \
     containerregistry.googleapis.com
   ```

3. Create Service Account for GitLab CI/CD:
   ```bash
   gcloud iam service-accounts create gitlab-ci \
     --display-name="GitLab CI Cloud Run"
   
   gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
     --member="serviceAccount:gitlab-ci@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
     --role="roles/run.admin"
   
   gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
     --member="serviceAccount:gitlab-ci@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
     --role="roles/storage.admin"
   ```

4. Generate and download service account key:
   ```bash
   gcloud iam service-accounts keys create key.json \
     --iam-account=gitlab-ci@YOUR_PROJECT_ID.iam.gserviceaccount.com
   ```

## 5. GitLab Repository Setup

1. Create a new project on GitLab

2. Add the following CI/CD variables in GitLab (Settings > CI/CD > Variables):
   - `GOOGLE_CLOUD_KEY`: Content of the `key.json` file (Base64 encoded)
   - `PROJECT_ID`: Your Google Cloud Project ID
   - `GOOGLE_PROJECT_ID`: Same as PROJECT_ID
   - `GOOGLE_COMPUTE_ZONE`: Your preferred compute zone (e.g., us-central1-a)

3. Push your code to the repository:
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   git branch -M main
   git remote add origin https://gitlab.com/USERNAME/REPO.git
   git push -u origin main
   ```

## 6. Testing the Pipeline

1. Make a change to your code
2. Commit and push to the main branch
3. Monitor the pipeline in GitLab CI/CD interface
4. Check Cloud Run console for the deployed service

## 7. Best Practices

1. Environment Configuration
   - Use GitLab environments for different deployment stages
   - Store sensitive data in GitLab CI/CD variables
   - Use environment variables for configuration

2. Pipeline Optimization
   - Use GitLab's cache feature for dependencies
   - Implement parallel jobs where possible
   - Use GitLab artifacts wisely

3. Security
   - Rotate service account keys regularly
   - Use protected branches and tags
   - Implement review apps for feature branches

4. Monitoring and Logging
   - Enable Spring Boot Actuator endpoints
   - Configure Cloud Monitoring
   - Set up Cloud Logging

5. Performance
   - Configure appropriate memory limits
   - Enable auto-scaling
   - Optimize container image size
   - Use GitLab's pipeline caching effectively
