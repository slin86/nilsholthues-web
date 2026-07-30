# nilsholthues-web — Deploy-Scaffold

Statische Portfolio-Seite, verpackt nach deinem gewohnten Schema
(Dockerfile → GitHub Actions → ghcr.io → ArgoCD → Kustomize-Overlays → Traefik).

## Struktur

```
Dockerfile              nginx:alpine + statische Dateien
nginx.conf              einfache Server-Config, Caching für Bilder
k8s/base/               Deployment + Service (Basis, env-unabhängig)
k8s/overlays/internal/  Ingress auf nilsholthues.home.lan (Phase A)
k8s/overlays/public/    IngressRoute auf nilsholthues.slin.io (Phase B),
                        Phase C für nils-holthues.de ist auskommentiert vorbereitet
.github/workflows/      Build & Push nach ghcr.io/slin86/nilsholthues-web
argocd/application.yaml In dein argocd-Repo übernehmen
```

`index.html` + die vier Bilder (`hero-landungsbruecken.jpg`, `portrait-nils.jpg`,
`harbor-tugboats-sunset.jpg`, `hamburg-flag.jpg`) kommen noch ins Repo-Root dazu,
dann greift der Dockerfile-Copy.

## Rollout

1. **Phase A (intern):** ArgoCD-App zeigt auf `overlays/internal` → Test unter
   `nilsholthues.home.lan`.
2. **Phase B (slin.io):** `targetRevision`/`path` in der ArgoCD-App auf
   `overlays/public` umstellen. `*.slin.io`-Wildcard-DNS und -Zertifikat greifen
   bereits, es ist nichts Neues an DNS nötig — nur den Secret-Namen in
   `ingressroute.yaml` auf deinen tatsächlichen Wildcard-Secret-Namen anpassen.
3. **Phase C (nils-holthues.de):** siehe Runbook unten, dann den auskommentierten
   Block in `ingressroute.yaml` aktivieren.

## Runbook: nils-holthues.de anbinden (liegt bei united-domains)

1. **Nameserver umstellen:** bei united-domains die NS für `nils-holthues.de` auf
   deSEC setzen (`ns1.desec.io` / `ns2.desec.io`) — genau wie bei `slin.io`.
2. **Zone in deSEC anlegen** und einen neuen, auf diese Domain **gescopten
   DynDNS-Token** erzeugen (separat vom slin.io-Token, least privilege).
3. **Dynamische IP-Updates:** Die FritzBox erlaubt i. d. R. nur einen
   benutzerdefinierten DynDNS-Eintrag gleichzeitig — der ist schon für
   `slin.io` belegt. Zwei Optionen:
   - **Schnell:** A-Record einmalig manuell in deSEC auf deine aktuelle
     öffentliche IP setzen (funktioniert, bis sich die IP ändert).
   - **Sauber:** kleiner CronJob im Cluster, der die eigene öffentliche IP
     ermittelt und bei Änderung `https://update.dedyn.io` mit dem
     nils-holthues.de-Token aufruft — läuft dann unabhängig von der FritzBox
     und passt gut zu deinem GitOps-Ansatz (Secret über Infisical).
4. **Zertifikat:** je nachdem, ob du Traefiks natives ACME (lego) oder
   cert-manager für die DNS-01-Challenge bei slin.io nutzt, dieselbe
   Konfiguration um `nils-holthues.de` (+ `www.nils-holthues.de`) erweitern.
5. **IngressRoute:** auskommentierten Block in `k8s/overlays/public/ingressroute.yaml`
   aktivieren, `secretName` auf das neue Zertifikat-Secret setzen.
