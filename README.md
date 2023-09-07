# mcs-testing-01
testing multi-cluster services

### setup

```
export CLUSTER_1=autopilot-cluster-us-central1
export CLUSTER_2=autopilot-cluster-us-east4

kubectl --context=$CLUSTER_1 create ns whereami-http-frontend
kubectl --context=$CLUSTER_2 create ns whereami-http-frontend

kubectl --context=$CLUSTER_1 create ns whereami-http-backend
kubectl --context=$CLUSTER_2 create ns whereami-http-backend

mkdir -p whereami-http-frontend/base
mkdir -p whereami-http-backend/base
mkdir whereami-http-backend/variant
mkdir whereami-http-frontend/variant

cat <<EOF > whereami-http-backend/base/kustomization.yaml 
resources:
  - github.com/GoogleCloudPlatform/kubernetes-engine-samples/whereami/k8s
EOF

cat <<EOF > whereami-http-backend/variant/cm-flag.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: whereami
data:
  BACKEND_ENABLED: "False" # assuming you don't want a chain of backend calls
  METADATA:        "backend"
EOF

cat <<EOF > whereami-http-backend/variant/service-type.yaml 
apiVersion: "v1"
kind: "Service"
metadata:
  name: "whereami"
spec:
  type: ClusterIP
EOF

cat <<EOF > whereami-http-backend/variant/kustomization.yaml 
nameSuffix: "-backend"
namespace: whereami-http-backend
commonLabels:
  app: whereami-backend
resources:
- ../base
patches:
- path: cm-flag.yaml
  target:
    kind: ConfigMap
- path: service-type.yaml
  target:
    kind: Service
EOF

kubectl --context=$CLUSTER_1 apply -k whereami-http-backend/variant
kubectl --context=$CLUSTER_2 apply -k whereami-http-backend/variant

cat <<EOF > whereami-http-frontend/base/kustomization.yaml 
resources:
  - github.com/GoogleCloudPlatform/kubernetes-engine-samples/whereami/k8s
EOF

mkdir whereami-http-frontend/variant

cat <<EOF > whereami-http-frontend/variant/cm-flag.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: whereami
data:
  BACKEND_ENABLED: "True"
  BACKEND_SERVICE:        "http://whereami-backend.whereami-http-backend.svc.cluster.local"
EOF


cat <<EOF > whereami-http-frontend/variant/kustomization.yaml 
nameSuffix: "-frontend"
namespace: whereami-http-frontend
commonLabels:
  app: whereami-frontend
resources:
- ../base
patches:
- path: cm-flag.yaml
  target:
    kind: ConfigMap
EOF

kubectl --context=$CLUSTER_1 apply -k whereami-http-frontend/variant
kubectl --context=$CLUSTER_2 apply -k whereami-http-frontend/variant
```