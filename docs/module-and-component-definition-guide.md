# Module and Component Definition Guide

## Overview

This guide walks through defining a complete Module with multiple Components, demonstrating the practical implementation of the meta-architecture. We'll create a user management Module with service, interface, and integration Components.

## Module Registry Structure

The Module Registry is implemented as a git repository with a defined structure:

```
module-registry/
├── registry.yaml                    # Generated index of all Modules
├── modules/
│   ├── user-management.module.yaml
│   ├── order-management.module.yaml
│   └── kafka-cluster.module.yaml
└── scripts/
    └── generate-registry.sh         # Regenerates registry.yaml from modules/
```

## Module Definition

A Module definition file describes the complete Module, its Components, and their relationships.

### modules/user-management.module.yaml

```yaml
apiVersion: modules.liquid-labs.com/v1
kind: Module
metadata:
  name: user-management
  version: 2.1.0
  description: Complete user management system with authentication and profile management
  team: platform-team
  maintainers:
    - name: Sarah Chen
      email: sarah.chen@company.com
    - name: Mike Johnson
      email: mike.johnson@company.com

components:
  service:    # Service Components - the business logic
    - name: user-service
      repository: https://github.com/company/user-service
      version: v1.5.0
      description: Core user management API service
      required: true    # Must be deployed for Module to function
      
    - name: user-auth-service
      repository: https://github.com/company/user-auth-service
      version: v1.2.0
      description: Dedicated authentication service (CQRS pattern)
      required: true
      
  infrastructure:    # Infrastructure Components - dedicated infra for this Module
    - name: user-cache-cluster
      repository: https://github.com/company/user-cache-cluster
      version: v1.0.0
      description: Dedicated Redis cluster for user sessions
      required: false    # Optional performance enhancement
      
  interfaces:    # How users interact with the Module
    - name: user-portal-ui
      repository: https://github.com/company/user-portal-ui
      version: v2.1.0
      description: React-based user self-service portal
      required: false
      
    - name: user-cli
      repository: https://github.com/company/user-cli
      version: v1.2.0
      description: Command-line interface for user administration
      required: false
      
  integrations:    # How the Module connects to external systems
    - name: user-events-publisher
      repository: https://github.com/company/user-events-publisher
      version: v1.1.0
      description: Publishes user lifecycle events to Kafka
      required: false

# Module-level contracts are aggregated from Components
# This section documents what the Module as a whole provides
contracts:
  - name: user-api
    type: openapi-3.0
    version: v1
    component: user-service    # Which Component provides this
    
  - name: user-events
    type: kafka-avro
    version: v1
    component: user-events-publisher

# Dependencies on other Modules
dependencies:
  - module: postgres-cluster
    version: ">=1.0.0 <2.0.0"
    purpose: primary-datastore
    
  - module: kafka-cluster
    version: ">=2.0.0"
    purpose: event-streaming

# Compliance and regulatory metadata
compliance:
  dataClassification: pii    # pii, confidential, internal, public
  regulations:
    - gdpr       # General Data Protection Regulation
    - ccpa       # California Consumer Privacy Act
  auditLogging: required
  encryptionAtRest: true
  encryptionInTransit: true

# Version compatibility notes
compatibility:
  notes: |
    - v2.1.0: Added multi-factor authentication support
    - v2.0.0: Breaking change - split auth into separate service
    - v1.x: Legacy single-service architecture (deprecated)
```

## Component Definitions

Each Component has its own definition file in its repository.

### Service Component

**user-service/component.yaml**

```yaml
apiVersion: components.liquid-labs.com/v1
kind: ServiceComponent
metadata:
  name: user-service
  module: user-management
  moduleRegistry: https://github.com/company/module-registry
  repository: https://github.com/company/user-service
  version: 1.5.0
  description: Core user management REST API
  status: stable    # prototype, alpha, beta, rc, stable
  languages:    # Languages used in this Component
    - java:17
    - sql:postgresql

# Contracts this Component provides
contracts:
  - name: user-api-v1
    type: openapi-3.0
    version: v1
    spec: api/v1/openapi.yaml    # File in this repo
    description: Main REST API for user management
    
  - name: user-api-v2
    type: openapi-3.0
    version: v2
    spec: api/v2/openapi.yaml
    description: Beta API with GraphQL-like field selection
    
  - name: user-grpc
    type: grpc-proto
    version: v1
    spec: api/grpc/    # Directory containing .proto files
    description: Internal gRPC API for service-to-service calls

# Dependencies on other Modules
dependencies:
  - module: postgres-cluster
    version: ">=1.0.0 <2.0.0"
    contract: postgres-connection    # Named contract from postgres-cluster
    
  - module: redis-cluster
    version: ">=2.0.0"
    contract: redis-connection
    optional: true    # Component works without it

# Configuration requirements
configuration:
  required:
    - DATABASE_URL
    - JWT_SECRET_KEY
  optional:
    - REDIS_URL
    - LOG_LEVEL
    - METRICS_PORT
```

