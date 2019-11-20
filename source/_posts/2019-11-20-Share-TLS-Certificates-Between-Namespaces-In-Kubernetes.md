---
title: Share TLS Certificates Between Namespaces In Kubernetes
date: 2019-11-20 15:32:42
tags: k8s traefik tls secret
---

## Problem Statement

There are cases you need to share wildcard SSL certificats between different namespaces through k8s ingress controller(traefik 2.x). From security perspective, you should not do this. But, but, but..., you still need it when it comes to reality.

{% asset_img ingress-multi-ssl.svg This is an image %}

<!--more-->

## Scenario

You deploy your kubernetes cluster in your private cloud and having lots of services deployed on top of it. Your service, however will only expose `domain-a.com`, `domain-b.com` to the world. You also have wildcard certificate files for both domain.

If you have more than 3 namespaces in your cluster, then you have to create those tls secrets for 3 different namespaces. When you renew your certificates, you need to modify them one by one too.

When we use Traefik 2.x for our ingress controller, then there comes to question, `Can I set namespace for TLS secretName in IngressRoute?`

## Manage TLS certification in Cluster

### User TLS certification in different namespaces

For IngressRoute - I would like to be able to set namespace for TLS secretName.
Currently, I am putting my IngressRoutes into the same namespace as the service they refer to.
And it is currently not possible to point the TLS secretName at another namespace.
So if I have more than one IngressRoute using the same certificate, then I need to create a secret for every namespace.

> Secrets are namespaced for isolation and security reasons. We should not support cross-namespace secret mounting, as it would allow users to fish for secrets.

I have my services in various namespaces other than default and want to have my IngressRoutes in the same namespace.
However, I am using the same wildcard certificate for multiple IngressRoutes, while I have to put the secret in a single namespace.

How do I go about managing this, if I don't want to create a secret per namespace?

### Solution

You can specify TLS certificates to Traefik (TLS secrets mounted in Traefik pod) as defined in https://docs.traefik.io/v2.0/https/tls/#user-defined .

Then, for your IngressRoute, you specify tls: {} and Traefik will match based on the router rule.

A quick example had been added into the community forum topic you started at https://community.containo.us/t/can-i-set-namespace-for-tls-secretname-in-ingressroute/2619.

### Create secrets for Traefik

For installation of Traefik, please refer to [official site](https://docs.traefik.io/getting-started/install-traefik/). Suppose you install traefik in `traefik-system` system.

Prepare your certificates, naming tls-a.key, tls-a.crt, tls-b.key, tls-b.crt. Get base64 of them for yaml files.

```bash
$ cat tls-a.key | base64
$ cat tls-a.crt | base64
$ cat tls-b.key | base64
$ cat tls-b.crt | base64
```

Your `tls.yaml` file.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: wildcard-domain-a-com-tls-cert
  namespace: traefik-system
type: kubernetes.io/tls
data:
  tls.crt: (base64 of your tls-a.crt)
  tls.key: (base64 of your tls-a.key)

---

apiVersion: v1
kind: Secret
metadata:
  name: wildcard-domain-b-com-tls-cert
  namespace: traefik-system
type: kubernetes.io/tls
data:
  tls.crt: (base64 of your tls-b.crt)
  tls.key: (base64 of your tls-b.key)

```

Apply it to your cluster.

```
$ kubectl apply -f tls.yaml
```

### Modify your traefik configuration

In your traefik yaml file, you have to add/modify as following.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: traefik-dynamic-config
data:
  dynamic.yaml: |
    # Dynamic configuration
    tls:
      certificates:
        - certFile: /wildcard-domain-a-com/tls.crt
          keyFile: /wildcard-domain-a-com/tls.key
        - certFile: /wildcard-domain-b-com/tls.crt
          keyFile: /wildcard-domain-b-com/tls.key
    stores:
      default: {}
    
---

apiVersion: apps/v1beta2
kind: DaemonSet
metadata:
  name: traefik
spec:
  selector:
    matchLabels:
      name: traefik
  template:
    metadata:
      labels:
        name: traefik
    spec:
      containers:
      - name: traefik
        image: traefik:v2.0.4
        imagePullPolicy: IfNotPresent
        args:
          - --entryPoints.web.address=:80
          - --entryPoints.websecure.address=:443
          - --api.insecure=true
          - --api.dashboard=true
          - --providers.kubernetescrd
          - --providers.file.filename=/config/dynamic.yaml
        volumeMounts:
          - name: certs-a
            mountPath: "/wildcard-domain-a-com"
            readOnly: true
          - name: certs-b
            mountPath: "/wildcard-domain-a-com"
            readOnly: true
          - name: config
            mountPath: "/config"
            readOnly: true     
      volumes:
        - name: certs-a
          secret:
            secretName: wildcard-domain-a-com-tls-cert
        - name: certs-b
          secret:
            secretName: wildcard-domain-b-com-tls-cert
        - name: config
          configMap:
            name: traefik-dynamic-config    
      dnsPolicy: ClusterFirstWithHostNet
      hostNetwork: true
      serviceAccountName: traefik-ingress-controller
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: RollingUpdate
```

We use `--providers.file.filename` for dynamic configuration for our traefik, and having volumeMounts[].mountPath same as certFile path, the traefik can retrieve those files when it needs it. Apply it to your traefik.

### Configure your IngressRoute

Here you have 2 services in two different namespaces. The service end points are:

* https://service1.domain-a.com
* https://service2.domain-b.com

then modify your `ingressroute.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service1
spec:
  ports:
    - port: 443
      targetPort: 8080
  selector:
    name: traefik
---
apiVersion: v1
kind: Service
metadata:
  name: service2
spec:
  ports:
    - port: 443
      targetPort: 8080
  selector:
    name: traefik
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: service1-ingress
spec:
  entryPoints:
    - websecure
  routes:
  - match: Host(`service1.domain-a.com`)
    kind: Rule
    priority: 1
    services:
    - name: service1
      port: 443
  tls: {}
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: service2-ingress
spec:
  entryPoints:
    - websecure
  routes:
  - match: Host(`service2.domain-a.com`)
    kind: Rule
    priority: 1
    services:
    - name: service2
      port: 443
  tls: {}
```

The most import part is `tls` as `{}`, no more `tls.secretName`.

Happy hacking...