# Setup Kubernetes load balancer with SSL on DigitalOcean

Recipe to setup a load balancer with Letsencrypt enabled SSL on a DigitalOcean managed Kubernetes using the nginx-ingress controller.
This recipe is extracted from the excellent tutorial ["How to Set Up an Nginx Ingress with Cert-Manager on DigitalOcean Kubernetes"](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes) by Hanif Jetha.

![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.png)

## Instructions (windows client)

Prerequisites: a DigitalOcean account, an domain name and the[chocolatey](https://chocolatey.org/) client.

Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-with-chocolatey-on-windows)
and [helm](https://docs.helm.sh/using_helm/#installing-helm) (>=2.12.1).

```bash
choco install kubernetes-cli
choco install kubernetes-helm
```

From a newly [created DigitalOcean cluster](https://www.digitalocean.com/docs/kubernetes/how-to/create-cluster/):

Retrieve the cluster configuration file and set the KUBECONFIG environment variable:

```bash
set KUBECONFIG="D:\Projects\digitalocean\kub\k8s-1-13-1-do-2-nyc1-XXXX-com-prod-kubeconfig.yaml"
```

Install *tiller* :

```bash
kubectl -n kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller
```

Set your email address in `prod_issuer.yaml` (in place of `XXXXXXXXXXX@XXXXXXXX.XXX`).

Setup *certmanager* and certificate issuer:

```
helm install --name cert-manager --namespace kube-system stable/cert-manager
kubectl create -f prod_issuer.yaml
```

Setup *nginx-ingress* and cloud load balancer:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/cloud-generic.yaml
```

Retrieve the load balancer external IP from `kubectl get svc --namespace=ingress-nginx --watch` and configure domain DNS A record appropriately.

Define the your base domain name in `echo_ingress.yaml` (replace the two `XXXXXXXXXXX` occurrences).

Create ingress rule (and generate SSL certificate):

```bash
kubectl apply -f echo_ingress.yaml
```

Monitor the certificate issuing process with `kubectl describe certificate letsencrypt-prod`.

Deploy the `echo` service (hello world service with 2 instances):

```bash
kubectl create -f echo.yaml
```

## Useful commands

```bash
kubectl cluster-info
kubectl get nodes
kubectl get pods --all-namespaces 
kubectl get ing -n default
kubectl describe ing echo-ingress  -n default
kubectl logs -f --since=1h -n  ingress-nginx <nginx-ingress-controller-id>
kubectl describe node
kubectl -n ingress-nginx get pod -o wide
kubectl get svc
kubectl get svc --namespace=ingress-nginx
kubectl describe certificate letsencrypt-prod
```

## Deploying the Kubernetes dashboard

Install the dashboard:

```bash
helm install stable/kubernetes-dashboard --namespace kube-system --name kubernetes-dashboard
```

Retrieve login token:

```bash
kubectl -n kube-system get secret # Choose one of the "service-account-token" secret (e.g. tiller-token-hf2xk)
kubectl -n kube-system describe secret <secret name> # print token
```

Start proxy:

```bash
kubectl proxy
```

Open `http://127.0.0.1:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:https/proxy` and login using the previously obtained token.



## Other resources

* [Sept.2018] [Modernizing Applications for Kubernetes](https://www.digitalocean.com/community/tutorials/modernizing-applications-for-kubernetes)
* [Feb. 2018] [Webinar Series: A Closer Look at Kubernetes](https://www.digitalocean.com/community/tutorials/webinar-series-a-closer-look-at-kubernetes)
* [Mar. 2018] [Webinar Series: Deploying and Scaling Microservices in Kubernetes](https://www.digitalocean.com/community/tutorials/webinar-series-deploying-and-scaling-microservices-in-kubernetes)
* [Mar. 2018]  [Kubernetes NodePort vs LoadBalancer vs Ingress? When should I use what?](https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0)
* [Aug. 2018] [How To Install Software on Kubernetes Clusters with the Helm Package Manager](https://www.digitalocean.com/community/tutorials/how-to-install-software-on-kubernetes-clusters-with-the-helm-package-manager)
* [Nginx Ingress Controller - Bare-metal considerations](https://kubernetes.github.io/ingress-nginx/deploy/baremetal/)
* [Which apiVersion should I use?](https://matthewpalmer.net/kubernetes-app-developer/articles/kubernetes-apiversion-definition-guide.html)