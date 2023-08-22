# Kong on Kubernetes

* Following this [guide](https://docs.konghq.com/gateway/latest/install/kubernetes/helm-quickstart/). Assumtions:

* `ClusterIssuer` called `letsencrypt-prod` that is configured to use Let's Encrypt to issue certificates
* We will also be using the `postgresql-ha` chart from Bitnami as our database, so make sure to set `pg_host` to the pgpool service
* the nginx ingress controller is installed

1. Create namespace: `kubectl create namespace kong`
2. Create Kong config and credential variables:

```console
kubectl create secret generic kong-config-secret -n kong \
    --from-literal=portal_session_conf='{"storage":"kong","secret":"super_secret_salt_string","cookie_name":"portal_session","cookie_same_site":"Lax","cookie_secure":false}' \
    --from-literal=admin_gui_session_conf='{"storage":"kong","secret":"super_secret_salt_string","cookie_name":"admin_session","cookie_same_site":"Lax","cookie_secure":false}' \
    --from-literal=pg_host="kong-postgresql-postgresql-ha-pgpool.kong-postgresql.svc" \
    --from-literal=pg_port="5432" \
    --from-literal=pg_user="postgres" \
    --from-literal=pg_password="<REDACTED>" \
    --from-literal=kong_admin_password=kong \
    --from-literal=password=kong
```

3. Create a Kong Enterprise license secret:
  
```console
kubectl create secret generic kong-enterprise-license --from-literal=license="'{}'" -n kong --dry-run=client -o yaml | kubectl apply -f -
```

4. Create cluster cert:

```console
openssl genrsa -out kong.key 4096
openssl req -new -x509 -days 3650 -key ca.key -out ca.pem
openssl req -new -key kong.key -out kong.csr
```

TNow create a CSR and have the cluster sign it:

```console
CSR=`cat kong.csr | base64 | tr -d '\n'`

cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: kong
spec:
  request: $CSR
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 31536000  # one day
  usages:
  - client auth
EOF
kubectl certificate approve kong
```

Finally, get the signed cert:

```console
kubectl get csr kong -o jsonpath='{.status.certificate}' | base64 --decode > kong.crt

kubectl create secret tls kong-kong-cluster-cert -n kong --key kong.key --cert kong.crt
```

5. Install Kong:

```console
helm upgrade --install --namespace kong \
  --values ./values.yml kong kong/kong
```

## Issues

Using the values specified in `values.yml`, I get the following error(s):

* 302 redirect loop when trying to access the admin UI 