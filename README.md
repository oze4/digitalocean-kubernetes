# digitalocean-kubernetes
Officially using DigitalOcean's Kubernetes offering

Using [this article](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes)

---

## The article linked above, but converted to markdown:
Just in case they change anything...

- Instead of using the snippets in the article below, you can use the manifests in this repo
- **There are a handful of URLs that you will need to change, though!**

---

```
!
```
"Contents

```
By Hanif Jetha
Become an author
```
## Introduction

Kubernetes Ingresses allow you to flexibly route traffic from outside your Kubernetes cluster to

Services inside of your cluster. This is accomplished using Ingress Resources, which define rules

for routing HTTP and HTTPS traffic to Kubernetes Services, and Ingress Controllers, which

implement the rules by load balancing traffic and routing it to the appropriate backend Services.

```
#
```
# How to Set Up an Nginx Ingress with Cert-

# Manager on DigitalOcean Kubernetes

PostedDecember 14, 2018 $61.5k KUBERNETES SECURITY NGINX SOLUTIONS LET'S ENCRYPT


Popular Ingress Controllers include Nginx, Contour, HAProxy, and Traefik. Ingresses provide a more

efficient and flexible alternative to setting up multiple LoadBalancer services, each of which uses

its own dedicated Load Balancer.

In this guide, we’ll set up the Kubernetes-maintained Nginx Ingress Controller, and create some

Ingress Resources to route traffic to several dummy backend services. Once we’ve set up the

Ingress, we’ll install cert-manager into our cluster to manage and provision TLS certificates for

encrypting HTTP traffic to the Ingress.

## Prerequisites

Before you begin with this guide, you should have the following available to you:

A Kubernetes 1.10+ cluster with role-based access control (RBAC) enabled

```
The kubectl command-line tool installed on your local machine and configured to connect
to your cluster. You can read more about installing kubectlin the official documentation.
```
```
A domain name and DNS A records which you can point to the DigitalOcean Load Balancer
used by the Ingress. If you are using DigitalOcean to manage your domain’s DNS records,
consult How to Manage DNS Records to learn how to create A records.
```
```
The Helm package manager installed on your local machine and Tiller installed on your
cluster, as detailed in How To Install Software on Kubernetes Clusters with the Helm Package
Manager. Ensure that you are using Helm v2.12.1 or later or you may run into issues installing
the cert-manager Helm chart. To check the Helm version you have installed, run helm
version on your local machine.
```
```
The wget command-line utility installed on your local machine. You can install wget using
the package manager built into your operating system.
```
Once you have these components set up, you’re ready to begin with this guide.

## Step 1 — Setting Up Dummy Backend Services

Before we deploy the Ingress Controller, we’ll first create and roll out two dummy echo Services to

which we’ll route external traffic using the Ingress. The echo Services will run the

hashicorp/http-echo web server container, which returns a page containing a text string passed

in when the web server is launched. To learn more about http-echo, consult its GitHub Repo, and

to learn more about Kubernetes Services, consult Services from the official Kubernetes docs.


On your local machine, create and edit a file called echo1.yaml using nano or your favorite editor:

```
$ nano echo1.yaml
```
Paste in the following Service and Deployment manifest:

```
apiVersion: v
kind: Service
metadata:
name: echo
spec:
ports:
```
- port: 80
targetPort: 5678
selector:
app: echo
---
apiVersion: apps/v
kind: Deployment
metadata:
name: echo
spec:
selector:
matchLabels:
app: echo
replicas: 2
template:
metadata:
labels:
app: echo
spec:
containers:
- name: echo
image: hashicorp/http-echo
args:
- "-text=echo1"
ports:
- containerPort: 5678

In this file, we define a Service called echo1 which routes traffic to Pods with the app: echo1 label

```
echo1.yaml
```

selector. It accepts TCP traffic on port 80 and routes it to port 5678 ,http-echo’s default port.

We then define a Deployment, also called echo1, which manages Pods with the app: echo

Label Selector. We specify that the Deployment should have 2 Pod replicas, and that the Pods

should start a container called echo1 running the hashicorp/http-echo image. We pass in the

text parameter and set it to echo1, so that the http-echo web server returns echo1. Finally, we

open port 5678 on the Pod container.

Once you’re satisfied with your dummy Service and Deployment manifest, save and close the file.

Then, create the Kubernetes resources using kubectl create with the -f flag, specifying the file

you just saved as a parameter:

```
$ kubectl create -f echo1.yaml
```
You should see the following output:

```
Output
service/echo1 created
deployment.apps/echo1 created
```
Verify that the Service started correctly by confirming that it has a ClusterIP, the internal IP on

which the Service is exposed:

```
$ kubectl get svc echo
```
You should see the following output:

```
Output
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
echo1 ClusterIP 10.245.222.129 <none> 80/TCP 60s
```
This indicates that the echo1 Service is now available internally at 10.245.222.129 on port 80. It

