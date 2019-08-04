# kubernetes-setup

## Create GKE cluster

```bash #gke
k8s_version=$(gcloud container get-server-config \
  --format=json \
  --region=europe-north1 \
  | jq -r '.validNodeVersions[0]' \
  | tee /dev/tty)

gcloud container clusters create cluster-1 \
  --cluster-version=${k8s_version} \
  --region=europe-north1 \
  --num-nodes=1 \
  --machine-type=n1-standard-2 \
  --preemptible \
  --enable-autorepair \
  --enable-autoupgrade \
  --enable-autoscaling --min-nodes=3 --max-nodes=9 \
  --no-enable-basic-auth \
  --no-issue-client-certificate \
  --enable-ip-alias \
  --metadata disable-legacy-endpoints=true

gcloud container clusters get-credentials cluster-1  \
  --region=europe-north1

kubectl create clusterrolebinding "cluster-admin-$(whoami)" \
  --clusterrole=cluster-admin \
  --user="$(gcloud config get-value core/account)"
```

## Setup Tiller

```bash #tiller
# create tiller service role
kubectl --namespace kube-system create sa tiller
kubectl create clusterrolebinding tiller \
  --clusterrole cluster-admin \
  --serviceaccount=kube-system:tiller

helm init --service-account tiller
```

## Install istio

```bash #istio
[ -d istio ] || git clone git@github.com:istio/istio.git
pushd ./istio
git checkout 1.2.3

kubectl create namespace istio-system

helm upgrade --install istio-init install/kubernetes/helm/istio-init \
  --namespace istio-system \
  --set certmanager.enabled=true \
  --wait

helm upgrade --install istio install/kubernetes/helm/istio \
  --namespace istio-system \
  --set gateways.istio-ingressgateway.sds.enabled=true \
  --set certmanager.enabled=true \
  --set certmanager.email=me@example.com

kubectl label namespace default istio-injection=enabled
```
