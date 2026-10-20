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

## 1. Completează formularul „Install Custom App”

În TrueNAS, deschide **Apps → Discover → ix-app → Install**. În formularul din
captură, completează câmpurile astfel:

| Secțiune / câmp | Valoare |
| --- | --- |
| **Application Name** | `graph-lab` |
| **Version** | Lasă versiunea oferită (`1.4.5` în captură). Aceasta este versiunea formularului/aplicației TrueNAS, nu tag-ul imaginii Docker. |
| **Notes** | Opțional; poți lăsa gol. |

Deschide apoi secțiunile din formular și configurează:

### Image Configuration

| Câmp | Valoare |
| --- | --- |
| **Repository** / **Image repository** | `ghcr.io/alex1612356356/graph-lab-truenas-image` |
| **Tag** | `latest` |
| **Pull Policy** | `Always`, dacă este disponibil; altfel lasă valoarea implicită. |

Tag-ul `latest` este public, deci nu trebuie să adaugi acreditări sau să te
autentifici în GitHub Container Registry.

### Container Configuration

Lasă **Entrypoint**, **Command** și variabilele de mediu necompletate/la
valorile implicite. Imaginea pornește singură Nginx pe portul `80`.

### Security Context Configuration

Păstrează valorile implicite. Nu activa **Privileged** și nu adăuga capabilități
speciale.

### Network Configuration

Adaugă o regulă de publicare a portului containerului:

| Câmp | Valoare |
| --- | --- |
| **Container Port** | `80` |
| **Host Port** / **Node Port** | `30080` |
| **Protocol** | `TCP` |
| **Port Binding** | Activat, dacă formularul afișează această opțiune. |

Dacă formularul are câmpurile **Host IP** sau **Bind Address**, lasă-le
necompletate/implicit pentru a permite accesul din rețeaua locală. Dacă
`30080` este ocupat, folosește alt port liber pentru **Host Port** și reține
portul ales.

### Portal Configuration

Opțional, activează portalul pentru aplicație și setează protocolul `HTTP` și
portul `30080` (sau portul Host ales mai sus). Portalul este doar o scurtătură
din interfața TrueNAS; publicarea portului din **Network Configuration** este
cea care face serviciul accesibil.

### Storage Configuration

Nu adăuga volume sau mount-uri. Fișierele site-ului și configurarea Nginx sunt
deja incluse în imagine.

Salvează formularul cu **Install** și așteaptă până când aplicația este
pornită. Dintr-un dispozitiv din aceeași rețea, deschide
`http://IP-TRUENAS:30080` (sau portul Host ales) și verifică dacă se încarcă
Graph Lab. Înlocuiește `IP-TRUENAS` cu adresa locală a NAS-ului.

> Dacă preferi instalarea prin YAML în locul formularului, folosește
> [`docker-compose.yaml`](./docker-compose.yaml). Nu lipi Compose YAML în
> câmpurile formularului **Install Custom App**.

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
