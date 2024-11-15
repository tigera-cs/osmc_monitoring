
# Monitoring Calico Components - DEMO

### Table of Contents

* [Thank You!](#thank-you)
* [Overview](#overview)
* [Before you begin...](#before-you-begin)
* [Demo](#demo)

### Thank you!

Thank you for attending the OSMC in Nuremberg and joining the presentation on **"Monitoring Calico Components"** by Tigera.

We hope you enjoyed the presentation! Feel free to download the slides from [here](etc/OSMC_Nuremberg_2024_Davide_Sellitri_Monitoring_Calico_Components.pdf).

We would also appreciate your feedback about the presentation or Project Calico. Please leave your feedback [here](https://forms.gle/GX8byFYZmACcYKHM6). Your input is valuable to us!

---

### Overview

In this demo, we will deploy Calico Open Source using Helm and monitor core Calico components: Typha and Felix. We’ll deploy Prometheus, Grafana, and Grafana dashboards to observe key metrics and establish baselines.

Next, we’ll upgrade Calico with intentionally low limits for Typha and run a script to stress Calico components. Finally, we’ll examine how the metrics change in the Grafana dashboard.

---

### Before you begin...

**IMPORTANT**

* This demo is **disruptive** and should not be run in any production cluster.
* **DO NOT** use the Typha limits shown in this demo in any cluster other than for this demo.
* The Grafana dashboards provided are not maintained by Tigera. They are for demonstration purposes only. Feel free to adapt them or use the metrics as needed.

**About Calico Felix and Typha**

* **Felix** is the primary Calico agent that runs on every node, enforcing network policy and managing routes.
* **Typha** is an optional component that enhances scalability by reducing datastore traffic between Calico nodes.

Both Felix and Typha can provide metrics for Prometheus. For more details, see the official documentation: [Typha](https://docs.tigera.io/calico/latest/reference/typha/) | [Felix](https://docs.tigera.io/calico/latest/reference/felix/).

---

### Demo

**1.** Set up a test cluster and deploy Calico Open Source using Helm, following this [guide](https://docs.tigera.io/calico/latest/getting-started/kubernetes/helm). Using Helm is essential, as we will later modify Typha resource limits via Helm values.

**2.** Implement basic monitoring of Calico with Prometheus by following [this tutorial](https://docs.tigera.io/calico/3.28/operations/monitor/monitor-component-metrics):

* Enable Calico metrics reporting.
* Create the necessary namespace and service account for Prometheus.
* Deploy and configure Prometheus.
* View metrics in the Prometheus dashboard and create a simple graph.

**3.** Set up Calico metrics dashboards in Grafana by following [this tutorial](https://docs.tigera.io/calico/latest/operations/monitor/monitor-component-visual).

**4.** Import the additional `Demo_Grafana_Dashboard.json` file from this repo, by following [these](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/import-dashboards/) instructions.

**5.** Open the `OSMC - Monitoring Calico Components` dashboard in Grafana. Set the timeframe to 15 minutes and the Auto refresh to 5 seconds, as shown below:

![dashboard_1](etc/dashboard_1.png)

**6.** Wait 5 to 10 minutes and take note of baselines of the metrics in the dashboard.

**7.** Upgrade Calico Open Source with very low Typha resource limits, using this Helm `values_osmc.yaml` file:

```
installation:
    enabled: true
    kubeletVolumePluginPath: "None"
    typhaDeployment:
      spec:
        template:
          spec:
            containers:
            - name: calico-typha
              resources:
                limits:
                  cpu: 1m
                  memory: 40Mi
```

Upgrade Calico with this command:

```
helm upgrade calico projectcalico/tigera-operator --values values_osmc.yaml -n tigera-operator
```

**(Optional)** Adjust these limits if the upgrade fails.

**8**. Wait 5–10 minutes and observe the new baseline metrics in the dashboard.

**9.** Deploy a `test-deployment` which we'll use to stress Calico components:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-deploy
  template:
    metadata:
      labels:
        app: test-deploy
    spec:
      containers:
      - name: test-container
        image: wbitt/network-multitool
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: "k8s-app"
                operator: In
                values:
                - calico-typha
            namespaceSelector:
              matchLabels:
                name: calico-system
            topologyKey: "kubernetes.io/hostname"

```

**NOTE**: Esure that `calico-typha` and `test-deploy` pods are not scheduled on the same nodes.

**10.** Wait 5–10 minutes, observe the new baseline metrics, and note any changes.

**11.** Stress Calico Typha by scaling up and down the `test-deploy` deployment, using this script:

```
while true; do kubectl scale deployment test-deploy --replicas=10; sleep 2; kubectl scale deployment test-deploy --replicas=1;sleep 5; done
```

**12.** Monitor the Grafana dashboard. Note significant metric changes and watch for a drop in the number of active Calico nodes.

**13.** Stop the script and cleanup the cluster.

> **Congratulations! You have completed 'Monitoring Calico Components' demo! Don’t forget to leave your feedback [here](https://forms.gle/GX8byFYZmACcYKHM6).**
>
