
# Victoria Metrics Operator


Operator serves to make running Victoria Metrics applications on top of Kubernetes as easy as possible, while preserving Kubernetes-native configuration options.

## installing operator

Obtain release from releases page:

https://github.com/VictoriaMetrics/operator/releases

```yaml
wget https://github.com/VictoriaMetrics/operator/releases/download/v0.1.0/bundle_crd.zip
zip -e bundle_crd.zip 
```

First of all, you  have to create crd
```yaml
kubectl apply -f install/crds/vm/
#if you dont have prometheus crds, add it with command:
kubectl apply -f install/crds/prom/
```
*NOTE if your kubernetes version less then 1.16, it`s no possible to use server side validation for crds and you have do disable it, when applying crds:
```yaml
kubectl apply -f install/crds/vm/ --validate=false

``` 
 
Then you need rbac for operator, relevant configuration for release can be found at install/operator/rbac.yaml
```
#change namespace for your installation:
sed -i -e 's/namespace: default/namespace: YOUR_NS' install/operator/rbac.yaml
kubectl apply -f install/operator/rbac.yaml
```

change configuration for operator at `install/operator/operator.yaml`, possible settings: [operator-settings][vars.MD] 
and apply it:
```
kubectl -n YOUR_NS apply -f install/operator/operator.yaml
```

check status of operator

```
kubectl -n YOUR_NS get pods
```


## Create related resources

The Victoria Metrics Operator introduces additional resources in Kubernetes to declare the desired state of a Victoria Metrics applications and Alertmanager cluster as well as the Prometheus resources configuration. The resources it introduces are:

* `VmAgent`
* `VmSingle`
* `VmAlert`
* `PrometheusRule`
* `ServiceMonitor`
* `VmAlertmanager`


## VmSingle

 It represent database for storing metrics:
 
 ```yaml
cat <<EOF | kubectl apply -f -
apiVersion: victoriametrics.com/v1beta1
kind: VmSingle
metadata:
  name: example-vmsingle-persisted
spec:
  retentionPeriod: "1"
  removePvcAfterDelete: true
  storage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi

EOF
```

configuration can be extended with extraFlags and extraEnv, check flag documentation at https://github.com/VictoriaMetrics/VictoriaMetrics/blob/master/docs/Single-server-VictoriaMetrics.md

for instance:
```yaml
cat <<EOF | kubectl apply -f -
apiVersion: victoriametrics.com/v1beta1
kind: VmSingle
metadata:
  name: example-vmsingle-persisted
spec:
  retentionPeriod: "1"
  extraArgs:
    - -dedup.minScrapeInterval=60s
EOF
```

flag can be replaced with envVar, it`s useful for retriving value from secret:
```yaml
cat <<EOF | kubectl apply -f  -
apiVersion: victoriametrics.com/v1beta1
kind: VmSingle
metadata:
  name: example-vmsingle-persisted
spec:
  retentionPeriod: "1"
  extraEnvs:
    - name: DEDUP_MINSCRAPEINTERVAL
      valueFrom: 
        secretKeyRef:
           name: vm-secret
           key: dedup
EOF

```


## VmAgent

 `VmAgent` needs access to kubernetes api and you must create rbac for it first, it can be found at `install/examples/vmagent_rbac.yaml`
 ```yaml
#change namespace
sed -i -e 's/namespace: default/namespace: YOUR_NS/' install/examples/vmagent_rbac.yaml 
kubectl apply -f install/examples/vmagent_rbac.yaml
```

 Modify `VmAgent` config parameters at `install/examples/vmagent.yaml` and apply it
 ```yaml
kubectl -n YOUR_NS apply -install/examples/vmagent.yaml
```
example:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: victoriametrics.com/v1beta1
kind: VmAgent
metadata:
  name: example-vmagent
spec:
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  replicas: 1
  serviceAccountName: vmagent
  remoteWrite:
    - url: "http://vmagent-example-vmsingle.default.svc:8429/api/v1/write"

