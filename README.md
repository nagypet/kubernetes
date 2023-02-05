# kubernetes

Please install the [prometheus-operator](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/user-guides/getting-started.md) first.

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
