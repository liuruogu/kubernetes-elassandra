# Elassandra deployment on Kubernetes Cluster

This recipe is a fork from https://github.com/mdoulaty/Scalable-Elassandra-deployment-on-Kubernetes. It uses statefulsets for storing data persistantly and it is easy to scale up and down! Elassandra docker images are available from [docker hub](https://hub.docker.com/r/strapdata/elassandra/) and built from [https://github.com/strapdata/docker-elassandra](https://github.com/strapdata/docker-elassandra).

It is assumed that you are already familiary with Cassandra, EalsticSearch and Kubernetes and you have a working Kubernetes cluster.
for Microsoft Azure Kubernetes deployment, see [Azure.md](Azure.md) .

The following assume that you work in the **default** kubernetes namespace. If you run your elassandra cluster in another namespace, switch to your context as follow:

    kubectl config use-context <namespace>

## Create k8s resources;

Create elassandra service, and pods through the statefulset definition.

WARNING: This statefulset use a storageClassName **managed-premium** defined on Microfot Azure. In order to run this statefulset on another cloud provider, change this storageClassName.

	kubectl create -f elassandra-service.yaml
	kubectl create -f elassandra-statefulset.yaml

### Accessing the cluster

Assuming that the default namespace was used, the service is accessible via `elassandra.svc.default.cluster.local`
Check k8s resources:

    kubectl cluster-info
    kubectl get statefulsets
    kubectl get nodes
    kubectl get pvc
    kubectl get pv
    kubectl get all -o wide

### Scaling up and down

With this sample script, only three persistant volumes are created - so if you want to scale to more than three pods, first create the pvs accordingly.

    kubectl scale statefulset elassandra --replicas=3

And to verify the state of the statefulset:

    kubectl get statefulsets

### Logs of a node

    kubectl logs elassandra-0
    
### Checking Cassandra nodes

    kubectl exec -ti elassandra-0 -- nodetool status
    kubectl exec -ti elassandra-0 -- cqlsh
    kubectl exec -ti elassandra-0 -- nodetool gossipinfo

### Checking Elasticsearch status

    kubectl exec -ti elassandra-0 -- curl http://elassandra-0.elassandra.default.svc.cluster.local:9200/_cat/nodes
    kubectl exec -ti elassandra-1 -- curl http://elassandra-1.elassandra.default.svc.cluster.local:9200/_cluster/state?pretty

### Bash in a container

    kubectl exec -ti elassandra-0 -- bash -il
    source $CASSANDRA_HOME/bin/aliases.sh
    state
    
    curl -XPUT -H "Content-Type: application/json" "http://elassandra-0.elassandra.default.svc.cluster.local:9200/my_index/my_type/1" -d'{ "foo":"bar" }'
    curl -XPUT -H "Content-Type: application/json" "http://elassandra-0.elassandra.default.svc.cluster.local:9200/my_index/my_type/2" -d'{ "foo":"bar2" }'
    
## Rolling upgrade of Elassandra docker image

See [updating-statefulsets](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/#updating-statefulsets)

    kubectl patch statefulset elassandra --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"strapdata.azurecr.io/strapdata/elassandra:5.5.0.22-rc1"}]'
    kubectl get all -o wide

    kubectl patch statefulset elassandra --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"strapdata.azurecr.io/strapdata/elassandra:6.2.3.5-rc1"}]'

    kubectl patch statefulset elassandra --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"strapdata.azurecr.io/strapdata/elassandra-enterprise:6.2.3.5-rc1"}]'
    
Check pods active image:

    for p in 0 1 2; do echo "$p"; kubectl get po elassandra-$p --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'; echo; done

## Restart with Elasticsearch disabled/enabled

You can rolling-restart Elassandra nodes with Elasticsearch disabled (Cassandra only) by setting the **CASSANDRA_DAEMON** env variable to **org.apache.cassandra.service.CassandraDaemon** as follow:

    kubectl set env statefulset -e CASSANDRA_DAEMON="org.apache.cassandra.service.CassandraDaemon" --all

To re-enable Elasticsearch on all nodes, restart nodes with the main class **org.apache.cassandra.service.ElassandraDaemon**:

    kubectl set env statefulset -e CASSANDRA_DAEMON="org.apache.cassandra.service.ElassandraDaemon" --all
    
List pods env variables:

    kubectl set env pods --all --list

## Elassandra major upgrade

* Run cassandra-only:

    kubectl set env statefulset -e CASSANDRA_DAEMON="org.apache.cassandra.service.CassandraDaemon" --all
    
* Drop the elastic_admin keyspacee

    kubectl exec -ti elassandra-0 -- cqlsh -e "DROP KEYSPACE elastic_admin;"
    
* Rolling-upgrade to the new version

    kubectl patch statefulset elassandra --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"strapdata.azurecr.io/strapdata/elassandra:6.2.3.5-rc1"}]'

