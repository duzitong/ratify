# Ratify OCI Artifact Verification Constraint Template

This constraint template extends Ratify's verification capabilities beyond container images to verify OCI artifacts referenced in Custom Resources (CRs).

## Overview

The `RatifyOCIArtifactVerification` constraint template allows you to:
- Verify OCI artifacts referenced in Custom Resources through configurable field paths
- Support nested field paths (e.g., `spec.chart.repository.url`)
- Validate artifact types against an allowlist
- Require or make verification optional
- Work with any CR that contains OCI artifact references

## Template Features

### Parameters

- `ociPathField` (string): The field path in the CR containing the OCI artifact reference
  - Supports nested paths like `spec.repository.url` or `data.artifactPath`
- `requireVerification` (boolean): Whether verification is mandatory
- `allowedArtifactTypes` (array): List of allowed OCI artifact MIME types

### Supported Field Path Examples

```yaml
# Simple field path
ociPathField: "data.artifactPath"

# Nested field path  
ociPathField: "spec.chart.repository.url"

# Deep nested path
ociPathField: "spec.source.helm.repository.oci.url"
```

## Usage Examples

### 1. Verify Helm Charts in Flux HelmReleases

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RatifyOCIArtifactVerification
metadata:
  name: verify-flux-helm-charts
spec:
  match:
    kinds:
      - apiGroups: ["helm.toolkit.fluxcd.io"]
        kinds: ["HelmRelease"]
  parameters:
    ociPathField: "spec.chart.spec.sourceRef.url"
    requireVerification: true
    allowedArtifactTypes:
      - "application/vnd.cncf.helm.chart.content.v1.tar+gzip"
```

### 2. Verify Artifacts in ConfigMaps

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RatifyOCIArtifactVerification
metadata:
  name: verify-configmap-artifacts
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["ConfigMap"]
  parameters:
    ociPathField: "data.artifactPath"
    requireVerification: true
```

### 3. Verify Custom Artifact References

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RatifyOCIArtifactVerification
metadata:
  name: verify-custom-artifacts
spec:
  match:
    kinds:
      - apiGroups: ["example.com"]
        kinds: ["ArtifactReference"]
  parameters:
    ociPathField: "spec.repository.url"
    requireVerification: true
    allowedArtifactTypes:
      - "application/vnd.cncf.notary.signature"
      - "application/vnd.in-toto+json"
```

## Example Custom Resources

### ConfigMap with Artifact Reference

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: policy-artifact
data:
  artifactPath: "oci://myregistry.azurecr.io/policies/security:v1.0"
  description: "Security policy artifact"
```

### Custom Resource with Nested OCI Path

```yaml
apiVersion: example.com/v1
kind: ArtifactReference
metadata:
  name: signed-policy
spec:
  name: "security-policy"
  repository:
    url: "oci://ghcr.io/myorg/policies/security:v1.0"
  verification:
    required: true
```

### Flux HelmRelease with OCI Chart

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: my-app
spec:
  chart:
    spec:
      chart: my-app
      sourceRef:
        url: "oci://myregistry.azurecr.io/helm/my-app:1.0.0"
```

## Deployment with Azure Policy

Since your cluster uses Azure Policy, you would need to:

1. Create an Azure Policy definition that includes this constraint template
2. Assign the policy to your AKS cluster
3. Azure Policy will automatically create the constraint template and any associated constraints

### Azure Policy Definition Structure

```json
{
  "policyRule": {
    "if": {
      "field": "type",
      "in": ["Microsoft.ContainerService/managedClusters"]
    },
    "then": {
      "effect": "[parameters('effect')]",
      "details": {
        "templateInfo": {
          "sourceType": "PublicURL",
          "url": "https://raw.githubusercontent.com/duzitong/ratify/v1.4.0/library/default/oci-artifact-template.yaml"
        },
        "constraintInfo": {
          "sourceType": "PublicURL", 
          "url": "https://raw.githubusercontent.com/duzitong/ratify/v1.4.0/library/default/samples/oci-artifact-constraints.yaml"
        }
      }
    }
  }
}
```

## Error Messages

The constraint provides detailed error messages:

- **Missing field**: `"Required OCI artifact path field 'spec.repository.url' is missing or empty"`
- **System error**: `"System error calling Ratify for OCI artifact 'oci://registry/path': connection failed"`  
- **Verification failed**: `"Failed to verify OCI artifact 'oci://registry/path' at 2025-09-22T12:00:00Z: application/vnd.oci.image.manifest.v1+json, trace-id: abc123"`
- **Disallowed type**: `"OCI artifact 'oci://registry/path' has disallowed type 'application/unknown'. Allowed types: [application/vnd.cncf.helm.chart.content.v1.tar+gzip]"`

## Integration with Ratify

This constraint template works with your existing Ratify setup by:
- Using the same `ratify-provider` external data provider
- Leveraging the same verification policies and certificate stores
- Supporting the same artifact types that Ratify can verify
- Providing the same level of signature verification for OCI artifacts

The constraint template extracts OCI paths from CRs and sends them to Ratify for verification, extending supply chain security beyond just container images to any OCI artifacts referenced in your Kubernetes resources.