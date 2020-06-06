# Kubernetes Federation

## Introduction - A cluster to rule them all.

### ¿Qué es la federación en Kubernetes?

La federación en Kubernetes consiste en la capacidad de gestionar y coordinar múltiples clústers de kubernetes desde un control plane maestro usando un conjunto de APIs que tendremos disponibles para este fin.

Tal como veremos más adelante nos ofrece gran flexibilidad con la intención de ajustarse a casos complejos como por ejemplo el despliegue de aplicaciones multi-región o situaciones de disaster recovery.

### Ventajas e inconvenientes

#### Ventajas

- Incrementa la disponibilidad de las aplicaciones usando clústers multi-region
- Posibilidad de despliegues cross-cloud provider (GCP, AWS, Azure)
- Posibilidad de despliegues conjuntos en cloud y on-premise
- Failover entre clústers redundantes (Disaster Recovery)
- Gestión centralizada de múltiples entornos (despliega en un solo lugar)

#### Desventajas

- Incremento considerable en la complejidad
- Menor fiabilidad si no se realiza una buena gestión

# LAB TIME! Step-by-step

> This lab assumes that you already have at least two kubernetes clusters available with kubeconfig setup.

> Also note that this demo is meant to be run on Google Cloud, although you can run this on any cloud where you have the chance to automatically create LoadBalancers through a kubernetes service. If you opt for Google Cloud, remember that you have $300 in credits as a new user. I expect this to cost less than $5 in credits if you use the resources for less than 24 hours.

## 0 - Pre-steps Install helm (be sure to be using latest helm binary)

Download the latest 2.x release:

```
wget https://get.helm.sh/helm-v2.16.7-linux-arm64.tar.gz
tar xf helm-v2.16.7-linux-arm64.tar.gz
sudo mv linux-arm64/helm /usr/local/bin
```

Now init helm:

```
helm init --service-account tiller
```

## 1 - Download the kubefedctl tool

Kubefed has its own ctl to perform some operations like joining/unjoining clústers, enable new APIs for federation and so on.

```
wget https://github.com/kubernetes-sigs/kubefed/releases/download/v0.3.0/kubefedctl-0.3.0-linux-amd64.tgz
xf kubefedctl-0.3.0-linux-amd64.tgz
sudo mv kubefedctl /usr/bin
```

## 2 - Install Kubefed helm chart

Note that we will install Kubefed on the MASTER. We will be using the official helm chart and will install the latest version available as of now (0.3.0):

```
helm repo add kubefed-charts https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
helm install kubefed-charts/kubefed --name kubefed --version=0.3.0 --namespace kube-federation-system
```

## 3 - Cluster registration

Now it's time for cluster registraiton. We will be joining the two clusters to the master cluster where kubefed is installed. As you can see **the master itself will join as a slave**.

```
kubefedctl join kubefed-demo1 --cluster-context kubefed-demo1 --host-cluster-context kubefed-demo1 --v=2
kubefedctl join kubefed-demo2 --cluster-context kubefed-demo2 --host-cluster-context kubefed-demo1 --v=2
```

Check status of the joined clusters:

```
kubectl -n kube-federation-system get kubefedclusters
```

## 4 - Deploy the demo application

Clone this repo if you haven't yet.

```
git clone git@github.com:GekoCloud/kubefed-demo-app.git
cd kubefed-demo-app
```

We will start by creating a standard namespace that will exist only on the master **to host the federated resources**, and a federatednamespace matching this namespace that will be replicated over all the federated clusters.

```
kubectl apply -f ./sample1/namespace.yaml -f ./sample1/federatednamespace.yaml
```

And now we will apply all the resources in the sample folder:

```
kubectl apply -R -f .
```

At this point we will most likely receive an error:

```
error: unable to recognize "sample1/federatedclusterrolebinding.yaml": no matches for kind "FederatedClusterRoleBinding" in version "types.kubefed.io/v1beta1"
```

This means we need to enable the clusterrolebinding for federation. If you recall, kubefed only enables a set of APIs to be federated but you can enable any additional API afterwards. This is what we are doing here:

```
kubefedctl enable clusterrolebinding
```

Apply all the sample folder again, you will see that it works properly now.

By inspecting the deployment you can see that this demo will create the following set of resources:

- Deployment: Will create a set of pods running nginx, with some overrides over the second cluster.
- Configmap: This configmap will be loaded by the nginx pods to show a custom index.html when accessing nginx.
- Ingress: An ingress will be created just to show off the overrides functionality, although we will not be accessing it.
- Service: This service will create a LoadBalancer on each cluster that will allow us to access the nginx service.

Now that all the resources are created we can check that nginx is working properly on both clusters:

```
for c in kubefed-demo1 kubefed-demo2; do
    NODE_PORT=$(kubectl --context=${c} -n test-namespace get service \
        test-service -o jsonpath='{.spec.ports[0].nodePort}')
    echo; echo ------------ ${c} ------------; echo
    LB=$(kubectl get svc --context=kubefed-demo1 | grep test-service | awk '{print $4}')
    curl ${LB}
    echo; echo
done
```
