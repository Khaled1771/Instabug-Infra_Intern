# Kubernetes Sealed Secrets Automation Task for Instabug infrastructure intern 2025

## Background

[Bitnami's sealed-secrets controller](https://github.com/bitnami-labs/sealed-secrets) (kubeseal) is a Kubernetes controller that provides secure secret management through asymmetric encryption. This allows for safe storage of encrypted secrets in version control systems while maintaining security.

### How It Works

![Kubeseal](./images/Kubeseal-Diagram.png)

The sealed-secrets controller operates using a sophisticated encryption process:

1. **Key Generation**
   - Controller generates public-private key pair
   - Private key remains securely within the Kubernetes cluster
   - Public key is distributed to users

2. **Encryption Process**
   - Users encrypt secrets using `kubeseal` CLI and public key
   - Generates encrypted "SealedSecret" objects
   - Example:
     ```bash
     kubeseal -o yaml < secret.yaml > sealed-secret.yaml
     ```

3. **Storage & Security**
   - SealedSecret objects can be safely stored in version control
   - Encrypted format ensures security in insecure storage mediums

4. **Decryption Process**
   - Controller automatically decrypts SealedSecrets using private key
   - Creates standard Kubernetes Secret objects
   - Applications access decrypted secrets normally


### Core Requirements

1. **Secret Discovery**
   - Identify all existing SealedSecrets in the cluster
   - Map dependencies and relationships

2. **Key Management**
   - Fetch all active public keys from controller
   - Maintain secure access to encryption keys

3. **Encryption Operations**
   - Decrypt SealedSecrets using existing keys
   - Re-encrypt with latest public key
   - Update SealedSecret objects in cluster


## Implementation Phases

1. **Planning Phase**
   - Review existing kubeseal codebase
   - Identify integration points
   - Design system architecture

2. **Development Phase**
   - Implement core functionality
   - Add monitoring capabilities
   - Develop security measures

3. **Testing Phase**
   - Unit testing
   - Integration testing
   - Performance testing
   - Security testing


## Resources

- [Sealed Secrets Documentation](https://github.com/bitnami-labs/sealed-secrets)
- [Kubernetes Secrets Documentation](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Key Rotation Best Practices](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)

# Automated Re-encryption Feature for Kubernetes SealedSecrets

## Overview

This document outlines the implementation of an automated re-encryption feature for Kubernetes SealedSecrets, addressing the manual re-encryption requirement during key rotation cycles.

## Problem Statement

Every 30 days, the sealed-secrets controller performs a key rotation by:
1. Generating a new key pair
2. Updating the controller with the new keys
3. Using new keys for new secrets

However, existing SealedSecrets continue using older keys and require manual re-encryption. This implementation automates this process.

## Implementation Plan

### 1. Command-Line Interface Extension

Add a new subcommand to kubeseal CLI:

```bash
kubeseal rotate-keys [flags]
```

Flags:
- `--namespace, -n`: Target specific namespace(s)
- `--all-namespaces`: Process all namespaces
- `--parallel`: Number of concurrent re-encryption operations
- `--dry-run`: Preview changes without applying
- `--backup`: Create backups before re-encryption
- `--timeout`: Operation timeout duration
- `--verbose`: Detailed logging output

### 2. Core Components

#### 2.1 Secret Discovery Module
```go
type SecretDiscoverer struct {
    kubeClient *kubernetes.Clientset
    namespace  string
    batchSize  int
}

func (sd *SecretDiscoverer) ListSealedSecrets() ([]*SealedSecret, error)
func (sd *SecretDiscoverer) WatchSealedSecrets(ctx context.Context) (<-chan *SealedSecret, error)
```

Responsibilities:
- List all SealedSecret objects in specified namespace(s)
- Implement pagination for large clusters
- Watch for changes during re-encryption
- Handle RBAC permissions

#### 2.2 Key Management Module
```go
type KeyManager struct {
    controller *SealedSecretsController
    keyCache   map[string]*certv1.Certificate
}

func (km *KeyManager) GetActiveKeys() ([]*certv1.Certificate, error)
func (km *KeyManager) GetLatestKey() (*certv1.Certificate, error)
```

Responsibilities:
- Fetch active public keys from controller
- Identify latest key for re-encryption
- Maintain key version mapping
- Secure key handling

#### 2.3 Re-encryption Module
```go
type ReEncryptor struct {
    keyManager *KeyManager
    parallel   int
    backup     bool
}

func (re *ReEncryptor) ReEncryptSecret(secret *SealedSecret) error
func (re *ReEncryptor) ProcessBatch(secrets []*SealedSecret) error
```

Responsibilities:
- Decrypt existing secrets
- Re-encrypt with latest key
- Validate re-encrypted content
- Handle concurrent operations

#### 2.4 Backup Module
```go
type BackupManager struct {
    storage    StorageBackend
    timeFormat string
}

func (bm *BackupManager) CreateBackup(secret *SealedSecret) error
func (bm *BackupManager) RestoreBackup(name string) error
```

Responsibilities:
- Create backups before re-encryption
- Manage backup retention
- Provide restoration capability
- Secure backup storage

### 3. Process Flow

1. **Initialization**
   ```mermaid
   graph TD
       A[Start] --> B[Parse Flags]
       B --> C[Initialize Components]
       C --> D[Validate Permissions]
       D --> E[Setup Logging]
   ```

2. **Discovery Phase**
   ```mermaid
   graph TD
       A[List Namespaces] --> B[Get SealedSecrets]
       B --> C[Group by Priority]
       C --> D[Create Work Queue]
   ```

3. **Re-encryption Phase**
   ```mermaid
   graph TD
       A[Process Queue] --> B[Create Backup]
       B --> C[Decrypt Secret]
       C --> D[Re-encrypt]
       D --> E[Validate]
       E --> F[Update Cluster]
   ```

### 4. Security Considerations

1. **Private Key Protection**
   - Never expose private keys
   - Use controller's secure endpoints
   - Implement secure communication

2. **Access Control**
   - Required RBAC permissions:
     ```yaml
     - apiGroups: ["bitnami.com"]
       resources: ["sealedsecrets"]
       verbs: ["get", "list", "watch", "update"]
     ```
   - Audit logging
   - Role-based access

3. **Backup Security**
   - Encrypted backup storage
   - Secure cleanup procedures
   - Access controls on backups

### 5. Performance Optimizations

1. **Batch Processing**
   - Worker pool implementation
   - Configurable concurrency
   - Resource limits

2. **Resource Management**
   ```go
   type ResourceManager struct {
       maxConcurrent int
       memoryLimit   int64
       cpuLimit      int64
   }
   ```

### 6. Logging and Monitoring

1. **Structured Logging**
   ```go
   type OperationLog struct {
       Timestamp   time.Time     `json:"timestamp"`
       Operation   string        `json:"operation"`
       SecretName  string        `json:"secretName"`
       Namespace   string        `json:"namespace"`
       Status      string        `json:"status"`
       Duration    time.Duration `json:"duration"`
       Error       string        `json:"error,omitempty"`
   }
   ```

2. **Metrics**
   - Total secrets processed
   - Processing duration
   - Success/failure rates
   - Resource utilization

## Usage Guide

### Basic Usage

```bash
# Re-encrypt all secrets in current namespace
kubeseal rotate-keys

# Re-encrypt in specific namespace
kubeseal rotate-keys -n production

# Re-encrypt across all namespaces
kubeseal rotate-keys --all-namespaces

# Dry run to preview changes
kubeseal rotate-keys --dry-run
```

### Advanced Usage

```bash
# Re-encrypt with increased parallelism
kubeseal rotate-keys --parallel 10

# Re-encrypt with backups
kubeseal rotate-keys --backup

# Re-encrypt with detailed logging
kubeseal rotate-keys --verbose
```

## Error Handling

1. **Failure Scenarios**
   - Network interruptions
   - Permission errors
   - Resource constraints
   - Timeout issues

2. **Recovery Procedures**
   ```go
   type ErrorHandler struct {
       maxRetries int
       backoff    time.Duration
   }
   ```

## Testing Strategy

1. **Unit Tests**
   - Component-level testing
   - Mock interfaces
   - Error scenarios

2. **Integration Tests**
   - End-to-end workflows
   - Cluster interaction
   - Performance testing

3. **Security Tests**
   - Penetration testing
   - Key handling validation
   - Access control verification




