# Traefik

Træfik can act as an Ingress controller for a Kubernetes cluster, see [documentation](https://docs.traefik.io/user-guide/kubernetes/).

## Traefik Helm deployment

1.Check the tiller (helm server) is properly running:

    kubectl -n kube-system get po | grep tiller

2.Change your domainename and namespace in traefik/traefik-values.yaml

    helm install --values values.yaml --name my-release stable/traefik
    
## Update a variable

    helm upgrade --set debug.enabled=true my-release stable/traefik

1. Get Traefik's load balancer IP/hostname:

NOTE: It may take a few minutes for this to become available.
You can watch the status by running:

    kubectl get svc my-release-traefik --namespace kube-system -w

Once 'EXTERNAL-IP' is no longer '<pending>':

    kubectl describe svc my-release-traefik --namespace kube-system | grep Ingress | awk '{print $3}'

2. Configure DNS records corresponding to Kubernetes ingress resources to point to the load balancer IP/hostname found in step 1, and check that you have access to the traefik-dashbord.

## Uninstalling the Treafik

To uninstall/delete the my-release deployment:

    helm delete my-release