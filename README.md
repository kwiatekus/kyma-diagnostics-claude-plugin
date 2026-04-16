# kyma-diagnose

A Claude Code plugin that diagnoses problems on SAP BTP Kyma instances. It runs `kyma alpha diagnose` commands (via the local Kyma CLI or the `dev-toolbox` Docker image) and produces a structured root-cause analysis report.

## Installation

**1. Add the marketplace (once):**

```
/plugin marketplace add kwiatekus/claude-plugins
```

**2. Install the plugin:**

```
/plugin install kyma-diagnose
```

## Usage

### Automatic — via the `kyma-diagnoser` agent

Just describe your problem in natural language. The `kyma-diagnoser` agent triggers automatically when it detects diagnostic intent:

```
My eventing module keeps going into error state
What's wrong with my Kyma cluster?
My app on Kyma is getting 503s, I think it's an Istio problem
```

### Manual — via the `/kyma-diagnose` slash command

```
/kyma-diagnose [optional module or area]
```

Examples:

```
/kyma-diagnose
/kyma-diagnose eventing
/kyma-diagnose api-gateway
```

In both cases, Claude Code will:
1. Detect whether the Kyma CLI is available locally or fall back to Docker
2. Confirm the `KUBECONFIG` before touching the cluster
3. Run cluster, Istio, and module-log diagnostics
4. Produce a structured report with overall health, degraded components, node health, and recommended actions