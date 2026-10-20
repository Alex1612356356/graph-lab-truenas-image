# Graph Lab pentru TrueNAS SCALE

Imagine Docker publicată în GitHub Container Registry:

```text
ghcr.io/alex1612356356/graph-lab-truenas-image:latest
```

## Instalare

În TrueNAS, deschide **Apps** și instalează aplicația prin YAML. Folosește
conținutul din [`docker-compose.yaml`](./docker-compose.yaml), schimbând portul
din stânga lui `30080:80` dacă este deja ocupat. Nu este necesară copierea
fișierelor web sau configurarea unui volum local.

Ghidul complet, inclusiv configurarea opțională Cloudflare Tunnel și Ollama,
este în [`INSTALL-GUIDE.md`](./INSTALL-GUIDE.md).

## Actualizarea imaginii

La fiecare push pe ramura `main`, GitHub Actions construiește și publică
imaginea cu etichetele `latest` și `sha-<commit>`. În TrueNAS, redeploy/restart
aplicația pentru a descărca imaginea nouă.

Imaginea conține build-ul Flutter Web din directorul `web/`. Pentru a publica o
versiune nouă a aplicației, înlocuiește conținutul acestui director cu un build
nou (`flutter build web --release`), apoi fă push pe `main`.
