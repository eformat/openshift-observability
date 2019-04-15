# openshift observability stack

Deploy standalone prometheus using openshift example templates

```
wget https://raw.githubusercontent.com/openshift/origin/master/examples/prometheus/prometheus-standalone.yaml
```

We adjust the template to allow the `prom` service account to scraping our application endpoints and use default images in our cluster

```
# add ClusterRole and ClusterRoleBinding
- apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: prometheus-scraper
  rules:
  - apiGroups: [""]
    resources:
    - services
    - endpoints
    - pods
    verbs: ["get", "list", "watch"]
  - apiGroups:
    - route.openshift.io
    resources:
    - routers/metrics
    verbs:
    - get
  - apiGroups:
    - image.openshift.io
    resources:
    - registry/metrics
    verbs:
    - get
- apiVersion: authorization.openshift.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: prometheus-scraper
  roleRef:
    name: prometheus-scraper
  subjects:
  - kind: ServiceAccount
    name: prom
    namespace: "${NAMESPACE}"

# adjust images to use
- description: The location of the proxy image
  name: IMAGE_PROXY
  value: openshift3/oauth-proxy:v3.11.82
- description: The location of the prometheus image
  name: IMAGE_PROMETHEUS
  value: openshift3/prometheus:v3.11.82
- description: The location of the alertmanager image
  name: IMAGE_ALERTMANAGER
  value: openshift3/prometheus-alertmanager:v3.11.82
- description: The location of alert-buffer image
  name: IMAGE_ALERT_BUFFER
  value: openshift3/prometheus-alert-buffer:v3.11.82
```

We will use a custom prometheus config file that allows us to scrape applications based on `Service` annotations.

Create prometheus

```
# Create project for our observability stack
oc new-project observability --display-name="Observability" --description="Observability"

# create secrets containing configuration
oc create secret generic prom --from-file=./prometheus.yml
oc create secret generic prom-alerts --from-file=./alertmanager.yml
oc create -f ./prometheus-htpasswd-secret.yml

# Create the prometheus instance
# this requires cluster admin so we can create a cluster role that can scrape our app configs
oc process -f prometheus-standalone.yaml -p NAMESPACE=observability | oc apply -f -
oc policy add-role-to-user view system:serviceaccount:$(oc project -q):prom

# If using multitenant plugin:
$ oc get clusternetwork default --template='{{.pluginName}}'
redhat/openshift-ovs-multitenant

# Then we need to make prometheus global so it can be seen by all projects
oc adm pod-network make-projects-global observability
```

Prometheus persistent data

```
-- prometheus persistent data

oc create -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: prometheus-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: netapp-nfs
EOF

oc volume statefulsets/prom --add --overwrite -t persistentVolumeClaim --claim-name=prometheus-data --name=prometheus-data --mount-path=/prometheus

oc create -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: alertmanager-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: netapp-nfs
EOF

oc volume statefulsets/prom --add --overwrite -t persistentVolumeClaim --claim-name=alertmanager-data --name=alertmanager-data --mount-path=/alertmanager
```

Application

```
# https://github.com/eformat/camel-springboot-rest-ose

# becuase we are not using fabric8 fragments to build and deploy (mvn fabric8:deploy), we can do these steps manually
oc edit svc camel-springboot-rest-ose-master

# recreate service as S2I does not get this corect
oc delete svc camel-springboot-rest-ose-master
oc expose dc camel-springboot-rest-ose-master --name=camel-springboot-rest-ose-master --port=8080 --generator=service/v1

# expose route and set it on our swagger endpoint
oc expose svc camel-springboot-rest-ose-master --port=8080
oc set env dc/camel-springboot-rest-ose-master SWAGGERUI_HOST=$(oc get route camel-springboot-rest-ose-master --template='{{ .spec.host }}')

# Annotate our SpringBoot service so it can be scraped
oc annotate svc camel-springboot-rest-ose-master --overwrite prometheus.io/path='/prometheus' prometheus.io/port='9779' prometheus.io/scrape='true'
```

Grafana deploy

The example template is here, we adjust for images and oauth config in our cluster

```
wget https://raw.githubusercontent.com/openshift/origin/master/examples/grafana/grafana.yaml -O grafana.yaml
```

Deploy Grafana with persistent storage for data and graphs

```
oc create secret generic grafana-datasources --from-file="prometheus.yaml=./grafana-prometheus-secret.json"

oc create -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: grafana-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: netapp-nfs
EOF

# create grafana
oc new-app -f grafana.yaml -p NAMESPACE=$(oc project -q)

# ouath delegation
oc adm policy add-cluster-role-to-user system:auth-delegator -z grafana -n observability

# set datasource and data volumes
oc set volume deployment/grafana --add --overwrite -t secret --secret-name=grafana-datasources --name=grafana-datasources --mount-path=/etc/grafana/provisioning/datasources --overwrite
oc set volume deployment/grafana --add --overwrite -t persistentVolumeClaim --claim-name=grafana-data --name=grafana-data --mount-path=/var/lib/grafana --overwrite
```
