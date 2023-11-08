# Tyk K8S API Platform Teams Demo

## Requirements
1. minikube
2. kubectl
3. helm
4. git

## Installation

#### Minikube

1. Fetch required helm repositories:
```
helm repo add tyk-helm https://helm.tyk.io/public/helm/charts/
helm repo add jetstack https://charts.jetstack.io
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

2. Start Minikube
```
minikube start
minikube addons enable ingress
```

#### Tyk

1. Install Redis - Tyk requirement
```
helm install tyk-redis tyk-helm/simple-redis \
   --namespace tyk \
   --create-namespace
```

2. Install Tyk API Gateway
```
APISecret=topsecretpassword
helm install tyk-gateway tyk-helm/tyk-oss \
   --namespace tyk \
   --set 'global.redis.addrs[0]=redis.tyk.svc:6379' \
   --set global.secrets.APISecret=$APISecret
```

#### Tyk Operator
1. Create Tyk Operator Secret to allow Tyk Operator to connect to Tyk Gateway
```
kubectl create secret generic tyk-operator-conf \
   --namespace tyk \
   --from-literal="TYK_MODE=ce" \
   --from-literal="TYK_URL=http://gateway-svc-tyk-gateway.tyk.svc:8080" \
   --from-literal="TYK_AUTH=$APISecret" \
   --from-literal="TYK_ORG=tyk"
```

2. Install CertManager - Tyk Operator requirement
```
helm install cert-manager jetstack/cert-manager \
   --version v1.10.1 \
   --namespace tyk \
   --set "installCRDs=true" \
   --set "prometheus.enabled=false"
```

3. Install Tyk Operator
```
helm install tyk-operator tyk-helm/tyk-operator \
   --namespace tyk
```

#### Keycloak
1. Install Keycloak Operator
```
kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/22.0.5/kubernetes/keycloaks.k8s.keycloak.org-v1.yml \
   --namespace tyk
kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/22.0.5/kubernetes/keycloakrealmimports.k8s.keycloak.org-v1.yml \
   --namespace tyk
kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/22.0.5/kubernetes/kubernetes.yml \
   --namespace tyk
```

2. Install Postgres - Keycloak requirement
```
   --version 11.9.7 \

POSTGRES_PASSWORD=topsecretpassword
helm install keycloak-postgres bitnami/postgresql \
   --namespace tyk \
   --set "auth.database=keycloak-db" \
   --set "auth.postgresPassword=$POSTGRES_PASSWORD"
```

3. Create Keycloak database credentials secret
```
kubectl create secret generic keycloak-db-secret \
   --namespace tyk \
   --from-literal="username=postgres" \
   --from-literal="password=$POSTGRES_PASSWORD" 
```

4. Create Keycloak credentials secret
```
KEYCLOAK_USERNAME=default@example.com
KEYCLOAK_PASSWORD=topsecretpassword
kubectl create secret generic keycloak-initial-admin \
   --namespace tyk \
   --from-literal="username=$KEYCLOAK_USERNAME" \
   --from-literal="password=$KEYCLOAK_PASSWORD"
```

5. Create Keycloak CRD to install a Keycloak instance using the Keycloak Operator
```
kubectl apply -f ./keycloak.yaml \
   --namespace tyk
```