### Web UI Interface Component

**user-portal-ui/component.yaml**

```yaml
apiVersion: components.liquid-labs.com/v1
kind: InterfaceComponent
metadata:
  name: user-portal-ui
  module: user-management
  moduleRegistry: https://github.com/company/module-registry
  repository: https://github.com/company/user-portal-ui
  version: 2.1.0
  description: React-based user self-service portal
  status: stable
  languages:
    - typescript:4.9
    - css:3

# UI Components often expose different types of contracts
contracts:
  - name: user-portal-mfe
    type: micro-frontend
    version: v1
    description: |
      Exposes UserProfile and UserSettings Components via Module Federation.
      Host apps should configure remotes: { userPortal: 'userPortal@[url]/remoteEntry.js' }
    
  - name: user-components
    type: npm-package
    version: v2.1.0
    spec: package.json
    description: Shared React components for user interfaces

dependencies:
  - module: user-management
    contract: user-api-v1    # References the named contract
    
  - module: design-system
    version: "^3.2.0"
    contract: ui-components    # NPM package from design system

configuration:
  required:
    - USER_API_URL
    - AUTH_DOMAIN
  optional:
    - FEATURE_FLAGS_URL
    - ANALYTICS_KEY
```

### CLI Interface Component

**user-cli/component.yaml**

```yaml
apiVersion: components.liquid-labs.com/v1
kind: InterfaceComponent
metadata:
  name: user-cli
  module: user-management
  moduleRegistry: https://github.com/company/module-registry
  repository: https://github.com/company/user-cli
  version: 1.2.0
  description: Command-line interface for user administration
  status: stable
  languages:
    - go:1.20

contracts:
  - name: user-cli-commands
    type: cli-commands
    version: v1
    spec: docs/cli-reference.yaml    # Structured command documentation
    description: CLI command structure and options
    
  - name: user-cli-brew
    type: homebrew-formula
    version: v1.2.0
    spec: deploy/homebrew/user-cli.rb
    description: Homebrew installation formula

dependencies:
  - module: user-management
    contract: user-api-v1    # Uses the main API
    
  - module: user-management
    contract: user-grpc      # Also uses gRPC for some operations

configuration:
  required:
    - USER_API_URL
    - USER_GRPC_URL
  optional:
    - API_KEY_FILE
    - OUTPUT_FORMAT
```

### Kafka Integration Component

**user-events-publisher/component.yaml**

```yaml
apiVersion: components.liquid-labs.com/v1
kind: IntegrationComponent
metadata:
  name: user-events-publisher
  module: user-management
  moduleRegistry: https://github.com/company/module-registry
  repository: https://github.com/company/user-events-publisher
  version: 1.1.0
  description: Publishes user lifecycle events to Kafka
  status: stable
  languages:
    - java:17

contracts:
  - name: user-events-v1
    type: kafka-avro
    version: v1
    spec: schemas/avro/    # Directory with .avsc files
    description: User lifecycle events (created, updated, deleted, authenticated)
    
  - name: user-events-api
    type: async-api-2.0
    version: v1
    spec: docs/async-api.yaml
    description: Complete event documentation including examples

dependencies:
  - module: kafka-cluster
    version: ">=2.0.0"
    contract: kafka-producer    # Standard Kafka producer contract
    
  - module: schema-registry
    version: ">=7.0.0"
    contract: registry-api

configuration:
  required:
    - KAFKA_BOOTSTRAP_SERVERS
    - SCHEMA_REGISTRY_URL
    - DATABASE_CONNECTION    # For CDC
  optional:
    - CONSUMER_GROUP_ID
    - BATCH_SIZE
```

### Infrastructure Component

**user-cache-cluster/component.yaml**

