# KRO Workshop
These are the instructions for the workshop. Follow them one step at a time

# Get a Google Cloud Free Account
- You workshop facilitator will share a way to create a free Google Cloud. Follow the steps and once your project is ready come back here
- Make sure the drop down for projects has a project selected.
- Start the Google Cloud Shell (Top right corner of the Google Cloud console).
- Git clone this repo
```
git clone 
cd kro-workshop
```
# Create GKE cluster
Export needed environment variables. Replace the value for PROJECT_ID. You will find it in the dropdown or URL

```
export CLUSTER_NAME=kro
export REGION=europe-west1
export PROJECT_ID=<Insert your project ID here>
```

Create an Autopilot GKE cluster

```
gcloud container clusters create-auto $CLUSTER_NAME --region $REGION
```

Get credentials

```
gcloud container clusters get-credentials $CLUSTER_NAME --region $REGION
```

Install KRO

```
export KRO_VERSION=$(curl -sL \
    https://api.github.com/repos/kubernetes-sigs/kro/releases/latest | \
    jq -r '.tag_name | ltrimstr("v")'
  )

helm install kro oci://ghcr.io/kro-run/kro/kro \
  --namespace kro \
  --create-namespace \
  --version=${KRO_VERSION}
```

Check KRO was Installed. Wait until the pod says Ready

```
kubectl get pods -n kro
```

Congrats you are ready to use KRO

# Deloy a simple RGD

In this example we will deploy nginx a Service and an Ingress.

```
kubectl apply -f simple-rgd/rgd.yaml
```

Check the RGD was deployed. It's status should say ACTIVE

```
kubectl get rgd simple-rgd
```

Now deploy an Instance of this new RGD.

```
kubectl apply -f simple-rgd/instance.yaml
```

This step will take time specially if you are running on GKE Autopilot.
Take a look at simple-rgd/rgd.yaml and simple-rgd/instance.yaml while the deployment is progressing

After a few minutes the deployment should have finished. Inspect it

```
kubectl get pods,svc,ingress
```

It will take the Ingress a few more minutes to be available. Once the Address field displays an IP, you can use that to check the Nginx deployment

Congrats you just used KRO to simplify deploying a web app

# Deloy cloud resources using RGD

```
gcloud services enable \
  container.googleapis.com  \
  cloudresourcemanager.googleapis.com \
  serviceusage.googleapis.com
```

Install KCC (Kubernetes Config Connector)

```
gcloud storage cp gs://configconnector-operator/latest/release-bundle.tar.gz release-bundle.tar.gz
tar zxvf release-bundle.tar.gz
kubectl apply -f operator-system/configconnector-operator.yaml
```

Setup IAM Permissions.

```
gcloud iam service-accounts create kcc-operator
gcloud projects add-iam-policy-binding ${PROJECT_ID}\
 --member="serviceAccount:kcc-operator@${PROJECT_ID}.iam.gserviceaccount.com" \
 --role="roles/owner"
gcloud iam service-accounts add-iam-policy-binding kcc-operator@${PROJECT_ID}.iam.gserviceaccount.com \
 --member="serviceAccount:${PROJECT_ID}.svc.id.goog[cnrm-system/cnrm-controller-manager]" \
 --role="roles/iam.workloadIdentityUser"
gcloud projects add-iam-policy-binding ${PROJECT_ID}\
 --member="serviceAccount:kcc-operator@${PROJECT_ID}.iam.gserviceaccount.com" \
 --role="roles/storage.admin"
```

```
kubectl apply -f - <<EOF
apiVersion: core.cnrm.cloud.google.com/v1beta1
kind: ConfigConnector
metadata:
  name: configconnector.core.cnrm.cloud.google.com
spec:
  mode: cluster
  googleServiceAccount: "kcc-operator@${PROJECT_ID?}.iam.gserviceaccount.com"
  stateIntoSpec: Absent
EOF
```

```
kubectl create namespace config-connector
kubectl annotate namespace config-connector cnrm.cloud.google.com/project-id=${PROJECT_ID?}
```

Verify Installation

```
kubectl wait -n cnrm-system --for=condition=Ready pod --all
```

Let's create another GKE Cluster now

```
kubectl apply -f gke-cluster/rgd.yaml
```

Verify the RGD is Active

```
kubectl get rgd gkecluster.kro.run
```

Create a cluster

```
kubectl apply -f gke-cluster/instance.yaml
```

Wait few minutes and check the cluster was created. You either use the CLI or the UI

```
kubectl get GKECluster -n config-connector
```