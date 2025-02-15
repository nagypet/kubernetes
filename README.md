# Kubernetes

<!-- TOC -->
* [Kubernetes](#kubernetes)
  * [Let's encrypt integration](#lets-encrypt-integration)
  * [Microk8s dashboard](#microk8s-dashboard)
    * [Accessing with port forward](#accessing-with-port-forward)
    * [Accessing with Ingress](#accessing-with-ingress)
    * [Token](#token)
  * [Monitoring](#monitoring)
  * [Log analysis with Kibana](#log-analysis-with-kibana)
<!-- TOC -->

## Let's encrypt integration

```
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.17.0 --set crds.enabled=true
```

Create a ClusterIssuer for receiving a certificate from Let's Encrypt
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: peter.nagy@perit.hu
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

## Microk8s dashboard

Enabling the dashboard
```
microk8s enable dashboard
```
### Accessing with port forward
Port forward
```
microk8s kubectl port-forward -n kube-system service/kubernetes-dashboard 10443:443
```
https://localhost:10443

### Accessing with Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubernetes-dashboard-ingress
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
    - host: dashboard.k8s-test.perit.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kubernetes-dashboard
                port:
                  number: 443
  tls:
    - hosts:
        - dashboard.k8s-test.perit.local
      secretName: kubernetes-dashboard-certs
```
https://dashboard.k8s-test.perit.local

### Token
```
microk8s kubectl -n kube-system get secret
microk8s kubectl -n kube-system describe secret microk8s-dashboard-token
```
if invalid:
```
kubectl -n kube-system create token kubernetes-dashboard
```

## Monitoring

Please install the [prometheus-operator](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/user-guides/getting-started.md) first.

```
helm install kube-prom-stack prometheus-community/kube-prometheus-stack -n observability
```

Then you can use these helm charts to install prometheus/grafana in the given namespace.

- `helm install prometheus prometheus -n wstemplate`
- `helm install grafana grafana -n wstemplate`

URLs:

- http://prometheus.wstemplate.k8s-test.perit.local
- http://grafana.wstemplate.k8s-test.perit.local

Please make sure, the service has the label `monitored: prometheus` and the port is named as http.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  namespace: wstemplate
  labels:
    monitored: prometheus
spec:
  type: ClusterIP
  selector:
    component: auth-service
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: http
```

## Log analysis with Kibana

```
microk8s enable fluentd
```

Port forward
```
microk8s.kubectl port-forward -n kube-system service/kibana-logging 8188:5601
```
http://localhost:8188

Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana-ingress
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: kibana.k8s-test.perit.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kibana-logging
                port:
                  number: 5601
```
