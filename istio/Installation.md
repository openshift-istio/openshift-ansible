# Installing Istio into an existing OCP 3.9 installation

This document describes the steps for installing the Istio Tech Preview release into an existing installation of OCP 3.9.

**Table of Contents**

- [Preparing the OCP 3.9 Installation](#preparing-the-ocp-39-installation)
   - [Updating the Master](#updating-the-master)
   - [Updating the Nodes](#updating-the-nodes)
- [Installing Istio](#installing-istio)
- [Verifying Installation](#verifying-installation)


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

The following steps will install Istio into an existing OCP 3.9 installation

- Upload the `istio_installer_template.yaml` template to the master node
- Execute the following commands, specifying the public URL of your master as the value of the parameter

```
oc new-project istio-system
oc create sa openshift-ansible
oc adm policy add-scc-to-user privileged -z openshift-ansible
oc adm policy add-cluster-role-to-user cluster-admin -z openshift-ansible
oc new-app istio_installer_template.yaml --param=OPENSHIFT_ISTIO_MASTER_PUBLIC_URL=<master public url> --param=OPENSHIFT_ISTIO_KIALI_USERNAME=<username> --param=OPENSHIFT_ISTIO_KIALI_PASSWORD=<password>
```

## Verifying Installation
The above instructions will create a job within the istio-system project to install Istio using ansible playbooks, the progress of the installation can be followed by either watching the pods or the log output from the `openshift-ansible-istio-job` pod.

To watch the progress of the pods execute the following command

```
oc get pods -n istio-system -w
```

Once the `openshift-ansible-istio-job` has completed run `oc get pods -n istio-system` and verify you have state similar to the following

```
NAME                                      READY     STATUS      RESTARTS   AGE
elasticsearch-0                           1/1       Running     0          1m
elasticsearch-1                           1/1       Running     0          1m
grafana-6f4fd4986f-gf2sp                  1/1       Running     0          1m
istio-ca-ddb878d84-cmgf2                  1/1       Running     0          1m
istio-ingress-76b5496c58-9cd4k            1/1       Running     0          1m
istio-mixer-56f49dc667-2phzt              3/3       Running     0          1m
istio-mixer-validator-65c7fccc64-2fkgc    1/1       Running     0          1m
istio-pilot-76dd785958-q545x              2/2       Running     0          1m
istio-sidecar-injector-599d8c454c-kxbwm   1/1       Running     0          1m
jaeger-agent-xvp8k                        1/1       Running     0          1m
jaeger-collector-575866c585-pjcjc         1/1       Running     0          1m
jaeger-query-657776775f-dj5k4             1/1       Running     0          1m
kiali-7f887bf646-prfdf                    1/1       Running     0          56s
openshift-ansible-istio-job-9b7gd         0/1       Completed   0          2m
prometheus-cf8456855-qgnbn                1/1       Running     0          1m
```

If you have also chosen to install the Farbic8 launcher then you should monitor the containers within the devex project until the following state has been reached

```
NAME                          READY     STATUS    RESTARTS   AGE
configmapcontroller-1-8rr6w   1/1       Running   0          1m
launcher-backend-2-2wg86      1/1       Running   0          1m
launcher-frontend-2-jxjsd     1/1       Running   0          1m
```
