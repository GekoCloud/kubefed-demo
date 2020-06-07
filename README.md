# Kubernetes Federation

## Introduction - A cluster to rule them all.

This repo will be used as a complement for the spainclouds meetup where Geko Cloud will be talking about Kubernetes Federation.

All the contents in this repo are supposed to go along with the presentation from the meetup although you can follow this lab regardless.

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

If you wish you can check all the resources that are **enabled by default** along with the long and short names for such resources:

```
kubectl api-resources | grep Federated
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
git clone git@github.com:GekoCloud/kubefed-demo.git
cd kubefed-demo
```

Start by creating a **standard namespace** that will exist only on the master **to host the federated resources**, and then create a **federatednamespace** matching this namespace that will be replicated over all the federated clusters.

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

This means we need to **enable the clusterrolebinding for federation**. If you recall from before, kubefed only enables a set of APIs to be federated but you can enable any additional API afterwards. This is what we are doing here:

```
kubefedctl enable clusterrolebinding
```

> NOTE: If you don't understand what do we mean by enabling the APIs, take a look at the official docs at this section: https://github.com/kubernetes-sigs/kubefed/blob/master/docs/userguide.md#enabling-federation-of-an-api-type

Apply the resources again, you will see that it works properly now.

By inspecting the deployment you can see that this demo will create the following set of resources:

- **Deployment**: Will create a set of pods running nginx, with some overrides over the second cluster.
- **Configmap**: This configmap will be loaded by the nginx pods to show a custom index.html when accessing nginx.
- **Ingress**: An ingress will be created just to show off the overrides functionality, although we will not be accessing it.
- **Service**: This service will create a LoadBalancer on each cluster that will allow us to access the nginx service.

Now that all the resources are created we can check that nginx is working properly on both clusters:

```
for c in kubefed-demo1 kubefed-demo2; do
    NODE_PORT=$(kubectl --context=${c} -n geko-demo get service \
        test-service -o jsonpath='{.spec.ports[0].nodePort}')
    echo; echo ------------ ${c} ------------; echo
    LB=$(kubectl get svc --context=$c | grep test-service | awk '{print $4}')
    echo "check ${LB}"
    curl ${LB}
    echo; echo
done
```

## 5 - Playing with our federated resources

### Example 1: Alter the number of replicas in the second cluster

This is a very simple one. To do this we will add the following override:

```
- path: "/spec/replicas"
    value: 1
```

If you apply the federateddeployment now and check the second cluster you will find that the additional pods that were present will get terminated and only one pod will be left.

### Example 2: Add a label in second cluster

We can easily add a new value anywhere, for example to add new labels we will add the following block:

```
- path: "/metadata/labels"
    op: "add"
    value:
      app: nginx-custom
      project: geko-project
```

Apply the resources and check the deployment using the --show-labels flag.

### Example 3: Mixing federated and non-federated resources.

#### Method 1.

Lets suppose we have two clusters and each one belongs to a different environment: we have one for PROD and one for STAGING, and we want to show different contents in the landing page of our application.

What we are gonna do is to create two **standard configmaps** and apply each one of them on each cluster separately:

```
kubectl create cm skin --from-literal=index.html="production" --context kubefed-demo1
kubectl create cm skin --from-literal=index.html="staging" --context kubefed-demo2
```

Now modify the volume in our deployment by referencing the new configmap:

```
volumes:
- name: geko-way
    configMap:
    name: app-skin
```

Apply the deployment again and see what happens.

What we are seeing now is an independent index.html on each cluster, although we are using the same deployment and the exact same settings in both pods.

#### Method 2.

This time we are gonna do the exact same thing but using a FederatedConfigMap with overrides.

First we will change the configmap in the deployment back to test-configmap. Then we will modify the federatedconfigmap.yaml and will add the appropiate overrides for the second cluster:

```
overrides:
- clusterName: kubefed-demo2
    clusterOverrides:
    - path: "/data"
        value:
        index.html: "staging - using overrides!"
```

Apply the resources and check the endpoint to see the changes reflected.

### Example 4: Disable the configmap on the second cluster

We can remove our volumeMounts with the custom index.html using this override:

```
- path: "/spec/template/spec/containers/0/volumeMounts"
    value: []
```

What this is gonna do is to replace the volumeMounts with an empty array, thus disabling the configmap.

Now apply the deployment again and see what happens when you access the service on both clusters. As you can see in "kubefed-demo2" now we have the default nginx landing page instead of our custom index.html, while "kubefed-demo1" still shows the same page.
