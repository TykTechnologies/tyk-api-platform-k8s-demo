# Tyk K8S API Platform Teams Demo
This repository will help you get started with Tyk OSS in kubernetes and allow 
you to leverage OSS tools such as Keycloak and the Tyk Operator to manage your
API authentication and authorization in k8s using OAuth2.0 

## Requirements
1. [minikube](https://minikube.sigs.k8s.io/docs/start/)
2. [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
3. [helm](https://helm.sh/docs/intro/install/)

## Installation

### Minikube & Helm Repositories

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

### Tyk

1. Install Redis. Redis is a requirement for Tyk Gateway. It is used as a 
performant key store. 
```
helm install tyk-redis tyk-helm/simple-redis \
   --namespace tyk \
   --create-namespace \
   --wait
```

2. Install Tyk API Gateway.
```
APISecret=topsecretpassword
helm install tyk-gateway tyk-helm/tyk-oss \
   --namespace tyk \
   --set 'global.redis.addrs[0]=redis.tyk.svc:6379' \
   --set global.secrets.APISecret=$APISecret \
   --wait 
```

You can expose the Tyk Gateway to your localhost using the following command:
```
kubectl port-forward svc/gateway-svc-tyk-gateway --namespace tyk 8080 &
```

You can check the state of the Tyk gateway using the following `curl` command:
```
curl localhost:8080/hello
```

### Tyk Operator

1. Create Tyk Operator k8s secret to allow Tyk Operator to connect to the Tyk Gateway.
```
kubectl create secret generic tyk-operator-conf \
   --namespace tyk \
   --from-literal="TYK_MODE=ce" \
   --from-literal="TYK_URL=http://gateway-svc-tyk-gateway.tyk.svc:8080" \
   --from-literal="TYK_AUTH=$APISecret" \
   --from-literal="TYK_ORG=tyk" 
```

2. Install CertManager. CertManager is a requirement for the Tyk Operator. 
```
helm install cert-manager jetstack/cert-manager \
   --version v1.10.1 \
   --namespace tyk \
   --set "installCRDs=true" \
   --set "prometheus.enabled=false" \
   --wait 
```

3. Install Tyk Operator.
```
helm install tyk-operator tyk-helm/tyk-operator \
   --namespace tyk \
   --wait
```

### Deploy HttpBin service and expose it through the Tyk Gateway

1. Deploy the HttpBin deployment and service
```
kubectl apply -f resources/httpbin/httpbin.yaml \
   --namespace tyk
```

2. Expose the HttpBin service through the Tyk Gateway
```
kubectl apply -f resources/httpbin/httpbin-api.yaml \
   --namespace tyk
```

You can access the httpbin api using the following curl command:
```
curl localhost:8080/httpbin/get
```

### Keycloak

1. Install Keycloak Operator.
```
kubectl apply -f resources/keycloak-operator/keycloaks.k8s.keycloak.org-v1.yml \
   --namespace tyk
kubectl apply -f resources/keycloak-operator/keycloakrealmimports.k8s.keycloak.org-v1.yml \
   --namespace tyk
kubectl apply -f resources/keycloak-operator/kubernetes.yml \
   --namespace tyk
```

2. Install Postgres. Postgres is a requirement for Keycloak as a backend. 
```
POSTGRES_PASSWORD=topsecretpassword
helm install keycloak-postgres bitnami/postgresql \
   --namespace tyk \
   --set "auth.database=keycloak-db" \
   --set "auth.postgresPassword=$POSTGRES_PASSWORD" \
   --wait
```

3. Create Keycloak database credentials secret. 
```
kubectl create secret generic keycloak-db-secret \
   --namespace tyk \
   --from-literal="username=postgres" \
   --from-literal="password=$POSTGRES_PASSWORD" 
```

4. Create Keycloak credentials secret.
```
KEYCLOAK_USERNAME=default@example.com
KEYCLOAK_PASSWORD=topsecretpassword
kubectl create secret generic keycloak-initial-admin \
   --namespace tyk \
   --from-literal="username=$KEYCLOAK_USERNAME" \
   --from-literal="password=$KEYCLOAK_PASSWORD"
```

5. Create Keycloak CRD to install a Keycloak instance using the Keycloak Operator.
```
kubectl apply -f resources/keycloak/keycloak.yaml \
   --namespace tyk
```

6. Import Keycloak realm with preconfigured users to test out the OAuth2.0 flow.
```
kubectl apply -f resources/keycloak/keycloak-realm.yaml \
   --namespace tyk
```

You can expose Keycloak to your localhost using the following command:
```
kubectl port-forward svc/keycloak-service --namespace tyk 7000 &
```

You can access the Keycloak instance in your browser at [localhost:7000](http://localhost:7000):
```
Username: default@example.com
Password: topsecretpassword
```

### Expose HttpBin through the Tyk Gateway and manage AuthN and AuthZ using OAuth2.0

1. Expose the HttpBin service through the Tyk Gateway
```
kubectl apply -f resources/httpbin/httpbin-keycloak.yaml \
   --namespace tyk
```

2. There are three user profiles available; you can generate a JWT associated 
with each profile using the following curl commands. The JWT can be passed to
the gateway under the `Authorization` header:

- Developer user, gives access to `/xml` endpoint:
```
curl -L --insecure -s -X POST 'http://localhost:7000/realms/keycloak-oauth/protocol/openid-connect/token' \
   -H 'Content-Type: application/x-www-form-urlencoded' \
   --data-urlencode 'client_id=keycloak-oauth' \
   --data-urlencode 'grant_type=password' \
   --data-urlencode 'client_secret=NoTgoLZpbrr5QvbNDIRIvmZOhe9wI0r0' \
   --data-urlencode 'scope=openid' \
   --data-urlencode 'username=developer@example.com' \
   --data-urlencode 'password=topsecretpassword' | jq -r '.access_token'
```

- Admin user, gives access to all endpoints:
```
curl -L --insecure -s -X POST 'http://localhost:7000/realms/keycloak-oauth/protocol/openid-connect/token' \
   -H 'Content-Type: application/x-www-form-urlencoded' \
   --data-urlencode 'client_id=keycloak-oauth' \
   --data-urlencode 'grant_type=password' \
   --data-urlencode 'client_secret=NoTgoLZpbrr5QvbNDIRIvmZOhe9wI0r0' \
   --data-urlencode 'scope=openid' \
   --data-urlencode 'username=admin@example.com' \
   --data-urlencode 'password=topsecretpassword' | jq -r '.access_token'
```

- Random user, does not give access even if it can generate a valid Keycloak JWT:
```
curl -L --insecure -s -X POST 'http://localhost:7000/realms/keycloak-oauth/protocol/openid-connect/token' \
   -H 'Content-Type: application/x-www-form-urlencoded' \
   --data-urlencode 'client_id=keycloak-oauth' \
   --data-urlencode 'grant_type=password' \
   --data-urlencode 'client_secret=NoTgoLZpbrr5QvbNDIRIvmZOhe9wI0r0' \
   --data-urlencode 'scope=openid' \
   --data-urlencode 'username=random@example.com' \
   --data-urlencode 'password=topsecretpassword' | jq -r '.access_token'
```

You can access the httpbin Keycloak managed api using the following curl command:
```
curl -L --insecure -s -X POST 'http://localhost:8080/httpbin-jwt/get' \
   -H 'Authorization: $JWT'
```