# Instalare Graph Lab pe TrueNAS SCALE

Imaginea aplicației este publicată în GitHub Container Registry și poate fi
instalată direct din TrueNAS. TrueNAS servește site-ul; Ollama și Cloudflare
Tunnel rămân pe laptop:

- TrueNAS descarcă imaginea `ghcr.io/alex1612356356/graph-lab-truenas-image:latest`
  și servește aplicația web pe portul `30080`.
- Cloudflare Tunnel rulează pe laptop și trimite traficul normal al site-ului
  la TrueNAS.
- Tunnel-ul trimite în continuare `/api/` la `127.0.0.1:11434` de pe laptop,
  unde rulează Ollama.

Laptopul trebuie să fie pornit și conectat la internet pentru ca domeniul și
funcțiile AI să fie disponibile, deoarece Tunnel-ul și Ollama rămân acolo.
Nu deschide portul Ollama `11434` către internet și nu configura port forwarding
pentru el pe router.

## 1. Instalează aplicația din imagine

1. În TrueNAS SCALE/Community Edition, deschide **Apps** și alege instalarea
   unei aplicații nișate / **Install via YAML** (numele opțiunii poate varia
   în funcție de versiune).
2. Copiază conținutul din [`docker-compose.yaml`](./docker-compose.yaml) în
   editorul YAML. Imaginea folosită este:

   ```text
   ghcr.io/alex1612356356/graph-lab-truenas-image:latest
   ```

3. Instalează aplicația și așteaptă să fie pornită. Nu trebuie să copiezi
   manual fișiere web pe TrueNAS și nici să configurezi volume.
4. Dintr-un dispozitiv din aceeași rețea, deschide
   `http://IP-TRUENAS:30080` și verifică dacă se încarcă Graph Lab. Înlocuiește
   `IP-TRUENAS` cu adresa locală a NAS-ului.

Dacă portul `30080` este deja ocupat, alege alt port liber în partea stângă a
mapării `ports` din YAML și folosește acel port în pașii următori. Lasă portul
din dreapta (`80`) neschimbat.

## 2. Actualizează ruta site-ului din Cloudflare Tunnel

Configurația existentă a Tunnel-ului este
`deploy/cloudflared.yml` din proiect. Păstrează în acel fișier valorile
existente pentru `tunnel:` și `credentials-file:`.

În lista `ingress`:

1. Păstrează regula `atestat.fdmedia.studio` pentru `path: ^/api/.*` direcționată
   către `http://127.0.0.1:11434`.
2. La regula generală, fără `path`, pentru `atestat.fdmedia.studio`, schimbă
   serviciul web la `http://IP-TRUENAS:30080`. Folosește IP-ul local al
   TrueNAS-ului și portul ales la pasul 1.
3. Păstrează regula de fallback `http_status:404` la final.

Regulile app-ului trebuie să arate astfel, cu IP-ul tău în locul
`IP-TRUENAS`:

```yaml
- hostname: atestat.fdmedia.studio
  path: ^/api/.*
  service: http://127.0.0.1:11434
  originRequest:
    httpHostHeader: localhost:11434
- hostname: atestat.fdmedia.studio
  service: http://IP-TRUENAS:30080
```

Asigură-te că regula `/api/` apare înaintea regulii generale. Dacă folosești și
`ollama.fdmedia.studio`, poți păstra regulile sale existente neschimbate.

Salvează fișierul după ce îl modifici. **Nu copia fișierul JSON de acreditări
Cloudflare în repository și nu-l publica.**

## 3. Pornește Ollama și Tunnel-ul pe laptop

Dacă Ollama rulează deja ca aplicație desktop, închide-l înainte să pornești
`ollama serve`, ca să nu încerce două instanțe să folosească același port. Într-un
terminal PowerShell pe laptop, pornește Ollama:

```powershell
$env:OLLAMA_HOST = '127.0.0.1:11434'
$env:OLLAMA_ORIGINS = 'https://atestat.fdmedia.studio,https://ollama.fdmedia.studio,http://localhost:8080,http://localhost:11434'
ollama serve
```

Dacă modelele nu sunt deja instalate pe laptop, deschide un alt terminal
PowerShell și descarcă-le:

```powershell
ollama pull qwen2.5vl:3b
ollama pull gemma4:e2b
```

Într-un alt terminal, pornește Cloudflare Tunnel cu fișierul de configurare
actualizat:

```powershell
cloudflared tunnel --config deploy\cloudflared.yml run
```

Lasă procesele Ollama și Tunnel pornite. Nu mai este necesar să rulezi
`python deploy\serve_web.py` pe laptop.

## 4. Verifică funcționarea

1. Confirmă că `http://IP-TRUENAS:30080` se deschide în rețeaua locală.
2. Deschide `https://atestat.fdmedia.studio` și verifică dacă aplicația se
   încarcă prin domeniu.
3. Testează o funcție AI. Dacă aplicația se încarcă, dar AI-ul nu răspunde,
   verifică mai întâi că Ollama și Tunnel-ul sunt pornite și că regula `/api/`
   din Tunnel încă pointează la `127.0.0.1:11434`.
4. Dacă site-ul prin domeniu nu se deschide, verifică IP-ul/portul din regula
   generală și că laptopul poate accesa `http://IP-TRUENAS:30080`.

## Publicarea unei versiuni noi

Imaginea conține fișierele web din directorul `web/` al acestui repository.
Pentru a publica o versiune nouă:

1. Construiește versiunea web actualizată cu `flutter build web --release`.
2. Înlocuiește conținutul directorului `web/` cu fișierele noi din `build\web`.
3. Fă push pe ramura `main`. GitHub Actions publică automat imaginea cu tag-ul
   `latest`.
4. În TrueNAS, redeploy/restart aplicația pentru a descărca imaginea nouă.
