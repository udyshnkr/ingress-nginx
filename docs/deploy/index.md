# Installation Guide

!!! attention
    The default configuration watches Ingress object from **all the namespaces**.

    To change this behavior use the flag `--watch-namespace` to limit the scope to a particular namespace.

!!! warning
    If multiple Ingresses define paths for the same host, the ingress controller **merges the definitions**.

!!! danger
    The [admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) require conectivity between Kubernetes API server and the ingress controller.

    In case [Network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) or additional firewalls, please allow access to port `8443`.

!!! attention
    The first time the ingress controller starts, two [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/) create the SSL Certificate used by the admission webhook.
    For this reason, there is an initial delay of up to two minutes until it is possible to create and validate Ingress definitions.

    You can wait until is ready to running the next command:

    ```yaml
    kubectl wait --namespace ingress-nginx \
      --for=condition=ready pod \
      --selector=app.kubernetes.io/component=controller \
      --timeout=120s
    ```

## Contents

- [Provider Specific Steps](#provider-specific-steps)
  - [Docker for Mac](#docker-for-mac)
  - [minikube](#minikube)
  - [AWS](#aws)
  - [GCE - GKE](#gce-gke)
  - [Azure](#azure)
  - [Digital Ocean](#digital-ocean)
  - [Bare-metal](#bare-metal)
  - [Verify installation](#verify-installation)
  - [Detect installed version](#detect-installed-version)
- [Using Helm](#using-helm)

### Provider Specific Steps

#### Docker for Mac

Kubernetes is available in Docker for Mac (from [version 18.06.0-ce](https://docs.docker.com/docker-for-mac/release-notes/#stable-releases-of-2018))

[enable]: https://docs.docker.com/docker-for-mac/#kubernetes

```console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud/deploy.yaml
```

#### minikube

For standard usage:

```console
minikube addons enable ingress
```

For development:

- Disable the ingress addon:

```console
minikube addons disable ingress
```

- Execute `make dev-env`
- Confirm the `ingress-nginx-controller` deployment exists:

```console
$ kubectl get pods -n ingress-nginx
NAME                                       READY     STATUS    RESTARTS   AGE
ingress-nginx-controller-fdcdcd6dd-vvpgs   1/1       Running   0          11s
```

#### AWS

In AWS we use a Network load balancer (NLB) to expose the NGINX Ingress controller behind a Service of `Type=LoadBalancer`.

##### Network Load Balancer (NLB)

```console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/aws/deploy.yaml
```

##### TLS termination in AWS Load Balancer (ELB)

In some scenarios is required to terminate TLS in the Load Balancer and not in the ingress controller.

For this purpose we provide a template:

- Download [deploy-tls-termination.yaml](https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/aws/deploy-tls-termination.yaml)

```console
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/aws/deploy-tls-termination.yaml
```

- Edit the file and change:

  - VPC CIDR in use for the Kubernetes cluster:

  `proxy-real-ip-cidr: XXX.XXX.XXX/XX`

  - AWS Certificate Manager (ACM) ID

  `arn:aws:acm:us-west-2:XXXXXXXX:certificate/XXXXXX-XXXXXXX-XXXXXXX-XXXXXXXX`

- Deploy the manifest:

```console
kubectl apply -f deploy-tls-termination.yaml
```

##### NLB Idle Timeouts

In some scenarios users will need to modify the value of the NLB idle timeout. Users need to ensure the idle timeout is less than the [keepalive_timeout](http://nginx.org/en/docs/http/ngx_http_core_module.html#keepalive_timeout) that is configured for NGINX.
By default NGINX `keepalive_timeout` is set to `75s`.

The default NLB idle timeout works for most scenarios, unless the NGINX [keepalive_timeout](http://nginx.org/en/docs/http/ngx_http_core_module.html#keepalive_timeout) has been modified, in which case the annotation

`service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout` value must be modified to ensure it is less than the configured `keepalive_timeout`.

!!! note ""
    An idle timeout of `3600` is recommended when using WebSockets

More information with regards to timeouts for can be found in the [official AWS documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancers.html#connection-idle-timeout)

#### GCE-GKE

!!! info
    Initialize your user as a cluster-admin with the following command:
    ```console
    kubectl create clusterrolebinding cluster-admin-binding \
      --clusterrole cluster-admin \
      --user $(gcloud config get-value account)
    ```

!!! danger
    For private clusters, you will need to either add an additional firewall rule that allows master nodes access port `8443/tcp` on worker nodes, or change the existing rule that allows access to ports `80/tcp`, `443/tcp` and `10254/tcp` to also allow access to port `8443/tcp`.

    See the [GKE documentation](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters#add_firewall_rules) on adding rules and the [Kubernetes issue](https://github.com/kubernetes/kubernetes/issues/79739) for more detail.


```console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud/deploy.yaml
```

!!! failure Important
    Proxy protocol is not supported in GCE/GKE

#### Azure

```console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud/deploy.yaml
```

#### Digital Ocean

```console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/do/deploy.yaml
```

#### Bare-metal

Using [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport):

```console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml
```

!!! tip
    For extended notes regarding deployments on bare-metal, see [Bare-metal considerations](./baremetal.md).

### Verify installation

!!! info
    In minikube the ingress addon is installed in the namespace **kube-system** instead of ingress-nginx

To check if the ingress controller pods have started, run the following command:

```console
kubectl get pods -n ingress-nginx \
  -l app.kubernetes.io/name=ingress-nginx --watch
```

Once the ingress controller pods are running, you can cancel the command typing `Ctrl+C`.

Now, you are ready to create your first ingress.

### Detect installed version

To detect which version of the ingress controller is running, exec into the pod and run `nginx-ingress-controller version` command.

```console
POD_NAMESPACE=ingress-nginx
POD_NAME=$(kubectl get pods -n $POD_NAMESPACE -l app.kubernetes.io/name=ingress-nginx --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')

kubectl exec -it $POD_NAME -n $POD_NAMESPACE -- /nginx-ingress-controller --version
```

## Using Helm

NGINX Ingress controller can be installed via [Helm](https://helm.sh/) using the chart from the project repository.
To install the chart with the release name `ingress-nginx`:

```console
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install my-release ingress-nginx/ingress-nginx
```

If you are using [Helm 2](https://v2.helm.sh/) then specify release name using `--name` flag

```console
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx/
helm install --name ingress-nginx ingress-nginx/ingress-nginx
```

## Detect installed version:

```console
POD_NAME=$(kubectl get pods -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD_NAME -- /nginx-ingress-controller --version
```
