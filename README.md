# GitLab Runner Kubernetes Executor with Google Kubernetes Engine Autopilot

This document details deployment of the [GitLab Runner](https://docs.gitlab.com/runner/) [Kubernetes Executor](https://docs.gitlab.com/runner/executors/kubernetes.html) [Helm Chart](https://gitlab.com/gitlab-org/charts/gitlab-runner) on [GKE Autopilot](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview).

While most elements work without modification, some Autopilot-specific changes are necessary.

## Changes to default [`values.yaml`](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/main/values.yaml)

### Required changes

Below are the minimum number of changes to the default `values.yaml` to run in GKE Autopilot.

* Set `gitlabUrl` to URL of instance ([~L52](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/779d45cd05abba6ea8b056380401bcfde2dbf51f/values.yaml#L52)):

      `gitlabUrl: <URL to base GitLab install>`

* Set `runnerRegistrationToken` for project/group/subgroup ([~L58](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/779d45cd05abba6ea8b056380401bcfde2dbf51f/values.yaml#L58)):

      `runnerRegistrationToken: "<runner registration token>"`

* Enable rbac ([~L141](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/779d45cd05abba6ea8b056380401bcfde2dbf51f/values.yaml#L141)):

      rbac:
        create: true

* Define a node selector to ensure executor lives in a [separate workload](https://cloud.google.com/kubernetes-engine/docs/how-to/node-auto-provisioning#workload_separation) to avoid preempting of the runner pod by jobs or cluster services ([~L463](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/779d45cd05abba6ea8b056380401bcfde2dbf51f/values.yaml#L463)):

      nodeSelector:
        group: autopilot-executor

* Set the corresponding toleration ([~L471](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/779d45cd05abba6ea8b056380401bcfde2dbf51f/values.yaml#L471)):

      tolerations:
        - key: group
          operator: Equal
          value: autopilot-executor
          effect: NoSchedule

### Recommended changes

* Set runner tag ([~L333](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/779d45cd05abba6ea8b056380401bcfde2dbf51f/values.yaml#L333)):

        tags: <runner tag>

* Disable running untagged ([~L349](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/779d45cd05abba6ea8b056380401bcfde2dbf51f/values.yaml#L349)):

        runUntagged: false

* Increase concurrency ([~L94](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/779d45cd05abba6ea8b056380401bcfde2dbf51f/values.yaml#L94)). Each 100 jobs consume approximately 0.25 CPU and 0.25 GB RAM.

      concurrent: 400

* Set higher runner pod resources ([~L447](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/779d45cd05abba6ea8b056380401bcfde2dbf51f/values.yaml#L447)):

      resources:
        requests:
          cpu: 1
          memory: 1Gi
          ephemeral-storage: 1Gi

* Provide additional updates to job pods config, run all job pods as [spot VMs](https://cloud.google.com/kubernetes-engine/docs/concepts/spot-vms) ([~L313](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/779d45cd05abba6ea8b056380401bcfde2dbf51f/values.yaml#L313)):

        config: |
          [[runners]]
            [runners.kubernetes]
              namespace = "{{.Release.Namespace}}"
              image = "ubuntu:22.04"
              poll_timeout = 3600
              cpu_request = "500m"
              cpu_request_overwrite_max_allowed = 50
              memory_request = "256M"
              memory_request_overwrite_max_allowed = "256G"
              ephemeral_storage_request = "10G"
              ephemeral_storage_request_overwrite_max_allowed = "5T"
              helper_cpu_request = "500m"
              helper_cpu_request_overwrite_max_allowed = "5"
              helper_memory_request = "256M"
              helper_memory_request_overwrite_max_allowed = "2G"
              helper_ephemeral_storage_request = "5G"
              helper_ephemeral_storage_request_overwrite_max_allowed = "20G"
              node_selector_overwrite_allowed = ".*"
            [runners.kubernetes.node_selector]
              "cloud.google.com/gke-spot" = "true"
 
## Deployment

Deploy with [helm](https://helm.sh), recommend deploying from [Google Cloud Shell](https://cloud.google.com/shell) after [setting a default kubectl cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl#default_cluster_kubectl):

      kubectl create ns gitlab-runner
      helm repo add gitlab https://charts.gitlab.io
      helm repo update
      helm install --namespace gitlab-runner gitlab-runner -f values.yaml gitlab/gitlab-runner

Subsequent updates to `values.yaml` configuration can be propagated with `helm upgrade`:

      helm upgrade --namespace gitlab-runner gitlab-runner -f values.yaml gitlab/gitlab-runner

## Usage

Run [computational pipelines](https://en.wikipedia.org/wiki/Pipeline_(computing)) with [GitLab CI/CD](https://docs.gitlab.com/ee/ci) defined in a repository's [`.gitlab-ci.yml`](https://docs.gitlab.com/ee/ci/yaml/gitlab_ci_yaml.html).

Resources can be specified on a per-job basis using the `KUBERNETES_` [container override variables](https://docs.gitlab.com/runner/executors/kubernetes.html#overwriting-container-resources).
