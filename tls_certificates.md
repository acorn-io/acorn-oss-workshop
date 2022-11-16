## Adding TLS certificate

### 1st option: with Let's Encrypt

I’ve created a clusterissuer:
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: le
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: luc.juggery@gmail.com
    privateKeySecretRef:
      name: letsencrypt-prod-issuer
    solvers:
    - http01:
        ingress:
          class: traefik
Then 2 certificates for both vote.k8sapp.xyz and result.k8sapps.xyz domains:
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: k8sapp-certs
spec:
  dnsNames:
  - vote.k8sapps.xyz
  issuerRef:
    kind: ClusterIssuer
    name: le
  secretName: k8sapps-tls-secret
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: k8sapp-certs
spec:
  dnsNames:
  - result.k8sapps.xyz
  issuerRef:
    kind: ClusterIssuer
    name: le
  secretName: k8sapps-result-tls-secret
Then run a demo app with:
acorn run -p vote.k8sapps.xyz:voteui -p result.k8sapps.xyz:resultui docker.io/lucj/voting:v1.0.26
(voteui and resultui are the 2 “frontend” containers of the app)
I can now access the app with https://vote.k8sapps.xyz and https://result.k8sapps.xyz (modifié) 

### 2nt option: directly using Acorn