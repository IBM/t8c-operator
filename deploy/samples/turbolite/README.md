# TurboLite Deployment Configuration

## Overview

TurboLite is a lightweight deployment mode for Turbonomic that reduces resource consumption by disabling non-essential components while maintaining core platform functionality. TurboLite is used as Common Data Layer in Concert Platform.

## Usage

To enable TurboLite mode, set the `global.turboLite.enabled` flag to `true` and enable the required `instana` component in your Custom Resource (CR) file:

```yaml
apiVersion: charts.helm.k8s.io/v1
kind: Xl
metadata:
  name: xl-release
spec:
  global:
    turboLite:
      enabled: true
    repository: icr.io/cpopen/turbonomic
    tag: 8.19.6-SNAPSHOT
    # ... other configuration
  
  # Enable Instana for TurboLite
  instana:
    enabled: true
```

**Note:** When deploying TurboLite, you must disable all probes (except instana) if any of them are already enabled:
- **Option 1**: Set all probes to `false` except instana (Refer: [`charts_v1alpha1_xl_cr.yaml`](../../crds/charts_v1alpha1_xl_cr.yaml))
- **Option 2**: Remove all configurations existing under `# Component selector - Probes` except instana

Apply the configuration:
```bash
kubectl apply -f charts_v1_turbolite_cr.yaml -n turbonomic
```

## Components Disabled by TurboLite

When `turboLite.enabled` is set to `true`, the following components which are enabled by default via Values.yaml are automatically disabled to reduce resource footprint:

### Core Services
- **action-orchestrator** - Action execution and orchestration service
- **cost** - Cost calculation and analysis service
- **history** - Historical data storage and retrieval
- **metadata** - Metadata management service
- **market** - Market analysis and optimization engine
- **plan-orchestrator** - Plan creation and orchestration

### Data Processing
- **extractor** - Data extraction service
- **metrics-processor** - Metrics processing pipeline
- **cloud-optimizer** - Cloud optimization service

### Monitoring & Observability
- **prometheus** - Metrics collection and storage
- **kube-state-metrics** - Kubernetes state metrics exporter
- **telemetry** - Telemetry data collection

### Additional Features
- **sustainability** - Sustainability metrics and reporting
- **container-action-cost** - Container action cost calculation
- **server-power-modeler** - Server power modeling
- **suspend** - Workload suspension capabilities
- **ui** - User interface (disabled in lite mode)

### Mediation Probes
- **mediation-webhook** - Webhook-based mediation
- **mediation-udt** - User-defined target mediation
- **mediation-kubecost** - Kubecost integration probe

## Components That Remain Active

The following essential platform components remain active in TurboLite mode:

### Core Platform

- **db** - Database (MariaDB)
- **api** - API gateway and services
- **auth** - Authentication and authorization
- **topology-processor** - Topology processing engine
- **group** - Group management
- **repository** - Data repository service

### Infrastructure
- **consul** - Service discovery and configuration
- **kafka** - Message streaming platform
- **redis** - In-memory data store
- **nginx-ingress** / **openshift-ingress** - Ingress controllers

### Security & Identity
- **oauth2** - OAuth2 authentication
- **secret-manager** - Secret management (if enabled)

## How Helper Templates Read turboLite.enabled

The TurboLite configuration is implemented through Helm helper templates in [`_helpers.tpl`](../../helm-charts/xl/templates/_helpers.tpl).

### 1. Component Disabling Logic

The `xl.isComponentDisabledByTurboLite` helper template checks if a component should be disabled:

```go
{{- define "xl.isComponentDisabledByTurboLite" -}}
  {{- $componentName := .componentName -}}
  {{- $global := .global | default dict -}}
  {{- $turboLite := $global.turboLite | default dict -}}
  
  {{- if $turboLite.enabled -}}
    {{/* Disable all listed components */}}
    {{- if or
      (eq $componentName "action-orchestrator")
      (eq $componentName "cost")
      (eq $componentName "history")
      # ... other components
    -}}
      true
    {{- end -}}
  {{- end -}}
{{- end -}}
```

**Usage in component templates:**
```yaml
{{- if not (include "xl.isComponentDisabledByTurboLite" (dict "componentName" .Chart.Name "global" .Values.global)) }}
# Component deployment YAML
{{- end }}
```

### 2. Template Evaluation Flow

1. **Component Template Evaluation**: Each component's deployment template checks the helper function
2. **Flag Check**: Helper reads `global.turboLite.enabled` from the CR
3. **Component Name Match**: Compares the component name against the disabled/enabled list
4. **Conditional Rendering**: Returns `true` if component should be disabled/enabled
5. **Template Rendering**: Helm conditionally renders or skips the component based on the result

### Example: Action Orchestrator

In [`action-orchestrator.yaml`](../../helm-charts/services/action-orchestrator/templates/action-orchestrator.yaml):

```yaml
{{- if not (include "xl.isComponentDisabledByTurboLite" (dict "componentName" .Chart.Name "global" .Values.global)) }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: action-orchestrator
  # ... rest of deployment
{{- end }}
```

When `turboLite.enabled: true`:
- Helper returns `true` (component should be disabled)
- `not` inverts it to `false`
- Entire deployment block is skipped
- No Kubernetes resources are created for this component

## Required Components for TurboLite

### Mediation Instana
Instana observability integration is **required** for TurboLite deployments. You must explicitly enable it in your CR:

```yaml
instana:
  enabled: true
```

This should be placed at the same level as other component configurations (after the `global` section).

**Important**: Unlike other components that are automatically managed by the `turboLite.enabled` flag, instana must be manually enabled as it is disabled by default in values.yaml but required for TurboLite functionality.

## Feature Flags

TurboLite also sets specific feature flags automatically:

### Automatic Feature Flags

When TurboLite is enabled, the following feature flags are automatically set:

- **ignoreAOErrorsInSearch**: `true` - Ignores action orchestrator errors in search operations

These can be overridden explicitly in the CR:

```yaml
global:
  turboLite:
    enabled: true
  featureFlags:
    ignoreAOErrorsInSearch: "false"  # Override default
```

## Configuration Best Practices

### 1. Resource Allocation
Since TurboLite disables many components, you can reduce resource requests/limits:

```yaml
global:
  turboLite:
    enabled: true
  # Adjust resource allocations for remaining components
```

### 2. External Dependencies
Ensure external dependencies are configured:

```yaml
global:
  turboLite:
    enabled: true
  externalIP: 10.0.2.15
  externalDbIP: 10.0.2.15
  externalTimescaleDBIP: 10.0.2.15
```

## Troubleshooting

### Component Not Disabled
If a component is not being disabled:
1. Verify `global.turboLite.enabled: true` is set in the CR
2. Check that the component template uses the helper function
3. Ensure the component name matches exactly in the helper template

### Component Not Starting
If essential components fail to start:
1. Check that the component is not in the disabled list
2. Verify resource availability in the cluster
3. Review component logs for specific errors

### Reverting to Full Deployment
To switch back to full deployment:
1. Set `global.turboLite.enabled: false` in the CR
2. Apply the updated configuration
3. Wait for all components to be deployed

## Related Files

- **CR Sample**: [`charts_v1_turbolite_cr.yaml`](./charts_v1_turbolite_cr.yaml)
- **Helper Templates**: [`_helpers.tpl`](../../helm-charts/xl/templates/_helpers.tpl)
- **Default Values**: [`values.yaml`](../../build/values.yaml)

## Additional Resources

- For component-specific configuration, refer to individual component values.yaml files
- For deployment mode documentation, see the main Turbonomic documentation
- For Helm chart structure, review the helm-charts directory structure