# Referentie: Mailstandaarden

Achtergrondkennis bij de `$inet-mail` skill. Dit document bevat server-specifieke
configuraties, key rotation-procedures en rapportage-analyse.

## Postfix-configuratie

### STARTTLS inschakelen

```conf
# /etc/postfix/main.cf

# TLS voor inkomende verbindingen (server)
smtpd_tls_cert_file = /etc/letsencrypt/live/mx.example.nl/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mx.example.nl/privkey.pem
smtpd_tls_security_level = may
smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtpd_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtpd_tls_mandatory_ciphers = medium
smtpd_tls_loglevel = 1

# TLS voor uitgaande verbindingen (client)
smtp_tls_security_level = dane
smtp_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtp_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtp_tls_loglevel = 1
smtp_dns_support_level = dnssec
```

### DKIM met OpenDKIM

```conf
# /etc/opendkim.conf
Syslog          yes
UMask           007
Mode            sv
SubDomains      no
Canonicalization relaxed/simple
KeyTable        /etc/opendkim/key.table
SigningTable     refile:/etc/opendkim/signing.table
ExternalIgnoreList refile:/etc/opendkim/trusted.hosts
InternalHosts   refile:/etc/opendkim/trusted.hosts
```

```conf
# /etc/opendkim/key.table
default._domainkey.example.nl example.nl:default:/etc/opendkim/keys/example.nl/default.private

# /etc/opendkim/signing.table
*@example.nl default._domainkey.example.nl

# /etc/opendkim/trusted.hosts
127.0.0.1
::1
localhost
example.nl
```

**Sleutelpaar genereren:**

```bash
mkdir -p /etc/opendkim/keys/example.nl
opendkim-genkey -b 2048 -d example.nl -D /etc/opendkim/keys/example.nl -s default -v
chown opendkim:opendkim /etc/opendkim/keys/example.nl/default.private
```

**Postfix koppelen aan OpenDKIM:**

```conf
# /etc/postfix/main.cf
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:8891
non_smtpd_milters = inet:localhost:8891
```

### Postfix en SPF-validatie

```conf
# /etc/postfix/main.cf
# Installeer pypolicyd-spf
policyd-spf_time_limit = 3600
smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination,
    check_policy_service unix:private/policyd-spf
```

## Microsoft Exchange / Microsoft 365

### SPF voor Microsoft 365

```dns
example.nl. IN TXT "v=spf1 include:spf.protection.outlook.com -all"
```

### DKIM voor Microsoft 365

DKIM wordt geconfigureerd via het Microsoft 365 Defender-portaal:
1. Ga naar Email & Collaboration > Policies > DKIM
2. Selecteer het domein
3. Klik op "Enable"
4. Publiceer de CNAME-records die Microsoft aangeeft:

```dns
selector1._domainkey.example.nl. IN CNAME selector1-example-nl._domainkey.example.onmicrosoft.com.
selector2._domainkey.example.nl. IN CNAME selector2-example-nl._domainkey.example.onmicrosoft.com.
```

### DMARC voor Microsoft 365

```dns
_dmarc.example.nl. IN TXT "v=DMARC1; p=reject; rua=mailto:dmarc@example.nl; fo=1"
```

## Google Workspace

### SPF voor Google Workspace

```dns
example.nl. IN TXT "v=spf1 include:_spf.google.com -all"
```

### DKIM voor Google Workspace

1. Ga naar Google Workspace Admin > Apps > Google Workspace > Gmail > Verificatie
2. Klik op "Genereer nieuw record"
3. Kies sleutellengte 2048-bit
4. Publiceer het TXT-record in DNS

```dns
google._domainkey.example.nl. IN TXT "v=DKIM1; k=rsa; p=MIIBIjANBgkqhki..."
```

## DKIM key rotation

### Waarom roteren?

- Beperkt de schade als een privesleutel wordt gecompromitteerd
- Aanbevolen: elke 6-12 maanden
- Gebruik een nieuwe selector bij elke rotatie

### Rotatieprocedure

```bash
# Stap 1: Genereer nieuwe sleutel met nieuwe selector
opendkim-genkey -b 2048 -d example.nl -D /etc/opendkim/keys/example.nl -s selector202602 -v

# Stap 2: Publiceer nieuwe DKIM-record in DNS
# selector202602._domainkey.example.nl. IN TXT "v=DKIM1; k=rsa; p=MIIBIjANBgkqhki..."

# Stap 3: Wacht tot DNS is gepropageerd (controleer TTL)
dig TXT selector202602._domainkey.example.nl +short

# Stap 4: Update OpenDKIM configuratie naar nieuwe selector
# In /etc/opendkim/key.table:
# selector202602._domainkey.example.nl example.nl:selector202602:/etc/opendkim/keys/example.nl/selector202602.private

# Stap 5: Herstart OpenDKIM
systemctl restart opendkim

# Stap 6: Wacht 1-2 weken, verwijder dan het oude DNS-record
```

## DMARC-rapportage analyseren

### Aggregaat rapporten (rua)

DMARC-aggregaatrapporten worden dagelijks verstuurd als XML (vaak gzipped).
Ze bevatten statistieken over alle e-mail die namens jouw domein is verstuurd.

**Structuur van een rapport:**

