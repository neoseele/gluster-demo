## Deploy Gluster

### Local SSD Alpha

```sh
PROJECT_ID=nmiu-play
SA=my-cluster-admin
# SA=nmiu-cluster-admin

gcloud iam service-accounts create ${SA} --display-name=${SA}

gcloud projects add-iam-policy-binding ${PROJECT_ID} --member=serviceAccount:${SA}@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/container.clusterAdmin

gcloud projects add-iam-policy-binding ${PROJECT_ID} --member=serviceAccount:${SA}@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/iam.serviceAccountActor

gcloud iam service-accounts keys create ${SA}-private-key.json --iam-account=${SA}@${PROJECT_ID}.iam.gserviceaccount.com

gcloud auth activate-service-account ${SA}@${PROJECT_ID}.iam.gserviceaccount.com --key-file=${SA}-private-key.json
```

```sh
export CLOUDSDK_CONTAINER_USE_V1_API_CLIENT=false
```
or
```sh
gcloud config set container/use_v1_api_client false
```

```sh
gcloud alpha container clusters list --log-http
```

*https://cloud.google.com/kubernetes-engine/docs/reference/api-organization#beta*


### Create the GKE cluster

_must have minimal 3 nodes to run a gluster cluster_

```sh
gcloud alpha container clusters create kube-test \
--machine-type=n1-standard-2 \
--num-nodes=3 \
--image-type=COS \
--cluster-version=1.9.2-gke.1 \
--node-labels=storagenode=glusterfs \
--tags=ssh \
--enable-kubernetes-alpha \
--local-ssd-volumes count=1,type=scsi,format=block \
--preemptible \
--scopes default,cloud-platform,cloud-source-repos,service-control
```

_unset use_v1_api_client_ 
```sh
gcloud config unset container/use_v1_api_client
```

### Modprobe / Cleanup SSD

```sh
i=1
for node in `kubectl get nodes -o jsonpath='{.items[*].metadata.name}'`;
do
  echo "* ${node}";
  gcloud compute ssh $node -- 'sudo sh -c "modprobe dm_thin_pool"'
  # gcloud compute ssh $node -- 'sudo sh -c "umount /dev/sdb && dd if=/dev/zero of=/dev/sdb bs=512 count=100"'
  ((i+=1))
done
```

### Pre Deploy

```sh
cat << EOF | kubectl apply -f -
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: default-rbac
subjects:
  - kind: ServiceAccount
    name: default
    namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```

```sh
kubectl create secret generic heketi-admin-secret \
  --from-literal=key='12qwaszx34erdfcv' \
  --type=kubernetes.io/glusterfs
```

```sh
cat << EOF | kubectl apply -f -
---
apiVersion: v1
kind: Pod
metadata:
  name: nm-toolbox
spec:
  containers:
  - name: nm-toolbox
    image: gcr.io/nmiu-play/nm-toolbox:latest
    command:
    - '/bin/sh'
    - '-c'
    - while true; do sleep 5; done;
    env:
    - name: HEKETI_ADMIN_SECRET
      valueFrom:
        secretKeyRef:
          name: heketi-admin-secret
          key: key
EOF
```

### Deploy

```sh
kubectl exec nm-toolbox deploy.sh
```

### Create service (ext)

```sh
cat << EOF | kubectl apply -f -
---
kind: Service
apiVersion: v1
metadata:
  name: heketi-ext
  labels:
    glusterfs: heketi-ext-service
    heketi: ext-service
  annotations:
    description: Exposes Heketi Service Externally
spec:
  type: LoadBalancer
  selector:
    glusterfs: heketi-pod
  ports:
  - name: heketi
    port: 18080
    targetPort: 8080
EOF
```

### Create storage class

```sh
HEKETI="$(kubectl describe service heketi-ext | grep Ingress: | awk '{print $3}'):18080"
echo $HEKETI
```
_use the heketi-ext-service for resturl_

```sh
cat << EOF | kubectl apply -f -
---
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: gluster-heketi
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://${HEKETI}"
  restuser: "admin"
  secretNamespace: "default"
  secretName: "heketi-admin-secret"
EOF
```
_mark gluster-heketi as default storage class_

```sh
kubectl patch storageclass gluster-heketi -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.beta.kubernetes.io/is-default-class":"false"}}}'
```

### Heketi Cli

_In nm-toolbox pod_

```sh
kubectl exec nm-toolbox -it bash
```

```sh
heketi-cli-admin volume list

## reload topology
build-topology.rb > topology.json
heketi-cli-admin topology load --json=topology.json
```

## Clean Up

* Delete the service
```sh
kubectl delete service heketi-ext
```

* Delete the cluster
```sh
gcloud container clusters delete kube-test
```
