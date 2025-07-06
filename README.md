# cert-manager-ops

## Upgrading guide

Special considerations may be required when upgrading the Helm chart, and these are documented in:
https://cert-manager.io/docs/installation/upgrade/

## Self-signed certificates (informative)

WARNING:
For demonstration purpose, in local environment, root CA certificate and private key have been
generated and tracked. They should NOT be used for protected environments because the encrypted
secrets are tracked and the demo SOPS private key is also tracked and public.

Following procedures illustrate how the certificate and key are generated, in case similar
procedures are needed for other environments:

```bash
mkdir -p secret
openssl genrsa -out secret/code-monkey-root-ca.key 2048
openssl req -x509 -new -nodes -key secret/code-monkey-root-ca.key -sha256 -days 3650 -out secret/code-monkey-root-ca.crt -subj "/CN=code-monkey"

CRT=$(cat secret/code-monkey-root-ca.crt | base64 -w0)
KEY=$(cat secret/code-monkey-root-ca.key | base64 -w0)

# Create local-secret.values.dec.yaml
echo "secrets:" >> secret/local-secret.values.dec.yaml
echo "  data:" >> secret/local-secret.values.dec.yaml
echo "    - name: code-monkey-root-ca-secret" >> secret/local-secret.values.dec.yaml
echo "      data:" >> secret/local-secret.values.dec.yaml
echo "        tls.crt: ${CRT}" >> secret/local-secret.values.dec.yaml
echo "        tls.key: ${KEY}" >> secret/local-secret.values.dec.yaml

# Encrypt the secret values
sops --config local.sops.yaml -e secret/local-secret.values.dec.yaml > helm-chart/local-secret.values.yaml
```

The encrypted secrets, after decrypted and templated, shall produce a Secret resource, similar to
the following:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: code-monkey-root-ca-secret
data:
  tls.crt: "LS0..."
  tls.key: "LS0..."
```

Create a CA ClusterIssuer Configure cert-manager to use the manually created root CA.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: code-monkey-root-ca-cluster-issuer
spec:
  ca:
    secretName: code-monkey-root-ca-secret
```

Request Certificates Using the Root CA. For example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: iam-keycloak-ing
  annotations:
    cert-manager.io/cluster-issuer: code-monkey-root-ca-cluster-issuer
spec:
  ingressClassName: nginx
  tls:
    # Overlayed
    - hosts:
        - iam.code-monkey.local
      secretName: code-monkey-local-tls-secret
  rules:
```

## References

- https://artifacthub.io/packages/helm/cert-manager/cert-manager
