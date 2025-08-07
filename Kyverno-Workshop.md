# Kyverno Workshop: Istio Security with Policy Automation

## Overview
This workshop demonstrates how to use Kyverno to automate Istio security policies, enabling network segmentation without traditional NetworkPolicies through policy-driven sidecar injection and authorization controls.

## Policy Types

### Mutate Policies
**Purpose**: Automatically modify resources during creation/update
- Add missing labels, annotations, or configuration
- Inject security defaults (resource limits, security contexts)
- Auto-configure Istio sidecar injection

### Generate Policies  
**Purpose**: Create new resources when triggers occur
- Generate ConfigMaps, Secrets, or RBAC resources
- Create Istio AuthorizationPolicies for new namespaces
- Auto-provision ServiceAccounts

### Validate Policies
**Purpose**: Enforce rules and block non-compliant resources
- Require specific security configurations
- Prevent privilege escalation
- Ensure Istio sidecar presence

### VerifyImages Policies
**Purpose**: Cryptographically verify container image signatures and attestations
- Verify image signatures using Cosign/Sigstore
- Validate software supply chain attestations
- Ensure images come from trusted sources
- Block unsigned or tampered images

### Cleanup Policies
**Purpose**: Automatically remove resources based on conditions
- TTL-based resource cleanup (delete after X time)
- Label-based cleanup (remove resources with specific labels)
- Conditional cleanup (remove when certain conditions are met)
- Namespace cleanup when empty

## Kyverno Controllers

### Admission Controller
Processes admission requests in real-time, enforcing mutate and validate policies as resources are created/updated.

### Background Controller  
Scans existing cluster resources against policies, useful for policy changes and compliance reporting.

### Cleanup Controller
Handles cleanup policies that remove resources based on conditions (TTL, labels, etc.).

### Reports Controller
Generates and manages policy violation reports, admission reports, and background scan results.

## Helm Charts

### Kyverno Core (`kyverno/kyverno`)
- Main policy engine with all controllers
- Policy exceptions and exclusions support
- Admission webhooks and CRDs

### Kyverno Policies (`kyverno/kyverno-policies`)  
- Pre-built security policies (Pod Security Standards)
- Best practices and compliance policies
- Configurable enforcement modes

### Policy Reporter UI (`policy-reporter/policy-reporter`)
- Web interface for policy violations
- Kyverno plugin for enhanced reporting
- Metrics and monitoring integration

### Istio Base (`istio-official/base`)
- Istio CRDs including AuthorizationPolicy
- Required for Kyverno generate policies
- RBAC configured for Kyverno integration

## Workshop Prerequisites

### Required Tools
```bash
# Install K3D
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Workshop Steps

### Step 1: Create K3D Cluster
```bash
# Create cluster
k3d cluster create kyverno-labs

# Set context
kubectl config use-context k3d-kyverno-labs

# Verify
kubectl cluster-info
```

### Step 2: Deploy Kyverno and Istio CRDs
```bash
# Add Helm repositories
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo add policy-reporter https://kyverno.github.io/policy-reporter
helm repo add istio-official https://istio-release.storage.googleapis.com/charts
helm repo update

# Install Istio CRDs
helm install istio-base istio-official/base \
  --version 1.27.0-rc.0 \
  --namespace istio-system \
  --create-namespace

# Install Kyverno core
helm install kyverno kyverno/kyverno \
  --version 3.5.0 \
  --namespace kyverno \
  --create-namespace \
  --set admissionController.replicas=1 \
  --set backgroundController.replicas=1 \
  --set cleanupController.replicas=1 \
  --set reportsController.replicas=1 \
  --set policyExceptions.enabled=true \
  --set features.policyExceptions.enabled=true \
  --set features.admissionReports.enabled=true \
  --set features.backgroundScan.enabled=true

