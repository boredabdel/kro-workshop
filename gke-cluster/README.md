# GKECluster

A **Platform Administrator** wants to give end users in their organization self-service access to create GKE clusters. The platform administrator creates a kro ResourceGraphDefinition called *gkecluster.kro.run* that defines the required Kubernetes resources and a CRD called *GKEcluster* that exposes only the options they want to be configurable by end users. The ResourceGraphDefinition would define the following resources (using [KCC](https://github.com/GoogleCloudPlatform/k8s-config-connector) to provide the mappings from K8s CRDs to Google Cloud APIs):

* GKE cluster
* Container Node Pool
* Network
* Subnetwork
* KMSKeyRing   - Encrypt BootDisk
* KMSCryptoKey - Encrypt BootDisk

Everything related to these resources would be hidden from the end user, simplifying their experience.  

![GKE Cluster Stack](gke-cluster.png)

## End User: GKECluster

The administrator needs to install the RGD first.
The end user creates a `GKECluster` resource something like this:

```yaml
apiVersion: kro.run/v1alpha1
kind: GKECluster
metadata:
  name: krodemo
  namespace: config-connector
spec:
  name: krodemo         # Name used for all resources created as part of this RGD
  location: us-central1 # Region where the GCP resources are created
  maxnodes: 4           # Max scaling limit for the nodes in the new nodepool
```

They can then check the status of the applied resource:

```
kubectl get gkeclusters
kubectl get gkeclusters krodemo -n config-connector -o yaml
```

Navigate to GKE Cluster page in the GCP Console and verify the cluster creation.

Once done, the user can delete the `GKECluster` instance:

```
kubectl delete gkecluster krodemo -n config-connector
```

## Administrator: ResourceGraphDefinition
The administrator needs to install the RGD in the cluster first before the user can consume it:

```
kubectl apply -f rgd.yaml
```

Validate the RGD is installed correctly:

```
kubectl get rgd gkecluster.kro.run
```

Once all user created instances are deleted, the administrator can choose to deleted the RGD.