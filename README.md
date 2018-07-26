# Kuberhealthy

Easy synthetic testing for [Kubernetes](https://kubernetes.io) clusters.  Supplements other solutions like [prometheus](https://prometheus.io/) nicely.

[![Docker Repository on Quay](https://quay.io/repository/comcast/kuberhealthy/status "Kuberhealthy Docker Repository on Quay")](https://quay.io/repository/comcast/kuberhealthy)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

## Installation

#### Helm Install

Installation via [Helm](https://helm.sh) template is recommended.

`helm template --set prometheus.enabled=true https://github.com/Comcast/kuberhealthy/raw/master/helm/kuberhealthy-0.1.1.tgz | kubectl apply -f -`

_If you do not have prometheus enabled in your cluster, make sure you set `prometheus.enabled=false` in the above command._

##### Service Exposure

After installation, Kuberhealthy will only be available from within the cluster (service `Type: ClusterIP`) at the service URL `kuberhealthy.kuberhealthy`.  To expose Kuberhealthy to an external checking service, you must edit (`kubectl -n kuberhealthy edit service kuberhealthy`) the service to set `Type: LoadBalancer`.

##### Helm Template Tweaks

If you need more options, or want to customize the helm chart, you can edit the helm chart and manually make updates:

`helm template https://github.com/Comcast/kuberhealthy/raw/master/helm/kuberhealthy-0.1.1.tgz > kubernetes.yaml`

Alternatively, you can tweak the helm `values.yaml` file by cloning the repository and visiting the `helm/kuberhealthy` directory.


## What is Kuberhealthy?

Kuberhealthy performs stynthetic tests from within Kubernetes clusters in order to catch issues that would otherwise go unnoticed.  Instead of trying to identify all the things that could potentially go wrong, Kuberhealthy replicates real workflow and watches carefully for the expected Kubernetes behavior to occur.  Kuberhealthy serves both a JSON status page and a [Prometheus](https://prometheus.io/) metrics endpoint for integration into your choice of alerting solution.  More checks will be added in future versions to better cover [service provisioning](https://github.com/Comcast/kuberhealthy/issues/11), [DNS resolution](https://github.com/Comcast/kuberhealthy/issues/16), [disk provisioning](https://github.com/Comcast/kuberhealthy/issues/9), and more.

Some examples of errors Kuberhealthy would detect:

- Pods stuck in `Terminating` due to CNI communication failures
- Pods stuck in `ContainerCreating` due to disk scheduler errors
- Pods stuck in `Pending` due to Docker daemon errors
- A node that can not provision or terminate pods for any reason
- A pod in the `kube-system` namespace that is restarting too quickly
- A cluster component that is in a non-ready state
- Intermittent failures to access or create custom resources
- Kubernetes system services remaining technically "healthy" while their underlying pods are crashing
  - kue-scheduler
  - kube-apiserver
  - kube-dns

#### Deployment and Status Page

Deploying Kuberhealthy is as simple as applying the [helm](https://helm.sh/) chart file in this repository:

```
cd helm
helm install .
```

##### Prometheus Integration

If you wish for Kuberhealthy's checks to write metrics for Prometheus to collect and alert on, this is already available and configured with an annotation after installing the helm chart.  Alertmanager configuration examples are available.

##### Status Page

If you choose to alert from the JSON status page, you can access the status on `http://kuberhealthy.kuberhealthy.svc.cluster.local`.  The status page displays server status in the format shown below.  The boolean `OK` field can be used to indicate up/down status, while the `Errors` array will contain a list of potential error descriptions.  Granular, per-check information, including the last time a check was run, and the Kuberhealthy pod that ran that specific check is available under the `CheckDetails` object.

```json
  {
  "OK": true,
  "Errors": [],
  "CheckDetails": {
    "ComponentStatusChecker": {
      "OK": true,
      "Errors": [],
      "LastRun": "2018-06-21T17:32:16.921733843Z",
      "AuthorativePod": "kuberhealthy-7cf79bdc86-m78qr"
    },
    "DaemonSetChecker": {
      "OK": true,
      "Errors": [],
      "LastRun": "2018-06-21T17:31:33.845218901Z",
      "AuthorativePod": "kuberhealthy-7cf79bdc86-m78qr"
    },
    "PodRestartChecker namespace kube-system": {
      "OK": true,
      "Errors": [],
      "LastRun": "2018-06-21T17:31:16.45395092Z",
      "AuthorativePod": "kuberhealthy-7cf79bdc86-m78qr"
    },
    "PodStatusChecker namespace kube-system": {
      "OK": true,
      "Errors": [],
      "LastRun": "2018-06-21T17:32:16.453911089Z",
      "AuthorativePod": "kuberhealthy-7cf79bdc86-m78qr"
    }
  },
  "CurrentMaster": "kuberhealthy-7cf79bdc86-m78qr"
}
```

#### High Availability

Kuberhealthy scales horizontally in order to be fault tolerant.  By default, two instances are used with a [pod disruption budget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) and [RollingUpdate](https://kubernetes.io/docs/tasks/run-application/rolling-update-replication-controller/) strategy to ensure high availability.  

##### Centralized Check State State

The state of checks is centralized as [custom resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) records for each check.  This allows Kuberhealthy to always serve the same result, no matter which node in the pool you hit.  The current master running checks is calculated by all nodes in the deployment by simply querying the Kubernetes API for 'Ready' Kuberhealthy pods of the correct label, and sorting them alphabetically by name.  The node that comes first is master.

## Checks

Kuberhealthy performs the following checks in parallel at all times:

#### Daemonset Deployment and Termination

Deploys a `daemonset` to the `kuberhealthy` namespace, waits for all pods to be in the 'Ready' state, then terminates them and ensures all pod terminations were successful.  Containers are deployed with their resource requirements set to 0 cores and 0 memory and use the pause container from Google (`gcr.io/google_containers/pause:0.8.0`), which is likely already cached on your nodes.  The `node-role.kubernetes.io/master` `NoSchedule` taint is tolerated by daemonset testing pods.  The pause container is already used by kubelet to do various tasks and should be cached at all times.  If a failure occurs anywhere in the daemonset deployment or tear down, an error is shown on the status page describing the issue.

- Namespace: kuberhealthy
- Timeout: 5 minutes
- Check Interval: 15 minutes
- Check name: `daemonSet`

#### Component Health

Checks for the state of cluster `componentstatuses`.  Kubernetes components include the ETCD and ETCD-event deployments, the Kubernetes scheduler, and the Kubernetes controller manager.  This is almost the same as running `kubectl get componentstatuses`.  If a `componentstatus` status is down for 5 minutes, an alert is shown on the status page.

- Timeout: 1 minute
- Check Interval: 2 minute
- Downtime toleration: 5 minutes
- Check name: `componentStatus`

#### Excessive Pod Restarts

Checks for excessive pod restarts in the `kube-system` namespace.  If a pod has restarted more than five times in an hour, an error is indicated on the status page.  The exact pod's name will be shown as one of the `Error` field's strings.

A command line flag exists `--podCheckNamespaces` which can optionally contain a comma-separated list of namespaces on which to run the podRestarts checks.  The default value is `kube-system`.  Each namespace for which the check is configured will require the `get` and `list` verbs on the `pods` resource within that namespace.

- Namespace: kube-system
- Timeout: 3 minutes
- Check Interval: 5 minutes
- Tolerated restarts per pod over 1 hour: 5
- Check name: `podRestarts`  

#### Pod Status

Checks for pods older than ten minutes in the `kube-system` namespace that are in an incorrect lifecycle phase (anything that is not 'Ready').  If a `podStatus` detects a pod down for 5 minutes, an alert is shown on the status page. When a pod is found to be in error, the exact pod's name will be shown as one of the `Error` field's strings.

A command line flag exists `--podCheckNamespaces` which can optionally contain a comma-separated list of namespaces on which to run the podStatus checks.  The default value is `kube-system`.  Each namespace for which the check is configured will require the `get` and `list` [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) verbs on the `pods` resource within that namespace.

- Namespace: kube-system
- Timeout: 1 minutes
- Check Interval: 2 minutes
- Error state toleration: 5 minutes
- Check name: `podStatus`

### Security Considerations

By default, Kuberhealthy exposes an insecure (non-HTTPS) status endpoint without authentication. You should never expose this endpoint to the public internet. Exposing Kuberhealthy's status page to the public internet could result in private cluster information being exposed to the public internet when errors occur and are displayed on the page.