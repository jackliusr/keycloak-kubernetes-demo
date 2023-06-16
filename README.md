# Keycloak Kubernetes Demo

This demo assumes you have Kind installed with the ingress addon enabled.

My environment is WSL2. I have to make some tweaks.


nip.io cann't be resolved in WSL2. I changed my dns resolver to 8.8.8.8 to resovle this issue.


## Setup URLs

The following URLs uses nip.io to prevent having to modify `/etc/hosts`.

```bash
    kind create cluster --config cluster.yaml
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
    export IP=$(ip -4 addr show eth0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
    export MINIKUBE_IP=$IP
    export KEYCLOAK_HOST=keycloak.$MINIKUBE_IP.nip.io
    export BACKEND_HOST=backend.$MINIKUBE_IP.nip.io
    export FRONTEND_HOST=frontend.$MINIKUBE_IP.nip.io
```

## Keycloak

```bash
    kubectl create -f keycloak/keycloak.yaml
    
    cat keycloak/keycloak-ingress.yaml | sed "s/KEYCLOAK_HOST/$KEYCLOAK_HOST/" \
    | kubectl create -f -

    echo https://$KEYCLOAK_HOST
```

The Keycloak admin console should now be opened in your browser. Ignore the warning caused by the self-signed certificate. Login with admin/admin. Create a new realm and import `keycloak/realm.json`.

The client config for the frontend allows any redirect-uri and web-origin. This is to simplify configuration for the demo. For a production system always use the real URL of the application for the redirect-uri and web-origin.

## Backend

```bash
    eval `minikube docker-env`
    docker build -t kube-demo-backend backend
    kind load docker-image kube-demo-backend

    cat backend/backend.yaml | sed "s/KEYCLOAK_HOST/$KEYCLOAK_HOST/" | \
    kubectl create -f -

    cat backend/backend-ingress.yaml | sed "s/BACKEND_HOST/$BACKEND_HOST/" | \
    kubectl create -f -

    echo https://$BACKEND_HOST/public
```
## Frontend

```bash
    docker build -t kube-demo-frontend frontend
    kind load docker-image  kube-demo-frontend

    cat frontend/frontend.yaml | sed "s/KEYCLOAK_HOST/$KEYCLOAK_HOST/" | \
    sed "s/BACKEND_HOST/$BACKEND_HOST/" | \
    kubectl create -f -

    cat frontend/frontend-ingress.yaml | sed "s/FRONTEND_HOST/$FRONTEND_HOST/" | \
    kubectl create -f - 

    echo https://$FRONTEND_HOST
```

The frontend application should now be opened in your browser. Login with stian/pass. You should be able to invoke public and invoke secured, but not invoke admin. To be able to invoke admin go back to the Keycloak admin console and add the `admin` role to the user `stian`.
