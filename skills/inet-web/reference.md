# Referentie: Webstandaarden

Achtergrondkennis bij de `$inet-web` skill. Dit document bevat protocol-details,
server-specifieke configuraties en bronverwijzingen.

## TLS protocol-details

### Aanbevolen cipher suites (NCSC)

Het NCSC adviseert de volgende cipher suites voor TLS 1.2:

**Goed (voorkeur):**
- TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
- TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
- TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
- TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
- TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
- TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256

**Voldoende:**
- TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
- TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256

**Niet toegestaan:**
- Alles met RC4, 3DES, NULL, EXPORT, DES, MD5
- TLS_RSA_* cipher suites (geen forward secrecy)

Voor TLS 1.3 zijn alleen deze cipher suites beschikbaar (allemaal goed):
- TLS_AES_128_GCM_SHA256
- TLS_AES_256_GCM_SHA384
- TLS_CHACHA20_POLY1305_SHA256

### Certificaatvereisten

- Minimaal 2048-bit RSA of 256-bit ECDSA
- SHA-256 of hoger als hash-algoritme
- Geldige keten naar een vertrouwde root CA
- Hostnaam in Subject Alternative Name (SAN)

### Renegotiation en OCSP (NCSC 2025-05)

Met de NCSC TLS-richtlijnen van mei 2025 (geïmplementeerd in Internet.nl v1.11.0):

- **Client-initiated renegotiation** is acceptabel mits de server het aantal renegotiations beperkt tot minder dan 10 per verbinding. Onbeperkte client renegotiation blijft een fout (DoS-risico).
- **Secure renegotiation** (RFC 5746) blijft verplicht.
- **Extended Master Secret** (RFC 7627) is een aparte test geworden voor TLS 1.2; ontbreken telt als fout. Voor TLS 1.3 niet van toepassing.
- **OCSP stapling** wordt niet langer als fout gerekend wanneer het certificaat zelf geen OCSP-endpoint bevat (`AIA` zonder OCSP URI). In dat geval is stapling technisch onmogelijk en wordt het als niet-getest gemeld.

