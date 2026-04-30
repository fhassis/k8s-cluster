# TLS Certificate Management

All web services (Jenkins, Grafana, Nexus, XWiki, GitLab, ArgoCD) are exposed via Traefik, which handles TLS termination. A single wildcard certificate for `*.fhassis.top` covers every subdomain from one Secret in `kube-system`.

## How it works

```
Browser → port 443 → Traefik (kube-system)
                          ↓ reads TLSStore default
                     kube-system/wildcard-tls (Secret)
                          ↓ terminates TLS
                     forwards plain HTTP to each pod
```

Traefik's `TLSStore` CRD (`tlsstore.yaml`, deployed by ArgoCD at wave -1) designates `kube-system/wildcard-tls` as the **default certificate** for all Ingress resources. Every service gets HTTPS from this single Secret — no per-namespace copies.

## Importing a certificate (DigiCert or any CA)

**1. Obtain the certificate from your CA**

You need two files:
- `cert.pem` — your certificate **plus the full intermediate chain**, concatenated:
  ```
  -----BEGIN CERTIFICATE-----
  (your wildcard cert)
  -----END CERTIFICATE-----
  -----BEGIN CERTIFICATE-----
  (intermediate CA cert)
  -----END CERTIFICATE-----
  ```
- `key.pem` — your private key (the one you generated the CSR with)

**2. Import as a Kubernetes TLS Secret**

```bash
kubectl create secret tls wildcard-tls \
  --cert=cert.pem \
  --key=key.pem \
  --namespace kube-system
```

Traefik picks up the new cert within seconds — no restart needed.

**3. Verify**

```bash
# Confirm the Secret exists
kubectl get secret wildcard-tls -n kube-system

# Test TLS from the command line (replace with any service hostname)
curl -v https://jenkins.fhassis.top 2>&1 | grep -E "subject|issuer|expire"
```

## Renewal

When your cert expires, delete the old Secret and re-create it:

```bash
kubectl delete secret wildcard-tls -n kube-system
kubectl create secret tls wildcard-tls \
  --cert=new-cert.pem \
  --key=new-key.pem \
  --namespace kube-system
```

Traefik reloads within seconds. No application restarts needed.

## Why not per-namespace secrets?

Kubernetes Secrets are namespace-scoped. With 5+ namespaces (jenkins, nexus, xwiki, gitlab, observability), per-namespace certs would require either a replication controller or manual copies on every renewal. The TLSStore default avoids this entirely: one Secret, all services.

## Switching CAs

Because all Ingress resources reference the TLSStore (not a per-service `secretName`), switching from one CA to another only requires replacing the `kube-system/wildcard-tls` Secret. Zero changes to any Helm values or application configs.
