# AI/ML Industrial Edge Blueprint Documentation

## About this Blueprint

Red Hat’s edge computing blueprints are repositories of configuration templates in the form of Kubernetes manifests that describe an edge computing stack fully declaratively and comprehensively; from its services down to the supporting infrastructure. Blueprints facilitate complex, highly reproducible deployments and are ideal for operating these deployments at scale using GitOps operational practices.

This “AI/ML Industrial Edge” blueprint demonstrates using OpenShift, ACM, AMQ Streams, OpenDataHub, and other products from Red Hat’s portfolio to address a common edge computing use case commonly found in manufacturing:
Machine inference-based anomaly detection on metric time-series sensor data at the edge, with a central data lake and ML model retraining.

[add high level overview figure plus brief explanation: multiple factory edges, central mgmt and data lake on public cloud]
See [link to architecture diagrams and explanation] for a detailed deployment architecture explaining what will be deployed where.


## Preparing for Deploying the Blueprint

### Forking the Blueprint GitHub Repos

[Fork](https://docs.github.com/en/free-pro-team@latest/github/getting-started-with-github/fork-a-repo) the following repos into your own GitHub org:

- https://github.com/redhat-edge-computing/blueprint-management-hub
- https://github.com/redhat-edge-computing/blueprint-industrial-edge
- https://github.com/redhat-edge-computing/manuela-gitops
- https://github.com/redhat-edge-computing/manuela-dev

It should look like this:

![alt text](images/github.png "Github org with forked repos")


### Adding Keys and Tokens

We assume the machine where these blueprints are going to be deployed from, it is a Linux machine (Fedora 32 in this case). In order to run this edge computing deployment, you will need to install some packages and create some files on a specific location:

Create a $HOME/.kni folder:

```
$ mkdir $HOME/.kni
```

and copy in the following files:
- id_rsa.pub: the SSH public key you want to use to access your nodes
- pull-secret.json: your pull secret, downloaded from https://cloud.redhat.com/openshift/install
- githubsecret.json: a JSON file containing your base64-encoded user name and personal access token (with “repo” permissions) for the GitHub account that can access the chosen GitOps repo


```
$ ls $HOME/.kni
githubsecret.json  id_rsa.pub  pull-secret.json
$ cat $HOME/.kni/githubsecret.json
{
  user: xxxxxxx
  token: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
}
```

For deploying the blueprint on Google Cloud Platform, store a GCP Service Account under $HOME/.gcp:

```
$ ls $HOME/.gcp
osServiceAccount.json
$ cat $HOME/.gcp/osServiceAccount.json
{
  "type": "service_account",
  "project_id": "openshift-my-project-xxx",
  "private_key_id": "xxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "private_key": "-----BEGIN PRIVATE KEY-----\nxxxxxxxxx\n-----END PRIVATE KEY-----\n",
  "client_email": "aos-serviceaccount@openshift-my-project-xxxx.iam.gserviceaccount.com",
  "client_id": "xxxxxxxxxxxxx",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/aos-serviceaccount%40openshift-my-project.iam.gserviceaccount.com"
}
```

Ensure that account meets the requirements in the [OpenShift documentation for installing on GCP](https://docs.openshift.com/container-platform/4.5/installing/installing_gcp/installing-gcp-account.html) so the installer can use the GCP APIs to create the needed infrastructure.

### Creating Quay Repos

[Create](https://access.redhat.com/documentation/en-us/red_hat_quay/3.3/html/use_red_hat_quay/use-quay-create-repo) the Quay image repositories to host the container images built as part of the GitOps workflow. You can leave these repositories empty since the initial pipeline runs will build the initial images, push them to your Quay repositories, and tag them:

- https://quay.io/organization/redhat-edge-computing/iot-frontend
- https://quay.io/organization/redhat-edge-computing/iot-consumer
- https://quay.io/organization/redhat-edge-computing/iot-software-sensor
- https://quay.io/organization/redhat-edge-computing/iot-anomaly-detection

You need to create a robot account and get the following token:

![alt text](images/robot.png "Robot account")

Set write permissions to all four repositories:

![alt text](images/permissions.png "Add write permissions")


Finally, copy the value of the .dockerconfigjson key in the Kubernetes secret and place it in $HOME/.kni/dockerconfig.json:

![alt text](images/secret.png "Copy dockerconfig")


### Adapting to Your Environment

Execute the following commands to clone the four forked repositories and replace the variables to adapt the blueprints to your own environment:

```
$ export MYGITHUBORG=your_github_org_name_here
$ export MYQUAYORG=your_quay_org_name_here
$ export MGMT_HUB_NAME=your_mgmt_hub_cluster_name
$ export MGMT_HUB_DOMAIN=your_mgmt_hub_cluster_domain
$ export EDGE_SITE_NAME=your_edge_cluster_name
$ export EDGE_SITE_DOMAIN=your_edge_cluster_domain

$ export REPOS=("blueprint-management-hub" "blueprint-industrial-edge" "manuela-gitops")
$ for r in ${REPOS[@]}; do
    git clone git@github.com:$MYGITHUBORG/$r
    find $r -not \( -path $r/.git -prune \) -type f -exec sed -i \
        -e "s/github.com\/redhat-edge-computing/github.com\/$MYGITHUBORG/g" \
        -e "s/quay.io\/redhat-edge-computing/quay.io\/$MYQUAYORG/g" \
        -e "s/edge-mgmt-hub.gcp.devcluster.openshift.com/$MGMT_HUB_NAME.$MGMT_HUB_DOMAIN/g" \
        -e "s/staging-edge.gcp.devcluster.openshift.com/$EDGE_NAME.$EDGE_DOMAIN/g" \
        {} \;
done


```

Generate new TLS certificates matching your environment:

```
$ openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem -subj "/C=DE/OU=Manuela/CN=*.apps.$MGMT_HUB_NAME.$MGMT_HUB_DOMAIN"

$ cat <<EOF >manuela-gitops/config/instances/manuela-data-lake/central-kafka-cluster/kafka-tls-certificate-and-key.yaml
apiVersion: v1
kind: Secret
metadata:
  name: kafka-tls-certificate-and-key
data:
  tls.crt: $(base64 -w0 <certificate.pem)
  tls.key: $(base64 -w0 <key.pem)
EOF

$ cat <<EOF >manuela-gitops/config/instances/manuela-data-lake/factory-mirror-maker/kafka-tls-certificate.yaml
apiVersion: v1
kind: Secret
metadata:
  name: kafka-tls-certificate
data:
  tls.crt: $(base64 -w0 <certificate.pem)
EOF
```

Change name and domain of your clusters:

```
$ pushd blueprint-management-hub >/dev/null
$ git mv sites/edge-mgmt-hub.gcp.devcluster.openshift.com sites/$MGMT_HUB_NAME.$MGMT_HUB_DOMAIN
$ sed -i -e "s|gcp.devcluster.openshift.com|$MGMT_HUB_DOMAIN|g" sites/$MGMT_HUB/00_install-config/install-config.patch.yaml
$ sed -i -e "s|edge-mgmt-hub|$MGMT_HUB_NAME|g" sites/$MGMT_HUB_NAME.$MGMT_HUB_DOMAIN/00_install-config/install-config.name.patch.yaml
$ popd >/dev/null

$ pushd blueprint-industrial-edge >/dev/null
$ git mv sites/staging-edge.gcp.devcluster.openshift.com sites/$EDGE_SITE_NAME.$EDGE_SITE_DOMAIN
$ sed -i -e "s|gcp.devcluster.openshift.com|$EDGE_SITE_DOMAIN|g" sites/$EDGE_SITE_NAME.$EDGE_SITE_DOMAIN/00_install-config/install-config.patch.yaml
$ sed -i -e "s|staging-edge|$EDGE_SITE_NAME|g" sites/$EDGE_SITE_NAME.$EDGE_SITE_DOMAIN/00_install-config/install-config.name.patch.yaml
$ popd >/dev/null
```


Push the changes back to GitHub:

```
$ export REPOS=("blueprint-management-hub" "blueprint-industrial-edge" "manuela-gitops")
for r in ${REPOS[@]}; do
    pushd $r >/dev/null
    git add .
    git commit -m "Customize URLs and update certificates"
    git push origin master
    popd >/dev/null
done
```

## Deploying the Blueprint

### Deploying the Management Hub

Start by deploying the management hub:

```
$ podman run --rm \
-v $HOME/.gcp:/root/.gcp:ro \
-v $HOME/.kni:/root/.kni \
quay.io/redhat-edge-computing/kni-install:latest create cluster --site-repo github.com/$MYGITHUBORG/blueprint-management-hub/sites/$MGMT_HUB_NAME.$MGMT_HUB_DOMAIN/
```

Ensure all workloads have been applied:

```
$ podman run --rm \
-v $HOME/.gcp:/root/.gcp:ro \
-v $HOME/.kni:/root/.kni \
quay.io/redhat-edge-computing/kni-install:latest create workloads --site-repo github.com/$MYGITHUBORG/blueprint-management-hub/sites/$MGMT_HUB_NAME.$MGMT_HUB_DOMAIN/
```

Once the deployment has finished and the workloads have been deployed, you can find the management hub credentials in the following folder:

```
$  ls ~/.kni/$MGMT_HUB_NAME.$MGMT_HUB_DOMAIN/final_manifests/auth/
kubeadmin-password  kubeconfig
```

### Priming the Seed Pipelines

The following commands will run 4 pipelines in order to build container images and feed the configuration in order to make the pods from the manuela-tst-all namespace work as expected.

Run:

```
$  oc apply -f ~/.kni/$MGMT_HUB_NAME.$MGMT_HUB_DOMAIN/blueprint/base/03_services/tekton/seed-iot-anomaly-detection-run.yaml -n manuela-ci
```

Wait for the pipeline run to finish, then run:

```
$  oc apply -f ~/.kni/$MGMT_HUB_NAME.$MGMT_HUB_DOMAIN/blueprint/base/03_services/tekton/seed-iot-consumer-run.yaml -n manuela-ci
```

Wait for the pipeline run to finish, then run:

```
$  oc apply -f ~/.kni/$MGMT_HUB_NAME.$MGMT_HUB_DOMAIN/blueprint/base/03_services/tekton/seed-iot-frontend-run.yaml -n manuela-ci
```

Wait for the pipeline run to finish, then run:

```
$  oc apply -f ~/.kni/$MGMT_HUB_NAME.$MGMT_HUB_DOMAIN/blueprint/base/03_services/tekton/seed-iot-software-sensor-run.yaml -n manuela-ci
```

The final result should look like this in the OpenShift console:

![alt text](images/seed.png "Seed pipelines")


### Verifying and Troubleshooting the ACM Subscriptions

The current version of ACM (2.0) has a known bug that is being resolved. The bug prevents the correct propagation of the subscription when its channel is created simultaneously. To check whether this is the case, run:

```
$ oc get appsub -A                                                                                                                             
NAMESPACE                 NAME                       STATUS              AGE                                                                                                      
argocd                    argocd-sub                 PropagationFailed   22m             
open-cluster-management   application-chart-sub      Subscribed          24m                                                                                                      
open-cluster-management   cert-manager-sub           Subscribed          25m             
open-cluster-management   cert-manager-webhook-sub   Subscribed          24m             
open-cluster-management   configmap-watcher-sub      Subscribed          24m             
open-cluster-management   console-chart-sub          Subscribed          24m             
open-cluster-management   grc-sub                    Subscribed          24m             
open-cluster-management   kui-web-terminal-sub       Subscribed          24m   
```

If the argocd-sub subscription is in PropagationFailed state, add a fake annotation to the object to trigger an update of the ArgoCD subscription:

```
$ oc edit appsub argocd-sub -n argocd
apiVersion: apps.open-cluster-management.io/v1
kind: Subscription
metadata:
  annotations:
    apps.open-cluster-management.io/deployables: ""
    apps.open-cluster-management.io/git-commit: fc79496f37b761d9d77e8b59af0fba00408043d2
    apps.open-cluster-management.io/github-path: base/02_cluster-addons/03_argocd
    apps.open-cluster-management.io/topo: ""
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps.open-cluster-management.io/v1","kind":"Subscription","metadata":{"annotations":{"apps.open-cluster-management.io/github-path":"base/02_cluster-addons/03_argocd"},"name":"argocd-sub","namespace":"argocd"},"spec":{"channel":"argocd-operator-gitops/argocd-operator-channel","name":"argocd-sub","placement":{"placementRef":{"apiGroup":"apps.open-cluster-management.io","kind":"PlacementRule","name":"apply-argocd-to-ready-clusters"}}}}
    open-cluster-management.io/user-group: c3lzdGVtOm1hc3RlcnMsc3lzdGVtOmF1dGhlbnRpY2F0ZWQ=
    open-cluster-management.io/user-identity: c3lzdGVtOmFkbWlu
    *test: fake #Add this annotation*
  creationTimestamp: "2020-10-28T10:21:06Z"
  generation: 2
  managedFields:
```

### Deploying the Factory Edge Site

The Management Hub cluster runs Red Hat's Advance Cluster Manager which has a list of three managed clusters preprovisioned. This will allow us to automatically register those three remote clusters into ACM once they are deployed. To perform this "self-registration" process, base64-encode the content of the management hub’s kubeconfig file and place it in $HOME/.kni/kubeconfighub.json:

```
$  base64 -w0 $HOME/.kni/$MGMT_HUB_NAME.$MGMT_HUB_DOMAIN/final_manifests/auth/kubeconfig > $HOME/.kni/kubeconfighub.json
```

Now you can deploy the factory edge site:

```
$ podman run --rm \
-v $HOME/.gcp:/root/.gcp:ro \
-v $HOME/.kni:/root/.kni \
quay.io/redhat-edge-computing/kni-install:latest create cluster --site-repo github.com/$MYGITHUBORG/blueprint-industrial-edge/sites/$EDGE_SITE_NAME.$EDGE_SITE_DOMAIN/
```

Once the deployment has finished, ensure all workloads have been applied:

```
$ podman run --rm \
-v $HOME/.gcp:/root/.gcp:ro \
-v $HOME/.kni:/root/.kni \
quay.io/redhat-edge-computing/kni-install:latest create workloads --site-repo github.com/$MYGITHUBORG/blueprint-industrial-edge/sites/$EDGE_SITE_NAME.$EDGE_SITE_DOMAIN/
```

### Using the Blueprint

Once the management hub and factory edge clusters are up and running, the platform is ready to perform some demo flows that will show how we can manage clusters and applications in the edge using GitOps.

- GitOps configuration management [preparation](https://github.com/redhat-edge-computing/industrial-edge-docs/blob/master/docs/module-configuration-management.md#Demo-preparation) - [demo execution](https://github.com/redhat-edge-computing/industrial-edge-docs/blob/master/docs/module-configuration-management.md#Demo-execution)
- Code change [preparation](https://github.com/redhat-edge-computing/industrial-edge-docs/blob/master/docs/module-code-change.md#Demo-preparation) - [demo execution](https://github.com/redhat-edge-computing/industrial-edge-docs/blob/master/docs/module-code-change.md#demo-execution)
- CI/CD pipeline & GitOps staging [preparation](https://github.com/redhat-edge-computing/industrial-edge-docs/blob/master/docs/module-ci-cd-pipeline.md#Demo-preparation) - [demo execution](https://github.com/redhat-edge-computing/industrial-edge-docs/blob/master/docs/module-ci-cd-pipeline.md#demo-execution)
- Event streaming from edge to core & filling the data lake [preparation](https://github.com/redhat-edge-computing/industrial-edge-docs/blob/master/docs/module-event-streaming.md#Demo-preparation) - [demo execution](https://github.com/redhat-edge-computing/industrial-edge-docs/blob/master/docs/module-event-streaming.md#demo-execution)
- Machine learning [preparation](https://github.com/redhat-edge-computing/industrial-edge-docs/blob/master/docs/module-machine-learning.md#Demo-preparation) - [demo execution](https://github.com/redhat-edge-computing/industrial-edge-docs/blob/master/docs/module-machine-learning.md#demo-execution)


![alt text](images/dashboard.png "IoT Dashboard")


