# k3s-server

## Get started

Install k3s:
```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -
```

Create services:
```
helmfile sync
```

## Setup network

### Configure stage acme server

If you want to test your domains, I recommend using another stage acme server without limits in `network/values.yaml`:
```
network:
  tls:
    issuerName: nginx-issuer
    secretName: nginx-tls
    server: https://acme-staging-v02.api.letsencrypt.org/directory
```

### Configure ACL for ingress

To configure ACL for ingress you should add the directive `acl` for site in `network/values.yaml`:
```
- host: k3s.example.com
  ingress:
    name: k3s-proxy
  svc:
    namespace: kubernetes-dashboard
    name: kubernetes-dashboard-kong-proxy
    port: 443
    protocol: HTTPS
  acl:
    - 10.42.0.0/16
```

P.S. 10.42.0.0/16 - default k3s local subnet.

### Set your own domain

Change example domain for ingress resources:
```
sed -i '.bak' '/host:/ s/.example.com/.newdomain.com/' network/values.yaml
```

### Apply configuration

Create ingresses:
```
helm install network network -n default
```

## Setup apps

### kubernetes-dashboard

Create user `admin-user` for namespace `kubernetes-dashboard`:
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

Give `cluster-admin` role for created user `admin-user`:
```
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

Generate access token for 10 hours:
```
kubectl create token -n kubernetes-dashboard admin-user --duration 10h
```

### marzban

You should change VLESS privateKey, SpiderX and shortIds in `helmfile.yaml`.

Generate VLESS privateKey:
```
kubectl exec <marzban pod> -n <marzban namespace> -- /usr/local/bin/xray x25519
```

Generate VLESS SpiderX:
```
pwgen -A0s 20
```

Generate VLESS shortIds:
```
openssl rand -hex 8
```

### grafana

You can change default username/password in `helmfile.yaml`.
