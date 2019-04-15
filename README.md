# openshift observability stack

Deploy standalone prometheus

```
wget https://raw.githubusercontent.com/openshift/origin/master/examples/prometheus/prometheus-standalone.yaml

# adjust images to use as required (these must be imported into your openshift cluster for offline install)
- description: The location of the proxy image
  name: IMAGE_PROXY
  value: openshift/oauth-proxy:v1.0.0
- description: The location of the prometheus image
  name: IMAGE_PROMETHEUS
  value: openshift/prometheus:v2.3.2
- description: The location of the alertmanager image
  name: IMAGE_ALERTMANAGER
  value: openshift/prometheus-alertmanager:v0.15.1
- description: The location of alert-buffer image
  name: IMAGE_ALERT_BUFFER
  value: openshift/prometheus-alert-buffer:v0.0.2
```

Get a default prometheus configuration

```
wget https://raw.githubusercontent.com/prometheus/prometheus/master/documentation/examples/prometheus.yml -O prometheus-default.yml
```

Get a default alertmanager configuration

```
wget https://raw.githubusercontent.com/prometheus/alertmanager/master/doc/examples/simple.yml -O alertmanager.yml
```

We will use a prometheus config file that allows us to scrape based on annotations

```
# e.g. annotate application service

prometheus.io/path: /prometheus
prometheus.io/port: '8081'
prometheus.io/scrape: 'true'
```

Create prometheus

```
# Create project for our observability stack
oc project observability --display-name="Observability" --description="Observability"
oc create secret generic prom --from-file=./prometheus.yml
oc create secret generic prom-alerts --from-file=./alertmanager.yml

# Create the prometheus instance
oc process -f prometheus-standalone.yaml | oc apply -f -
oc policy add-role-to-user view system:serviceaccount:$(oc project -q):prom

```
