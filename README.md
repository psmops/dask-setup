# dask
Dask in the NEO cluster

These are my notes about the Dask setup in the PS-MOPS NEO cluster and
the kubernetes configuration files.

Dask is here: https://dask.org/

Kubernetes (more preciselyy microk8s) has been deployed on some nodes in the cluster.

For reference, microk8s current setup (I doubt all those are necessary though):
  enabled:
    dashboard            # The Kubernetes dashboard
    dns                  # CoreDNS
    fluentd              # Elasticsearch-Fluentd-Kibana logging and monitoring
    ha-cluster           # Configure high availability on the current node
    helm                 # Helm 2 - the package manager for Kubernetes
    helm3                # Helm 3 - Kubernetes package manager
    metallb              # Loadbalancer for your Kubernetes cluster
    metrics-server       # K8s Metrics Server for API access to service metrics
    rbac                 # Role-Based Access Control for authorisation
    registry             # Private image registry exposed on localhost:32000
    storage              # Storage class; allocates storage from host directory

Despite my efforts (I didn't insist much though), I haven't been able
to set a dask cluster with helm, helm2 or helm3. Not sure why.

# Objective

The Dask Scheduler and Workers will be managed through Kubernetes:
* One Service for production
    * https://kubernetes.io/docs/concepts/services-networking/service/
    * Namespace: `psmops-dask-production`
    * 1 (2?) replica for Dask-Scheduler
    * 1 replica/node for Dask-Worker (maybe more?)
        * Note: https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
* One or more Services for testing
    * Each will mimic the Service for production in its own namespace
    *  Should there be always one for production integration testing? (ns = `psmops-dask-test-production`)
    *  Namespaces all prefixed with `psmops-dask-test-`

Why different namespaces? Mostly to clean up things quickly (and I
don't want the test instances to mess up the production dbs)

# Production Setup
In `neopipe@nmops32:~/k8s/psmops-dask-production`

## Useful Commands

Used to check the status 
```
alias kubectl='microk8s.kubectl'
kubectl --namespace psmops-dask-production get pods
kubectl --namespace psmops-dask-production describe pods 
kubectl --namespace psmops-dask-production get endpoints
kubectl --namespace psmops-dask-production logs dask-scheduler-7c47c875fb-z7cms
```

## Namespace
Definition in `namespace.yaml`

```
neopipe@nmops32:~/k8s/psmops-dask-production$ kubectl create -f ./namespace.yaml 
namespace/psmops-dask-production created
```

Delete with `kubectl delete namespaces psmops-dask-production`


## Scheduler Service

In scheduler-dask.yaml

Inspired by:
https://docs.prefect.io/orchestration/recipes/k8s_dask.html#kubernetes-manifests

```
neopipe@nmops32:~/k8s/psmops-dask-production$ kubectl create -f ./scheduler-dask.yaml 
deployment.apps/dask-scheduler created
service/dask-scheduler created
```

## Worker Service
In dask-worker.yaml

Inspired by:
https://docs.prefect.io/orchestration/recipes/k8s_dask.html#kubernetes-manifests

```
neopipe@nmops32:~/k8s/psmops-dask-production$ kubectl create -f ./worker-dask.yaml 
deployment.apps/dask-worker created
```

## Check

```
neopipe@nmops32:~/k8s/psmops-dask-production$ kubectl --namespace psmops-dask-production get pods
NAME                              READY   STATUS    RESTARTS   AGE
dask-scheduler-7c47c875fb-z7cms   1/1     Running   0          49m
dask-worker-7c4f4598f-slxtx       1/1     Running   0          34m
dask-worker-7c4f4598f-ddxnl       1/1     Running   0          34m
dask-worker-7c4f4598f-wkpjg       1/1     Running   0          34m
dask-worker-7c4f4598f-fb7v2       1/1     Running   0          34m
dask-worker-7c4f4598f-gkbgw       1/1     Running   0          34m
dask-worker-7c4f4598f-8snbl       1/1     Running   0          34m
neopipe@nmops32:~/k8s/psmops-dask-production$ kubectl --namespace psmops-dask-production get ep
NAME             ENDPOINTS          AGE
dask-scheduler   10.1.61.142:8786   49m
dask-dashboard   10.1.61.142:8787   49m
neopipe@nmops32:~/k8s/psmops-dask-production$ kubectl --namespace psmops-dask-production get svc
NAME             TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
dask-scheduler   LoadBalancer   10.152.183.197   10.11.0.145   8786:30526/TCP   49m
dask-dashboard   LoadBalancer   10.152.183.145   10.11.0.146   8787:32090/TCP   49m
```

Note:
* scheduler is at 10.11.0.145:8786
* dashboard is at 10.11.0.146:8786

## Traffic forwarding

Forward traffic to localhost host (vpn):
```
schastel@toes:~$ ssh -L8787:10.11.0.146:8787 -L8786:10.11.0.145:8786 nmops15
```

Check http://localhost:8787 to see dask dashboard 

## Notebook
```
schastel@toes:~$ docker run --rm -it --network host daskdev/dask-notebook
[...]
To access the server...
[...]
```

and in a python console/notebook:
```
from dask.distributed import Client
c = Client("localhost:8786")
c.ncores()
```

Now... I just have to use it

# Test environment

TODO: Write a script to build one yaml file so that only one `kubectl create` needs to be run.

Input parameters:
* namespace
* workers replica
* port for scheduler
* port for dashboard? Not sure it's needed if test is one-shot
