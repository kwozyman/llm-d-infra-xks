# LLM-D Conformance Tests

Conformance tests validate that LLM-D deployments meet expected specifications for each deployment profile/guide.

## Quick Start

```bash
# Run conformance test with auto-detection
./verify-llm-d-deployment.sh --namespace llm-d

# Test specific profile
./verify-llm-d-deployment.sh --profile inference-scheduling --namespace llm-d

# List available profiles
./verify-llm-d-deployment.sh --list-profiles
```

## Available Profiles

Profiles match official llm-d guides: https://github.com/llm-d/llm-d/tree/main/guides

| Profile | Description | Key Validations |
|---------|-------------|-----------------|
| `inference-scheduling` | Intelligent inference scheduling | EPP, ModelService decode, InferencePool, Gateway |
| `pd-disaggregation` | Prefill/Decode disaggregation | Prefill workers, Decode workers, P/D routing |
| `wide-ep-lws` | Wide Expert Parallelism with LWS | LeaderWorkerSet, multi-worker pods |
| `precise-prefix-cache-aware` | Prefix cache aware routing | Cache config, session affinity |
| `tiered-prefix-cache` | Tiered prefix caching | CPU offload, tiered cache config |
| `simulated-accelerators` | Testing without GPUs | Simulated workers, no GPU requests |
| `quickstart` | Basic deployment | EPP, ModelService, InferencePool |

## What Gets Tested

1. **Cluster Connectivity** - kubectl/oc access
2. **Namespace Validation** - Namespace exists, pods healthy
3. **Helm Releases** - Deployed Helm charts
4. **Profile Components** - Profile-specific pods, deployments, CRDs
5. **Inference Readiness** - Port-forward, /v1/models, inference test
6. **Monitoring Stack** - Prometheus, Grafana, ServiceMonitors, PodMonitors
7. **Recent Events** - Warning/Error events

## Monitoring Validation

The script validates the monitoring stack required for metrics and autoscaling:

- **Auto-detects** Prometheus namespace (monitoring, openshift-monitoring, etc.)
- **Prometheus** - Checks for running Prometheus pods
- **Grafana** - Checks for Grafana (optional but recommended)
- **ServiceMonitor CRD** - Prometheus Operator installed
- **PodMonitor CRD** - For vLLM metrics collection
- **llm-d Monitors** - ServiceMonitors/PodMonitors in llm-d namespace
- **Prometheus Targets** - Validates llm-d metrics are being scraped

### Expected Monitors

When llm-d is deployed with monitoring enabled:
- **ServiceMonitor for EPP** - `inferenceExtension.monitoring.prometheus.enabled: true`
- **PodMonitor for vLLM prefill** - `prefill.monitoring.podmonitor.enabled: true`
- **PodMonitor for vLLM decode** - `decode.monitoring.podmonitor.enabled: true`

## Inference Testing

The script tests actual inference capability:
- **Port Detection**: Prefers port 80 or 8000 over status ports (15021)
- **Model Detection**: Queries `/v1/models` endpoint
- **Inference Test**: Sends request to `/v1/completions`
- **FAIL if**: Model detection fails OR inference request fails (HTTP != 200)

Note: If modelservice pods are scaled to 0, inference will fail (expected behavior).

## Deployment Checks

The script detects deployments that exist but are scaled to 0:
- Reports as WARNING: "deployment exists but scaled to 0"
- Useful for identifying incomplete deployments

## Auto-Detection

The script automatically detects:
- **Monitoring Namespace**: Scans common namespaces for Prometheus pods
- **Inference Service**: Scans for services matching gateway patterns
- **Inference Port**: Prefers port 80/8000 over status ports
- **Model Name**: Queries `/v1/models` endpoint

## Usage Examples

```bash
# Fully automatic
./verify-llm-d-deployment.sh -n llm-d

# Specify profile
./verify-llm-d-deployment.sh --profile pd-disaggregation -n llm-d

# Override model (skip auto-detection)
./verify-llm-d-deployment.sh -n llm-d -m "meta-llama/Llama-3.1-8B-Instruct"

# Skip inference test (faster, just check components)
./verify-llm-d-deployment.sh --profile inference-scheduling --skip-inference -n llm-d

# Skip monitoring validation
./verify-llm-d-deployment.sh -n llm-d --skip-monitoring

# Longer timeout for slow clusters
./verify-llm-d-deployment.sh -n llm-d --timeout 300
```

## CLI Options

| Option | Description | Default |
|--------|-------------|---------|
| `-p, --profile NAME` | Deployment profile to validate | `inference-scheduling` |
| `-n, --namespace NAME` | LLM-D namespace | `llm-d` |
| `-t, --timeout SECONDS` | Timeout for wait operations | `120` |
| `-m, --model NAME` | Model name for inference test | auto-detect |
| `--skip-inference` | Skip the inference test | - |
| `--skip-monitoring` | Skip monitoring stack validation | - |
| `--monitoring-namespace` | Override monitoring namespace | auto-detect |
| `--list-profiles` | List available profiles | - |
| `-h, --help` | Show help message | - |

## Adding New Profiles

1. Add profile name to `AVAILABLE_PROFILES` array
2. Create config function:

```bash
profile_my_profile_config() {
    PROFILE_DESCRIPTION="My custom profile"

    # Pod patterns to check
    EXPECTED_POD_PATTERNS="epp modelservice"
    EXPECTED_DEPLOYMENT_PATTERNS="epp modelservice"
    EXPECTED_SERVICE_PATTERNS="epp"
    EXPECTED_CRDS="inferencepool"

    # Inference service pattern for auto-detect
    INFERENCE_SERVICE_PATTERN="inference-gateway"

    # Feature flags
    VALIDATE_EPP="true"
    VALIDATE_MODELSERVICE="true"
    VALIDATE_GPU="true"
}
```

3. Optionally add custom validation:

```bash
profile_my_profile_validate() {
    log_info "Running custom validations..."

    check_pod_pattern "epp" "EPP"
    check_pod_pattern "modelservice" "ModelService"
    check_inferencepool
}
```

## Exit Codes

- `0` - All checks passed
- `1` - One or more checks failed

## Requirements

- `kubectl` or `oc` CLI with cluster access
- `curl` for inference testing
- `jq` for JSON parsing and Prometheus target validation (optional)
