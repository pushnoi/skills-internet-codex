# Referentie: Implementatiegidsen (Toolbox)

Achtergrondkennis bij de `$inet-toolbox` skill. Dit document bevat links naar
de toolbox-wiki en geavanceerde configuraties.

## Toolbox-wiki bronnen

De [toolbox-wiki](https://github.com/internetstandards/toolbox-wiki) bevat
gedetailleerde gidsen per standaard. Raadpleeg de wiki voor de meest actuele
instructies.

```bash
# Alle gidsen in de toolbox-wiki opvragen
gh api repos/internetstandards/toolbox-wiki/git/trees/main?recursive=1 \
  --jq '.tree[] | select(.type=="blob" and (.path | endswith(".md"))) | .path'

# Specifieke gids ophalen
gh api repos/internetstandards/toolbox-wiki/contents/DNSSEC.md \
  --jq '.content' | base64 -d

# Zoeken in de wiki
gh api "search/code?q=DANE+repo:internetstandards/toolbox-wiki" \
  --jq '.items[] | "\(.path): \(.html_url)"'
```

## DNSSEC - geavanceerde configuratie

### Key rollover schema

**ZSK rollover (aanbevolen: elke 3 maanden):**

| Dag | Actie |
|-----|-------|
| 0 | Genereer nieuwe ZSK, publiceer in DNSKEY RRset |
| 1 | Wacht tot oude TTL verlopen is (24-48 uur) |
| 2-3 | Begin te ondertekenen met nieuwe ZSK |
| 4 | Verwijder oude ZSK na TTL-periode |

**KSK rollover (aanbevolen: jaarlijks):**

| Dag | Actie |
|-----|-------|
| 0 | Genereer nieuwe KSK |
| 1 | Publiceer DS van nieuwe KSK bij registrar (naast oude) |
| 2-7 | Wacht tot DS gepropageerd is (afhankelijk van registrar) |
| 8 | Onderteken DNSKEY RRset met nieuwe KSK |
| 14 | Verwijder oude DS bij registrar |
| 21 | Verwijder oude KSK |

### Algoritme-keuze

| Algoritme | Code | Status |
|-----------|------|--------|
| ECDSAP256SHA256 | 13 | Aanbevolen (compact, snel) |
| ECDSAP384SHA384 | 14 | Goed (sterker, groter) |
| ED25519 | 15 | Goed (nieuwste, compact) |
| RSASHA256 | 8 | Voldoende (breed ondersteund) |

**Aanbeveling:** Gebruik ECDSAP256SHA256 (algoritme 13) voor nieuwe zones.
Compacte handtekeningen, brede ondersteuning en goede prestaties.

### OpenDNSSEC

Voor grotere organisaties met veel zones:

```bash
# OpenDNSSEC installeren
apt install opendnssec softhsm2

# SoftHSM initialiseren
softhsm2-util --init-token --slot 0 --label OpenDNSSEC --so-pin 1234 --pin 1234

# OpenDNSSEC configureren
# /etc/opendnssec/conf.xml - Repository en beleid
# /etc/opendnssec/kasp.xml - Key and Signing Policy

# Zone toevoegen
ods-enforcer zone add --zone example.nl

# Sleutels genereren en zone ondertekenen
ods-enforcer key generate --zone example.nl
ods-signer sign example.nl
```

## HTTPS/TLS - geavanceerde configuratie

### Mozilla SSL Configuration Generator

Gebruik de [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
voor server-specifieke configuratie. Kies "Intermediate" voor de beste balans
tussen beveiliging en compatibiliteit.

### Certificate Transparency monitoring

```bash
# Controleer CT-logs voor jouw domein
curl -s "https://crt.sh/?q=example.nl&output=json" | \
  python3 -c "import sys,json; [print(c['common_name'], c['not_before']) for c in json.load(sys.stdin)[:5]]"
```

### Let's Encrypt automatisering met certbot

```bash
# Automatische vernieuwing met systemd timer
systemctl enable certbot.timer
systemctl start certbot.timer

# Post-renewal hook voor Nginx
# /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
#!/bin/bash
systemctl reload nginx

# Post-renewal hook voor Apache
# /etc/letsencrypt/renewal-hooks/deploy/reload-apache.sh
#!/bin/bash
systemctl reload apache2
```

### Wildcard-certificaat

```bash
# Let's Encrypt wildcard (vereist DNS-challenge)
certbot certonly \
  --manual \
  --preferred-challenges dns \
  -d "example.nl" \
  -d "*.example.nl"
```

## E-mailbeveiliging - geavanceerde configuratie

### Postfix TLS-configuratie (volledig)

```conf
# /etc/postfix/main.cf

# === Inkomend (SMTP server) ===
smtpd_tls_cert_file = /etc/letsencrypt/live/mx.example.nl/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mx.example.nl/privkey.pem
smtpd_tls_security_level = may
smtpd_tls_auth_only = yes
smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtpd_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtpd_tls_mandatory_ciphers = medium
smtpd_tls_exclude_ciphers = aNULL, eNULL, EXPORT, DES, RC4, MD5, PSK, aECDH, EDH-DSS-DES-CBC3-SHA, EDH-RSA-DES-CBC3-SHA, KRB5-DES, CBC3-SHA
smtpd_tls_dh1024_param_file = /etc/ssl/private/dhparams.pem
smtpd_tls_loglevel = 1
smtpd_tls_received_header = yes
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtpd_tls_session_cache_timeout = 3600s

# === Uitgaand (SMTP client) ===
smtp_tls_security_level = dane
smtp_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtp_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtp_tls_mandatory_ciphers = medium
smtp_tls_loglevel = 1
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
smtp_tls_session_cache_timeout = 3600s
smtp_dns_support_level = dnssec

# DH-parameters genereren:
# openssl dhparam -out /etc/ssl/private/dhparams.pem 2048
```

### DKIM key rotation automatiseren

```bash
#!/bin/bash
# /usr/local/bin/rotate-dkim.sh
# Draai maandelijks via cron

DOMAIN="example.nl"
KEYDIR="/etc/opendkim/keys"
NEW_SELECTOR="dkim$(date +%Y%m)"

# Genereer nieuwe sleutel
opendkim-genkey -b 2048 -d "$DOMAIN" -D "$KEYDIR" -s "$NEW_SELECTOR"

# Toon het DNS-record dat gepubliceerd moet worden
echo "Publiceer dit DNS-record:"
cat "${KEYDIR}/${NEW_SELECTOR}.txt"

echo ""
echo "Na DNS-propagatie: update /etc/opendkim/key.table"
echo "en herstart OpenDKIM."
```

### SPF-flattening

Wanneer je tegen de 10 DNS-lookup limiet aanloopt:

```bash
# Voorbeeld: te veel includes
# v=spf1 include:_spf.google.com include:spf.protection.outlook.com include:sendgrid.net include:mailgun.org -all
# Dit zijn al 4 includes, die elk weer includes hebben

# Oplossing 1: IP-adressen direct opnemen
# Resolv de IP-ranges van de providers
dig TXT _spf.google.com +short
# Vervang include door ip4:/ip6: regels

# Oplossing 2: Subdomeinen gebruiken
# marketing.example.nl. IN TXT "v=spf1 include:sendgrid.net -all"
# example.nl. IN TXT "v=spf1 mx include:_spf.google.com -all"
```

### DMARC rapportage met parsedmarc

```bash
# parsedmarc installeren
pip install parsedmarc

# Rapporten analyseren uit mailbox
parsedmarc -e dmarc@example.nl \
  --imap-host imap.example.nl \
  --imap-user dmarc@example.nl \
  --imap-password 'wachtwoord' \
  --output-format json \
  > dmarc_analyse.json

# Of van een lokaal bestand
parsedmarc rapport.xml.gz --output-format json
```

## DANE - geavanceerde configuratie

### Meerdere TLSA-records (voor rollover)

```dns
; Huidig certificaat
_25._tcp.mx.example.nl. IN TLSA 3 1 1 abc123...huidige_hash
; Toekomstig certificaat (pre-publish)
_25._tcp.mx.example.nl. IN TLSA 3 1 1 def456...nieuwe_hash
```

### DANE voor meerdere MX-servers

```dns
; MX-records
example.nl.              IN MX 10 mx1.example.nl.
example.nl.              IN MX 20 mx2.example.nl.

; TLSA per MX
_25._tcp.mx1.example.nl. IN TLSA 3 1 1 <hash-mx1>
_25._tcp.mx2.example.nl. IN TLSA 3 1 1 <hash-mx2>
```

### DANE monitoring

```bash
# Controleer DANE voor alle MX-servers
for mx in $(dig MX example.nl +short | awk '{print $2}' | sed 's/\.$//' ); do
  echo "=== $mx ==="
  dig TLSA "_25._tcp.${mx}" +short
  echo "Certificaat-hash:"
  openssl s_client -connect "${mx}:25" -starttls smtp </dev/null 2>/dev/null | \
    openssl x509 -pubkey -noout | \
    openssl pkey -pubin -outform DER | \
    openssl dgst -sha256 -binary | xxd -p -c 32
  echo ""
done
```

## IPv6 - geavanceerde configuratie

### Firewall-regels (nftables)

```nft
# /etc/nftables.conf
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        # Bestaande verbindingen
        ct state established,related accept

        # Loopback
        iifname lo accept

        # ICMPv6 (nodig voor IPv6)
        ip6 nexthdr icmpv6 accept

        # SSH, HTTP, HTTPS
        tcp dport { 22, 80, 443 } accept

        # SMTP (als mailserver)
        tcp dport 25 accept
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}
```

### Dual-stack testen

```bash
# Test IPv4 en IPv6 naast elkaar
echo "=== IPv4 ==="
curl -4 -sI https://example.nl | head -3
echo "=== IPv6 ==="
curl -6 -sI https://example.nl | head -3

# DNS-records controleren
echo "=== A-record ==="
dig A example.nl +short
echo "=== AAAA-record ==="
dig AAAA example.nl +short

# Nameserver IPv6
dig NS example.nl +short | while read ns; do
  echo "$ns: $(dig AAAA "$ns" +short)"
done
```

## Externe bronnen

- [toolbox-wiki](https://github.com/internetstandards/toolbox-wiki) - Alle implementatiegidsen
- [internet.nl](https://internet.nl) - Test het resultaat
- [Forum Standaardisatie](https://www.forumstandaardisatie.nl/open-standaarden) - Standaardenstatus
- [NCSC - TLS Security Guidelines](https://www.ncsc.nl/en/transport-layer-security/ICT-beveiligingsrichtlijnen-voor-TLS)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
- [Let's Encrypt documentatie](https://letsencrypt.org/docs/)