* Recreate elasticsearch indices mapping (or insert data)
* Rebuild indices on all nodes:

    for p in 0 1 2; do echo "$p"; kubectl exec -ti elassandra-$p -- nodetool rebuild_index <ks> <table> elastic_<table>_idx
    
## Enable Elassandra Enterprise security

Elassandra Enterprise provides elasticsearch transport and client-to-node SSL encryption based on Cassandra X509 certificate and elasticsearch authentication based on Cassandra roles. The Elassandra Enterprise docker image enable cassandra SSL encryption and user authentication.

* Disable elasticsearch

    kubectl set env statefulset -e CASSANDRA_DAEMON="org.apache.cassandra.service.CassandraDaemon" --all
    
* Upgrade to elassandra-enterprise docker image with SSL and authentication enabled. 

    kubectl patch statefulset elassandra --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"strapdata.azurecr.io/strapdata/elassandra-enterprise:6.2.3.5-rc1"}]'
 
  The strapdata enterprise docker image includes a cacert.pem, a truststore and a keystore containing a Strapdata root CA and generic node certificate. This generic SSL certificate with **CN=elassandra-0.elassandra.default.svc.cluster.local** also matches **localhost**, **127.0.0.1** or any DNS names suffixed by **elassandra.default.svc.cluster.local**. To overwrite these settings, overwrite the strapdata docker image by COPYing yours in /opt/elassandra-<version>/conf/[cacert.pem, .truststore, .keystore].

* Open a cqlsh session:

    * Alter the keyspace *system_auth* replication map with RF >= 2 to avoid authentication issue when a node is down.
    * Create an admin role with superuser=true to avoid connection issue when a node id down.
    * Change the default cassandra password to a secure one.
    * Create a monitor ROLE with superuser=false and login=true. This role is by default authorized to monitor the elasticsearch cluster and is used by the k8s ready_probe.sh to check for elasticsearch node availability.
    
    kubectl exec -ti elassandra-0 -- cqlsh --ssl -u cassandra -p ***** -e "\
    ALTER KEYSPACE system_auth WITH replication = {'class': 'NetworkTopologyStrategy', 'DC1': '2'};\
    CREATE ROLE IF NOT EXISTS admin WITH PASSWORD = '*****' AND LOGIN = true AND SUPERUSER = true;\
    ALTER ROLE cassandra WITH PASSWORD = '*****';\
    CREATE ROLE IF NOT EXISTS monitor WITH PASSWORD = 'monitor' AND LOGIN = true AND SUPERUSER = false;"

* Add an env variable ELASSANDRA_MONITOR_PASSWORD and enable elasticsearch as follow:
    
    kubectl set env statefulset -e ELASSANDRA_MONITOR_PASSWORD="monitor" -e CASSANDRA_DAEMON="org.apache.cassandra.service.ElassandraDaemon" --all

* Once nodes are running, check your elasticsearch cluster and nodes status:

    kubectl exec -ti elassandra-0 -- curl --user monitor:$ELASSANDRA_MONITOR_PASSWORD https://elassandra-0.elassandra.default.svc.cluster.local:9200/_cluster/state?pretty
    kubectl exec -ti elassandra-0 -- curl --user monitor:$ELASSANDRA_MONITOR_PASSWORD https://elassandra-0.elassandra.default.svc.cluster.local:9200/_cat/nodes?v
    
* In order to allow rolling upgarde to the elassandra enterprise image, the cassandra configuration does not enforce SSL encryption, clear node-to-node and client-to-node connection are still allowed. As soon as all nodes in your cluster are SSL-enabled, you can enforce SSL encryption by setting the following variables. I may also require to update the TCP livenessProbe port, because port 7000/tcp is closed when the cassandra inter-node encryption is set to all.

    * CASSANDRA\__client\_encryption\_options\__optional="false"
    * CASSANDRA\__server\_encryption\_options\__internode_encryption="all"
    
    kubectl patch statefulset elassandra -p '{"spec":{"template":{"spec":{"containers":[{"name":"elassandra","livenessProbe":{"tcpSocket":{"port":7001}}}]}}}}'
    kubectl set env statefulset -e CASSANDRA__client_encryption_options__optional="false" -e CASSANDRA__server_encryption_options__internode_encryption="all" --all

To disable cassandra SSL encryption (but keeps the elasticsearch SSL encryption enable):

    kubectl patch statefulset elassandra -p '{"spec":{"template":{"spec":{"containers":[{"name":"elassandra","livenessProbe":{"tcpSocket":{"port":7000}}}]}}}}'
    kubectl set env statefulset -e CASSANDRA__client_encryption_options__enabled="false" -e CASSANDRA__server_encryption_options__internode_encryption="none" --all

## Destroy your cluster

	kubectl delete statefulset -l app=elassandra
	Wait for statefulset termination...
	kubectl delete pvc,pv,service -l app=elassandra

