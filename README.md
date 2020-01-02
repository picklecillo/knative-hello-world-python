# Hackday: Knative + flask

# Env vars
* Configure env vars
```sh
export PROJECT_NAME=<project_name>
export CLUSTER_NAME=<cluster_name>
export CLUSTER_ZONE=<cluster_zone>
```
* `source env_vars`


# Cluster
```sh
gcloud beta container clusters create $CLUSTER_NAME \
  --addons=HorizontalPodAutoscaling,HttpLoadBalancing,Istio \
  --machine-type=g1-small \
  --cluster-version=latest --zone=$CLUSTER_ZONE \
  --enable-stackdriver-kubernetes --enable-ip-alias \
  --enable-autoscaling --min-nodes=1 --max-nodes=10 \
  --enable-autorepair \
  --scopes cloud-platform
```

```sh
# configure kubectl
gcloud container clusters get-credentials $CLUSTER_NAME --zone $CLUSTER_ZONE --project $PROJECT_NAME

# Enable APIs
gcloud services enable \
cloudapis.googleapis.com \
container.googleapis.com \
containerregistry.googleapis.com

# Cluster Role Binding
kubectl create clusterrolebinding cluster-admin-binding \
 --clusterrole=cluster-admin \
 --user=$(gcloud config get-value core/account)

```

## Delete cluster
```sh
gcloud container clusters delete $CLUSTER_NAME --zone $CLUSTER_ZONE
```

# Install Istio
*  https://knative.dev/docs/install/installing-istio/#installing-istio-without-sidecar-injection

## `istio-system` namespace

```sh
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
 name: istio-system
 labels:
   istio-injection: disabled
   knative-eventing-injection: enabled
EOF

```

# Install knative
```sh
kubectl apply --selector knative.dev/crd-install=true \
--filename https://github.com/knative/serving/releases/download/v0.11.0/serving.yaml \
--filename https://github.com/knative/eventing/releases/download/v0.11.0/release.yaml \
--filename https://github.com/knative/serving/releases/download/v0.11.0/monitoring.yaml

kubectl apply --filename https://github.com/knative/serving/releases/download/v0.11.0/serving.yaml \
--filename https://github.com/knative/eventing/releases/download/v0.11.0/release.yaml \
--filename https://github.com/knative/serving/releases/download/v0.11.0/monitoring.yaml

# Test with...
kubectl get pods --namespace knative-serving
kubectl get pods --namespace knative-eventing
kubectl get pods --namespace knative-monitoring

```

# SET Istio ingress ip

```sh

kubectl edit cm config-domain --namespace knative-serving

```

Algo así:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-domain
  namespace: knative-serving
data:
  # xip.io is a "magic" DNS provider, which resolves all DNS lookups for:
  # *.{ip}.xip.io to {ip}.
  34.83.80.117.xip.io: ""
```

```sh
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export INGRESS_HOST=35.239.214.60
gcloud compute firewall-rules create allow-gateway-http --allow tcp:$INGRESS_PORT
```

# Registry


```sh

# Build image
docker build -t <registry>/hello-world <my-service/code>
# Push to registry
docker push <registry>/hello-world

```

# Apply service
```sh
kubectl apply --filename app_service.yaml

kubectl get service istio-ingressgateway --namespace istio-system
kubectl patch configmap config-domain --namespace knative-serving --patch \
  '{"data": {"example.com": null, "[EXTERNAL-IP].xip.io": ""}}'
  

curl http://helloworld-python.default.35.239.214.60.xip.io/asdf
```