```xml
<feedback>
  <report_metadata>
    <org_name>google.com</org_name>
    <date_range><begin>1700000000</begin><end>1700086400</end></date_range>
  </report_metadata>
  <policy_published>
    <domain>example.nl</domain>
    <p>reject</p>
  </policy_published>
  <record>
    <row>
      <source_ip>192.0.2.1</source_ip>
      <count>42</count>
      <policy_evaluated>
        <disposition>none</disposition>
        <dkim>pass</dkim>
        <spf>pass</spf>
      </policy_evaluated>
    </row>
  </record>
</feedback>
```

**Analyseren met commandline:**

```bash
# Rapport uitpakken en bekijken
gunzip < rapport.xml.gz | xmllint --format -

# Alle IP-adressen met failed SPF/DKIM
gunzip < rapport.xml.gz | xmllint --xpath '//record/row[policy_evaluated/spf="fail" or policy_evaluated/dkim="fail"]' -
```

**Tooling voor DMARC-rapportage:**
- Open-source: parsedmarc (Python)
- Online diensten: dmarcian, Postmark, Report URI

### Forensische rapporten (ruf)

Forensische rapporten bevatten individuele e-mailberichten die DMARC-verificatie
niet doorstaan. Let op: niet alle mailproviders versturen ruf-rapporten.

## DANE - TLSA record beheer

### TLSA-record aanmaken voor Postfix

```bash
# Haal het certificaat op
openssl s_client -connect mx.example.nl:25 -starttls smtp </dev/null 2>/dev/null > cert.pem

# Genereer TLSA-hash (usage=3, selector=1, matching=1)
openssl x509 -in cert.pem -pubkey -noout | \
  openssl pkey -pubin -outform DER | \
  openssl dgst -sha256 -binary | xxd -p -c 32

# Resultaat: abc123...def456 (64 hex karakters)
# DNS-record:
# _25._tcp.mx.example.nl. IN TLSA 3 1 1 abc123...def456
```

### TLSA bijwerken bij certificaatvernieuwing

Bij gebruik van Let's Encrypt met automatische vernieuwing:

1. **Pre-publish methode:** Publiceer het TLSA-record van het nieuwe certificaat
   voordat het actief wordt
2. Gebruik `DANE-EE (3) + SPKI (1)` zodat alleen de publieke sleutel telt, niet
   het volledige certificaat
3. Als je dezelfde privesleutel hergebruikt bij vernieuwing, hoeft het TLSA-record
   niet te veranderen

**Automatisering met certbot hook:**

```bash
#!/bin/bash
# /etc/letsencrypt/renewal-hooks/deploy/update-tlsa.sh
DOMAIN="mx.example.nl"
HASH=$(openssl x509 -in /etc/letsencrypt/live/$DOMAIN/cert.pem -pubkey -noout | \
  openssl pkey -pubin -outform DER | \
  openssl dgst -sha256 -binary | xxd -p -c 32)
echo "Nieuw TLSA-record: _25._tcp.$DOMAIN. IN TLSA 3 1 1 $HASH"
# Hier: update via DNS API van je provider
```

## MTA-STS configuratie

> **Let op:** MTA-STS wordt niet getest door internet.nl. Het is een aanvullende
> best practice naast DANE.

### Vereisten

1. DNS TXT-record: `_mta-sts.example.nl`
2. HTTPS-website op `mta-sts.example.nl` met geldig certificaat
3. Beleidsbestand op `https://mta-sts.example.nl/.well-known/mta-sts.txt`

### Nginx-configuratie voor MTA-STS

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name mta-sts.example.nl;

    ssl_certificate /etc/letsencrypt/live/mta-sts.example.nl/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mta-sts.example.nl/privkey.pem;

    location /.well-known/mta-sts.txt {
        root /var/www/mta-sts;
        default_type text/plain;
    }
}
```

### SMTP TLS Reporting (TLSRPT)

Voeg een TLSRPT DNS-record toe om rapporten te ontvangen over TLS-fouten:

```dns
_smtp._tls.example.nl. IN TXT "v=TLSRPTv1; rua=mailto:tls-reports@example.nl"
```

## Relatie tussen mailstandaarden

```
Verzendende server                    Ontvangende server
     |                                      |
     |-- DKIM ondertekening -->             |
     |-- SPF (IP check via DNS) -->         |
     |-- STARTTLS (versleuteling) -------->  |
     |      \-- DANE (cert verificatie) --> |
     |                                      |
     |                              DMARC beleid toepassen
     |                              (op basis van SPF+DKIM)
     |                                      |
     |  <-- DMARC rapportage (rua/ruf) --   |
     |  <-- TLSRPT rapportage ----------    |
```

## Bronnen

- [Forum Standaardisatie - DMARC](https://www.forumstandaardisatie.nl/open-standaarden/dmarc)
- [Forum Standaardisatie - SPF](https://www.forumstandaardisatie.nl/open-standaarden/spf)
- [Forum Standaardisatie - DKIM](https://www.forumstandaardisatie.nl/open-standaarden/dkim)
- [Forum Standaardisatie - STARTTLS](https://www.forumstandaardisatie.nl/open-standaarden/starttls-en-dane)
- [Forum Standaardisatie - DANE](https://www.forumstandaardisatie.nl/open-standaarden/starttls-en-dane)
- [NCSC - TLS Security Guidelines](https://www.ncsc.nl/en/transport-layer-security/ICT-beveiligingsrichtlijnen-voor-TLS)
- [toolbox-wiki](https://github.com/internetstandards/toolbox-wiki) - Implementatiegidsen
