# Referentie: Internet.nl Batch API

Achtergrondkennis bij de `$inet-api` skill. Dit document bevat gedetailleerde
API-schema's, rate limits en foutcodes.

## API-versie

De huidige versie is **v2** van de batch API. De documentatie staat in de
[Internet.nl-API-docs](https://github.com/internetstandards/Internet.nl-API-docs) repository.

```bash
# Actuele API-documentatie ophalen
gh api repos/internetstandards/Internet.nl-API-docs/contents/README.md \
  --jq '.content' | base64 -d

# Alle bestanden in de docs-repo
gh api repos/internetstandards/Internet.nl-API-docs/git/trees/main?recursive=1 \
  --jq '.tree[] | select(.type=="blob") | .path'
```

## Authenticatie

De API gebruikt HTTP Basic Authentication.

```
Authorization: Basic base64(gebruikersnaam:wachtwoord)
```

Registratie voor API-toegang via [internet.nl](https://internet.nl).

## Request-formaat

### Batch indienen

**Endpoint:** `POST /api/batch/v2/requests`

**Headers:**
- `Content-Type: application/json`
- `Authorization: Basic <credentials>`

**Body:**

```json
{
  "name": "Beschrijvende naam voor de scan",
  "type": "web",
  "domains": [
    "domein1.nl",
    "domein2.nl"
  ]
}
```

**Velden:**

| Veld | Type | Verplicht | Beschrijving |
|------|------|-----------|-------------|
| `name` | string | Nee | Naam van de batch, max 255 tekens (voor eigen administratie) |
| `type` | string | Ja | Testtype: `"web"` of `"mail"` |
| `domains` | array[string] | Ja | Lijst van te testen domeinen |

### Beperkingen

| Beperking | Waarde |
|-----------|--------|
| Maximaal domeinen per batch | 5.000 |
| Maximaal batch requests per week | 2 |
| Verwerking per gebruiker | Sequentieel (FIFO, 1 actieve batch tegelijk) |

Bron: [Terms of Use](https://github.com/internetstandards/Internet.nl-API-docs/blob/main/terms-of-use.md)

## Response-formaat

### Batch submit response

```json
{
  "success": true,
  "data": {
    "request_id": "abcdef1234567890abcdef1234567890",
    "request_type": "web",
    "name": "Beschrijvende naam",
    "status": "registering",
    "submission_date": "2026-02-22T10:00:00Z",
    "finished_date": null,
    "num_domains": 3
  }
}
```

### Status response

```json
{
  "success": true,
  "data": {
    "request_id": "abcdef1234567890abcdef1234567890",
    "request_type": "web",
    "name": "Beschrijvende naam",
    "status": "running",
    "submission_date": "2026-02-22T10:00:00Z",
    "finished_date": null,
    "num_domains": 3,
    "progress": {
      "total": 3,
      "done": 1,
      "percentage": 33
    }
  }
}
```

### Resultaten response (webtest)

```json
{
  "success": true,
  "data": {
    "request_id": "abcdef1234567890abcdef1234567890",
    "request_type": "web",
    "status": "done",
    "finished_date": "2026-02-22T10:30:00Z",
    "domains": {
      "example.nl": {
        "status": "ok",
        "scoring": {
          "percentage": 92
        },
        "results": {
          "categories": {
            "ipv6": {
              "verdict": "passed",
              "tests": {
                "web_ipv6_ns_address": {"verdict": "passed"},
                "web_ipv6_ns_reach": {"verdict": "passed"},
                "web_ipv6_ws_address": {"verdict": "passed"},
                "web_ipv6_ws_reach": {"verdict": "passed"},
                "web_ipv6_ws_similar": {"verdict": "passed"}
              }
            },
            "dnssec": {
              "verdict": "passed",
              "tests": {
                "web_dnssec_exist": {"verdict": "passed"},
                "web_dnssec_valid": {"verdict": "passed"}
              }
            },
            "tls": {
              "verdict": "passed",
              "tests": {
                "web_tls_https_exists": {"verdict": "passed"},
                "web_tls_https_forced": {"verdict": "passed"},
                "web_tls_https_hsts": {"verdict": "passed"},
                "web_tls_http_redirect": {"verdict": "passed"},
                "web_tls_http_compress": {"verdict": "passed"},
                "web_tls_http_available": {"verdict": "passed"},
                "web_tls_cert_chain": {"verdict": "passed"},
                "web_tls_cert_pubkey": {"verdict": "passed"},
                "web_tls_cert_sig": {"verdict": "passed"},
                "web_tls_cert_domain": {"verdict": "passed"},
                "web_tls_dane_exist": {"verdict": "info"},
                "web_tls_dane_valid": {"verdict": "info"},
                "web_tls_version": {"verdict": "passed"},
                "web_tls_ciphers": {"verdict": "passed"},
                "web_tls_cipher_order": {"verdict": "passed"},
                "web_tls_keyexchange": {"verdict": "passed"},
                "web_tls_compress": {"verdict": "passed"},
                "web_tls_secreneg": {"verdict": "passed"},
                "web_tls_clientreneg": {"verdict": "passed"},
                "web_tls_ocsp": {"verdict": "passed"},
                "web_tls_zero_rtt": {"verdict": "passed"},
                "web_tls_kex_hash_func": {"verdict": "passed"},
                "web_tls_cert_trusted": {"verdict": "passed"}
              }
            },
            "appsecpriv": {
              "verdict": "warning",
              "tests": {
                "web_appsecpriv_csp": {"verdict": "warning"},
                "web_appsecpriv_referrer_policy": {"verdict": "passed"},
                "web_appsecpriv_x_content_type": {"verdict": "passed"},
                "web_appsecpriv_x_frame": {"verdict": "passed"},
                "web_appsecpriv_securitytxt": {"verdict": "passed"}
              }
            },
            "rpki": {
              "verdict": "passed",
              "tests": {
                "web_rpki_exists": {"verdict": "passed"}
              }
            }
          }
        },
        "report": {
          "url": "https://internet.nl/site/example.nl/12345/"
        }
      }
    }
  }
}
```

### Resultaten response (mailtest)

De mailtest bevat andere categorieen:

```json
{
  "categories": {
    "ipv6": {
      "tests": {
        "mail_ipv6_ns_address": {},
        "mail_ipv6_ns_reach": {},
        "mail_ipv6_mx_address": {},
        "mail_ipv6_mx_reach": {}
      }
    },
    "dnssec": {
      "tests": {
        "mail_dnssec_exist": {},
        "mail_dnssec_valid": {},
        "mail_dnssec_mx_exist": {},
        "mail_dnssec_mx_valid": {}
      }
    },
    "auth": {
      "tests": {
        "mail_auth_spf_exist": {},
        "mail_auth_spf_policy": {},
        "mail_auth_dkim_exist": {},
        "mail_auth_dmarc_exist": {},
        "mail_auth_dmarc_policy": {}
      }
    },
    "tls": {
      "tests": {
        "mail_tls_starttls_exist": {},
        "mail_tls_version": {},
        "mail_tls_ciphers": {},
        "mail_tls_cert_chain": {},
        "mail_tls_cert_pubkey": {},
        "mail_tls_cert_sig": {},
        "mail_tls_cert_domain": {},
        "mail_tls_dane_exist": {},
        "mail_tls_dane_valid": {},
        "mail_tls_dane_rollover": {}
      }
    },
    "rpki": {
      "tests": {
        "mail_rpki_exists": {}
      }
    }
  }
}
```

## Foutcodes

| HTTP-code | Betekenis | Actie |
|-----------|-----------|-------|
| 200 | Succes | Verwerk de response |
| 400 | Ongeldige request | Controleer JSON-formaat en verplichte velden |
| 401 | Niet geautoriseerd | Controleer credentials |
| 403 | Geen toegang | Neem contact op met internet.nl |
| 404 | Batch niet gevonden | Controleer het request-ID |
| 429 | Rate limit bereikt | Wacht en probeer later opnieuw |
| 500 | Serverfout | Probeer later opnieuw, meld bij aanhoudend probleem |

## Error response

```json
{
  "success": false,
  "error": {
    "code": "invalid_domains",
    "message": "One or more domains are invalid",
    "details": {
      "invalid_domains": ["niet-bestaand..nl"]
    }
  }
}
```

## Geavanceerd Python-voorbeeld met foutafhandeling

```python
"""Internet.nl batch API client met uitgebreide foutafhandeling."""

import json
import logging
import time
from pathlib import Path

import requests

logger = logging.getLogger(__name__)


class InternetNLClient:
    """Client voor de internet.nl batch API."""

    def __init__(self, api_url: str, username: str, password: str):
        self.api_url = api_url.rstrip("/")
        self.auth = (username, password)
        self.session = requests.Session()
        self.session.auth = self.auth

    def submit_web(self, name: str, domains: list[str]) -> str:
        """Dien een webtest batch in."""
        return self._submit("web", name, domains)

    def submit_mail(self, name: str, domains: list[str]) -> str:
        """Dien een mailtest batch in."""
        return self._submit("mail", name, domains)

    def _submit(self, test_type: str, name: str, domains: list[str]) -> str:
        """Generieke batch submit."""
        url = f"{self.api_url}/api/batch/v2/requests"
        response = self.session.post(
            url,
            json={"name": name, "type": test_type, "domains": domains},
            timeout=30,
        )
        response.raise_for_status()
        data = response.json()
        if not data.get("success"):
            raise RuntimeError(f"API fout: {data.get('error', {}).get('message', 'Onbekend')}")
        return data["data"]["request_id"]

    def get_status(self, request_id: str) -> dict:
        """Haal status op (eenmalig)."""
        url = f"{self.api_url}/api/batch/v2/requests/{request_id}"
        response = self.session.get(url, timeout=30)
        response.raise_for_status()
        return response.json()

    def get_results(self, request_id: str) -> dict:
        """Haal resultaten op (als status done is)."""
        url = f"{self.api_url}/api/batch/v2/requests/{request_id}/results"
        response = self.session.get(url, timeout=30)
        response.raise_for_status()
        return response.json()

    def wait_for_results(
        self,
        request_id: str,
        interval: int = 30,
        max_wait: int = 3600,
    ) -> dict:
        """Poll tot resultaten beschikbaar zijn."""
        start = time.time()
        while time.time() - start < max_wait:
            data = self.get_status(request_id)
            status = data["data"]["status"]
            progress = data["data"].get("progress", {})
            logger.info(
                "Status: %s (%s%%)",
                status,
                progress.get("percentage", "?"),
            )

            if status == "done":
                return self.get_results(request_id)
            if status in ("error", "cancelled"):
                raise RuntimeError(f"Batch mislukt: {status}")

            time.sleep(interval)

        raise TimeoutError(f"Resultaten niet beschikbaar na {max_wait}s")

    def save_results(self, results: dict, output_path: str) -> None:
        """Sla resultaten op als JSON."""
        Path(output_path).write_text(
            json.dumps(results, indent=2, ensure_ascii=False),
            encoding="utf-8",
        )
        logger.info("Resultaten opgeslagen in %s", output_path)
```

## TLS-velddetails sinds v1.11.0 (API v2.7.0)

De v1.11.0 release (NCSC TLS-richtlijnen 2025-05) wijzigt enkele TLS-velden in de batch-resultaten:

| Veld | Wijziging |
|------|-----------|
| `web_tls_clientreneg` / `mail_tls_clientreneg` | Status was boolean, is nu een enum: `not_allowed` (good), `allowed_with_low_limit` (info, < 10 renegotiations), `allowed_with_too_high_limit` (failed) |
| `web_tls_ocsp` / `mail_tls_ocsp` | Nieuwe status `not_in_cert` (not_tested) wanneer het certificaat geen OCSP-endpoint bevat; stapling is dan technisch onmogelijk en wordt niet als fout gerekend |
| `web_tls_extended_master_secret` / `mail_tls_extended_master_secret` | Nieuw veld (RFC 7627). Statussen: `supported` (good), `not_supported` (failed), `na_no_tls_1_2` (good, alleen TLS 1.3 onderhandeld), `unknown` (not_tested) |
| `web_tls_cipher_order` | Status `not_prescribed` en `not_seclevel` zijn vervallen; verkeerde voorkeur of geen voorkeur is nu allebei `bad` |
| `cert_signature_phase_out` | Nieuw veld in TLS-details, naast bestaande `cert_signature_bad`. Lijst met certificaat-signature-algoritmes op phase-out niveau (warning) |

Een client die `clientreneg` als boolean parseert, breekt op v2.7.0. Pas de parser aan op string-enum.

## Bronnen

- [Internet.nl-API-docs](https://github.com/internetstandards/Internet.nl-API-docs) - Officieel API-documentatie
- [Internet.nl](https://github.com/internetstandards/Internet.nl) - Testsuite broncode
- [internet.nl](https://internet.nl) - Website en API-registratie
- [Internet.nl v1.11.0 release notes](https://github.com/internetstandards/Internet.nl/releases/tag/v1.11.0)