will forward traffic to containerPort 5678 on the Pods it selects.

Now that the echo1 Service is up and running, repeat this process for the echo2 Service.


Create and open a file called echo2.yaml:

```
apiVersion: v
kind: Service
metadata:
name: echo
spec:
ports:
```
- port: 80
targetPort: 5678
selector:
app: echo
---
apiVersion: apps/v
kind: Deployment
metadata:
name: echo
spec:
selector:
matchLabels:
app: echo
replicas: 1
template:
metadata:
labels:
app: echo
spec:
containers:
- name: echo
image: hashicorp/http-echo
args:
- "-text=echo2"
ports:
- containerPort: 5678

Here, we essentially use the same Service and Deployment manifest as above, but name and

relabel the Service and Deployment echo2. In addition, to provide some variety, we create only 1

Pod replica. We ensure that we set the text parameter to echo2 so that the web server returns

the text echo2.

Save and close the file, and create the Kubernetes resources using kubectl:

```
echo2.yaml
```

```
$ kubectl create -f echo2.yaml
```
You should see the following output:

```
Output
service/echo2 created
deployment.apps/echo2 created
```
Once again, verify that the Service is up and running:

```
$ kubectl get svc
```
You should see both the echo1 and echo2 Services with assigned ClusterIPs:

```
Output
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
echo1 ClusterIP 10.245.222.129 <none> 80/TCP 6m6s
echo2 ClusterIP 10.245.128.224 <none> 80/TCP 6m3s
kubernetes ClusterIP 10.245.0.1 <none> 443/TCP 4d21h
```
Now that our dummy echo web services are up and running, we can move on to rolling out the

Nginx Ingress Controller.

## Step 2 — Setting Up the Kubernetes Nginx Ingress Controller

In this step, we’ll roll out v0.24.1 of the Kubernetes-maintained Nginx Ingress Controller. Note

that there are several Nginx Ingress Controllers; the Kubernetes community maintains the one used

in this guide and Nginx Inc. maintains kubernetes-ingress. The instructions in this tutorial are based

on those from the official Kubernetes Nginx Ingress Controller Installation Guide.

The Nginx Ingress Controller consists of a Pod that runs the Nginx web server and watches the

Kubernetes Control Plane for new and updated Ingress Resource objects. An Ingress Resource is

essentially a list of traffic routing rules for backend Services. For example, an Ingress rule can

specify that HTTP traffic arriving at the path /web1 should be directed towards the web1 backend

web server. Using Ingress Resources, you can also perform host-based routing: for example,

routing requests that hit web1.your_domain.com to the backend Kubernetes Service web1.


In this case, because we’re deploying the Ingress Controller to a DigitalOcean Kubernetes cluster,

the Controller will create a LoadBalancer Service that spins up a DigitalOcean Load Balancer to

which all external traffic will be directed. This Load Balancer will route external traffic to the Ingress

Controller Pod running Nginx, which then forwards traffic to the appropriate backend Services.

We’ll begin by first creating the Kubernetes resources required by the Nginx Ingress Controller.

These consist of ConfigMaps containing the Controller’s configuration, Role-based Access

Control (RBAC) Roles to grant the Controller access to the Kubernetes API, and the actual Ingress

Controller Deployment which uses v0.24.1 of the Nginx Ingress Controller image. To see a full list of

these required resources, consult the manifest from the Kubernetes Nginx Ingress Controller’s

GitHub repo.

To create these mandatory resources, use kubectl apply and the -f flag to specify the manifest

file hosted on GitHub:

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.24.
```
We use apply instead of create here so that in the future we can incrementally apply changes

to the Ingress Controller objects instead of completely overwriting them. To learn more about

apply, consult Managing Resources from the official Kubernetes docs.

You should see the following output:

```
Output
namespace/ingress-nginx created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
deployment.extensions/nginx-ingress-controller created
```
This output also serves as a convenient summary of all the Ingress Controller objects created from

the mandatory.yaml manifest.


Next, we’ll create the Ingress Controller LoadBalancer Service, which will create a DigitalOcean

Load Balancer that will load balance and route HTTP and HTTPS traffic to the Ingress Controller

Pod deployed in the previous command.

To create the LoadBalancer Service, once again kubectl apply a manifest file containing the

Service definition:

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.24.
```
You should see the following output:

```
Output
service/ingress-nginx created
```
Now, confirm that the DigitalOcean Load Balancer was successfully created by fetching the

Service details with kubectl:

```
$ kubectl get svc --namespace=ingress-nginx
```
You should see an external IP address, corresponding to the IP address of the DigitalOcean Load

Balancer:

```
Output
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
ingress-nginx LoadBalancer 10.245.247.67 203.0.113.0 80:32486/TCP,443:32096/TCP 20h
```
Note down the Load Balancer’s external IP address, as you’ll need it in a later step.

