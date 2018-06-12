# Installing Istio into an existing OCP 3.9 installation

This document describes the steps for installing the Istio Tech Preview release into an existing installation of OCP 3.9.

**Table of Contents**

- [Preparing the OCP 3.9 Installation](#preparing-the-ocp-39-installation)
   - [Updating the Master](#updating-the-master)
   - [Updating the Nodes](#updating-the-nodes)
- [Installing Istio](#installing-istio)
- [Verifying Installation](#verifying-installation)
- [Removing Istio](#removing-istio)

## Preparing the OCP 3.9 Installation

Before Istio can be installed into an OCP 3.9 installation it is necessary to make a number of changes to the master configuration and each of the schedulable nodes.  These changes will enable features required within Istio and also ensure Elasticsearch will function correctly.

### Updating the Master

To enable the automatic injection of the Istio sidecar we first need to modify the master configuration on each master to include support for webhooks and signing of Certificate Siging Requests (CSRs).

Make the following changes on each master within your OCP 3.9 installation.

- Change to the directory containing the master configuration file (master-config.yaml)
- create a file named master-config.patch with the following contents (also in `master-config.patch`)

```
admissionConfig:
  pluginConfig:
    MutatingAdmissionWebhook:
      configuration:
        apiVersion: v1
        disable: false
        kind: DefaultAdmissionConfig
kubernetesMasterConfig:
  controllerArguments:
    cluster-signing-cert-file:
    - ca.crt
    cluster-signing-key-file:
    - ca.key
```

- within the same directory issue the following commands

```
cp -p master-config.yaml master-config.yaml.prepatch
oc ex config patch master-config.yaml.prepatch -p "$(cat master-config.patch)" > master-config.yaml
systemctl restart atomic-openshift-master*
```

### Updating the Nodes
In order to run the Elasticsearch application it is necessary to make a change to the kernel configuration on each node, this change will be handled through the sysctl service.

Make the following changes on each node within your OCP 3.9 installation

- Create a file named /etc/sysctl.d/99-elasticsearch.conf with the following contents

`vm.max_map_count = 262144`

- Execute the following command

```
sysctl vm.max_map_count=262144
```

## Installing Istio

The following steps will install Istio into an existing OCP 3.9 installation, they can be executed from any host with access to the cluster

```
oc new-project istio-system
oc create sa openshift-ansible
oc adm policy add-scc-to-user privileged -z openshift-ansible
oc adm policy add-cluster-role-to-user cluster-admin -z openshift-ansible
oc new-app istio_installer_template.yaml --param=OPENSHIFT_ISTIO_MASTER_PUBLIC_URL=<master public url> --param=OPENSHIFT_ISTIO_KIALI_USERNAME=<username> --param=OPENSHIFT_ISTIO_KIALI_PASSWORD=<password>
```

## Verifying Installation
The above instructions will create a job within the istio-system project to install Istio using ansible playbooks, the progress of the installation can be followed by either watching the pods or the log output from the `openshift-ansible-istio-installer-job` pod.

To watch the progress of the pods execute the following command

```
oc get pods -n istio-system -w
```

Once the `openshift-ansible-istio-installer-job` has completed run `oc get pods -n istio-system` and verify you have state similar to the following

```
NAME                                          READY     STATUS      RESTARTS   AGE
elasticsearch-0                               1/1       Running     0          41s
elasticsearch-1                               1/1       Running     0          31s
grafana-6f4fd4986f-czr6s                      1/1       Running     0          45s
istio-ca-7cd8c577cc-ng7wc                     1/1       Running     0          1m
istio-ingress-57866f979-rgj4l                 1/1       Running     0          1m
istio-mixer-7494d796df-vx9tx                  3/3       Running     0          1m
istio-mixer-validator-56488d76d7-h46kt        1/1       Running     0          59s
istio-pilot-fdd46f9bb-vtr5x                   2/2       Running     0          1m
istio-sidecar-injector-6b58cdf5b8-f2p2s       1/1       Running     0          53s
jaeger-agent-lrhw5                            1/1       Running     0          29s
jaeger-collector-575866c585-8bmv2             1/1       Running     0          29s
jaeger-query-657776775f-nc2ls                 1/1       Running     0          29s
kiali-795b86cfc7-w7b85                        1/1       Running     0          26s
openshift-ansible-istio-installer-job-9svtx   0/1       Completed   0          1m
prometheus-cf8456855-r96hb                    1/1       Running     0          44s
```

If you have also chosen to install the Farbic8 launcher then you should monitor the containers within the devex project until the following state has been reached

```
NAME                          READY     STATUS    RESTARTS   AGE
configmapcontroller-1-8rr6w   1/1       Running   0          1m
launcher-backend-2-2wg86      1/1       Running   0          1m
launcher-frontend-2-jxjsd     1/1       Running   0          1m
```

## Removing Istio

The following step will remove Istio from an existing installation, it can be executed from any host with access to the cluster.

```
oc new-app istio_removal_template.yaml
```
