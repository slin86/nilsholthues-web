# nilsholthues-web — Deploy Scaffold

Static portfolio site, packaged according to your established pattern
(Dockerfile → GitHub Actions → ghcr.io → ArgoCD → Kustomize overlays → Traefik).

## Structure

```
Dockerfile              nginx:alpine + static files
nginx.conf              simple server config, caching for images
k8s/base/               Deployment + Service (base, environment-independent)
k8s/overlays/internal/  Ingress on nilsholthues.home.lan (Phase A)
k8s/overlays/public/    IngressRoute on nilsholthues.slin.io (Phase B),
                        Phase C for nils-holthues.de is prepared in commented form
.github/workflows/      Build & push to ghcr.io/slin86/nilsholthues-web
argocd/application.yaml  Adopt into your argocd repo
```

`index.html` + the four images (`hero-landungsbruecken.jpg`, `portrait-nils.jpg`,
`harbor-tugboats-sunset.jpg`, `hamburg-flag.jpg`) will be added to the repo root,
then the Dockerfile copy will take effect.

## Rollout

1. **Phase A (internal):** ArgoCD app points to `overlays/internal` → testing under
   `nilsholthues.home.lan`.
2. **Phase B (slin.io):** Change `targetRevision`/`path` in the ArgoCD app to
   `overlays/public`. The `*.slin.io` wildcard DNS and certificate are already
   in place — no new DNS changes needed, just adjust the secret name in
   `ingressroute.yaml` to your actual wildcard secret name.
3. **Phase C (nils-holthues.de):** see runbook below, then activate the commented
   block in `ingressroute.yaml`.

## Runbook: connecting nils-holthues.de (registered with united-domains)

1. **Change nameservers:** At united-domains, set the NS for `nils-holthues.de` to
   deSEC (`ns1.desec.io` / `ns2.desec.io`) — same as for `slin.io`.
2. **Create zone in deSEC** and generate a new DynDNS Token scoped to this domain
   (separate from the slin.io token, following least privilege).
3. **Dynamic IP updates:** The FritzBox typically allows only one custom
   DynDNS entry at a time — the one for `slin.io` is already in use. Two options:
    - **Quick:** Set an A-Record manually once in deSEC to your current public
      IP (works until the IP changes).
    - **Clean:** Small CronJob in the cluster that detects its public IP and,
      upon change, calls `https://update.dedyn.io` with the
      nils-holthues.de token — runs independently of the FritzBox and fits well
      with your GitOps approach (secret via Infisical).
4. **Certificate:** Depending on whether you use Traefik's native ACME (lego) or
   cert-manager for the DNS-01 challenge at slin.io, extend the same
   configuration to include `nils-holthues.de` (+ `www.nils-holthues.de`).
5. **IngressRoute:** Activate the commented block in `k8s/overlays/public/ingressroute.yaml`, set `secretName` to the new certificate secret.