# Configure RBAC for Istio integration
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kyverno-istio-permissions
rules:
- apiGroups: ["security.istio.io"]
  resources: ["authorizationpolicies"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kyverno-istio-permissions
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kyverno-istio-permissions
subjects:
- kind: ServiceAccount
  name: kyverno-admission-controller
  namespace: kyverno
- kind: ServiceAccount
  name: kyverno-background-controller
  namespace: kyverno
EOF

# Verify deployments
kubectl get pods -n kyverno
kubectl get pods -n istio-system
kubectl get crd | grep istio
```

### Step 3: Deploy Custom Policies
```bash
# Deploy Istio security policies
kubectl apply -f kyverno-lab/mutate/
kubectl apply -f kyverno-lab/validate/  
kubectl apply -f kyverno-lab/generate/

# Verify policies
kubectl get clusterpolicies
```

**Note**: Istio CRDs and RBAC are configured automatically to enable AuthorizationPolicy generation.

### Step 4: Test Policy Effects
```bash
# Deploy test application (already included in validate folder)
kubectl apply -f kyverno-lab/validate/demo-app.yaml

# Check mutations applied
kubectl get namespace demo-app -o yaml | grep istio-injection
# Expected output: istio-injection: enabled

kubectl get pods -n demo-app -o yaml | grep sidecar.istio.io/inject
# Expected output: sidecar.istio.io/inject: "true"

# Check policy reports
kubectl get policyreports -A

# Test generate policy with new namespace
kubectl create namespace test-generate
kubectl get authorizationpolicy -n test-generate
# Expected output: default-deny-all policy created by Kyverno
```

### Step 5: Access Policy Reporter UI
```bash
# Port forward to UI
kubectl port-forward -n policy-reporter svc/policy-reporter-ui 8080:8080

# Access at: http://localhost:8080
```

### Step 6: Demonstrate Policy Violations
```bash
# Try to create non-compliant pod (notice automatic mutation)
kubectl run test-pod --image=nginx --dry-run=server -o yaml
# Look for: sidecar.istio.io/inject: "true" (automatically added)

# Check detailed policy reports
kubectl get policyreports -A
kubectl get policyreport -n demo-app -o yaml | grep -A 5 "message:"
```

## Expected Outcomes

1. **✅ Namespace Creation**: Auto-labeled with `istio-injection=enabled`
2. **✅ Pod Creation**: Auto-annotated with `sidecar.istio.io/inject=true`  
3. **✅ AuthorizationPolicy**: Generated deny-all policy per namespace
4. **✅ Policy Reports**: Violations visible and logged
5. **✅ Audit Trail**: All policy decisions recorded

## Test Results Summary

### ✅ Working Features
- **Mutate policies**: Automatically add Istio labels and annotations
- **Generate policies**: Create AuthorizationPolicy for new namespaces
- **Validate policies**: Detect missing sidecar containers and bypass attempts
- **Policy reports**: Background scanning generates compliance reports
- **Real-time mutation**: New resources get modified during creation

### 📊 Policy Report Examples
```
NAMESPACE   NAME                                   PASS   FAIL   WARN   ERROR   SKIP
demo-app    059ff11a-1e52-4750-832a-7fe10366e904   1      2      0      0       0
```

### 🔍 Common Violations Detected
- Missing `istio-proxy` container in pods
- Sidecar injection annotation conflicts
- Non-compliant security configurations

### 🎯 Generated Resources
```
NAME               ACTION   AGE
default-deny-all   DENY     7s
```

## Cleanup
```bash
# Remove Kyverno and Istio
helm uninstall kyverno -n kyverno
helm uninstall istio-base -n istio-system
kubectl delete namespace kyverno istio-system

# Remove RBAC
kubectl delete clusterrole kyverno-istio-permissions
kubectl delete clusterrolebinding kyverno-istio-permissions

# Delete cluster
k3d cluster delete kyverno-labs
```

## Key Takeaways
- **✅ Kyverno enables policy-as-code** for Kubernetes security
- **✅ Istio security can be automated** without manual configuration
- **✅ Audit mode allows gradual policy enforcement** without blocking resources
- **✅ Policy exceptions provide flexibility** during transitions
- **✅ Background scanning catches existing** non-compliant resources
- **✅ Real-time mutations work seamlessly** during resource creation
- **⚠️ Generate policies require target CRDs** to be installed first

