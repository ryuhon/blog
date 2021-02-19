---
layout: default
title: 03 ingress 설치
parent: 쿠버네티스
nav_order: 3
---

# Ingress 설치

## 인그레스 설치파일 (베어메탈 기준 )

1. 파일 다운로드 
    - [https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml](https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml)
2. [rbac.authorization.k8s.io/v1](http://rbac.authorization.k8s.io/v1)beta1 을 [rbac.authorization.k8s.io/v1](http://rbac.authorization.k8s.io/v1) 으로 변경 

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      # wait up to five minutes for the drain of connections
      terminationGracePeriodSeconds: 300
      serviceAccountName: nginx-ingress-serviceaccount
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 101
            runAsUser: 101
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
            - name: https
              containerPort: 443
              protocol: TCP
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          lifecycle:
            preStop:
              exec:
                command:
                  - /wait-shutdown

---

apiVersion: v1
kind: LimitRange
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  limits:
  - min:
      memory: 90Mi
      cpu: 100m
    type: Container
```

## Ingress 설치

```bash
$ kubectl apply -f ingress-init.yaml
```

## Ingress 설치 확인

```bash
$ kubectl get namespaces
NAME              STATUS   AGE
cert-manager      Active   36h
default           Active   5d14h
ingress-nginx     Active   36h
kube-node-lease   Active   5d14h
kube-public       Active   5d14h
kube-system       Active   5d14h
$ kubectl get pod --namespace ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-54b86f8f7b-znbr4   1/1     Running   0          36h
```

## Ingress 서비스 파일 작성

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 31080
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      nodePort: 31443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```

## Ingress 서비스 적용

```bash
$ kubectl apply -f ingress-service.yaml
```

## 프록시 프로토콜 활성화

- ingress-nginx 구성 맵 의 이름 확인

```bash
$ kubectl get configmap --namespace ingress-nginx
NAME                              DATA   AGE
ingress-controller-leader-nginx   0      36h
nginx-configuration               1      36h
tcp-services                      0      36h
udp-services                      0      36h
```

configMap 수정 

- 파일이름을 nginx-configuration.yaml

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
data:
  use-proxy-protocol: "true"
```

- 적용

```yaml
$ kubectl apply -f nginx-configuration.yaml 
```

## HAProxy 구성

- 쿠버네티스 앞단에서 ingress 로 로드밸런싱을 담당할 서버
- 리눅스 서버에 HAProxy 설치되어서 트래픽을 Ingress 로 넘겨주는 역활을 함.

```bash
frontend kube-ssl
  bind *:443
  mode tcp
  option tcplog
  timeout client 1m
  default_backend ssl-backend

frontend kube-http
  bind *:80
  mode tcp
  option tcplog
  timeout client 1m
  default_backend http-backend

backend ssl-backend
  mode tcp
  option tcplog
  option log-health-checks
  option redispatch
  log global
  balance roundrobin
  timeout connect 10s
  timeout server 1m
  server node1 <NODE1_IP>:31443 check-send-proxy inter 10s send-proxy
  server node2 <NODE2_IP>:31443 check-send-proxy inter 10s send-proxy

backend http-backend
  mode tcp
  option tcplog
  option log-health-checks
  option redispatch
  log global
  balance roundrobin
  timeout connect 10s
  timeout server 1m
  server node1 <NODE1_IP>:31080 check-send-proxy inter 10s send-proxy
  server node2 <NODE2_IP>:31080 check-send-proxy inter 10s send-proxy
```

## 테스트앱 배포

- http-go 앱 pod , service , ingress yaml 작성

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
spec:
  rules:
  - host: http-go.도메인.컴
    http:
      paths:
      - backend:
          serviceName: http-go
          servicePort: 80
```

배포 

```yaml
$ kubectl apply -f http-go.yaml 
```