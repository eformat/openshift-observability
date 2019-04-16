# OpenShift Observability Stack

There are 3 general areas for Observability of applications. Currently this is performed by

- Monitoring
- Log analysis
- Tracing

We will examine some opensource tools to perform these functions as part of a Proof of Concept atop OpenShift.

## Monitoring Stack

Prometheus only stores a (configurable) amount of metrics data.

A full scalable production stack requires a set of components that can be composed into a highly available  metric system with unlimited storage capacity.

[Thanos](https://github.com/improbable-eng/thanos) is an opensource project that can be added seamlessly on top of existing Prometheus deployments to provide this scaling layer.

Object storage is provided by [rook.io](https://rook.io/) which is based on ceph and provides an S3 endpoint to Thanos.

```
   x--------------------x
---| Grafana Dashboard  |
|  x--------------------X
|
|  x--------------------x   x--------------------x
|  | Prometheus Metrics |---|    Applications    |
|  x--------------------X   x--------------------x
|            |
|  x--------------------x
|->|   Thanos Scaling   |
   x--------------------X
             |
   x--------------------x
   |   rook.io Storage  |
   x--------------------X
```

The initial proof of concept will be deployed using NFS. Object Storage and software defined storage will be required for long term storage and requires storage design and requirements gathering.

## Prometheus

Deploy a standalone prometheus using OpenShift example templates.

In future 4.X versions of OpenShift the prometheus operator will become supported for standalone monitoring of applications (i.e. can deploy a multi-tenant prometheus operator for application stack). A blog discussing how this will look is here - https://coreos.com/blog/the-prometheus-operator.html

We start with the default prometheus OpenShift example template

```
wget https://raw.githubusercontent.com/openshift/origin/master/examples/prometheus/prometheus-standalone.yaml
```

We have adjusted the template to allow the `prom` service account to scrape our application endpoints and use the provided images available in our cluster.

```
# Add ClusterRole and ClusterRoleBinding
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

# Adjust the default images versions to use based on the current OpenShift cluster version
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

We will use a prometheus configuration file that allows us to scrape applications based on `Service` annotations. This allows a developer to annotate their service so that the exposed metrics are available to be scraped.

Login as an OpenShift that has cluster admin privilege (this is required to create ClusterRole and ClusterBindings)

Create a prometheus deployment.

```
# Create project to host our observability stack
oc new-project observability --display-name="Observability" --description="Observability"

# Create secrets containing configuration
oc create secret generic prom --from-file=./prometheus.yml
oc create secret generic prom-alerts --from-file=./alertmanager.yml
oc create -f ./prometheus-htpasswd-secret.yml

# Create the prometheus instance
oc process -f prometheus-standalone.yaml -p NAMESPACE=observability | oc apply -f -

# Allow view access to namespace for prom service account
oc policy add-role-to-user view system:serviceaccount:$(oc project -q):prom

# (Optional) if using OpenShift multitenant sdn plugin - check using:
oc get clusternetwork default --template='{{.pluginName}}'
redhat/openshift-ovs-multitenant

# Then we need to make prometheus global so it can be seen by all projects
oc adm pod-network make-projects-global observability
```

Prometheus and alert manager persistent data using nfs (size appropriately)

```
# prometheus persistent data

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

oc set volume statefulsets/prom --add --overwrite -t persistentVolumeClaim --claim-name=prometheus-data --name=prometheus-data --mount-path=/prometheus

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

oc set volume statefulsets/prom --add --overwrite -t persistentVolumeClaim --claim-name=alertmanager-data --name=alertmanager-data --mount-path=/alertmanager
```

## Application

Deploy an example SpringBoot Application using FIS S2I image. The underlying fuse image already has configuration to make prometheus metrics available on port 9779.

The source code application is based here: https://github.com/eformat/camel-springboot-rest-ose

```
# Create an application project
oc new-project my-app

# If using mvn fabric8 plugin, deploy the app using
mvn fabric8:deploy

# If NOT not using fabric8 fragments to build and deploy, we can do these steps manually
oc new-app fuse-java-openshift:1.2~https://github.com/eformat/camel-springboot-rest-ose.git

oc delete svc camel-springboot-rest-ose
oc expose dc camel-springboot-rest-ose --name=camel-springboot-rest-ose --port=8080,8778,9779 --generator=service/v1

# expose route and set it on our swagger endpoint
oc expose svc camel-springboot-rest-ose --port=8080
oc set env dc/camel-springboot-rest-ose SWAGGERUI_HOST=$(oc get route camel-springboot-rest-ose --template='{{ .spec.host }}')

# Annotate our SpringBoot service so it can be scraped
oc annotate svc camel-springboot-rest-ose --overwrite prometheus.io/path='/prometheus' prometheus.io/port='9779' prometheus.io/scrape='true'
```

If the `jolokia/hawt.io` console is not available for the fuse application, check the deployment config has these ports enabled (edit and replace them)

```
          ports:
            - containerPort: 8778
              name: jolokia
              protocol: TCP
            - containerPort: 9779
              name: prometheus
              protocol: TCP
            - containerPort: 8080
              name: http
              protocol: TCP
```

## Grafana

The example template is here.

```
wget https://raw.githubusercontent.com/openshift/origin/master/examples/grafana/grafana.yaml -O grafana.yaml
```

We adjust the template to use the images available in OpenShift.

Deploy Grafana with persistent storage for data and graphs

```
# Switch back to monitoring project
oc project observability

# Create prometheus datasource
oc create secret generic grafana-datasources --from-file="prometheus.yaml=./grafana-prometheus-secret.json"

# Create a PVC for storing grafana dashboards and configuration
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

# Create standalone grafana
oc new-app -f grafana.yaml -p NAMESPACE=$(oc project -q)

# Allow Ouath delegation
oc adm policy add-cluster-role-to-user system:auth-delegator -z grafana -n observability

# Add datasource and data volumes to pod
oc set volume deployment/grafana --add --overwrite -t secret --secret-name=grafana-datasources --name=grafana-datasources --mount-path=/etc/grafana/provisioning/datasources --overwrite
oc set volume deployment/grafana --add --overwrite -t persistentVolumeClaim --claim-name=grafana-data --name=grafana-data --mount-path=/var/lib/grafana --overwrite
```

## Examples

Login to grafana and Import the `helloservice-grafana-dashboard.json` dashboard. Try scaling the Application pod to 2 manually, you should see metrics being collected in prometheus and grafana. If you browse to the root URL of the Application, you can try out the `hello` swagger API endpoints to generate metrics traffic.

![image](images/grafana-dashboard.png)

## Tracing Stack

```
   x--------------------x   x--------------------x
   |  Jaeger Tracing    |---|    Applications    |
   x--------------------X   x--------------------x
             |
   x--------------------x
   | Cassandra Storage  |
   x--------------------X
```

[Opentracing](https://opentracing.io/) contains vendor-neutral APIs and instrumentation for distributed tracing.

Production deployment is documented [here](https://github.com/jaegertracing/jaeger-openshift#production-setup) where Cassandra or Elasticsearch can be used as backing storage for the Jaeger Collector and Query.

The `Jaeger` all-in-one deployment suitable for development and testing can be downloaded here

```
wget https://raw.githubusercontent.com/jaegertracing/jaeger-openshift/master/all-in-one/jaeger-all-in-one-template.yml
```

We can deploy Jaeger Opentracing into OpenShift using

```
oc project observability
oc process -f ./jaeger-all-in-one-template.yml | oc create -f -
```

Set the Application's Jaeger endpoint in the deploymentconfig

```
oc project my-app
oc set env dc/camel-springboot-rest-ose JAEGER_ENDPOINT=http://jaeger-collector.observability.svc.cluster.local:14268/api/traces
```
