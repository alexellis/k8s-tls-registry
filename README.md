## Setup a private Docker registry with TLS on Kubernetes

This tutorial will show you how to deploy your own registry on Kubernetes for storing Docker images. You will also learn how to setup TLS certificates which will be issued for free from [LetsEncrypt.com](https://letsencrypt.com).

### Do I need my own registry?

At present there are managed Docker registries offered by almost every cloud provider. Even companies who do not offer compute are starting to offer registries such as [jFrog](https://jfrog.com), [GitLab.com](https://docs.gitlab.com/ee/user/project/container_registry.html), and [GitHub.com](https://github.com/features/package-registry).

So why would you want to set up your own?

* latency

Hosting a registry inside your Kubernetes cluster is the fastest possible way to push and pull images. This matters for use-case such as auto-scaling and affects the overall speed to deploy from a CI/CD pipeline.

* costs

Bandwidth in and out of a datacentre is rarely free, let alone across regions. By hosting Docker images where they are produced and consumed keeps costs to the absolute minimum.

* regulations

Some regulations and legal restrictions such as GDPR may mean that storing artifacts with a SaaS provider is just not tenable.

* security

Although we don't explore it in the scope of this tutorial, additional security can be added to self-hosted registries using Open Source software like the [CNCF's Harbor](https://goharbor.io). Harbor scans Docker images for CVEs and other vulnerabilities.

* automation & portability

You may be able to automate a hosted registry on AWS, but completely different code is required to automate a registry on GCP. By using an Open Source registry that we can self-host, we regain the portability aspect.

* ease of use

It is relatively easy to integrate one or more registries into an existing Kubernetes cluster, in any availability region that you choose.

### Pre-reqs

* A domain name, or sub-domain which you own and can update the DNS records for.

* [Docker](https://www.docker.com/) - we'll use Docker to generate some of our configuration

* [Kubernetes](https://kubernetes.io/) - this tutorial is written with k3s in mind, but also works on Kubernetes with a few tweaks.

For Civo users we'll be hosting our registry on Civo Cloud. You can use the new managed k3s service, if you have early access to the #Kube100 program, or use [k3sup (ketchup)](https://www.civo.com/learn/kubernetes-on-civo-in-5-minutes-flat) to deploy to your own instances.

* [helm](https://helm.sh) - a packaging tool used to install cert-manager and docker-registry

* [cert-manager](https://github.com/jetstack/cert-manager) - provides TLS certificates

* [docker-registry](https://hub.docker.com/_/registry) - Docker's own free, open source registry

> Note: If you are using k3s, you can skip installing Nginx IngressController,

* [nginx-ingress](https://kubernetes.github.io/ingress-nginx/) - The Nginx IngressController configures instances of [Nginx](https://nginx.com) to handle incoming HTTP/s traffic.

### Tutorial

We'll first install helm, then tiller, then Kubernetes users can add [Nginx](https://nginx.com/) in HostMode and k3s users can skip this because they will be using [Traefik](https://traefik.io). After that we'll add cert-manager and an Issuer to obtain certificates, followed by the registry. After everything is installed, we can then make use of our registry using the password created during the tutorial. You'll finish off by testing everything end-to-end, and if you get stuck, there are some helpful tips on how to troubleshoot.

Some components are installed in their own namespaces such as cert-manager, all others will be installed into the `default` namespace. You can control the namespace with `kubectl get --namespace/-n NAME` or `kubectl get --all-namespaces/-A`.

There will also be some ways to take the tutorial further in the appendix.

#### Conceptual architecture

![Registry](/images/registry.png)

Learn how each part works together by following the tutorial.

#### Install the helm CLI/client

Instructions for latest Helm install

* On MacOS and Linux:

      curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash

* Or via Homebrew on Mac:

      brew install kubernetes-helm

For Windows users, go to [helm.sh](https://helm.sh).

#### Install tiller

* Create RBAC permissions for tiller

```sh
kubectl -n kube-system create sa tiller \
  && kubectl create clusterrolebinding tiller \
  --clusterrole cluster-admin \
  --serviceaccount=kube-system:tiller
```

* Install the server-side Tiller component on your cluster

```sh
helm init --skip-refresh --upgrade --service-account tiller
```

> Note: this step installs a server component in your cluster. It can take anywhere between a few seconds to a few minutes to be installed properly. You should see tiller appear on: `kubectl get pods -n kube-system`.

* Now wait for tiller to become ready:

```sh
kubectl rollout status -n kube-system deploy/tiller-deploy

deployment "tiller-deploy" successfully rolled out
```

#### Your built-in IngressController with k3s

Given that neither Civo's k3s service nor k3sup offer a cloud LoadBalancer, we need to use an IngressController. Fortunately k3s comes with one called [Traefik](https://traefik.io/).

For k3s, don't install an IngressController, you already have one, skip ahead.

#### Add an IngressController if not using k3s

**If you're not using k3s**, then install Nginx Ingress instead:

```
helm install stable/nginx-ingress --name nginxingress --set rbac.create=true,controller.hostNetwork=true,controller.daemonset.useHostPort=true,dnsPolicy=ClusterFirstWithHostNet,controller.kind=DaemonSet
```

#### Install cert-manager

You can now install cert-manager, the version used is v0.9.1.

```sh
# Install the CustomResourceDefinition resources separately
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/deploy/manifests/00-crds.yaml

# Create the namespace for cert-manager
kubectl create namespace cert-manager

# Label the cert-manager namespace to disable resource validation
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install \
  --name cert-manager \
  --namespace cert-manager \
  --version v0.9.1 \
  jetstack/cert-manager

```

See also: [cert-manager v0.9.0 docs](https://docs.cert-manager.io/en/release-0.9/)

#### Create a ClusterIssuer

The way that cert-manager issues certificates is through an [Issuer](https://docs.cert-manager.io/en/release-0.9/tutorials/acme/http-validation.html). The `Issuer` can issue certificates for the namespace it is created in, but a `ClusterIssuer` can create certificates for any namespace, so that's the one we will use today.

Save `issuer.yaml`:

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: default
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: user@example.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: traefik
```

* Edit the file:

Edit the line: `email: user@example.com`.

If using Nginx instead of k3s and Traefik, then edit the following:

```
    solvers:
    - http01:
        ingress:
          class: nginx
```

Then run `kubectl apply -f issuer.yaml`.

> Note you may receive an error, if you do then wait 1-2 minutes and try again whilst cert-manager registers itself

You can check the status of your issuer like this:

```
kubectl describe clusterissuer/letsencrypt-prod
```

Look for it to become `Ready`.

#### Configure DNS

For this tutorial a domain `on-k3s.dev` was purchased from Google Domains to show a full worked example.

![](/images/buy-dns.png)

Once you have purchased your domain, you need to point the DNS records at the hosts in the k3s cluster where Nginx is going to be listening on port 80 (HTTP) and port 443 (HTTPS/TLS).

![](/images/add-dns.png)

You can find your IP addresses with the Civo UI, or by typing in `civo instance ls` through the CLI.

#### Install the registry

At this stage we can install the registry, but we are going to install it without persistence. If you need persistence see the appendix for how to do this.

Save the following as `install-registry.sh`:

```sh
export SHA=$(head -c 16 /dev/urandom | shasum | cut -d " " -f 1)
export USER=admin

echo $USER > registry-creds.txt
echo $SHA >> registry-creds.txt

docker run --entrypoint htpasswd registry:2 -Bbn admin $SHA > ./htpasswd

helm install stable/docker-registry \
  --name private-registry \
  --namespace default \
  --set persistence.enabled=false \
  --set secrets.htpasswd=$(cat ./htpasswd)
```

You will need to have `docker` installed and ready for this step. If it's not started, then start it up now.

Then run the script:

```
chmod +x install-registry.sh
./install-registry.sh
```

It will install the Docker registry from [the docker-registry](https://github.com/helm/charts/tree/master/stable/docker-registry) chart.

Later, when you want to use your registry you can find your username and password in the `registry-creds.txt` file.

#### Get a TLS certificate for the registry

Now let's get a TLS certificate for the registry.

Save `ingress.yaml`, then edit it:

```yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: registry
  namespace: default
  annotations:
    certmanager.k8s.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: "traefik"
    nginx.ingress.kubernetes.io/proxy-body-size: 50m
  labels:
    app: docker-registry
spec:
  tls:
  - hosts:
    - registry.example.com
    secretName: registry.example.com-cert
  rules:
  - host: registry.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: private-registry-docker-registry
          servicePort: 5000
```

Update the file:

* Everywhere that you see `registry.example.com`, replace it for your domain.
* If using Nginx, then change this line: `kubernetes.io/ingress.class:`

Note the special setting: `.ingress.kubernetes.io/proxy-body-size: 50m`. This value can be customized and allows large Docker images to be stored in the registry.

Now run:

```sh
kubectl apply -f ingress.yaml
```

### Check the certificate

Now check the certificate with the following:

```
kubectl get cert -n default

NAME                       READY   SECRET                     AGE
registry.on-k3s.dev-cert   True    registry.on-k3s.dev-cert   47s
```

For any of the entries listed, you can check the status with `kubectl describe`:

```
kubectl describe cert/registry.on-k3s.dev-cert

Status:
  Conditions:
    Last Transition Time:  2019-08-29T13:26:20Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2019-11-27T12:26:18Z
Events:
  Type    Reason              Age   From          Message
  ----    ------              ----  ----          -------
  Normal  Generated           64s   cert-manager  Generated new private key
  Normal  GenerateSelfSigned  64s   cert-manager  Generated temporary self signed certificate
  Normal  OrderCreated        63s   cert-manager  Created Order resource "registry.on-k3s.dev-cert-3194477141"
  Normal  OrderComplete       31s   cert-manager  Order "registry.on-k3s.dev-cert-3194477141" completed successfully
  Normal  CertIssued          31s   cert-manager  Certificate issued successfully
```

Look for hints in the *Status* and *Events* sections.

### Now let's test the registry

```
export DOCKER_PASSWORD="" # Populate this with your password used above
export DOCKER_USERNAME="admin"
export SERVER="registry.example.com"

echo $DOCKER_PASSWORD | docker login $SERVER --username $DOCKER_USERNAME --password-stdin
```

> Replace `example.com` with your domain

Sometimes it can take a few minutes for your new domain to become available. If it's an existing domain, then the DNS record should be synchronised already.

Once logged in, you can tag an image from the Docker Hub and push it into your own registry.

```sh
export SERVER="registry.example.com"

docker pull functions/figlet:latest

docker tag functions/figlet:latest $SERVER/functions/figlet:latest

docker push $SERVER/functions/figlet:latest
```

Now that we can log into our registry and push images, we need to enable the same from within our cluster. This is done by attaching an image pull secret to the namespace's service account.

```sh
export DOCKER_PASSWORD="" # Populate this with your password used above
export DOCKER_USERNAME="admin"

kubectl create secret docker-registry my-private-repo \
    --docker-username=$DOCKER_USERNAME \
    --docker-password=$DOCKER_PASSWORD \
    --docker-server=$SERVER \
    --namespace default
```

Now edit the service account and grant it permission to access the secret:

```sh
kubectl edit serviceaccount default -n default
```

Add the following and save:

```yaml
imagePullSecrets:
- name: my-private-repo
```

To check that it's available in Kubernetes, you can run the following [OpenFaaS function](https://github.com/openfaas/faas/), which prints an ASCII logo and then exits.

```sh
export SERVER=""

kubectl run --rm -t -i figlet --restart Never --image $SERVER/functions/figlet:latest -- figlet Kubernetes

 _  __     _                          _            
| |/ /   _| |__   ___ _ __ _ __   ___| |_ ___  ___ 
| ' / | | | '_ \ / _ \ '__| '_ \ / _ \ __/ _ \/ __|
| . \ |_| | |_) |  __/ |  | | | |  __/ ||  __/\__ \
|_|\_\__,_|_.__/ \___|_|  |_| |_|\___|\__\___||___/     

pod "figlet" deleted                                      
```

This will print out the Kubernetes logo in ASCII art and then delete the Pod used to run the code.

If it didn't work, find out why with this command:

```
kubectl get events --sort-by=.metadata.creationTimestamp -n default
```

### Take things further (appendix)

You can take things further and start to explore more advanced use-cases for your registry.

#### Enable persistence

It is desirable, but not essential to enable persistence for a registry. When available, persistence means that if the registry crashes, then the images can be recovered.

There are two routes to enable persistence.

* Use S3, or S3-compatible buckets
    S3 is a protocol and standard for storing objects. You can use [an AWS account](https://aws.amazon.com/s3/) and S3 as a backing for your registry's storage, or you can [install Minio onto your Civo instances](https://www.civo.com/learn/back-up-your-data-using-restic-minio-and-civo) and use it as an S3 target instead.

* Use PersistentVolumes in Kubernetes
    Storage in Kubernetes comes in the shape of Volumes. When volumes are not ephemeral, then they are called PersistentVolumes or (PVs). In order to use PVs with k3s, you'll have to install [Rancher's Longhorn project](https://www.civo.com/learn/cloud-native-stateful-storage-for-kubernetes-with-rancher-labs-longhorn).

The helm chart explains the options for using PVs or S3: [docker-registry chart](https://github.com/helm/charts/tree/master/stable/docker-registry).

## Wrapping up

We have now built a Kubernetes cluster using k3s and have a working registry with TLS, authentication and a public URL.

* helm provided us with charts (packaged software for Kubernetes)
* docker-registry gave us a registry with authentication
* cert-manager provided TLS certificates from LetsEncrypt
* Traefik was built into k3s, or we used Nginx on upstream Kubernetes.

You can now share the registry with your team or use it in your CI/CD pipeline using a tool like Jenkins to build and ship Docker images. You may like to try installing other software to start building applications on Kubernetes such as [OpenFaaS](https://www.civo.com/learn/deploy-openfaas-with-k3s-on-civo).
