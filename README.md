# Scalable multi-node Elassandra deployment on Kubernetes Cluster

This recipe is a fork from https://github.com/mdoulaty/Scalable-Elassandra-deployment-on-Kubernetes. It uses statefulsets for storing data persistantly and it is easy to scale up and down!

It is assumed that you are already familiary with Cassandra, EalsticSearch and Kubernetes and you have a working Kubernetes cluster.
for Microsoft Azure Kubernetes deployment, see [Azure.md](Azure.md) .

## Create k8s resources;

```
$ kubectl create -f elassandra-service.yaml
$ kubectl create -f local-volumes.yaml
$ kubectl create -f elassandra-statefulset.yaml
```

## Elassandra service

First the Elassandra service is defined and the relevant ports are exposed:

`$ kubectl -f elassandra-service.yaml`

Check to verify the service is created:

`$ kubectl get svc elassandra`


## Local volumes

Then the local volumes are created:

`$ kubectl create -f local-volumes.yaml`

To verify it:

`$ kubectl get pv`

## Statefulsets

Then the pods are created using the statefulset definition:

`$ kubectl create -f elassandra-statefulset.yaml`

And to verify the statefulset:

`$ kubectl get statefulsets`

## Scaling up and down

With this sample script, only three persistant volumes are created - so if you want to scale to more than three pods, first create the pvs accordingly.

`$ kubectl scale statefulset elassandra --replicas=3`

And to verify the state of the statefulset:

`$ kubectl get statefulsets`

## Logs of a node

    kubectl logs elassandra-0
    
## Accessing the cluster

Assuming that the default namespace was used, the service is accessible via `elassandra.svc.default.cluster.local`
Check k8s resources:

    kubectl cluster-info
    kubectl get nodes
    kubectl get pv
    kubectl get all

## Checking Cassandra nodes:

    kubectl exec -ti elassandra-0 nodetool status
    kubectl exec -ti elassandra-0 nodetool gossipinfo

## Checking ElasticSearch status:

    kubectl exec -ti elassandra-0 curl http://elassandra-0.elassandra.default.svc.cluster.local:9200/_cat/nodes
    kubectl exec -ti elassandra-0 curl http://elassandra-0.elassandra.default.svc.cluster.local:9200/_cluster/state?pretty

## Bash in a container:

    kubectl exec -ti elassandra-0 -- bash -il
    export CASSANDRA_HOME=/opt/elassandra-5.5.0.20/
    source $CASSANDRA_HOME/bin/aliases.sh
    state
    
## Rolling upgrade of Elassandra docker image

See [updating-statefulsets](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/#updating-statefulsets)

    kubectl patch statefulset elassandra --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"strapdata.azurecr.io/strapdata/elassandra:5.5.0.21"}]'
    kubectl get all -o wide
    
Check pods active image:

    for p in 0 1 2; do echo "$p"; kubectl get po elassandra-$p --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'; echo; done
