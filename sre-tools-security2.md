# SRE/DevOps: Tools, Security & Scripting
## Infrastructure, CI/CD, Security & Practical Examples

---

# 1. INFRASTRUCTURE AS CODE (IaC)

---

## 1.1 Terraform

### Core Concepts

```hcl
# Terraform workflow
terraform init      # Download providers, initialize backend
terraform plan      # Preview changes (dry-run)
terraform apply     # Apply changes
terraform destroy   # Tear down infrastructure

# State management
# State tracks real infrastructure vs config
# NEVER edit state manually
# Use remote state for teams
```

### Essential Patterns

```hcl
# Provider configuration
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  # Remote state (critical for teams)
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/infrastructure.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"  # State locking
  }
}

# Variables with validation
variable "environment" {
  type        = string
  description = "Environment name"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_type" {
  type    = string
  default = "t3.medium"
}

# Local values (computed variables)
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
    Project     = "myproject"
  }
  
  name_prefix = "${var.project}-${var.environment}"
}

# Data sources (read existing resources)
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

data "aws_vpc" "main" {
  tags = {
    Name = "main-vpc"
  }
}

# Resource with lifecycle rules
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  subnet_id     = data.aws_vpc.main.id
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-app"
  })
  
  lifecycle {
    create_before_destroy = true  # Zero-downtime updates
    prevent_destroy       = true  # Prevent accidental deletion
    
    ignore_changes = [
      tags["LastModified"],  # Ignore external tag changes
    ]
  }
}

# Outputs
output "instance_ip" {
  value       = aws_instance.app.public_ip
  description = "Public IP of the app server"
}
```

### Modules - Reusable Components

```hcl
# Module structure
# modules/
#   vpc/
#     main.tf
#     variables.tf
#     outputs.tf

# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  
  tags = {
    Name = var.name
  }
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnets)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnets[count.index]
  availability_zone = var.azs[count.index]
  
  map_public_ip_on_launch = true
}

# modules/vpc/variables.tf
variable "name" {
  type = string
}

variable "cidr_block" {
  type    = string
  default = "10.0.0.0/16"
}

variable "public_subnets" {
  type    = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "azs" {
  type = list(string)
}

# modules/vpc/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

# Using the module
module "vpc" {
  source = "./modules/vpc"
  
  name           = "production-vpc"
  cidr_block     = "10.0.0.0/16"
  public_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  azs            = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# Reference module outputs
resource "aws_instance" "app" {
  subnet_id = module.vpc.public_subnet_ids[0]
  # ...
}
```

### FAQ: Terraform state is corrupted/out of sync

```bash
# View current state
terraform state list
terraform state show aws_instance.app

# Remove resource from state (doesn't destroy real resource)
terraform state rm aws_instance.app

# Import existing resource into state
terraform import aws_instance.app i-1234567890abcdef0

# Move resource in state (rename)
terraform state mv aws_instance.app aws_instance.web

# Force unlock state (use carefully)
terraform force-unlock LOCK_ID

# Refresh state from real infrastructure
terraform refresh  # deprecated, use:
terraform apply -refresh-only

# Recover from corrupted state
# 1. Check for backup: terraform.tfstate.backup
# 2. Use remote state versioning (S3 versioning)
# 3. Worst case: terraform import all resources
```

### FAQ: How to structure Terraform for multiple environments?

```
Option 1: Workspaces (simple)
terraform workspace new staging
terraform workspace new prod
terraform workspace select prod
# Same code, different state files
# Use: terraform.workspace in conditionals

Option 2: Directory per environment (recommended)
environments/
├── dev/
│   ├── main.tf
│   ├── terraform.tfvars
│   └── backend.tf
├── staging/
│   └── ...
└── prod/
    └── ...
modules/
├── vpc/
├── eks/
└── rds/

# Each environment references shared modules
# Different tfvars per environment
# Separate state files

Option 3: Terragrunt (DRY)
# Wrapper that reduces duplication
# Handles remote state configuration
# Good for large organizations
```

---

## 1.2 CloudFormation (AWS)

### Template Structure

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Production web application stack'

# Parameters - inputs
Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]
    Default: dev
  
  InstanceType:
    Type: String
    Default: t3.medium
    AllowedValues: [t3.small, t3.medium, t3.large]

# Mappings - static lookup tables
Mappings:
  RegionAMI:
    us-east-1:
      HVM64: ami-0123456789abcdef0
    us-west-2:
      HVM64: ami-0fedcba9876543210

# Conditions
Conditions:
  IsProd: !Equals [!Ref Environment, prod]
  CreateReadReplica: !Equals [!Ref Environment, prod]

# Resources
Resources:
  # VPC
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-vpc'

  # EC2 Instance
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap [RegionAMI, !Ref 'AWS::Region', HVM64]
      InstanceType: !If [IsProd, t3.large, !Ref InstanceType]
      SubnetId: !Ref PublicSubnet
      SecurityGroupIds:
        - !Ref WebSecurityGroup
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-web'
    
    # Creation policy - wait for signal
    CreationPolicy:
      ResourceSignal:
        Timeout: PT15M

  # Auto Scaling Group
  ASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !GetAtt LaunchTemplate.LatestVersionNumber
      MinSize: !If [IsProd, 2, 1]
      MaxSize: !If [IsProd, 10, 3]
      DesiredCapacity: !If [IsProd, 2, 1]
      VPCZoneIdentifier:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
      TargetGroupARNs:
        - !Ref ALBTargetGroup
    
    # Rolling update policy
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: 1
        MaxBatchSize: 1
        PauseTime: PT5M
        WaitOnResourceSignals: true

# Outputs
Outputs:
  VPCId:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub '${Environment}-VPCId'
  
  WebServerURL:
    Description: URL of the web server
    Value: !Sub 'http://${ALB.DNSName}'
```

### Cross-Stack References

```yaml
# Stack A - exports VPC
Outputs:
  VPCId:
    Value: !Ref VPC
    Export:
      Name: shared-vpc-id

# Stack B - imports VPC
Resources:
  Instance:
    Type: AWS::EC2::Instance
    Properties:
      SubnetId: !ImportValue shared-vpc-id
```

### FAQ: CloudFormation vs Terraform?

```
CloudFormation:
+ Native AWS, no state file to manage
+ Deep AWS integration, same-day feature support
+ Drift detection built-in
+ StackSets for multi-account/region
- AWS only
- YAML/JSON can be verbose
- Slower rollback on failures

Terraform:
+ Multi-cloud (AWS, GCP, Azure, etc.)
+ HCL is more readable
+ Faster plan/apply cycle
+ Better module ecosystem
- State file management required
- New AWS features may lag
- State locking needs separate setup

Recommendation:
- AWS-only shop: Either works, CFN simpler operationally
- Multi-cloud: Terraform
- Complex infra: Terraform (better abstractions)
```

---

## 1.3 Ansible

### Playbook Structure

```yaml
# site.yml - main playbook
---
- name: Configure web servers
  hosts: webservers
  become: yes
  vars:
    http_port: 80
    app_version: "1.2.3"
  
  vars_files:
    - vars/{{ env }}.yml
  
  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"
  
  roles:
    - common
    - nginx
    - app
  
  tasks:
    - name: Ensure app is running
      systemd:
        name: myapp
        state: started
        enabled: yes
  
  handlers:
    - name: Restart nginx
      systemd:
        name: nginx
        state: restarted

# Inventory file (inventory/prod)
[webservers]
web1.example.com
web2.example.com

[dbservers]
db1.example.com

[prod:children]
webservers
dbservers

[prod:vars]
env=production
ansible_user=deploy
```

### Common Modules

```yaml
# Package management
- name: Install packages
  apt:
    name:
      - nginx
      - python3
      - htop
    state: present

