---
layout: default
title: k8s 시리즈 04 - cert-manager 설치
nav_order: 8
---

# Cert-Manager 설치

# 설치

- 현 시스템 k8s 버전에 따라 다름. 1.16 이상이면 상단 스크립트 사용

```yaml
# Kubernetes 1.16+
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.0.3/cert-manager.yaml

# Kubernetes <1.16
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.2.0/cert-manager-legacy.yaml
```

# 설치확인

```bash
$ kubectl get pods --namespace cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-64cdb5b965-bvtpr              1/1     Running   0          37h
cert-manager-cainjector-597859f766-dsxm7   1/1     Running   0          37h
cert-manager-webhook-7d749b578f-f6bxt      1/1     Running   0          37h
```

# 인증서 이슈 매니저 생성

- issuer.yaml

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # The ACME server URL
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: address@yourdns.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging
    # Enable the HTTP-01 challenge provider
    solvers:
    # An empty 'selector' means that this solver matches all domains
    - selector: {}
      http01:
        ingress:
          class: nginx

---
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: address@yourdns.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx
```

# 이슈 매니저 적용

```bash
$ kubectl apply -f issuer.yaml
```

# 인그레스 배포 파일 편집.

- http-go.yaml 수정

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: http-go
spec:
  replicas: 5
  selector:
    matchLabels:
      app: http-go
  template:
    metadata:
      labels:
        app: http-go
    spec:
      containers:
      - image: ryuhon/http-go
        name: http-go
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: http-go
spec:
  ports:
    - protocol: TCP
      port: 80
  selector:
    app: http-go

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: http-go-ingress
  annotations:
    ingress.kubernetes.io/ssl-redirect: "true"
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - http-go.yourdns.com
    secretName: http-go-devbox-kr-cert
  rules:
  - host: http-go.yourdns.com
    http:
      paths:
      - backend:
          serviceName: http-go
          servicePort: 80
```

# Ingress 확인

```yaml
$ kubectl describe ingress http-go-ingress
```

# 인증서 확인

```yaml
$ kubectl get certificate
```