```
Note: By default the Nginx Ingress LoadBalancer Service has
service.spec.externalTrafficPolicy set to the value Local, which routes all load balancer
traffic to nodes running Nginx Ingress Pods. The other nodes will deliberately fail load balancer
health checks so that Ingress traffic does not get routed to them. External traffic policies are beyond
the scope of this tutorial, but to learn more you can consult A Deep Dive into Kubernetes External
Traffic Policies and Source IP for Services with Type=LoadBalancer from the official Kubernetes
docs.
```

This load balancer receives traffic on HTTP and HTTPS ports 80 and 443, and forwards it to the

Ingress Controller Pod. The Ingress Controller will then route the traffic to the appropriate backend

Service.

We can now point our DNS records at this external Load Balancer and create some Ingress

Resources to implement traffic routing rules.

## Step 3 — Creating the Ingress Resource

Let’s begin by creating a minimal Ingress Resource to route traffic directed at a given subdomain

to a corresponding backend Service.

In this guide, we’ll use the test domain example.com. You should substitute this with the domain

name you own.

We’ll first create a simple rule to route traffic directed at echo1.example.com to the echo

backend service and traffic directed at echo2.example.com to the echo2 backend service.

Begin by opening up a file called echo_ingress.yaml in your favorite editor:

```
$ nano echo_ingress.yaml
```
Paste in the following ingress definition:

```
apiVersion: extensions/v1beta
kind: Ingress
metadata:
name: echo-ingress
spec:
rules:
```
- host: echo1.example.com
[http:](http:)
paths:
- backend:
serviceName: echo
servicePort: 80
- host: echo2.example.com

```
echo_ingress.yaml
```

```
http:
paths:
```
- backend:
serviceName: echo
servicePort: 80

When you’ve finished editing your Ingress rules, save and close the file.

Here, we’ve specified that we’d like to create an Ingress Resource called echo-ingress, and route

traffic based on the Host header. An HTTP request Host header specifies the domain name of the

target server. To learn more about Host request headers, consult the Mozilla Developer Network

definition page. Requests with host echo1.example.com will be directed to the echo1 backend set

up in Step 1, and requests with host echo2.example.com will be directed to the echo2 backend.

You can now create the Ingress using kubectl:

```
$ kubectl apply -f echo_ingress.yaml
```
You’ll see the following output confirming the Ingress creation:

```
Output
ingress.extensions/echo-ingress created
```
To test the Ingress, navigate to your DNS management service and create A records for

echo1.example.com and echo2.example.com pointing to the DigitalOcean Load Balancer’s

external IP. The Load Balancer’s external IP is the external IP address for the ingress-nginx

Service, which we fetched in the previous step. If you are using DigitalOcean to manage your

domain’s DNS records, consult How to Manage DNS Records to learn how to create A records.

Once you’ve created the necessary echo1.example.com and echo2.example.com DNS records,

you can test the Ingress Controller and Resource you’ve created using the curl command line

utility.

From your local machine, curl the echo1 Service:

```
$ curl echo1.example.com
```

You should get the following response from the echo1 service:

```
Output
echo
```
This confirms that your request to echo1.example.com is being correctly routed through the Nginx

ingress to the echo1 backend Service.

Now, perform the same test for the echo2 Service:

```
$ curl echo2.example.com
```
You should get the following response from the echo2 Service:

```
Output
echo
```
This confirms that your request to echo2.example.com is being correctly routed through the Nginx

ingress to the echo2 backend Service.

At this point, you’ve successfully set up a basic Nginx Ingress to perform virtual host-based routing.

In the next step, we’ll install cert-manager using Helm to provision TLS certificates for our Ingress

and enable the more secure HTTPS protocol.

## Step 4 — Installing and Configuring Cert-Manager

In this step, we’ll use Helm to install cert-manager into our cluster. cert-manager is a Kubernetes

service that provisions TLS certificates from Let’s Encrypt and other certificate authorities and

manages their lifecycles. Certificates can be requested and configured by annotating Ingress

Resources with the certmanager.k8s.io/issuer annotation, appending a tls section to the

Ingress spec, and configuring one or more Issuers to specify your preferred certificate authority. To

learn more about Issuer objects, consult the official cert-manager documentation on Issuers.

```
Note: Ensure that you are using Helm v2.12.1 or later before installing cert-manager. To check the
Helm version you have installed, run helm version on your local machine.
```

Before using Helm to install cert-manager into our cluster, we need to create the cert-manager

Custom Resource Definitions (CRDs). Create these by applying them directly from the cert-

manager GitHub repository :

```
$ kubectl apply \
$ -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.8/deploy/manifests/00-crds.ya
```
You should see the following output:

```
Output
customresourcedefinition.apiextensions.k8s.io/certificates.certmanager.k8s.io created
customresourcedefinition.apiextensions.k8s.io/issuers.certmanager.k8s.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.certmanager.k8s.io created
customresourcedefinition.apiextensions.k8s.io/orders.certmanager.k8s.io created
customresourcedefinition.apiextensions.k8s.io/challenges.certmanager.k8s.io created
```
Next, we’ll add a label to the kube-system namespace, where we’ll install cert-manager, to enable

advanced resource validation using a webhook:

```
$ kubectl label namespace kube-system certmanager.k8s.io/disable-validation="true"
```
Now, we’ll add the Jetstack Helm repository to Helm. This repository contains the cert-manager

Helm chart.

```
$ helm repo add jetstack https://charts.jetstack.io
```
Finally, we can install the chart into the kube-system namespace:

```
$ helm install --name cert-manager --namespace kube-system jetstack/cert-manager --version v0.8.
```
You should see the following output:

```
Output
```
...
NOTES:


```
cert-manager has been deployed successfully!
```
```
In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).
```
```
More information on the different types of issuers and how to configure them
can be found in our documentation:
```
```
https://cert-manager.readthedocs.io/en/latest/reference/issuers.html
```
```
For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:
```
```
https://cert-manager.readthedocs.io/en/latest/reference/ingress-shim.html
```
This indicates that the cert-manager installation succeeded.

Before we begin issuing certificates for our Ingress hosts, we need to create an Issuer, which

specifies the certificate authority from which signed x509 certificates can be obtained. In this

guide, we’ll use the Let’s Encrypt certificate authority, which provides free TLS certificates and

offers both a staging server for testing your certificate configuration, and a production server for

rolling out verifiable TLS certificates.

Let’s create a test Issuer to make sure the certificate provisioning mechanism is functioning

correctly. Open a file named staging_issuer.yaml in your favorite text editor:

```
nano staging_issuer.yaml
```
Paste in the following ClusterIssuer manifest:

```
apiVersion: certmanager.k8s.io/v1alpha
kind: ClusterIssuer
metadata:
name: letsencrypt-staging
spec:
acme:
# The ACME server URL
```
```
staging_issuer.yaml
```

```
server: https://acme-staging-v02.api.letsencrypt.org/directory
# Email address used for ACME registration
email: your_email_address_here
# Name of a secret used to store the ACME account private key
privateKeySecretRef:
name: letsencrypt-staging
# Enable the HTTP-01 challenge provider
http01: {}
```
Here we specify that we’d like to create a ClusterIssuer object called letsencrypt-staging, and

use the Let’s Encrypt staging server. We’ll later use the production server to roll out our certificates,

but the production server may rate-limit requests made against it, so for testing purposes it’s best

to use the staging URL.

We then specify an email address to register the certificate, and create a Kubernetes Secret called

letsencrypt-staging to store the ACME account’s private key. We also enable the HTTP-

challenge mechanism. To learn more about these parameters, consult the official cert-manager

documentation on Issuers.

Roll out the ClusterIssuer using kubectl:

```
$ kubectl create -f staging_issuer.yaml
```
You should see the following output:

```
Output
clusterissuer.certmanager.k8s.io/letsencrypt-staging created
```
Now that we’ve created our Let’s Encrypt staging Issuer, we’re ready to modify the Ingress

Resource we created above and enable TLS encryption for the echo1.example.com and

echo2.example.com paths.

Open up echo_ingress.yaml once again in your favorite editor:

```
$ nano echo_ingress.yaml
```
Add the following to the Ingress Resource manifest:


```
apiVersion: extensions/v1beta
kind: Ingress
metadata:
name: echo-ingress
annotations:
kubernetes.io/ingress.class: nginx
certmanager.k8s.io/cluster-issuer: letsencrypt-staging
spec:
tls:
```
- hosts:
    - echo1.example.com
    - echo2.example.com
    secretName: letsencrypt-staging
rules:
- host: echo1.example.com
[http:](http:)
paths:
- backend:
serviceName: echo
servicePort: 80
- host: echo2.example.com
[http:](http:)
paths:
- backend:
serviceName: echo
servicePort: 80

Here we add some annotations to specify the ingress.class, which determines the Ingress

Controller that should be used to implement the Ingress Rules. In addition, we define the

cluster-issuer to be letsencrypt-staging, the certificate Issuer we just created.

Finally, we add a tls block to specify the hosts for which we want to acquire certificates, and

specify a secretName. This secret will contain the TLS private key and issued certificate.

When you’re done making changes, save and close the file.

We’ll now update the existing Ingress Resource using kubectl apply:

```
$ kubectl apply -f echo_ingress.yaml
```
```
echo_ingress.yaml
```

You should see the following output:

```
Output
ingress.extensions/echo-ingress configured
```
You can use kubectl describe to track the state of the Ingress changes you’ve just applied:

```
$ kubectl describe ingress
```
```
Output
Events:
Type Reason Age From Message
---- ------ ---- ---- -------
Normal CREATE 14m nginx-ingress-controller Ingress default/echo-ingress
Normal UPDATE 1m (x2 over 13m) nginx-ingress-controller Ingress default/echo-ingress
Normal CreateCertificate 1m cert-manager Successfully created Certificate
```
Once the certificate has been successfully created, you can run an additional describe on it to

further confirm its successful creation:

```
$ kubectl describe certificate
```
You should see the following output in the Events section:

```
Output
Events:
Type Reason Age From Message
---- ------ ---- ---- -------
Normal Generated 63s cert-manager Generated new private key
Normal OrderCreated 63s cert-manager Created Order resource "letsencrypt-staging-147606226"
Normal OrderComplete 19s cert-manager Order "letsencrypt-staging-147606226" completed successfully
Normal CertIssued 18s cert-manager Certificate issued successfully
```
This confirms that the TLS certificate was successfully issued and HTTPS encryption is now active

for the two domains configured.


We’re now ready to send a request to a backend echo server to test that HTTPS is functioning

correctly.

Run the following wget command to send a request to echo1.example.com and print the

response headers to STDOUT:

```
$ wget --save-headers -O- echo1.example.com
```
You should see the following output:

```
Output
URL transformed to HTTPS due to an HSTS policy
--2018-12-11 14:38:24-- https://echo1.example.com/
Resolving echo1.example.com (echo1.example.com)... 203.0.113.
Connecting to echo1.example.com (echo1.example.net)|203.0.113.0|:443... connected.
ERROR: cannot verify echo1.example.com's certificate, issued by ‘CN=Fake LE Intermediate X1’:
Unable to locally verify the issuer's authority.
To connect to echo1.example.com insecurely, use `--no-check-certificate'.
```
This indicates that HTTPS has successfully been enabled, but the certificate cannot be verified as

it’s a fake temporary certificate issued by the Let’s Encrypt staging server.

Now that we’ve tested that everything works using this temporary fake certificate, we can roll out

production certificates for the two hosts echo1.example.com and echo2.example.com.

## Step 5 — Rolling Out Production Issuer

In this step we’ll modify the procedure used to provision staging certificates, and generate a valid,

verifiable production certificate for our Ingress hosts.

To begin, we’ll first create a production certificate ClusterIssuer.

Open a file called prod_issuer.yaml in your favorite editor:

```
nano prod_issuer.yaml
```
Paste in the following manifest:


```
apiVersion: certmanager.k8s.io/v1alpha
kind: ClusterIssuer
metadata:
name: letsencrypt-prod
spec:
acme:
# The ACME server URL
server: https://acme-v02.api.letsencrypt.org/directory
# Email address used for ACME registration
email: your_email_address_here
# Name of a secret used to store the ACME account private key
privateKeySecretRef:
name: letsencrypt-prod
# Enable the HTTP-01 challenge provider
http01: {}
```
Note the different ACME server URL, and the letsencrypt-prod secret key name.

When you’re done editing, save and close the file.

Now, roll out this Issuer using kubectl:

```
$ kubectl create -f prod_issuer.yaml
```
You should see the following output:

```
Output
clusterissuer.certmanager.k8s.io/letsencrypt-prod created
```
Update echo_ingress.yaml to use this new Issuer:

```
$ nano echo_ingress.yaml
```
Make the following changes to the file:

```
apiVersion: extensions/v1beta
```
```
prod_issuer.yaml
```
```
echo_ingress.yaml
```

```
kind: Ingress
metadata:
name: echo-ingress
annotations:
kubernetes.io/ingress.class: nginx
certmanager.k8s.io/cluster-issuer: letsencrypt-prod
spec:
tls:
```
- hosts:
- echo1.example.com
- echo2.example.com
secretName: letsencrypt-prod
rules:
- host: echo1.example.com
[http:](http:)
paths:
- backend:
serviceName: echo
servicePort: 80
- host: echo2.example.com
[http:](http:)
paths:
- backend:
serviceName: echo
servicePort: 80

Here, we update both the ClusterIssuer and secret name to letsencrypt-prod.

Once you’re satisfied with your changes, save and close the file.

Roll out the changes using kubectl apply:

```
$ kubectl apply -f echo_ingress.yaml
```
```
Output
ingress.extensions/echo-ingress configured
```
Wait a couple of minutes for the Let’s Encrypt production server to issue the certificate. You can

track its progress using kubectl describe on the certificate object:


```
$ kubectl describe certificate letsencrypt-prod
```
Once you see the following output, the certificate has been issued successfully:

```
Output
Events:
Type Reason Age From Message
---- ------ ---- ---- -------
Normal Generated 82s cert-manager Generated new private key
Normal OrderCreated 82s cert-manager Created Order resource "letsencrypt-prod-2626449824"
Normal OrderComplete 37s cert-manager Order "letsencrypt-prod-2626449824" completed successfully
Normal CertIssued 37s cert-manager Certificate issued successfully
```
We’ll now perform a test using curl to verify that HTTPS is working correctly:

```
$ curl echo1.example.com
```
You should see the following:

```
Output
<html>
<head><title>308 Permanent Redirect</title></head>
<body>
<center><h1>308 Permanent Redirect</h1></center>
<hr><center>nginx/1.15.9</center>
</body>
</html>
```
This indicates that HTTP requests are being redirected to use HTTPS.

Run curl on https://echo1.example.com:

```
$ curl https://echo1.example.com
```
You should now see the following output:


```
By Hanif Jetha
```
###### Was this helpful? Yes No % & '

Report an issue

```
Output
echo1
```
You can run the previous command with the verbose -v flag to dig deeper into the certificate

handshake and to verify the certificate information.

At this point, you’ve successfully configured HTTPS using a Let’s Encrypt certificate for your Nginx

Ingress.

## Conclusion

In this guide, you set up an Nginx Ingress to load balance and route external requests to backend

Services inside of your Kubernetes cluster. You also secured the Ingress by installing the cert-

manager certificate provisioner and setting up a Let’s Encrypt certificate for two host paths.

There are many alternatives to the Nginx Ingress Controller. To learn more, consult Ingress

controllers from the official Kubernetes documentation.

#### (^64

###### Related


```
TT UU TT OO RR I I AA L L
```
```
How To Set Up Flask
with MongoDB and
Docker
```
```
As a micro web framework
built on Python, Flask
provides an extensible way
for developers to grow...
```
```
TT UU TT OO RR I I AA L L
```
```
How to Protect Private
Kubernetes Services
Behind a GitHub Login
with oauth2_proxy
```
```
Kubernetes ingresses
make it easy to expose...
```
```
TT UU TT OO RR I I AA L L
```
```
How To Build and
Deploy a Node.js
Application To
DigitalOcean
Kubernetes Using
Semaphore Continuous
Integration and
Delivery
```
In this tutorial, you’ll build...

```
TT UU TT OO RR I I AA L L
```
```
How To Build and
Deploy Packages for
Your FreeBSD Servers
Using Buildbot and
Poudriere
```
```
Poudriere is the standard
tool on FreeBSD to build,...
```
### Still looking for an answer?

##### ) Ask a question * Search for more help


### 66 44 CC oo mm mm ee nn t t s s

Comment

```
Notify me of replies
to my comment
```
Leave a comment...

```
Logged in as:
matthewpoestreich
```
(^2) I like the article very much and alternatively you can expose the ingress controller through NodePort if
you want to save money on the load balancer/domain and access the application as below:
curl --resolve echo1.example.com:$IC_HTTPS_PORT:$IC_IP https://echo1.example.com:$IC_HTTPS_PORT/ --ins
Reply Report
samirshamsi100December 19, 2018

-

(^1) Awesome tutorial! Thanks for putting this together. For the installation of cert-manager via helm, I was
able to create with --set createCustomResources=true from the start and avoid creating with the
flag set to false and then updating it to true.
helm install --name cert-manager --namespace kube-system stable/cert-manager --set createCustomResourc
Reply Report
scrawfordDecember 21, 2018

-

(^0) This is the correct way to do it. See my comment below.
Reply Report
markhorrocksDecember 29, 2018

-

```
hjet MM O O D D January 3, 2019
```

```
0
```
```
Thank you! I’m glad you found the tutorial helpful. That --set createCustomResources
workaround was due to this bug which has since been resolved by upgrading Helm to v2.12.1. I’ve
updated the tutorial accordingly.
Reply•Report
```
(^1) Execllent tutorial!
FYI -
On my Windows 10 machine, in order to get the cert-manager to recognize the ClusterIssuer, I needed
to run this to setup the CRDs
kubectl apply -f 00-crds.yaml
00-crds.yaml
Reply Report
chadicusDecember 26, 2018

-

(^2) I had an issue while creating the ClusterIssuer, whether for staging or production.
kubectl create -f staging_issuer.yaml
error: unable to recognize “staging_issuer.yaml”: no matches for kind “ClusterIssuer” in version
“certmanager.k8s.io/v1alpha1”
I fixed this by running helm del --purge cert-manager
and then
helm install --name cert-manager --namespace kube-system stable/cert-manager --set createCustomResourc
Reply Report
markhorrocksDecember 29, 2018

-

(^0) Thanks for catching this! That --set createCustomResources workaround was due to this bug
which has since been resolved by upgrading Helm to v2.12.1. I’ve updated the tutorial accordingly.
Reply Report
hjetMM O O D D January 3, 2019

-

0

```
DaveMcCJanuary 4, 2019
```

Great tutorial - thanks very much.

Reply•Report

(^1) Thanks for this article. I can now actually use my DO K8s cluster with https setup.
Reply Report
sonamhavaJanuary 7, 2019

-

(^0) When I look at my DigitalOcean Loadbalancer I see that it has “a issue” where it shows that one of my
two droplets as being in Status of “Down”.
I increased the resource by adding another node and it shows that being down too. Is that a concern?
Reply Report
sonamhavaJanuary 8, 2019

-

(^0) Thanks for your comment!
This is expected, see the note in Step 2. Since the default Nginx Ingress Deployment only runs 1
Ingress Pod, the K8S node running the Pod will appear as “healthy” with the other nodes appearing
“down” so that Ingress traffic does not get routed to them. See the attached links in the
aforementioned note for more details.
Hope this is helpful!
HH o o ww t t o o S S e e t t UU p p a a n n NN g g i i n n x x I I n n g g r r e e s s s s ww i i t t h h CC e e r r t t - - MM a a n n a a g g e e r r......
by Hanif Jetha
In this tutorial, learn how to set up and secure an Nginx
Ingress Controller with Cert-Manager on DigitalOcean
Kubernetes.
Reply Report
hjetMM O O D D January 8, 2019

-

(^1) Thanks for the reply DO.
Reply Report
sonamhavaJanuary 8, 2019

-

(^0) Was you able to fix this? I’m also having this problem currently.
Update: Okay simple comment out the externalTrafficPolicy: Local on the lb yaml and
user72ac8c36d27f69339ca May 12, 2019


```
both are available.
Reply•Report
```
(^0) Thanks for the tutorial.
I tried to installed cert-manager using static manifest (without helm)
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.6/deploy/manifests/
However, I can’t get certificate issued
Events:
Type Reason Age From Message
---- ------ ---- ---- -------
Normal Generated 32m cert-manager Generated new private key
Normal OrderCreated 32m cert-manager Created Order resource "letsencrypt-staging-714061730"
Do you know what might be the problem here?
Thanks
Reply Report
arifsetiawanJanuary 16, 2019

-

(^0) ok. seems like I still use example.com in the hostname.
Reply Report
arifsetiawanJanuary 17, 2019

-

(^0) What is the proper way to setup to forward real ip from http requests?
Reply Report
SethiJanuary 19, 2019

-

(^2) Great tutorial! Demystified a lot for me, thanks for posting!
Reply Report
xennialJanuary 20, 2019

-

(^0) According to the docs cross-namespace ingress isn’t possible, yet in your example the 2 echo servers
are in the default namespace but the ingress-nginx namespace holds the info for the 2 ingress paths.
StubbsJanuary 28, 2019


I’ve followed your example and noticed you are goin across namespaces too, am I right or have I missed

something?

Reply•Report

(^0) That’s correct! Cross-namespace ingress isn’t possible per https://github.com/kubernetes
/kubernetes/issues/17088.
The echo services live in the default namespace, and the Ingress Controller lives in the
ingress-nginx namespace. The Ingress Resource (also commonly just called Ingress) which
defines the routing rules does not have a namespace specified and so lives in the default
namespace along with the echo services. So the Ingress/Ingress Resource and services are
actually in the same namespace.
Hopefully this helps, and thanks for reading!
Reply Report
hjetMM O O D D February 6, 2019

-

(^0) After creating the ClusterIssuer and updating the Ingress, inspecting the ingress shows that it
does not create the certificate. Why would it not create the certificate?
kubectl describe ingress
Events:
Type Reason Age From Message
—- —— —- —- ——-
Normal CREATE 28m nginx-ingress-controller Ingress default/echo-ingress
Normal UPDATE 27m nginx-ingress-controller Ingress default/echo-ingress
kubectl describe certificate
Empty
Reply Report
andrew804January 28, 2019

-

(^0) This could be a lot of things! Did you change the hosts from example.com to your own domain,
and point the A records to your DO load balancer accordingly?
hjetMM O O D D February 6, 2019


```
Reply•Report
```
(^0) all set
Events:
Type Reason Age From Message
Normal Generated 40m cert-manager Generated new private key
Normal OrderCreated 40m cert-manager Created Order resource “letsencrypt-staging-4138224086”
Normal OrderComplete 40m cert-manager Order “letsencrypt-staging-4138224086” completed
successfully
Normal CertIssued 40m cert-manager Certificate issued successfully
but at when opening certificate from browser it is not opening on https://
Reply Report
harshmanvarJanuary 29, 2019

-

(^0) Just leaving this as a note.
Had to do a couple of things to get this working with 0.6.0 of the cert-manager helm chart (may or may
not be related to the version number). Both were documented here.

1. Add the custom CRDs

```
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.6/deploy/manifests/
```
1. Disable cert-manager validation on the kube-system namespace since it’s not a new
    namespace.

```
kubectl label namespace kube-system certmanager.k8s.io/disable-validation="true"
```
```
Reply Report
```
```
scnewmanJanuary 31, 2019
```
-

(^0) Awesome, thanks for surfacing this! Will update the guide ASAP with these changes.
Reply Report
hjetMM O O D D February 6, 2019

-

```
hjet MM O O D D March 6, 2019
```

```
0
```
```
Tutorial has been updated to work with v0.6.0 of cert-manager. Thanks again for flagging
the changes!
Reply•Report
```
(^0) Hey @hjet thanks for the tutorial!
Please challenge me on this but I believe you have mistaken the relationship between the
privateKeySecretRef in the cluster issuer and the secretName in the ingress resources
(spec.tls.hosts.secretName). They are NOT the same secret. Per the documentation, the
privateKeySecretRef stores the “ACME account private key” where the the ingress’s secretName
stores the TLS private key and certificate.
In your example you say “Finally, we add a tls block to specify the hosts for which we want to acquire
certificates, and specify the private key we created earlier.” which I believe is incorrect and misleading.
Please let me know your thoughts on this.
Reply Report
briananstettFebruary 3, 2019

-

(^0) You are correct! Have updated the language in the tutorial accordingly. Thanks for the catch.
Reply Report
hjetMM O O D D February 6, 2019

-

(^0) Hello. how can I use the nginx ingress controller without the external load balancer to be automatically
created? I have a single node cluster. dont need External load balancer.
Thank you.
Reply Report
brpazFebruary 6, 2019

-

(^0) Thanks for your question! You can also use a ServiceType: NodePort. To learn more, consult
the official Ingress Controller docs. To learn more about NodePort services, consult the
Kubernetes docs.
Reply Report
hjetMM O O D D February 7, 2019

-

(^1) Thanks for this awesome manual.
Trying to execute
slotixFebruary 15, 2019


```
$ helm install --name cert-manager --namespace kube-system stable/cert-manager --version v0.5.2
```
I’ve got the following error

```
Error: namespaces "kube-system" is forbidden: User "system:serviceaccount:kube-system:default" cannot get
```
You should add the following commands before installing cert-manager into a cluster

```
$ kubectl create serviceaccount --namespace kube-system tiller
$ kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube
$ kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAcc
```
Reply•Report

(^2) Hey slotix, thanks for the feedback!!
Those steps are actually covered in How To Install Software on Kubernetes Clusters with the Helm
Package Manager which we link to in the Tutorial’s Prerequisites section.
Glad you found the tutorial useful!
HH o o ww TT o o I I n n s s t t a a l l l l S S o o f f t t ww a a r r e e o o n n K K u u b b e e r r n n e e t t e e s s CC l l u u s s t t e e r r s s ww i i......
by Brian Boucheron
Helm is a package manager for Kubernetes that allows
developers and operators to more easily configure and
deploy applications on Kubernetes clusters. In this
Reply Report
hjetMM O O D D February 15, 2019

-

(^0) Thank you for pointing it out!
I’ve stuck with one more problem.
Please, can you provide information about how to set up Nginx Ingress to support wss:// web
sockets?
Unfortunately, I didn’t find clear info about that.
Reply Report
slotixFebruary 15, 2019

-


```
This work is licensed under a Creative
Commons Attribution-NonCommercial-
ShareAlike 4.0 International License.
```
```
Load More Comments
```
```
leo1234May 28, 2019
```
```
0
```
```
Hey, were you able to get Web Sockets to work?
Reply•Report
```
(^2) Dude, you are the fucking man.
Reply Report
alexanderkleinhans234March 10, 2019

-


```
BECOME A CONTRIBUTOR
```
###### You get paid; we donate to tech

###### nonprofits.

```
CONNECT WITH OTHER DEVELOPERS
```
###### Find a DigitalOcean Meetup

###### near you.

```
GET OUR BIWEEKLY NEWSLETTER
```
###### Sign up for Infrastructure as a

###### Newsletter.


```
Featured on CommunityIntro to Kubernetes Learn Python 3 Machine Learning in Python
Getting started with Go Migrate Node.js to Kubernetes
```
DigitalOcean ProductsDroplets Managed Databases Managed Kubernetes Spaces Object Storage
Marketplace

### Welcome to the developer cloud

###### DigitalOcean makes it simple to launch in the

###### cloud and scale up as you grow – whether you’re

###### running one virtual machine or ten thousand.

Learn More


© 2019 DigitalOcean, LLC. All rights reserved.

```
Company
```
```
About
Leadership
Blog
Careers
Partners
Referral Program
Press
Legal & Security
```
```
Products
```
```
Products Overview
Pricing
Droplets
Kubernetes
Managed Databases
Spaces
Marketplace
Load Balancers
Block Storage
Tools & Integrations
API
Documentation
Release Notes
```
```
Community
```
```
Tutorials
Q&A
Tools and Integrations
Tags
Product Ideas
Meetups
Write for DOnations
Droplets for Demos
Hatch Startup Program
Shop Swag
Research Program
Currents Research
Open Source
```
```
Contact
```
```
Support
Sales
Report Abuse
System Status
```