EOF
```

## VmAlert


example

```yaml
cat << EOF | kubectl apply -f -
apiVersion: victoriametrics.com/v1beta1
kind: VmAlert
metadata:
  name: example-vmalert
spec:
  replicas: 1
  datasource:
    url: "http://vmagent-example-vmsingle.default.svc:8429"
  notifierURL: "http://alertmanager-operated:9093"
  evaluationInterval: "30s"
  ruleSelector: {}
EOF
```

## VmAlertmanager

create secret with config
```yaml
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-example-alertmanager
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
    route:
      group_by: ['job']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'webhook'
    receivers:
    - name: 'webhook'
      webhook_configs:
      - url: 'http://alertmanagerwh:30500/'
EOF
```

and `VmAlertmanager`

```yaml
cat << EOF | kubectl apply -f -
apiVersion: victoriametrics.com/v1beta1
kind: Alertmanager
metadata:
  name: example-alertmanager
spec:
  # Add fields here
  replicas: 1
EOF
```

## Deploy example-app


First, deploy three instances of a simple example application, which listens and exposes metrics on port `8080`.

[embedmd]:# (../../example/user-guides/getting-started/example-app-deployment.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: example-app
  template:
    metadata:
      labels:
        app: example-app
    spec:
      containers:
      - name: example-app
        image: fabxc/instrumented_app
        ports:
        - name: web
          containerPort: 8080
```

The ServiceMonitor has a label selector to select Services and their underlying Endpoint objects. The Service object for the example application selects the Pods by the `app` label having the `example-app` value. The Service object also specifies the port on which the metrics are exposed.

[embedmd]:# (../../example/user-guides/getting-started/example-app-service.yaml)
```yaml
kind: Service
apiVersion: v1
metadata:
  name: example-app
  labels:
    app: example-app
spec:
  selector:
    app: example-app
  ports:
  - name: web
    port: 8080
```

This Service object is discovered by a ServiceMonitor, which selects in the same way. The `app` label must have the value `example-app`.

[embedmd]:# (../../example/user-guides/getting-started/example-app-service-monitor.yaml)
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
  labels:
    team: frontend
spec:
  selector:
    matchLabels:
      app: example-app
  endpoints:
  - port: web
```


## Expose the VmSingle API


> WARNING: Enabling the external access to VmSingle highly recommend to protect with https and authentication, it allows to  to delete data.

example for kubernetes nginx ingress [doc](https://kubernetes.github.io/ingress-nginx/examples/auth/basic/)
```yaml
#generate creds
htpasswd -c auth foo

#create basic auth secret
cat <<EOF | kubectl apply -f -
apiVersion: v1
stringData:
  auth: foo:$apr1$wQ0fkxnl$KKOA/OqIZy48CuCB/A1Rc.
kind: Secret
metadata:
  name: vm-auth
type: Opaque
EOF

#create ingress rule
cat << EOF | kubectl apply -f -
apiVersion: networking/v1beta1                                                                                                                                                                                                                                                                                               
kind: Ingress                                                                                                                                                                                                                                                                                                                
metadata:                                                                                                                                                                                                                                                                                                                    
  annotations:                                                                                                                                                                                                                                                                                                               
    nginx.ingress.kubernetes.io/auth-realm: Authentication Required                                                                                                                                                                                                                                                          
    nginx.ingress.kubernetes.io/auth-secret: vm-auth                                                                                                                                                                                                                                                                         
    nginx.ingress.kubernetes.io/auth-type: basic
  labels:
    app: victoria-metrics-single
  name: victoria-metrics-single
spec:
  rules:
  - host: vmsingle.example.com
    http:
      paths:
      - backend:
          serviceName: vmsingle-example-vmsingle-persisted
          servicePort: 8428
        path: /
  tls:
  - hosts:
    - vmsingle.example.com
    secretName: ssl-name

EOF

#now you can access victoria metrics
curl -u foo:bar https://vmsingle.example/com/metrics
```