```yaml
apiVersion: components.liquid-labs.com/v1
kind: InfrastructureComponent
metadata:
  name: user-cache-cluster
  module: user-management
  moduleRegistry: https://github.com/company/module-registry
  repository: https://github.com/company/user-cache-cluster
  version: 1.0.0
  description: Dedicated Redis cluster for user sessions
  status: stable
  languages:
    - terraform:1.5
    - hcl:2

# Infrastructure Components provide infrastructure contracts
contracts:
  - name: redis-connection
    type: terraform-output
    version: v1
    spec: outputs.tf
    description: |
      Provides connection details as Terraform outputs:
      - redis_endpoint: Redis cluster endpoint
      - redis_port: Redis port
      - redis_auth_token: Authentication token (if enabled)
    
  - name: redis-k8s-service
    type: kubernetes-service
    version: v1
    spec: k8s/service.yaml
    description: Kubernetes service for in-cluster access

dependencies:
  - module: kubernetes-cluster
    version: ">=1.0.0"
    contract: k8s-namespace    # Where to deploy
    
  - module: network-foundation
    version: ">=2.0.0"
    contract: vpc-config       # Network configuration

configuration:
  required:
    - ENVIRONMENT
    - CLUSTER_SIZE
    - REGION
  optional:
    - REDIS_VERSION
    - BACKUP_ENABLED
    - MONITORING_ENABLED
```

## Module Registry Index

The registry index is auto-generated from the Module definitions.

### registry.yaml

```yaml
# AUTO-GENERATED - DO NOT EDIT
# Generated from modules/* by scripts/generate-registry.sh
# Last updated: 2024-03-20T14:30:00Z

apiVersion: registry.liquid-labs.com/v1
kind: ModuleRegistry
metadata:
  version: 1.0.0
  generated: 2024-03-20T14:30:00Z
  
modules:
  - name: user-management
    latestVersion: 2.1.0
    availableVersions: [2.1.0, 2.0.3, 1.5.2]
    supportedVersions: [2.1.0, 2.0.3]    # Still receiving updates
    description: Complete user management system with authentication and profile management
    
  - name: order-management
    latestVersion: 1.5.0
    availableVersions: [1.5.0, 1.4.2, 1.3.0]
    supportedVersions: [1.5.0, 1.4.2]
    description: Order processing and fulfillment system
    
  - name: kafka-cluster
    latestVersion: 3.0.0
    availableVersions: [3.0.0, 2.8.1, 2.8.0]
    supportedVersions: [3.0.0, 2.8.1]
    description: Managed Kafka cluster infrastructure
    
  - name: postgres-cluster
    latestVersion: 1.2.0
    availableVersions: [1.2.0, 1.1.0, 1.0.0]
    supportedVersions: [1.2.0, 1.1.0]
    description: PostgreSQL cluster with automatic failover
    
  - name: design-system
    latestVersion: 3.2.0
    availableVersions: [3.2.0, 3.1.0, 3.0.0]
    supportedVersions: [3.2.0, 3.1.0]
    description: Shared UI components and design tokens

# Module count
summary:
  totalModules: 5
  totalComponents: 23    # Calculated from all Module definitions
```

## Contract Type Registry

Standardized contract types used across the system:

```yaml
# Standard API Contracts
- openapi-3.0         # REST APIs using OpenAPI 3.0 spec
- openapi-3.1         # REST APIs using OpenAPI 3.1 spec
- grpc-proto          # gRPC service definitions
- graphql-schema      # GraphQL schemas
- thrift-idl          # Apache Thrift interfaces

# Event/Message Contracts
- kafka-avro          # Kafka with Avro schemas
- kafka-protobuf      # Kafka with Protocol Buffers
- kafka-json-schema   # Kafka with JSON Schema
- async-api-2.0       # AsyncAPI 2.0 specifications
- cloudevents-1.0     # CloudEvents specification

# Infrastructure Contracts
- terraform-output    # Terraform output values
- terraform-module    # Terraform module interface
- helm-chart          # Helm chart values
- kubernetes-service  # Kubernetes service definition
- kubernetes-crd      # Custom Resource Definitions

# Data Store Contracts
- postgres-connection # PostgreSQL connection details
- redis-connection    # Redis connection details
- s3-bucket          # S3 bucket access details
- mongodb-connection  # MongoDB connection details

# UI/Frontend Contracts
- micro-frontend      # Module federation config
- npm-package         # NPM package exports
- web-components      # Web Components

# CLI Contracts
- cli-commands        # Structured CLI documentation
- homebrew-formula    # Homebrew installation

# Deployment Contracts
- docker-compose      # Docker Compose services
- k8s-namespace      # Kubernetes namespace access
- vpc-config         # VPC network configuration

# Custom/Domain-specific
- custom:*            # Prefix for non-standard types
```

## Summary

This structure provides:

1. **Clear Module boundaries** with the Module definition file
2. **Detailed Component specifications** for implementation teams
3. **Named contracts** for explicit dependencies between Components
4. **Comprehensive registry** maintained as a git repository
5. **Version compatibility tracking** at both Module and Component levels
6. **Compliance and regulatory tracking** at the Module level

Each Component can evolve independently within its version constraints, while the Module version ensures a tested, compatible combination for applications to consume.