- name: Install specific version
  apt:
    name: nginx=1.18.0-0ubuntu1
    state: present

# File operations
- name: Copy config file
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
  notify: Restart nginx

- name: Ensure directory exists
  file:
    path: /opt/myapp
    state: directory
    owner: deploy
    mode: '0755'

# Service management
- name: Start and enable service
  systemd:
    name: nginx
    state: started
    enabled: yes
    daemon_reload: yes

# Command execution
- name: Run database migration
  command: /opt/myapp/migrate.sh
  args:
    chdir: /opt/myapp
  register: migration_result
  changed_when: "'Migrations applied' in migration_result.stdout"

# User management
- name: Create app user
  user:
    name: deploy
    groups: www-data
    shell: /bin/bash
    create_home: yes

# Git operations
- name: Clone repository
  git:
    repo: https://github.com/company/app.git
    dest: /opt/myapp
    version: "{{ app_version }}"
  notify: Restart app
```

### Roles Structure

```
roles/
└── nginx/
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── templates/
    │   └── nginx.conf.j2
    ├── files/
    │   └── ssl-cert.pem
    ├── vars/
    │   └── main.yml
    ├── defaults/
    │   └── main.yml
    └── meta/
        └── main.yml

# roles/nginx/tasks/main.yml
---
- name: Install nginx
  apt:
    name: nginx
    state: present

- name: Configure nginx
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: Restart nginx

- name: Enable site
  file:
    src: /etc/nginx/sites-available/default
    dest: /etc/nginx/sites-enabled/default
    state: link
  notify: Reload nginx

# roles/nginx/handlers/main.yml
---
- name: Restart nginx
  systemd:
    name: nginx
    state: restarted

- name: Reload nginx
  systemd:
    name: nginx
    state: reloaded

