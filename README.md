# Sonarqube Kubernetes Installation via GitOps
This repository can be used in conjunction with Argo CD to deploy the Sonarqube Helm Chart via GitOps.

## Prerequisites
1. You have forked this repo into your own account.
1. You have a Konvoy Kubernetes cluster up and running.
1. You have a domain name that you can create/update a CNAME to point at your traefik load balancer address.

## Create the sonarqube namespace
Run the following to create the sonarqube namespace:
```bash
kubectl create ns sonarqube
```

## Create or Update a CNAME
First, create or update your DNS to point at the Traefik Load Balancer. We will be using Traefik as the Ingress to Sonarqube.
You will need to have the DNS CNAME created/updated prior to trying to create the certificate.

## Setup the Let's Encrypt Certificate
You need to setup a ClusterIssuer to create certificates. This is a cert-manager object that tells the system to use Let's Encrypt.

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: <YOUREMAILADDRESS>
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: letsencrypt-private-key
    solvers:
    - http01:
        ingress:
          class: traefik
```
Save this to a file called `letsencrypt-clusterissuer.yaml`. 

NOTE: Make sure to update the email address with your email address.

Then run:
```bash
kubectl apply -f letsencrypt-clusterissuer.yaml
```

Next, you need to create the certificate.

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: sonar-cert
  namespace: sonarqube
spec:
  commonName: <YOUR_SONAR_DOMAIN>
  secretName: sonar-cert
  issuerRef:
    name: letsencrypt
    kind: ClusterIssuer
```
Save this to a file called `sonar-certificate.yaml`.

NOTE: Make sure you update the commonName with your domain name.

Then run:
```bash
kubectl apply -f sonar-certificate.yaml
```

This will create the certificate request.  You can check on the status by running:

```bash
kubectl get certificate -n sonarqube
```
The output will look like the following when it is ready:
```bash
NAME         READY   SECRET       AGE
sonar-cert   True    sonar-cert   32s
```
If it isn't becoming ready, you can try describing the certificate. 
The biggest thing to double check is to make sure the CNAME is pointing at the Traefik Load Balancer.

## Setup Dispatch Service Account
You will need a Github Token for the connectivity to the GitOps repository.  The Dispatch documentation covers creating a token and which permissions it needs: https://docs.d2iq.com/ksphere/dispatch/1.2/tutorials/ci_tutorials/credentials/. Click on the GitHub black text to expand the instructions under step 1 of "Setting up Github credentials".

Run the following to setup your environment variables:
```bash
export GITHUB_USERNAME=<YOUR GITHUB USERNAME>
export GITHUB_TOKEN=<YOUR GITHUB TOKEN>
```

The following is safe to run multiple times as it checks for existence before creating the objects.
```bash
kubectl -n dispatch get serviceaccount dispatch-sa
if [ $? -eq 1 ]; then
    dispatch serviceaccount create dispatch-sa --namespace dispatch
fi
kubectl -n dispatch get secret dispatch-sa-basic-auth
if [ $? -eq 1 ]; then
    dispatch login github --user ${GITHUB_USERNAME} --token ${GITHUB_TOKEN} --service-account dispatch-sa --namespace dispatch --insecure-webhook-skip-tls-verify
fi
docker login
kubectl -n dispatch get secret dispatch-sa-docker-auth
if [ $? -eq 1 ]; then
    dispatch login docker --service-account dispatch-sa --namespace dispatch
fi
```

## Installing Sonarqube
First, make sure you have updated the `values.yaml` file in this repository:
```yaml
sonarqube:
  image:
    pullPolicy: IfNotPresent
  ingress:
    enabled: true
    hosts:
    - name: YOUR_DOMAIN
    tls:
    - hosts:
        - YOUR_DOMAIN
      secretName: sonar-cert
  persistence:
    enabled: true
```
NOTE: Make sure to update the YOUR_DOMAIN twice in this file. It will have the same value in both spots.

Next, we are going to use Argo to manage this installation via Dispatch.
```bash
dispatch gitops app create sonarqube --repository=https://github.com/${GITHUB_USERNAME}/sonar-gitops/ --service-account dispatch-sa --namespace dispatch --dest-namespace sonarqube
```
NOTE: Make sure you have a GITHUB_USERNAME environment variable set.  The instructions were above.

You can now go into the Argo GUI to watch the objects get created.  When its complete, you should be able to browse to your URL and view the Sonarqube GUI.
