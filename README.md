# openshift observability stack

Deploy standalone prometheus

```
wget https://raw.githubusercontent.com/openshift/origin/master/examples/prometheus/prometheus-standalone.yaml

# adjust images to use as required
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

The default prometheus configuration is here

```
wget https://raw.githubusercontent.com/prometheus/prometheus/master/documentation/examples/prometheus.yml -O prometheus-default.yml
```

Get a default alertmanager configuration

```
wget https://raw.githubusercontent.com/prometheus/alertmanager/master/doc/examples/simple.yml -O alertmanager.yml
```

We will use a custom prometheus config file that allows us to scrape applications based on annotations

```
# example annotate a service so it can be scraped
oc annotate svc my-service --overwrite prometheus.io/path='/prometheus' prometheus.io/port='8081' prometheus.io/scrape='true'
```

Create prometheus

```
# Create project for our observability stack
oc new-project observability --display-name="Observability" --description="Observability"
oc create secret generic prom --from-file=./prometheus.yml
oc create secret generic prom-alerts --from-file=./alertmanager.yml

# Create the prometheus instance
oc process -f prometheus-standalone.yaml | oc apply -f -
oc policy add-role-to-user view system:serviceaccount:$(oc project -q):prom

# If using multitenant plugin:
oc get clusternetwork default --template='{{.pluginName}}'
redhat/openshift-ovs-multitenant

# then we need to make prometheus global
oc adm pod-network make-projects-global observability

```

Prometheus persistent data

```
-- prometheus persistent data

oc create -n rabbitmq -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: prometheus-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

oc volume statefulsets/prom --add --overwrite -t persistentVolumeClaim --claim-name=prometheus-data --name=prometheus-data --mount-path=/prometheus

oc create -n rabbitmq -f - <<EOF
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
EOF

oc volume statefulsets/prom --add --overwrite -t persistentVolumeClaim --claim-name=alertmanager-data --name=alertmanager-data --mount-path=/alertmanager
```

Application

```
# https://github.com/eformat/camel-springboot-rest-ose

-- becuase we are not using fabric8 fragments, do this manually
oc edit svc camel-springboot-rest-ose-master

# add port
  - name: 8080-tcp
    port: 8080
    protocol: TCP
    targetPort: 8080

-- expose route and set it on our swagger endpoint
oc expose svc camel-springboot-rest-ose-master --port=8080
oc set env dc/camel-springboot-rest-ose-master SWAGGERUI_HOST=$(oc get route camel-springboot-rest-ose-master --template='{{ .spec.host }}')

```