# roles/nginx/defaults/main.yml
---
nginx_worker_processes: auto
nginx_worker_connections: 1024
```

### Jinja2 Templates

```jinja2
{# templates/nginx.conf.j2 #}
worker_processes {{ nginx_worker_processes }};

events {
    worker_connections {{ nginx_worker_connections }};
}

http {
    upstream backend {
        {% for host in groups['appservers'] %}
        server {{ hostvars[host]['ansible_host'] }}:8080;
        {% endfor %}
    }
    
    server {
        listen {{ http_port }};
        server_name {{ server_name | default('_') }};
        
        {% if ssl_enabled | default(false) %}
        listen 443 ssl;
        ssl_certificate /etc/ssl/certs/{{ ssl_cert }};
        ssl_certificate_key /etc/ssl/private/{{ ssl_key }};
        {% endif %}
        
        location / {
            proxy_pass http://backend;
        }
    }
}
```

### FAQ: Ansible vs Terraform?

```
Ansible:
- Configuration management (what's ON the server)
- Procedural (do this, then this)
- Agentless (SSH)
- Good for: App deployment, config, orchestration

Terraform:
- Infrastructure provisioning (the server itself)
- Declarative (desired end state)
- API-based
- Good for: Cloud resources, infrastructure

Common pattern: Use both
1. Terraform creates infrastructure
2. Ansible configures it

# Terraform outputs inventory for Ansible
output "web_servers" {
  value = aws_instance.web[*].public_ip
}

# Or use dynamic inventory
# ansible-inventory -i aws_ec2.yml --list
```

---

# 2. CI/CD PIPELINES

---

## 2.1 Jenkins

### Declarative Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: maven
                image: maven:3.8-openjdk-11
                command: ['sleep', 'infinity']
              - name: docker
                image: docker:20-dind
                securityContext:
                  privileged: true
            '''
        }
    }
    
    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        APP_NAME = 'myapp'
        KUBECONFIG = credentials('kubeconfig-prod')
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'])
        booleanParam(name: 'SKIP_TESTS', defaultValue: false)
        string(name: 'VERSION', defaultValue: '', description: 'Version to deploy')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                    env.VERSION = params.VERSION ?: env.GIT_COMMIT_SHORT
                }
            }
        }
        
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }
        
        stage('Test') {
            when {
                expression { !params.SKIP_TESTS }
            }
            parallel {
                stage('Unit Tests') {
                    steps {
                        container('maven') {
                            sh 'mvn test'
                        }
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }
                }
                stage('Integration Tests') {
                    steps {
                        container('maven') {
                            sh 'mvn verify -Pintegration'
                        }
                    }
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                container('maven') {
                    sh 'mvn dependency-check:check'
                }
            }
            post {
                always {
                    dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                container('docker') {
                    script {
                        docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-creds') {
                            def image = docker.build("${APP_NAME}:${VERSION}")
                            image.push()
                            image.push('latest')
                        }
                    }
                }
            }
        }
        
        stage('Deploy to Dev') {
            when {
                branch 'develop'
            }
            steps {
                deployToK8s('dev')
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                deployToK8s('staging')
            }
        }
        
        stage('Deploy to Production') {
            when {
                allOf {
                    branch 'main'
                    expression { params.ENVIRONMENT == 'prod' }
                }
            }
            steps {
                // Manual approval
                timeout(time: 1, unit: 'HOURS') {
                    input message: 'Deploy to production?', ok: 'Deploy'
                }
                deployToK8s('prod')
            }
        }
    }
    
    post {
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "✅ ${APP_NAME} ${VERSION} deployed to ${params.ENVIRONMENT}"
            )
        }
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: "❌ ${APP_NAME} build failed: ${BUILD_URL}"
            )
        }
        always {
            cleanWs()
        }
    }
}

// Shared function
def deployToK8s(String env) {
    container('kubectl') {
        sh """
            kubectl set image deployment/${APP_NAME} \
                ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${VERSION} \
                -n ${env}
            kubectl rollout status deployment/${APP_NAME} -n ${env} --timeout=5m
        """
    }
}
```

### Shared Libraries

```groovy
// vars/deployToKubernetes.groovy (in shared library repo)
def call(Map config) {
    def namespace = config.namespace
    def appName = config.appName
    def image = config.image
    def timeout = config.timeout ?: '5m'
    
    sh """
        kubectl set image deployment/${appName} ${appName}=${image} -n ${namespace}
        kubectl rollout status deployment/${appName} -n ${namespace} --timeout=${timeout}
    """
}

// Usage in Jenkinsfile
@Library('my-shared-library') _

pipeline {
    stages {
        stage('Deploy') {
            steps {
                deployToKubernetes(
                    namespace: 'production',
                    appName: 'myapp',
                    image: 'registry/myapp:1.0.0'
                )
            }
        }
    }
}
```

### FAQ: Jenkins pipeline best practices?

```
1. Use declarative pipelines (not scripted)
   - Cleaner syntax
   - Better visualization
   - Easier to maintain

2. Keep Jenkinsfile in repo (Pipeline as Code)
   - Version controlled
   - Code review for pipeline changes
   - Same branching strategy as code

3. Use shared libraries for common code
   - DRY principle
   - Centralized updates
   - Tested once, used everywhere

4. Parallelize where possible
   parallel {
       stage('Test A') { ... }
       stage('Test B') { ... }
   }

5. Fail fast
   options {
       timeout(time: 30, unit: 'MINUTES')
   }

6. Clean workspace
   post {
       always {
           cleanWs()
       }
   }

7. Use credentials properly
   environment {
       AWS_CREDS = credentials('aws-creds')
   }
```

---

## 2.2 GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - security
  - deploy

variables:
  DOCKER_REGISTRY: registry.gitlab.com
  APP_NAME: myapp

# Templates for reuse
.docker_build: &docker_build
  image: docker:20
  services:
    - docker:20-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

# Build stage
build:
  stage: build
  <<: *docker_build
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  rules:
    - if: $CI_COMMIT_BRANCH

# Test stage
unit_tests:
  stage: test
  image: python:3.9
  script:
    - pip install -r requirements.txt
    - pytest tests/ --junitxml=report.xml
  artifacts:
    reports:
      junit: report.xml
  coverage: '/TOTAL.+ ([0-9]{1,3}%)/'

integration_tests:
  stage: test
  image: python:3.9
  services:
    - postgres:13
  variables:
    POSTGRES_DB: test
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    DATABASE_URL: postgresql://test:test@postgres/test
  script:
    - pip install -r requirements.txt
    - pytest tests/integration/ --junitxml=integration-report.xml

# Security scanning
sast:
  stage: security
  image: returntocorp/semgrep
  script:
    - semgrep --config=auto --json --output=semgrep.json .
  artifacts:
    reports:
      sast: semgrep.json
  allow_failure: true

container_scan:
  stage: security
  image: aquasec/trivy
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  allow_failure: true

# Deploy stages
deploy_staging:
  stage: deploy
  image: bitnami/kubectl
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - kubectl set image deployment/$APP_NAME $APP_NAME=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA -n staging
    - kubectl rollout status deployment/$APP_NAME -n staging
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

deploy_production:
  stage: deploy
  image: bitnami/kubectl
  environment:
    name: production
    url: https://example.com
  script:
    - kubectl set image deployment/$APP_NAME $APP_NAME=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA -n production
    - kubectl rollout status deployment/$APP_NAME -n production
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
  needs:
    - deploy_staging
```

---

## 2.3 GitHub Actions

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov
      
      - name: Run tests
        run: pytest --cov=app --cov-report=xml
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost/postgres
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'

  build-image:
    needs: [build-and-test, security-scan]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build-image
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: staging
    
    steps:
      - name: Deploy to staging
        uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBE_CONFIG_STAGING }}
      
      - name: Deploy
        run: |
          kubectl set image deployment/myapp myapp=${{ needs.build-image.outputs.image-tag }} -n staging
          kubectl rollout status deployment/myapp -n staging

  deploy-production:
    needs: [build-image, deploy-staging]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: 
      name: production
      url: https://example.com
    
    steps:
      - name: Deploy to production
        uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBE_CONFIG_PROD }}
      
      - name: Deploy
        run: |
          kubectl set image deployment/myapp myapp=${{ needs.build-image.outputs.image-tag }} -n production
          kubectl rollout status deployment/myapp -n production
```

---

## 2.4 Artifactory / Nexus (Artifact Management)

### Artifactory Concepts

```
Repository Types:
├── Local: Store your artifacts
├── Remote: Proxy to external repos (Maven Central, npm, PyPI)
└── Virtual: Aggregate multiple repos under one URL

Common Repositories:
├── maven-local, maven-remote, maven-virtual
├── docker-local, docker-remote, docker-virtual
├── npm-local, npm-remote, npm-virtual
└── generic-local (for arbitrary files)
```

### Integration Examples

```groovy
// Jenkins + Artifactory
pipeline {
    stages {
        stage('Build') {
            steps {
                rtMavenDeployer(
                    id: 'deployer',
                    serverId: 'artifactory',
                    releaseRepo: 'libs-release-local',
                    snapshotRepo: 'libs-snapshot-local'
                )
                
                rtMavenRun(
                    tool: 'maven-3.8',
                    pom: 'pom.xml',
                    goals: 'clean install',
                    deployerId: 'deployer'
                )
            }
        }
        
        stage('Publish Build Info') {
            steps {
                rtPublishBuildInfo(serverId: 'artifactory')
            }
        }
    }
}
```

```yaml
# Docker with Artifactory
# Pull through Artifactory (caching)
docker pull myartifactory.com/docker-remote/nginx:latest

# Push to Artifactory
docker tag myapp:1.0 myartifactory.com/docker-local/myapp:1.0
docker push myartifactory.com/docker-local/myapp:1.0

# .npmrc for npm
registry=https://myartifactory.com/artifactory/api/npm/npm-virtual/
//myartifactory.com/artifactory/api/npm/:_authToken=${NPM_TOKEN}

# pip.conf for Python
[global]
index-url = https://myartifactory.com/artifactory/api/pypi/pypi-virtual/simple
```

### FAQ: Why use artifact repository?

```
Benefits:
1. Caching: Faster builds, reduced external dependency
2. Security: Scan artifacts, control what enters your org
3. Reliability: Available even if upstream is down
4. Traceability: Know exactly what went into production
5. Governance: Retention policies, access control
6. Promotion: Move artifacts through environments

Best practices:
- Use virtual repos (single URL, multiple sources)
- Enable vulnerability scanning
- Set retention policies
- Use immutable versions (no overwriting)
- Integrate with CI/CD for build info
```

---

# 3. SECURITY

---

## 3.1 Security Fundamentals

### Defense in Depth

```
                    Internet
                        │
                  ┌─────▼─────┐
                  │   WAF     │  Layer 7 protection
                  └─────┬─────┘
                        │
                  ┌─────▼─────┐
                  │ DDoS      │  Volumetric attack protection
                  │ Protection│
                  └─────┬─────┘
                        │
                  ┌─────▼─────┐
                  │ Load      │  SSL termination
                  │ Balancer  │
                  └─────┬─────┘
                        │
            ┌───────────┴───────────┐
            │    Public Subnet      │
            │  ┌─────────────────┐  │
            │  │   Web Servers   │  │
            │  │ (Reverse Proxy) │  │
            │  └────────┬────────┘  │
            │           │           │
     ─ ─ ─ ┼ ─ ─ ─ ─ ─ ┼ ─ ─ ─ ─ ─ ┼ ─ ─  Security Group
            │           │           │
            │   Private Subnet      │
            │  ┌────────▼────────┐  │
            │  │  App Servers    │  │
            │  └────────┬────────┘  │
            └───────────┼───────────┘
                        │
            ┌───────────┴───────────┐
            │   Database Subnet     │
            │  ┌─────────────────┐  │
            │  │   Databases     │  │
            │  │ (Encrypted)     │  │
            │  └─────────────────┘  │
            └───────────────────────┘
```

### Security Groups vs NACLs

```
Security Groups (Stateful):
- Instance/ENI level
- Allow rules only (implicit deny)
- Stateful: Return traffic automatic
- Easier to manage

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]  # From ALB only
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

NACLs (Stateless):
- Subnet level
- Allow AND deny rules
- Stateless: Must allow return traffic explicitly
- Rule order matters (lowest number first)
- Use for: Blocking known bad IPs, subnet isolation
```

## 3.2 Secrets Management

### HashiCorp Vault

```bash
# Start Vault (dev mode for testing)
vault server -dev

# Authenticate
export VAULT_ADDR='http://127.0.0.1:8200'
vault login <token>

# Store secret
vault kv put secret/myapp/database \
    username="dbuser" \
    password="secretpassword"

# Read secret
vault kv get secret/myapp/database
vault kv get -field=password secret/myapp/database

# Dynamic secrets (database)
vault secrets enable database

vault write database/config/mydb \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(mysql:3306)/" \
    allowed_roles="readonly" \
    username="vault" \
    password="vaultpass"

vault write database/roles/readonly \
    db_name=mydb \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}'; GRANT SELECT ON *.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"

# Get dynamic credentials
vault read database/creds/readonly
# Returns: username + password with 1h TTL
```

### AWS Secrets Manager

```python
import boto3
import json

def get_secret(secret_name):
    client = boto3.client('secretsmanager')
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response['SecretString'])

# Usage
db_creds = get_secret('prod/myapp/database')
connection_string = f"postgresql://{db_creds['username']}:{db_creds['password']}@{db_creds['host']}/{db_creds['database']}"

# Terraform
resource "aws_secretsmanager_secret" "db" {
  name = "prod/myapp/database"
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id = aws_secretsmanager_secret.db.id
  secret_string = jsonencode({
    username = "dbuser"
    password = random_password.db.result
    host     = aws_db_instance.main.endpoint
    database = "myapp"
  })
}
```

### Kubernetes Secrets

```yaml
# Create secret
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:  # Plain text (encoded on creation)
  username: dbuser
  password: secretpassword

---
# Use in Pod
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: myapp
    env:
      - name: DB_USERNAME
        valueFrom:
          secretKeyRef:
            name: db-credentials
            key: username
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef:
            name: db-credentials
            key: password
    
    # Or mount as files
    volumeMounts:
      - name: secrets
        mountPath: /etc/secrets
        readOnly: true
  
  volumes:
    - name: secrets
      secret:
        secretName: db-credentials

# External Secrets (sync from Vault/AWS SM)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-credentials
  data:
    - secretKey: username
      remoteRef:
        key: prod/myapp/database
        property: username
    - secretKey: password
      remoteRef:
        key: prod/myapp/database
        property: password
```

## 3.3 Security Scanning

### Container Security

```yaml
# Trivy - vulnerability scanner
# Scan image
trivy image myregistry/myapp:1.0

# Scan filesystem
trivy fs .

# Scan in CI/CD (fail on HIGH/CRITICAL)
trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:1.0

# In Kubernetes (Trivy Operator)
apiVersion: aquasecurity.github.io/v1alpha1
kind: VulnerabilityReport
metadata:
  name: deployment-myapp
# Auto-generated reports for all workloads
```

### SAST (Static Application Security Testing)

```yaml
# Semgrep example
# Run locally
semgrep --config=auto .

# Custom rule
rules:
  - id: hardcoded-password
    patterns:
      - pattern: password = "..."
    message: "Hardcoded password found"
    severity: ERROR
    languages: [python]

# GitLab CI integration
sast:
  stage: test
  image: returntocorp/semgrep
  script:
    - semgrep --config=auto --json -o semgrep.json .
  artifacts:
    reports:
      sast: semgrep.json
```

### Dependency Scanning

```bash
# Python - safety
pip install safety
safety check -r requirements.txt

# Node.js - npm audit
npm audit
npm audit fix

# Java - OWASP Dependency Check
mvn dependency-check:check

# Snyk (multi-language)
snyk test
snyk container test myimage:tag
snyk iac test terraform/
```

## 3.4 Network Security

### Web Application Firewall (WAF)

```hcl
# AWS WAF with common rules
resource "aws_wafv2_web_acl" "main" {
  name  = "main-waf"
  scope = "REGIONAL"
  
  default_action {
    allow {}
  }
  
  # AWS Managed Rules - Common
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1
    
    override_action {
      none {}
    }
    
    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesCommonRuleSet"
      }
    }
    
    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSCommonRules"
    }
  }
  
  # SQL Injection Protection
  rule {
    name     = "AWSManagedRulesSQLiRuleSet"
    priority = 2
    
    override_action {
      none {}
    }
    
    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesSQLiRuleSet"
      }
    }
    
    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSSQLiRules"
    }
  }
  
  # Rate limiting
  rule {
    name     = "RateLimit"
    priority = 3
    
    action {
      block {}
    }
    
    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }
    
    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimit"
    }
  }
  
  visibility_config {
    sampled_requests_enabled   = true
    cloudwatch_metrics_enabled = true
    metric_name                = "MainWAF"
  }
}
```

### mTLS (Mutual TLS)

```yaml
# Istio mTLS (service mesh)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # Require mTLS for all services

---
# Destination rule
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: default
  namespace: production
spec:
  host: "*.production.svc.cluster.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

## 3.5 Security FAQ

### FAQ: How to handle secrets in CI/CD?

```
DO:
✓ Use CI/CD secret variables (masked in logs)
✓ Use external secret managers (Vault, AWS SM)
✓ Rotate secrets regularly
✓ Use short-lived credentials (OIDC, IAM roles)
✓ Audit secret access

DON'T:
✗ Commit secrets to git (even private repos)
✗ Echo secrets in scripts
✗ Pass secrets as command arguments (visible in ps)
✗ Store secrets in container images
✗ Use long-lived API keys

# GitHub Actions OIDC (no long-lived secrets)
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v2
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions
    aws-region: us-east-1
    # No access keys needed!
```

### FAQ: Container security best practices?

```dockerfile
# Dockerfile security
# 1. Use specific version, not latest
FROM python:3.9-slim-bullseye

# 2. Don't run as root
RUN useradd -r -s /bin/false appuser

# 3. Copy only needed files
COPY --chown=appuser:appuser requirements.txt .
COPY --chown=appuser:appuser app/ ./app/

# 4. Install dependencies first (layer caching)
RUN pip install --no-cache-dir -r requirements.txt

# 5. Remove unnecessary packages
RUN apt-get purge -y --auto-remove gcc \
    && rm -rf /var/lib/apt/lists/*

# 6. Use non-root user
USER appuser

# 7. Don't store secrets in image
# Use runtime env vars or mounted secrets

# 8. Set read-only filesystem where possible
# In K8s: securityContext.readOnlyRootFilesystem: true

# Runtime security (Kubernetes)
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

---

# 4. SCRIPTING & AUTOMATION

---

## 4.1 Bash Scripting

### Production-Ready Script Template

```bash
#!/usr/bin/env bash
#
# Script: deploy.sh
# Description: Deploy application to Kubernetes
# Usage: ./deploy.sh -e <environment> -v <version>
#

set -euo pipefail  # Exit on error, undefined vars, pipe failures
IFS=$'\n\t'        # Safer word splitting

# Constants
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "${BASH_SOURCE[0]}")"
readonly LOG_FILE="/var/log/${SCRIPT_NAME%.*}.log"

# Defaults
ENVIRONMENT=""
VERSION=""
DRY_RUN=false
VERBOSE=false

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Logging functions
log() {
    local level="$1"
    shift
    local message="$*"
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo -e "${timestamp} [${level}] ${message}" | tee -a "${LOG_FILE}"
}

info()  { log "INFO" "$*"; }
warn()  { log "${YELLOW}WARN${NC}" "$*"; }
error() { log "${RED}ERROR${NC}" "$*" >&2; }
success() { log "${GREEN}SUCCESS${NC}" "$*"; }

# Usage information
usage() {
    cat << EOF
Usage: ${SCRIPT_NAME} [OPTIONS]

Deploy application to specified environment.

Options:
    -e, --environment   Target environment (dev|staging|prod) [required]
    -v, --version       Version to deploy [required]
    -d, --dry-run       Show what would be done without making changes
    -V, --verbose       Enable verbose output
    -h, --help          Show this help message

Examples:
    ${SCRIPT_NAME} -e staging -v 1.2.3
    ${SCRIPT_NAME} --environment prod --version 1.2.3 --dry-run

EOF
    exit 1
}

# Cleanup function (runs on exit)
cleanup() {
    local exit_code=$?
    if [[ ${exit_code} -ne 0 ]]; then
        error "Script failed with exit code ${exit_code}"
    fi
    # Add cleanup tasks here (temp files, locks, etc.)
    exit ${exit_code}
}
trap cleanup EXIT

# Parse arguments
parse_args() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -e|--environment)
                ENVIRONMENT="$2"
                shift 2
                ;;
            -v|--version)
                VERSION="$2"
                shift 2
                ;;
            -d|--dry-run)
                DRY_RUN=true
                shift
                ;;
            -V|--verbose)
                VERBOSE=true
                set -x  # Enable debug output
                shift
                ;;
            -h|--help)
                usage
                ;;
            *)
                error "Unknown option: $1"
                usage
                ;;
        esac
    done
}

# Validate arguments
validate_args() {
    local valid=true
    
    if [[ -z "${ENVIRONMENT}" ]]; then
        error "Environment is required"
        valid=false
    elif [[ ! "${ENVIRONMENT}" =~ ^(dev|staging|prod)$ ]]; then
        error "Invalid environment: ${ENVIRONMENT}"
        valid=false
    fi
    
    if [[ -z "${VERSION}" ]]; then
        error "Version is required"
        valid=false
    elif [[ ! "${VERSION}" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
        error "Invalid version format: ${VERSION} (expected: X.Y.Z)"
        valid=false
    fi
    
    if [[ "${valid}" == "false" ]]; then
        echo
        usage
    fi
}

# Check prerequisites
check_prerequisites() {
    info "Checking prerequisites..."
    
    local missing=()
    
    for cmd in kubectl docker jq; do
        if ! command -v "${cmd}" &> /dev/null; then
            missing+=("${cmd}")
        fi
    done
    
    if [[ ${#missing[@]} -gt 0 ]]; then
        error "Missing required commands: ${missing[*]}"
        exit 1
    fi
    
    # Check Kubernetes connectivity
    if ! kubectl cluster-info &> /dev/null; then
        error "Cannot connect to Kubernetes cluster"
        exit 1
    fi
    
    info "Prerequisites check passed"
}

# Run command with dry-run support
run_cmd() {
    if [[ "${DRY_RUN}" == "true" ]]; then
        info "[DRY-RUN] Would run: $*"
    else
        info "Running: $*"
        "$@"
    fi
}

# Main deployment function
deploy() {
    info "Starting deployment of version ${VERSION} to ${ENVIRONMENT}"
    
    local namespace="${ENVIRONMENT}"
    local image="myregistry.com/myapp:${VERSION}"
    
    # Check if image exists
    info "Verifying image exists..."
    if ! docker manifest inspect "${image}" &> /dev/null; then
        error "Image not found: ${image}"
        exit 1
    fi
    
    # Backup current deployment
    info "Backing up current deployment..."
    run_cmd kubectl get deployment myapp -n "${namespace}" -o yaml > "/tmp/myapp-backup-$(date +%s).yaml"
    
    # Update deployment
    info "Updating deployment..."
    run_cmd kubectl set image deployment/myapp myapp="${image}" -n "${namespace}"
    
    # Wait for rollout
    info "Waiting for rollout to complete..."
    if [[ "${DRY_RUN}" != "true" ]]; then
        if kubectl rollout status deployment/myapp -n "${namespace}" --timeout=5m; then
            success "Deployment completed successfully"
        else
            error "Rollout failed, initiating rollback..."
            kubectl rollout undo deployment/myapp -n "${namespace}"
            exit 1
        fi
    fi
    
    # Verify deployment
    info "Verifying deployment..."
    local ready
    ready=$(kubectl get deployment myapp -n "${namespace}" -o jsonpath='{.status.readyReplicas}')
    local desired
    desired=$(kubectl get deployment myapp -n "${namespace}" -o jsonpath='{.spec.replicas}')
    
    if [[ "${ready}" == "${desired}" ]]; then
        success "All ${ready} replicas are ready"
    else
        warn "Only ${ready}/${desired} replicas are ready"
    fi
}

# Main execution
main() {
    parse_args "$@"
    validate_args
    check_prerequisites
    deploy
    success "Deployment completed!"
}

main "$@"
```

### Common Bash Patterns

```bash
# Check if command exists
command_exists() {
    command -v "$1" &> /dev/null
}

# Retry with backoff
retry() {
    local max_attempts="${1:-3}"
    local delay="${2:-5}"
    local cmd="${@:3}"
    local attempt=1
    
    until ${cmd}; do
        if (( attempt >= max_attempts )); then
            error "Command failed after ${max_attempts} attempts: ${cmd}"
            return 1
        fi
        
        warn "Attempt ${attempt} failed. Retrying in ${delay}s..."
        sleep "${delay}"
        ((attempt++))
        delay=$((delay * 2))  # Exponential backoff
    done
}

# Usage: retry 3 5 curl -f http://example.com/health

# Parallel execution
parallel_exec() {
    local pids=()
    
    for host in "${HOSTS[@]}"; do
        (
            ssh "${host}" "sudo systemctl restart nginx"
        ) &
        pids+=($!)
    done
    
    # Wait for all and check exit codes
    local failed=0
    for pid in "${pids[@]}"; do
        if ! wait "${pid}"; then
            ((failed++))
        fi
    done
    
    return ${failed}
}

# Lock file (prevent concurrent execution)
acquire_lock() {
    local lockfile="/var/run/${SCRIPT_NAME}.lock"
    exec 200>"${lockfile}"
    
    if ! flock -n 200; then
        error "Another instance is running"
        exit 1
    fi
    
    echo $$ >&200  # Write PID
}

# Temporary directory with cleanup
TMPDIR=$(mktemp -d)
trap 'rm -rf "${TMPDIR}"' EXIT

# Read config file
read_config() {
    local config_file="$1"
    
    if [[ ! -f "${config_file}" ]]; then
        error "Config file not found: ${config_file}"
        exit 1
    fi
    
    # Source bash config
    source "${config_file}"
    
    # Or parse key=value
    while IFS='=' read -r key value; do
        [[ -z "$key" || "$key" =~ ^# ]] && continue
        declare -g "${key}=${value}"
    done < "${config_file}"
}

# JSON parsing with jq
parse_json() {
    local json_file="$1"
    local query="$2"
    
    jq -r "${query}" "${json_file}"
}

# Example: parse_json config.json '.database.host'

# Array operations
declare -a SERVERS=("web1" "web2" "web3")

# Loop
for server in "${SERVERS[@]}"; do
    echo "Processing ${server}"
done

# Check if in array
in_array() {
    local needle="$1"
    shift
    local item
    for item in "$@"; do
        [[ "${item}" == "${needle}" ]] && return 0
    done
    return 1
}

# String operations
string="hello-world-test"

# Split by delimiter
IFS='-' read -ra parts <<< "${string}"
echo "${parts[0]}"  # hello

# Substring
echo "${string:0:5}"   # hello
echo "${string#*-}"    # world-test (remove prefix)
echo "${string%-*}"    # hello-world (remove suffix)
echo "${string//-/_}"  # hello_world_test (replace all)
```

## 4.2 Python Scripting

### Production Script Template

```python
#!/usr/bin/env python3
"""
Deploy application to Kubernetes.

Usage:
    deploy.py -e <environment> -v <version>
    deploy.py --help

Examples:
    deploy.py -e staging -v 1.2.3
    deploy.py --environment prod --version 1.2.3 --dry-run
"""

import argparse
import logging
import subprocess
import sys
import json
from datetime import datetime
from pathlib import Path
from typing import Optional, List, Dict, Any
from dataclasses import dataclass

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    handlers=[
        logging.StreamHandler(sys.stdout),
        logging.FileHandler('/var/log/deploy.log')
    ]
)
logger = logging.getLogger(__name__)


@dataclass
class DeploymentConfig:
    """Deployment configuration."""
    environment: str
    version: str
    namespace: str
    image_registry: str = "myregistry.com"
    app_name: str = "myapp"
    timeout: int = 300
    dry_run: bool = False
    
    @property
    def image(self) -> str:
        return f"{self.image_registry}/{self.app_name}:{self.version}"


class KubernetesClient:
    """Wrapper for kubectl commands."""
    
    def __init__(self, dry_run: bool = False):
        self.dry_run = dry_run
    
    def run(self, args: List[str], check: bool = True) -> subprocess.CompletedProcess:
        """Run kubectl command."""
        cmd = ["kubectl"] + args
        
        if self.dry_run:
            logger.info(f"[DRY-RUN] Would run: {' '.join(cmd)}")
            return subprocess.CompletedProcess(cmd, 0, "", "")
        
        logger.debug(f"Running: {' '.join(cmd)}")
        
        try:
            result = subprocess.run(
                cmd,
                capture_output=True,
                text=True,
                check=check
            )
            return result
        except subprocess.CalledProcessError as e:
            logger.error(f"Command failed: {e.stderr}")
            raise
    
    def get_deployment(self, name: str, namespace: str) -> Dict[str, Any]:
        """Get deployment details."""
        result = self.run([
            "get", "deployment", name,
            "-n", namespace,
            "-o", "json"
        ])
        return json.loads(result.stdout)
    
    def set_image(self, deployment: str, container: str, 
                  image: str, namespace: str) -> None:
        """Update deployment image."""
        self.run([
            "set", "image",
            f"deployment/{deployment}",
            f"{container}={image}",
            "-n", namespace
        ])
    
    def rollout_status(self, deployment: str, namespace: str, 
                       timeout: int) -> bool:
        """Wait for rollout to complete."""
        try:
            self.run([
                "rollout", "status",
                f"deployment/{deployment}",
                "-n", namespace,
                f"--timeout={timeout}s"
            ])
            return True
        except subprocess.CalledProcessError:
            return False
    
    def rollout_undo(self, deployment: str, namespace: str) -> None:
        """Rollback deployment."""
        self.run([
            "rollout", "undo",
            f"deployment/{deployment}",
            "-n", namespace
        ])


class Deployer:
    """Handle deployment logic."""
    
    def __init__(self, config: DeploymentConfig):
        self.config = config
        self.k8s = KubernetesClient(dry_run=config.dry_run)
    
    def validate_prerequisites(self) -> None:
        """Check all prerequisites before deployment."""
        logger.info("Validating prerequisites...")
        
        # Check kubectl connectivity
        try:
            self.k8s.run(["cluster-info"], check=True)
        except Exception as e:
            raise RuntimeError(f"Cannot connect to Kubernetes: {e}")
        
        # Verify image exists (would need Docker SDK or API call)
        logger.info(f"Verifying image: {self.config.image}")
        # Implementation depends on registry
        
        logger.info("Prerequisites validated")
    
    def backup_current(self) -> Path:
        """Backup current deployment state."""
        logger.info("Backing up current deployment...")
        
        deployment = self.k8s.get_deployment(
            self.config.app_name,
            self.config.namespace
        )
        
        backup_path = Path(f"/tmp/{self.config.app_name}-backup-{datetime.now():%Y%m%d%H%M%S}.json")
        backup_path.write_text(json.dumps(deployment, indent=2))
        
        logger.info(f"Backup saved to {backup_path}")
        return backup_path
    
    def deploy(self) -> bool:
        """Execute deployment."""
        logger.info(f"Starting deployment: {self.config.app_name} "
                   f"v{self.config.version} to {self.config.environment}")
        
        try:
            # Backup
            self.backup_current()
            
            # Update image
            logger.info(f"Updating image to {self.config.image}")
            self.k8s.set_image(
                self.config.app_name,
                self.config.app_name,
                self.config.image,
                self.config.namespace
            )
            
            # Wait for rollout
            logger.info("Waiting for rollout...")
            if not self.k8s.rollout_status(
                self.config.app_name,
                self.config.namespace,
                self.config.timeout
            ):
                raise RuntimeError("Rollout failed")
            
            # Verify
            self.verify_deployment()
            
            logger.info("Deployment completed successfully!")
            return True
            
        except Exception as e:
            logger.error(f"Deployment failed: {e}")
            logger.info("Initiating rollback...")
            self.k8s.rollout_undo(self.config.app_name, self.config.namespace)
            return False
    
    def verify_deployment(self) -> None:
        """Verify deployment health."""
        logger.info("Verifying deployment...")
        
        deployment = self.k8s.get_deployment(
            self.config.app_name,
            self.config.namespace
        )
        
        ready = deployment.get('status', {}).get('readyReplicas', 0)
        desired = deployment.get('spec', {}).get('replicas', 0)
        
        if ready == desired:
            logger.info(f"All {ready} replicas ready")
        else:
            logger.warning(f"Only {ready}/{desired} replicas ready")


def parse_args() -> argparse.Namespace:
    """Parse command line arguments."""
    parser = argparse.ArgumentParser(
        description="Deploy application to Kubernetes",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Examples:
    %(prog)s -e staging -v 1.2.3
    %(prog)s --environment prod --version 1.2.3 --dry-run
        """
    )
    
    parser.add_argument(
        '-e', '--environment',
        required=True,
        choices=['dev', 'staging', 'prod'],
        help='Target environment'
    )
    
    parser.add_argument(
        '-v', '--version',
        required=True,
        help='Version to deploy (format: X.Y.Z)'
    )
    
    parser.add_argument(
        '-d', '--dry-run',
        action='store_true',
        help='Show what would be done'
    )
    
    parser.add_argument(
        '--verbose',
        action='store_true',
        help='Enable verbose output'
    )
    
    return parser.parse_args()


def main():
    """Main entry point."""
    args = parse_args()
    
    if args.verbose:
        logging.getLogger().setLevel(logging.DEBUG)
    
    config = DeploymentConfig(
        environment=args.environment,
        version=args.version,
        namespace=args.environment,
        dry_run=args.dry_run
    )
    
    deployer = Deployer(config)
    
    try:
        deployer.validate_prerequisites()
        success = deployer.deploy()
        sys.exit(0 if success else 1)
        
    except KeyboardInterrupt:
        logger.info("Deployment cancelled by user")
        sys.exit(130)
    except Exception as e:
        logger.exception(f"Unexpected error: {e}")
        sys.exit(1)


if __name__ == '__main__':
    main()
```

### Useful Python Patterns

```python
import subprocess
import requests
import concurrent.futures
from functools import wraps
from time import sleep
import boto3

# Retry decorator with exponential backoff
def retry(max_attempts=3, base_delay=1, exceptions=(Exception,)):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts - 1:
                        raise
                    delay = base_delay * (2 ** attempt)
                    logger.warning(f"Attempt {attempt + 1} failed: {e}. Retrying in {delay}s")
                    sleep(delay)
        return wrapper
    return decorator

@retry(max_attempts=3, exceptions=(requests.RequestException,))
def fetch_url(url):
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    return response.json()

# Parallel execution
def parallel_health_check(hosts: list, timeout: int = 5) -> dict:
    """Check health of multiple hosts in parallel."""
    
    def check_host(host):
        try:
            response = requests.get(
                f"http://{host}/health",
                timeout=timeout
            )
            return host, response.status_code == 200
        except:
            return host, False
    
    results = {}
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        futures = {executor.submit(check_host, h): h for h in hosts}
        
        for future in concurrent.futures.as_completed(futures):
            host, healthy = future.result()
            results[host] = healthy
    
    return results

# AWS SDK example
def get_ec2_instances(filters: dict = None) -> list:
    """Get EC2 instances with optional filters."""
    ec2 = boto3.client('ec2')
    
    params = {}
    if filters:
        params['Filters'] = [
            {'Name': k, 'Values': v if isinstance(v, list) else [v]}
            for k, v in filters.items()
        ]
    
    instances = []
    paginator = ec2.get_paginator('describe_instances')
    
    for page in paginator.paginate(**params):
        for reservation in page['Reservations']:
            instances.extend(reservation['Instances'])
    
    return instances

# Usage
web_servers = get_ec2_instances({
    'tag:Environment': 'production',
    'instance-state-name': 'running'
})

# Context manager for timing
from contextlib import contextmanager
import time

@contextmanager
def timer(description: str):
    start = time.time()
    try:
        yield
    finally:
        elapsed = time.time() - start
        logger.info(f"{description} took {elapsed:.2f}s")

# Usage
with timer("Database migration"):
    run_migration()

# Configuration management
from dataclasses import dataclass, field
from typing import Optional
import yaml
import os

@dataclass
class AppConfig:
    database_url: str
    redis_url: str
    debug: bool = False
    workers: int = 4
    allowed_hosts: list = field(default_factory=list)
    
    @classmethod
    def from_yaml(cls, path: str) -> 'AppConfig':
        with open(path) as f:
            data = yaml.safe_load(f)
        return cls(**data)
    
    @classmethod
    def from_env(cls) -> 'AppConfig':
        return cls(
            database_url=os.environ['DATABASE_URL'],
            redis_url=os.environ.get('REDIS_URL', 'redis://localhost'),
            debug=os.environ.get('DEBUG', 'false').lower() == 'true',
            workers=int(os.environ.get('WORKERS', 4)),
        )
```

---

# 5. ARCHITECTURE PATTERNS

---

## 5.1 Infrastructure Architecture

```
                              ┌─────────────────────────────────────────────────────────────────┐
                              │                         INTERNET                                 │
                              └───────────────────────────────┬─────────────────────────────────┘
                                                              │
                              ┌───────────────────────────────▼─────────────────────────────────┐
                              │                        CloudFlare / WAF                          │
                              │                    DDoS Protection, CDN, WAF Rules               │
                              └───────────────────────────────┬─────────────────────────────────┘
                                                              │
┌─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────┐
│                                                   AWS / GCP / Azure                                                        │
│                                                                                                                            │
│  ┌────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │                                              VPC (10.0.0.0/16)                                                      │   │
│  │                                                                                                                     │   │
│  │   ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────┐  │   │
│  │   │                                     PUBLIC SUBNETS (10.0.1.0/24, 10.0.2.0/24)                               │  │   │
│  │   │                                                                                                              │  │   │
│  │   │     ┌──────────────────┐          ┌───────────────────────────────────────┐                                 │  │   │
│  │   │     │ NAT Gateway      │          │         Application Load Balancer     │                                 │  │   │
│  │   │     │ (for private     │          │         - SSL Termination             │                                 │  │   │
│  │   │     │  subnet egress)  │          │         - Path-based Routing          │                                 │  │   │
│  │   │     └──────────────────┘          │         - Health Checks               │                                 │  │   │
│  │   │                                   └───────────────────┬───────────────────┘                                 │  │   │
│  │   └───────────────────────────────────────────────────────┼─────────────────────────────────────────────────────┘  │   │
│  │                                                           │                                                         │   │
│  │   ┌───────────────────────────────────────────────────────┼─────────────────────────────────────────────────────┐  │   │
│  │   │                           PRIVATE SUBNETS (10.0.10.0/24, 10.0.11.0/24)                                      │  │   │
│  │   │                                                       │                                                      │  │   │
│  │   │   ┌───────────────────────────────────────────────────┼───────────────────────────────────────────────────┐ │  │   │
│  │   │   │                            EKS / Kubernetes Cluster                                                    │ │  │   │
│  │   │   │                                                   │                                                    │ │  │   │
│  │   │   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐│  ┌─────────────┐  ┌─────────────┐                 │ │  │   │
│  │   │   │  │   API       │  │   API       │  │   API       ││  │  Worker     │  │  Worker     │                 │ │  │   │
│  │   │   │  │   Pod       │  │   Pod       │  │   Pod       ││  │  Pod        │  │  Pod        │                 │ │  │   │
│  │   │   │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘│  └─────────────┘  └─────────────┘                 │ │  │   │
│  │   │   │         │                │                │       │                                                    │ │  │   │
│  │   │   │         └────────────────┼────────────────┘       │                                                    │ │  │   │
│  │   │   │                          │                        │                                                    │ │  │   │
│  │   │   │                  ┌───────▼───────┐                │                                                    │ │  │   │
│  │   │   │                  │   Service     │                │                                                    │ │  │   │
│  │   │   │                  │   Mesh        │                │                                                    │ │  │   │
│  │   │   │                  │   (Istio)     │                │                                                    │ │  │   │
│  │   │   │                  └───────┬───────┘                │                                                    │ │  │   │
│  │   │   └──────────────────────────┼────────────────────────┼────────────────────────────────────────────────────┘ │  │   │
│  │   │                              │                        │                                                      │  │   │
│  │   └──────────────────────────────┼────────────────────────┼──────────────────────────────────────────────────────┘  │   │
│  │                                  │                        │                                                         │   │
│  │   ┌──────────────────────────────┼────────────────────────┼──────────────────────────────────────────────────────┐  │   │
│  │   │                    DATABASE SUBNETS (10.0.20.0/24, 10.0.21.0/24)                                             │  │   │
│  │   │                              │                        │                                                      │  │   │
│  │   │     ┌────────────────────────▼─────┐    ┌─────────────▼──────────────┐    ┌───────────────────────┐         │  │   │
│  │   │     │      RDS PostgreSQL          │    │      ElastiCache Redis     │    │    S3 / Object        │         │  │   │
│  │   │     │      - Multi-AZ              │    │      - Cluster Mode        │    │    Storage            │         │  │   │
│  │   │     │      - Read Replicas         │    │      - Replication         │    │                       │         │  │   │
│  │   │     │      - Encrypted             │    │                            │    │                       │         │  │   │
│  │   │     └──────────────────────────────┘    └────────────────────────────┘    └───────────────────────┘         │  │   │
│  │   │                                                                                                              │  │   │
│  │   └──────────────────────────────────────────────────────────────────────────────────────────────────────────────┘  │   │
│  │                                                                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                                            │
│   ┌─────────────────────────────────┐   ┌─────────────────────────────────┐   ┌─────────────────────────────────┐         │
│   │        CloudWatch / Prometheus   │   │       Secrets Manager / Vault   │   │       ECR / Artifactory         │         │
│   │        Monitoring & Alerting     │   │       Secrets Management        │   │       Container Registry        │         │
│   └─────────────────────────────────┘   └─────────────────────────────────┘   └─────────────────────────────────┘         │
│                                                                                                                            │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## 5.2 CI/CD Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                            CI/CD PIPELINE FLOW                                                   │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

   Developer                                                                                              Production
       │                                                                                                       │
       │  git push                                                                                             │
       ▼                                                                                                       │
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐          │
│   GitHub /   │────▶│   Jenkins /  │────▶│   Build &    │────▶│   Security   │────▶│   Artifact   │          │
│   GitLab     │     │   GitLab CI  │     │   Test       │     │   Scan       │     │   Registry   │          │
│              │     │              │     │              │     │              │     │              │          │
│ - Code       │     │ - Webhook    │     │ - Compile    │     │ - SAST       │     │ - Docker     │          │
│ - PR/MR      │     │ - Pipeline   │     │ - Unit Test  │     │ - DAST       │     │ - Helm       │          │
│ - Branch     │     │ - Trigger    │     │ - Lint       │     │ - Dep Scan   │     │ - Versioned  │          │
└──────────────┘     └──────────────┘     │ - Coverage   │     │ - Container  │     └───────┬──────┘          │
                                          └──────────────┘     └──────────────┘             │                 │
                                                                                            │                 │
                                          ┌─────────────────────────────────────────────────┘                 │
                                          │                                                                   │
                                          ▼                                                                   │
                           ┌──────────────────────────────────────────────────────────────────────────────────┤
                           │                          DEPLOYMENT STAGES                                       │
                           │                                                                                  │
                           │   ┌────────────┐      ┌────────────┐      ┌────────────┐      ┌────────────┐    │
                           │   │    DEV     │─────▶│  STAGING   │─────▶│   CANARY   │─────▶│    PROD    │    │
                           │   │            │      │            │      │            │      │            │    │
                           │   │ Auto-deploy│      │ Auto-deploy│      │ 5% traffic │      │ Full       │    │
                           │   │ Every PR   │      │ Main branch│      │ Monitoring │      │ Rollout    │    │
                           │   │            │      │            │      │ Manual gate│      │            │    │
                           │   └────────────┘      └─────┬──────┘      └─────┬──────┘      └────────────┘    │
                           │                             │                   │                               │
                           │                    ┌────────▼───────┐    ┌──────▼───────┐                       │
                           │                    │ Integration    │    │ Smoke Tests  │                       │
                           │                    │ Tests          │    │ Health Check │                       │
                           │                    │ E2E Tests      │    │ Metrics OK?  │                       │
                           │                    └────────────────┘    └──────────────┘                       │
                           │                                                                                  │
                           └──────────────────────────────────────────────────────────────────────────────────┘


GitOps Flow (Alternative):

  ┌─────────────┐        ┌─────────────┐        ┌─────────────┐        ┌─────────────┐
  │ App Repo    │───────▶│ CI Pipeline │───────▶│ Update      │───────▶│ GitOps Repo │
  │ (Code)      │        │ Build Image │        │ Image Tag   │        │ (K8s YAML)  │
  └─────────────┘        └─────────────┘        └─────────────┘        └──────┬──────┘
                                                                              │
                                                                              │ Sync
                                                                              ▼
                                                                       ┌─────────────┐
                                                                       │ ArgoCD /    │
                                                                       │ Flux        │
                                                                       └──────┬──────┘
                                                                              │
                                                                              │ Apply
                                                                              ▼
                                                                       ┌─────────────┐
                                                                       │ Kubernetes  │
                                                                       │ Cluster     │
                                                                       └─────────────┘
```

## 5.3 Observability Architecture

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                        OBSERVABILITY STACK                                              │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

                                    ┌─────────────────────────────────┐
                                    │           DASHBOARDS            │
                                    │   Grafana / Datadog / Kibana    │
                                    └────────────────┬────────────────┘
                                                     │
                    ┌────────────────────────────────┼────────────────────────────────┐
                    │                                │                                │
           ┌────────▼────────┐             ┌────────▼────────┐             ┌─────────▼───────┐
           │     METRICS     │             │      LOGS       │             │     TRACES      │
           │   Prometheus    │             │  Elasticsearch  │             │     Jaeger      │
           │   Thanos/Mimir  │             │  Loki           │             │     Tempo       │
           └────────┬────────┘             └────────┬────────┘             └────────┬────────┘
                    │                               │                               │
           ┌────────▼────────┐             ┌────────▼────────┐             ┌────────▼────────┐
           │   Exporters     │             │   Log Shipper   │             │   Instrumented  │
           │ - node_exporter │             │ - Fluentd       │             │   Application   │
           │ - kube-state    │             │ - Fluent Bit    │             │ - OpenTelemetry │
           │ - custom        │             │ - Vector        │             │ - Jaeger Agent  │
           └────────┬────────┘             └────────┬────────┘             └────────┬────────┘
                    │                               │                               │
                    └───────────────────────────────┼───────────────────────────────┘
                                                    │
                                    ┌───────────────▼───────────────┐
                                    │      APPLICATION PODS         │
                                    │   Instrumented with:          │
                                    │   - Prometheus metrics /metrics│
                                    │   - Structured JSON logs      │
                                    │   - Trace context propagation │
                                    └───────────────────────────────┘


ALERTING FLOW:

┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Prometheus  │────▶│ Alertmanager│────▶│  PagerDuty  │────▶│  On-Call    │
│ Alert Rules │     │  Routing    │     │  Slack      │     │  Engineer   │
│             │     │  Grouping   │     │  Email      │     │             │
└─────────────┘     │  Silencing  │     └─────────────┘     └─────────────┘
                    └─────────────┘

Example Alert Rule:
- name: HighErrorRate
  rules:
    - alert: APIHighErrorRate
      expr: |
        sum(rate(http_requests_total{status=~"5.."}[5m])) /
        sum(rate(http_requests_total[5m])) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate on {{ $labels.service }}"
```

---

# 6. ADDITIONAL TOOLS QUICK REFERENCE

---

## 6.1 Service Mesh (Istio)

```yaml
# VirtualService - traffic routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: myapp
            subset: canary
    - route:
        - destination:
            host: myapp
            subset: stable
          weight: 90
        - destination:
            host: myapp
            subset: canary
          weight: 10

---
# DestinationRule - subsets and policies
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
  subsets:
    - name: stable
      labels:
        version: v1
    - name: canary
      labels:
        version: v2
```

## 6.2 GitOps (ArgoCD)

```yaml
# Application definition
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/k8s-manifests.git
    targetRevision: HEAD
    path: apps/myapp/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
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

## 6.3 Helm Charts

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 1.0.0
appVersion: "2.3.4"

# values.yaml
replicaCount: 3
image:
  repository: myregistry.com/myapp
  tag: "2.3.4"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

```bash
# Helm commands
helm repo add stable https://charts.helm.sh/stable
helm search repo nginx
helm install myrelease ./mychart -f values-prod.yaml
helm upgrade myrelease ./mychart --set image.tag=2.3.5
helm rollback myrelease 1
helm uninstall myrelease
helm template ./mychart  # Preview rendered manifests
```

## 6.4 Log Management (ELK / Loki)

```yaml
# Fluent Bit ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Parsers_File  parsers.conf

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            docker
        Tag               kube.*
        Refresh_Interval  5

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log           On
        K8S-Logging.Parser  On

    [OUTPUT]
        Name            es
        Match           *
        Host            elasticsearch
        Port            9200
        Logstash_Format On
        Replace_Dots    On
        Retry_Limit     False
```

---

# 7. QUICK DECISION MATRIX

---

| Scenario | Tool Choice | Why |
|----------|-------------|-----|
| Multi-cloud IaC | Terraform | Provider-agnostic, strong ecosystem |
| AWS-only IaC | CloudFormation or Terraform | CFN: native integration; TF: better abstractions |
| Config Management | Ansible | Agentless, simple, good for app config |
| Container orchestration | Kubernetes | Industry standard, rich ecosystem |
| CI/CD (simple) | GitHub Actions | Native integration, easy setup |
| CI/CD (complex) | Jenkins / GitLab CI | More control, self-hosted options |
| Secrets (cloud) | AWS Secrets Manager / GCP Secret Manager | Native, rotation support |
| Secrets (multi-cloud) | HashiCorp Vault | Dynamic secrets, advanced policies |
| Monitoring | Prometheus + Grafana | Open source, K8s native |
| Logging | Loki or ELK | Loki: simpler, less storage; ELK: more features |
| Tracing | Jaeger / Tempo | Open source, K8s native |
| Service Mesh | Istio or Linkerd | Istio: more features; Linkerd: simpler |
| GitOps | ArgoCD or Flux | ArgoCD: better UI; Flux: lighter weight |

---

*This guide covers tools and patterns for modern DevOps/SRE roles. Keep it handy for interviews and daily work!*