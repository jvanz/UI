# Kubewarden UI

> _Note_: The UI is still in development and has not yet been fully released

## Install

The first distribution of the UI will be alongside the Rancher Dashboard. Currently it's being built in a [forked repo](https://github.com/jordojordo/dashboard/tree/kubewarden), but is able to be tested with a [Docker image](https://hub.docker.com/repository/docker/jordonleach/kubewarden).

## Run Rancher UI

Run the latest [Docker image](https://hub.docker.com/repository/docker/jordonleach/kubewarden) with 1 environment variable:

- `CATTLE_UI_DASHBOARD_INDEX` - This must be set to ensure the dashboard is using the prebuilt dashboard with the Kubewarden branch. (e.g. `https://localhost/dashboard/index.html`)

For example:

```sh
docker run -d --name kubewarden \
  --restart=unless-stopped \
  --privileged \
  -p 80:80 -p 443:443 \
  -e CATTLE_UI_DASHBOARD_INDEX=https://localhost/dashboard/index.html \
  jordonleach/kubewarden:latest
```

---

## Add Kubewarden to Rancher

> Adapted from the [Kubewarden helm chart](https://charts.kubewarden.io/) install.

To add Kubewarden to Rancher you will need to install [`cert-manager`](https://cert-manager.io/docs/installation/), and then install the kubewarden-controller chart.

From your local cluster open the `kubectl` shell (ctrl+\`) and input:

```sh
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml
helm repo add kubewarden https://charts.kubewarden.io
helm install --create-namespace -n kubewarden kubewarden-crds kubewarden/kubewarden-crds
helm install --wait -n kubewarden kubewarden-controller kubewarden/kubewarden-controller
```

### Adding Cluster Admission Policies

Currently you will be unable to create [currated policies](https://hub.kubewarden.io/) from the UI. However, you can still install them using `kubectl`.

For instance:

```yml
kubectl apply -f - <<EOF
---
apiVersion: policies.kubewarden.io/v1alpha2
kind: ClusterAdmissionPolicy
metadata:
  name: privileged-pods
spec:
  policyServer: default
  module: registry://ghcr.io/kubewarden/policies/pod-privileged:v0.1.9
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations:
        - CREATE
        - UPDATE
  mutating: false
EOF
```

---

### Enabling metrics

Rancher has a Cluster Tool for Monitoring that uses Grafana and Prometheus. You can utilize this tool and integrate it with Kubewarden to view certain metrics pertaining to a given policy.

> _Note_: You will need a cluster with 4 or more cores to install the Monitoring tool

**To install**:

1. In the cluster explorer click on `Cluster Tools` from the side navigation
  1a. `Install` the Monitoring tool
  1b. Edit the YAML to include a Service Monitor for Kubewarden
  1c. You need to specify the correct namespace where you installed Kubewarden

> Adapted from [here](https://docs.kubewarden.io/operator-manual/telemetry/metrics/01-quickstart.html#install-prometheus)

```yml
prometheus:
  additionalPodMonitors: []
  additionalRulesForClusterRole: []
  additionalServiceMonitors:
    - name: kubewarden
      selector:
        matchLabels:
          app: kubewarden-policy-server-default
      namespaceSelector:
        matchNames:
          - kubewarden
      endpoints:
        - port: metrics
          interval: 10s
  annotations: {}
```

2. Add the [ConfigMap](https://grafana.com/grafana/dashboards/15314)
  2a. Navigate to More Resources -> Core -> ConfigMaps
  2b. Click `Create`
  2c. Be sure to select the `cattle-dashboards` namespace
  2d. Download the [JSON](https://grafana.com/api/dashboards/15314/revisions/1/download) for the Kubewarden Grafana dashboard
  2e. Choose `Read from File` and select the JSON file you downloaded
  2f. Add the necessary Annotations and Labels for Rancher monitoring

```yml
annotations:
  meta.helm.sh/release-name: rancher-monitoring
  meta.helm.sh/release-namespace: cattle-monitoring-system
labels:
  app: rancher-monitoring-grafana
  app.kubernetes.io/instance: rancher-monitoring
  app.kubernetes.io/managed-by: Helm
  app.kubernetes.io/part-of: rancher-monitoring
  app.kubernetes.io/version: 100.1.0_up19.0.3
  chart: rancher-monitoring-100.1.0_up19.0.3
  grafana_dashboard: "1"
  heritage: Helm
  release: rancher-monitoring
```

3. Enable telemetry for your `kubewarden-controller` resource
  3a. Navigate to Apps & Marketplace -> Installed Apps
  3b. Select the `Edit/Upgrade` action for your `kubewarden-controller` resource
  3c. Edit the YAML for `telemetry` to be `enabled: "true"` and ensure the metrics port is correct

```yml
telemetry:
  enabled: True
policyServer:
    metrics:
      port: 8080
```

> _Note_: You may need to redeploy your Monitoring resources for the new ConfigMap to be loaded. You can easily do this from Workloads -> Deployments. Select all the resources in the `cattle-monitoring-system` namespace and click on the `Redeploy` action.

You should be able to view the metrics for any given Cluster Admission Policy on the detail page for a given policy. You can also view the Kubewarden dashboard itself in the Grafana UI.