# ChartMuseum Helm Chart

Deploy your own private ChartMuseum.

## Prerequisites

* [If enabled] A persistent storage resource and RW access to it
* [If enabled] Kubernetes StorageClass for dynamic provisioning
* Tiller is mandatory . If not installed , create service account for tiller with required privelege ( Follow Tiller section )
* Create PV & local storage as a pre-requisites so that chart-museum persistency is bound to it. 

## Configuration

By default this chart will not have persistent storage, and the API service
will be *DISABLED*.  This protects against unauthorized access to the API
with default configuration values.

## Tiller configuration
Create service account for Tiller with required privilege on cluster 

Apply it using kubectl command as shown below 
```
kubectl apply -f helm-service-account.yaml
```

Initialise helm with tiller service account & verify using version command.
```
helm init --service-account tiller
helm version
```

### Using with local filesystem storage
By default chartmuseum uses local filesystem storage.
So , by Default persistency is enabled . At present persistent volume is mounted to local storage.
But in future it will be mounted to Ceph storage


## Uninstall

By default, a deliberate uninstall will result in the persistent volume
claim being deleted.

```shell
helm delete my-chartmuseum
```

To delete the deployment and its history:
```shell
helm delete --purge my-chartmuseum
```

## Jenkin pipeline to install chart-museum

Update the values.yaml & run jenkin pipeline job to install chart-musuem.
Jenkin pipeline peforms below step:
```
1> Create chart-museum secret with credential(admin,admin) 
2> Create namesace chart-museum
3> Install chart-musuem with customized values in above created namespace
```