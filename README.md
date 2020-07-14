# Sonarqube Kubernetes Installation via GitOps
This repository can be used in conjunction with Argo CD to deploy the Sonarqube Helm Chart via GitOps.

## Prerequisites
1. You have a Konvoy Kubernetes cluster up and running.
1. You have a domain name that you can create/update a CNAME to point at your traefik load balancer address.

## Setup the Let's Encrypt Certificate
First, you need to setup a ClusterIssuer. This is a cert-manager object that tells the system to use Let's Encrypt.

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