Bronnen:
- [NCSC - TLS Security Guidelines](https://www.ncsc.nl/en/transport-layer-security/ICT-beveiligingsrichtlijnen-voor-TLS)
- [Forum Standaardisatie - TLS](https://www.forumstandaardisatie.nl/open-standaarden/tls)

## DNSSEC protocol-details

### Sleuteltypen

- **KSK (Key Signing Key):** Ondertekent de DNSKEY RRset. Hash hiervan staat als DS
  bij de parent zone. Typisch RSA 2048-bit of ECDSA P-256.
- **ZSK (Zone Signing Key):** Ondertekent alle andere RRsets in de zone. Kan kleiner
  zijn (RSA 1024-bit of ECDSA P-256) vanwege frequentere rollover.

### Key rollover

**ZSK rollover (pre-publish methode):**
1. Publiceer nieuwe ZSK in DNSKEY RRset (naast de oude)
2. Wacht tot TTL verlopen is
3. Onderteken zone met nieuwe ZSK
4. Verwijder oude ZSK na TTL-periode

**KSK rollover (double-DS methode):**
1. Genereer nieuwe KSK
2. Publiceer DS van nieuwe KSK bij registrar (naast oude DS)
3. Wacht tot TTL verlopen is
4. Onderteken DNSKEY RRset met nieuwe KSK
5. Verwijder oude DS bij registrar

### Diagnostische commando's

```bash
# Alle DNSKEY records opvragen
dig DNSKEY example.nl +multi

# NSEC/NSEC3 chain controleren
dig NSEC example.nl +dnssec

# Validatie met specifieke resolver
dig @9.9.9.9 example.nl +dnssec +cd  # cd = checking disabled
dig @9.9.9.9 example.nl +dnssec      # validatie aan

# DS-record vergelijken met DNSKEY
dig DS example.nl +short
dig DNSKEY example.nl +short | dnssec-dsfromkey -f - example.nl
```

Bronnen:
- [Forum Standaardisatie - DNSSEC](https://www.forumstandaardisatie.nl/open-standaarden/dnssec)
- [SIDN - DNSSEC](https://www.sidn.nl/dnssec)

## IPv6 configuratie per platform

### Linux (netplan)

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 192.0.2.1/24
        - 2001:db8::1/64
      gateway4: 192.0.2.254
      gateway6: 2001:db8::fffe
      nameservers:
        addresses: [2001:4860:4860::8888, 8.8.8.8]
```

### Nginx dual-stack

```nginx
server {
    listen 80;
    listen [::]:80;       # IPv6
    listen 443 ssl;
    listen [::]:443 ssl;  # IPv6 met TLS
    server_name example.nl;
}
```

### Apache dual-stack

```apache
Listen 80
Listen [::]:80
Listen 443
Listen [::]:443

<VirtualHost *:443 [::]:443>
    ServerName example.nl
    SSLEngine on
</VirtualHost>
```

### DNS-records

```dns
; IPv4
example.nl.    IN  A     192.0.2.1
www.example.nl IN  A     192.0.2.1

; IPv6
example.nl.    IN  AAAA  2001:db8::1
www.example.nl IN  AAAA  2001:db8::1

; Nameservers moeten ook IPv6 hebben
ns1.example.nl IN  A     192.0.2.10
ns1.example.nl IN  AAAA  2001:db8::10
```

Bronnen:
- [Forum Standaardisatie - IPv6](https://www.forumstandaardisatie.nl/open-standaarden/ipv6)
- [Forum Standaardisatie - DNSSEC](https://www.forumstandaardisatie.nl/open-standaarden/dnssec)

## HSTS-details

### Preload-lijst

HSTS preloading zorgt ervoor dat browsers je domein altijd via HTTPS benaderen,
zelfs bij het eerste bezoek. Vereisten voor opname in de preload-lijst:

1. Geldig TLS-certificaat
2. Redirect van HTTP naar HTTPS op hetzelfde domein
3. HSTS-header met:
   - `max-age` minimaal 31536000 (1 jaar)
   - `includeSubDomains` directive
   - `preload` directive

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

Aanmelden: https://hstspreload.org/

**Let op:** `preload` is onomkeerbaar op korte termijn. Zorg dat alle subdomeinen
HTTPS ondersteunen voordat je dit inschakelt.

## Security headers - details

### Content-Security-Policy (CSP)

CSP voorkomt cross-site scripting (XSS) en data-injectie door te specificeren
welke bronnen de browser mag laden.

**Gefaseerde aanpak:**

1. Begin met `Content-Security-Policy-Report-Only` om impact te analyseren
2. Configureer een report-uri om schendingen te monitoren
3. Verscherp het beleid geleidelijk
4. Schakel over naar afdwingend beleid

**Veelgebruikte directives:**

| Directive | Doel |
|-----------|------|
| `default-src` | Fallback voor alle resource-types |
| `script-src` | JavaScript-bronnen |
| `style-src` | CSS-bronnen |
| `img-src` | Afbeeldingsbronnen |
| `font-src` | Lettertypebronnen |
| `connect-src` | XHR, WebSocket, fetch |
| `frame-ancestors` | Wie mag de pagina embedden (vervangt X-Frame-Options) |
| `base-uri` | Beperkt `<base>` element |
| `form-action` | Beperkt formulier-acties |

### Referrer-Policy waarden

| Waarde | Gedrag |
|--------|--------|
| `no-referrer` | Nooit een Referer header meesturen |
| `strict-origin` | Alleen origin meesturen, alleen bij HTTPS->HTTPS |
| `strict-origin-when-cross-origin` | Volledig pad bij same-origin, origin bij cross-origin (aanbevolen) |
| `same-origin` | Alleen bij same-origin requests |

## security.txt - RFC 9116

### Verplichte velden

| Veld | Beschrijving |
|------|-------------|
| `Contact` | URL of e-mailadres voor melden van kwetsbaarheden (minimaal 1) |
| `Expires` | Vervaldatum in ISO 8601 formaat (maximaal 1 jaar vooruit) |

### Optionele velden

| Veld | Beschrijving |
|------|-------------|
| `Encryption` | URL naar PGP-sleutel voor versleutelde communicatie |
| `Acknowledgments` | URL naar pagina met bedankingen voor melders |
| `Preferred-Languages` | Voorkeurstalen (bijv. `nl, en`) |
| `Canonical` | Canonieke URL van het security.txt bestand |
| `Policy` | URL naar het responsible disclosure beleid |
| `Hiring` | URL naar vacatures in het beveiligingsteam |

### PGP-ondertekening

```bash
# security.txt ondertekenen
gpg --clearsign --output security.txt.signed security.txt
mv security.txt.signed .well-known/security.txt
```

Bronnen:
- [Forum Standaardisatie - security.txt](https://www.forumstandaardisatie.nl/open-standaarden/securitytxt)
- [NCSC - Kwetsbaarheid melden (CVD)](https://www.ncsc.nl/dienstverlening/kwetsbaarheid-melden-cvd)

## RPKI - details

### Hoe RPKI werkt

1. **ROA (Route Origin Authorisation):** Een digitaal ondertekende verklaring die
   een IP-prefix koppelt aan een AS-nummer (het netwerk dat de prefix mag aankondigen).
2. **ROV (Route Origin Validation):** Routers controleren BGP-aankondigingen tegen
   de gepubliceerde ROA's.
3. **Validatiestatus:**
   - `Valid` - BGP-aankondiging komt overeen met een ROA
   - `Invalid` - BGP-aankondiging conflicteert met een ROA
   - `NotFound` - Geen ROA gevonden voor de prefix

### ROA aanmaken

ROA's worden aangemaakt in het RIPE NCC-portaal (voor Europese netwerken):
1. Log in op https://my.ripe.net
2. Ga naar RPKI Dashboard
3. Maak een ROA aan voor je prefix en AS-nummer

### RPKI-status controleren

```bash
# Via RIPE Stat
curl -s "https://stat.ripe.net/data/rpki-validation/data.json?resource=AS1234&prefix=192.0.2.0/24"

# Via Cloudflare RPKI checker
curl -s "https://rpki.cloudflare.com/api/v1/prefix/192.0.2.0/24"
```

Bronnen:
- [Forum Standaardisatie - RPKI](https://www.forumstandaardisatie.nl/open-standaarden/rpki)
- [RIPE NCC - RPKI](https://www.ripe.net/manage-ips-and-asns/resource-management/rpki/)

## Verkenning van de testsuite

```bash
# Bekijk de testcategorieen in de Internet.nl broncode
gh api repos/internetstandards/Internet.nl/git/trees/main?recursive=1 \
  --jq '.tree[] | select(.path | startswith("checks/")) | .path'

# Bekijk de scoring-logica
gh api repos/internetstandards/Internet.nl/contents/checks \
  --jq '.[].name'
```
