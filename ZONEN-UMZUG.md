# Runbook — aivantum.com von Squarespace zu DomainFactory

**Stand:** 2026-07-26 · **Anlass:** Website-Abo gekündigt, endet ~04.05.2027 · **Status:** vorbereitet, nichts ausgeführt
**Warum überhaupt:** Nicht die Domain ist das Problem (eigener Posten, 18 €, Auto-Verlängerung aktiv), sondern die **DNS-Zone**. Sie liegt auf `nsb1–4.squarespacedns.com`. Daran hängen **18 Subdomains** und der **komplette Mailversand**. Ob Squarespace das Panel für eine Domain ohne Website-Abo weiterführt, ist nicht gemessen — und diese Annahme trägt jetzt ein Datum.

**Der Ausweg ist bequemer als gedacht:** Jay ist seit rund 20 Jahren bei DomainFactory und hat dort bereits `coachjay.de` **und** `aivantum.de`. `coachjay.de` läuft dort nachweislich auf GitHub Pages (A-Records, seit 25.07.). Das Muster ist also erprobt, nur eben für eine andere Domain.

---

## Die Reihenfolge, die nichts kaputt macht

Der Fehler wäre, die Nameserver umzustellen und die Zone danach zu bauen. Dazwischen lägen Stunden ohne Subdomains und ohne Mail. Deshalb: **Zone zuerst, Umschaltung zuletzt.**

### 1. Zone bei DomainFactory anlegen — Squarespace bleibt unangetastet

Alle Einträge unten in der DomainFactory-Verwaltung für `aivantum.com` anlegen. Solange die Nameserver noch auf Squarespace zeigen, hat das **keinerlei Wirkung nach außen** — es ist ein stiller Zwilling.

### 2. Den Zwilling messen, bevor jemand ihn benutzt

Direkt gegen DomainFactorys Nameserver fragen, ohne die Zone offiziell zu aktivieren:

```bash
for h in @ www neu check truth tomo itr ai-act shoshin ready ciao restart \
         gallery dashboard tomodachi yaruki recall oldschool hikidashi zanshin; do
  n=$([ "$h" = "@" ] && echo aivantum.com || echo $h.aivantum.com)
  printf '%-12s %s\n' "$h" "$(dig +short $n CNAME @ns67.domaincontrol.com; dig +short $n A @ns67.domaincontrol.com | tr '\n' ' ')"
done
dig +short aivantum.com MX  @ns67.domaincontrol.com
dig +short aivantum.com TXT @ns67.domaincontrol.com
dig +short k1._domainkey.aivantum.com     TXT @ns67.domaincontrol.com
dig +short resend._domainkey.aivantum.com TXT @ns67.domaincontrol.com
dig +short litesrv._domainkey.aivantum.com CNAME @ns67.domaincontrol.com
dig +short _dmarc.aivantum.com TXT @ns67.domaincontrol.com
```

**Erst wenn diese Ausgabe der Tabelle unten entspricht, geht es weiter.** Ein fehlender DKIM-Eintrag fällt sonst erst auf, wenn Mails im Spam landen.

### 3. Nameserver umstellen (der eigentliche Umschalt-Moment)

Bei Squarespace unter `Domains → aivantum.com` die Nameserver auf DomainFactory ändern. Ab jetzt beantwortet DomainFactory die Zone. Weil der Zwilling schon steht, merkt niemand etwas.

Danach gegen **fremde** Resolver messen, nicht gegen den eigenen:

```bash
dig +short aivantum.com NS @1.1.1.1
dig +short neu.aivantum.com @1.1.1.1
dig +short aivantum.com MX @8.8.8.8
```

### 4. Registrierung übertragen (optional, in Ruhe)

Bei Squarespace: **Domainsperre lösen** (steht aktuell auf An — `clientTransferProhibited`), Übertragungscode anfordern, bei DomainFactory den Transfer starten. Dauert üblicherweise fünf bis sieben Tage und verlängert die Domain in der Regel um ein Jahr. Danach ist Squarespace vollständig aus dem Spiel.

**Nicht nötig für die Sicherheit** — nach Schritt 3 hängt nichts Betriebliches mehr an Squarespace. Schritt 4 ist Aufräumen.

### 5. Apex und www auf die neue Seite legen

Sinnvollerweise gleich mit Schritt 1 mitgebaut, aber erst nach dem Umschalten wirksam:

| Art | Name | Wert | statt bisher |
| :-- | :-- | :-- | :-- |
| A | `@` | `185.199.108.153` · `.109.153` · `.110.153` · `.111.153` | 4 × Squarespace-IP |
| CNAME | `www` | `jaymanuzon.github.io.` | `ext-sq.squarespace.com.` |

Dazu im Repo `jaymanuzon/aivantum-neu` die `CNAME`-Datei auf `aivantum.com` ändern, danach Zertifikat abwarten. **Achtung:** GitHub Pages bedient pro Repo genau eine Custom-Domain — `neu.aivantum.com` fällt damit weg beziehungsweise wird zur Weiterleitung.

---

## Die Zone, gemessen am 2026-07-26

Quelle: `dig` gegen `nsb1.squarespacedns.com` (autoritativ). Nichts hiervon ist geraten.

### Subdomains → GitHub Pages (18 Stück, alle CNAME auf `jaymanuzon.github.io.`)

`neu` · `check` · `truth` · `tomo` · `itr` · `ai-act` · `shoshin` · `ready` · `ciao` · `restart` · `gallery` · `dashboard` · `tomodachi` · `yaruki` · `recall` · `oldschool` · `hikidashi` · `zanshin`

### Apex und www (Stand heute — bei Schritt 5 ersetzen)

| Art | Name | Wert |
| :-- | :-- | :-- |
| A | `@` | `198.185.159.144` · `198.185.159.145` · `198.49.23.144` · `198.49.23.145` |
| CNAME | `www` | `ext-sq.squarespace.com.` |

### Mail — der Teil, bei dem ein Fehler richtig weh tut

| Art | Name | Wert |
| :-- | :-- | :-- |
| MX | `@` | `10 mxa.mailgun.org.` |
| MX | `@` | `10 mxb.mailgun.org.` |
| TXT | `@` | `v=spf1 include:mailgun.org a include:_spf.mlsend.com mx ~all` |
| TXT | `@` | `mailerlite-domain-verification=f5cbbdf712cc2b497d88006e6d3990e95973b2fb` |
| TXT | `k1._domainkey` | `k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDNTiNhaxeBbpm1O2euuvKkXI/t/6/SC618uDW2us/FBVl/oykKr3FaKI4EYMyzAsnW81U0mFsVXZb6YDXGewrrkz6HC9msHApA28SlApRME6smZRRyKvTlmr6ij5fkDT0zeZd34T85Un8/fVFTSq+r2lRV6CHkw21x6A90PQjSHwIDAQAB` |
| TXT | `resend._domainkey` | `p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDFXMs+T8FGAZJB1AJgg26YjLJfBuuqABOwHTzz5Jop6vC/NqyHf5Q2Ov+gsS+4kcxVBU2fuT+ORevoMLqIkigFbiNe9sl3iZ0xr8iBIrCK0MgWuAckLgVUrL9d7ZFVVtH3qopnApuma+MHcKzBpzORyEiHkcjjpPJLskizlYwVgQIDAQAB` |
| CNAME | `litesrv._domainkey` | `litesrv._domainkey.mlsend.com.` |
| TXT | `_dmarc` | `v=DMARC1; p=reject; pct=100; fo=1; ri=3600; sp=reject; adkim=s; aspf=s; rua=mailto:32a1088@dmarc.mailgun.org,mailto:7a82e45f@inbox.ondmarc.com; ruf=mailto:32a1088@dmarc.mailgun.org,mailto:7a82e45f@inbox.ondmarc.com;` |

> **DMARC steht auf `p=reject`.** Das heißt: Stimmt SPF oder DKIM nach dem Umzug nicht, werden Mails nicht etwa markiert, sondern **abgewiesen**. Die drei DKIM-Einträge und der SPF gehören deshalb zeichengenau übernommen — ein abgeschnittener Schlüssel ist der wahrscheinlichste Fehler beim Kopieren.

---

## Was mit aivantum.de ist

Sie liegt bereits bei DomainFactory, ist aber **ungenutzt**: `A` zeigt auf `80.67.16.8`, den DomainFactory-Platzhalter, ein Aufruf liefert eine 302 ins Leere, HTTPS antwortet gar nicht. Genau der Zustand, in dem `coachjay.de` vor dem 25.07. war.

Sie ist kein Ersatz für die `.com` — daran hängen 18 Subdomains und die gesamte Mail-Zustellung, ein Wechsel der Hauptdomain würde all das brechen. Sinnvoll wäre sie später als deutsche Zweitadresse, die auf die `.com` weiterleitet. Eigener Vorgang, nicht Teil dieses Umzugs.


---

## Nachtrag 02.09.2026 02:40 — Zone frisch aus dem Squarespace-Panel gelesen (Jay eingeloggt, Tomo read-only)

**Zwei Prämissen geprüft:** (1) Das Squarespace-Domain-Panel ist **trotz gekündigtem Website-Abo voll zugänglich** (Domains → aivantum.com → DNS-Einstellungen, Verlängerung 04.05.2027 für 18 €, Auto-Renew AN, Domainsperre AN). (2) **aivantum.com liegt NICHT bei DomainFactory** (dort nur coachjay.de + aivantum.de); DF nähme sie nur als kostenpflichtige „externe Domain" (4-Schritt-Bestellung, abgebrochen bei Schritt 2). → Zwilling entweder DF-extern (Gebühr) oder **Cloudflare** (Konto vorhanden, gratis, Scan-Import) — Jays Entscheid am Wachtag.

**Zone, Stand 02.09. (Panel-Scan per DOM + zwei Werte per dig @nsb1.squarespacedns.com):**

```
MX | @ | 10 | mxa.mailgun.org
MX | @ | 10 | mxb.mailgun.org
TXT | @ | v=spf1 include:mailgun.org ~all
TXT | _dmarc | (im Panel-Scan geblockt, per dig:) "v=DMARC1; p=reject; pct=100; fo=1; ri=3600; sp=reject; adkim=s; aspf=s; rua=mailto:32a1088@dmarc.mailgun.org,mailto:7a82e45f@inbox.ondmarc.com; ruf=mailto:32a1088@dmarc.mailgun.org,mailto:7a82e45f@inbox.ondmarc.com;"
TXT | k1._domainkey | (im Panel-Scan geblockt, per dig:) "k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDNTiNhaxeBbpm1O2euuvKkXI/t/6/SC618uDW2us/FBVl/oykKr3FaKI4EYMyzAsnW81U0mFsVXZb6YDXGewrrkz6HC9msHApA28SlApRME6smZRRyKvTlmr6ij5fkDT0zeZd34T85Un8/fVFTSq+r2lRV6CHkw21x6A90PQjSHwIDAQAB"
A | @ | 198.185.159.144 · 198.185.159.145 · 198.49.23.144 · 198.49.23.145
CNAME | www | ext-sq.squarespace.com
HTTPS | @ | 1 . alpn="h2,http/1.1" ipv4hint="198.185.159.144,198.185.159.145,198.49.23.144,198.49.23.145"
CNAME | _domainconnect | _domainconnect.domains.squarespace.com
CNAME | litesrv._domainkey | litesrv._domainkey.mlsend.com
TXT | @ | mailerlite-domain-verification=f5cbbdf712cc2b497d88006e6d3990e95973b2fb
CNAME × 21 → jaymanuzon.github.io | ai-act · check · ciao · dashboard · family · gallery · hikidashi · itr · neu · okay · oldschool · ready · recall · restart · shoshin · tomo · tomodachi · truth · waechter · yaruki
MX | send | 10 | feedback-smtp.eu-west-1.amazonses.com
TXT | send | v=spf1 include:amazonses.com ~all
TXT | _github-pages-challenge-jaymanuzon | a319373d0f3b5262824f0bd6eb8b76
TXT | resend._domainkey | p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDFXMs+T8FGAZJB1AJgg26YjLJfBuuqABOwHTzz5Jop6vC/NqyHf5Q2Ov+gsS+4kcxVBU2fuT+ORevoMLqIkigFbiNe9sl3iZ0xr8iBIrCK0MgWuAckLgVUrL9d7ZFVVtH3qopnApuma+MHcKzBpzORyEiHkcjjpPJLskizlYwVgQIDAQAB
```

**Diff gegen die Messung vom 26.07.:**
- NEU seit 26.07.: CNAMEs `family`, `okay`, `waechter` (jetzt **20** GitHub-Pages-Subdomains statt 18) · `send` MX + SPF (Resend über Amazon SES) · `_github-pages-challenge-jaymanuzon` TXT · `_domainconnect` CNAME (Squarespace-intern, beim Umzug NICHT mitnehmen) · `HTTPS`-Record am Apex (Squarespace-intern, entfällt mit Schritt 5).
- WEG seit 26.07.: `zanshin` (der dangling CNAME aus dem Fußabdruck-Audit ist raus — dig bestätigt: leer).
- SPF am Apex im Panel: `v=spf1 include:mailgun.org ~all` — **ohne** `include:_spf.mlsend.com a mx`, das die 26.07.-Messung zeigte. Per dig heute: `"v=spf1 a include:_spf.mlsend.com include:mailgun.org mx ~all" ‖ "mailerlite-domain-verification=f5cbbdf712cc2b497d88006e6d3990e95973b2fb"`. **Aufgelöst per dig:** autoritativ gilt weiterhin `v=spf1 a include:_spf.mlsend.com include:mailgun.org mx ~all` — die Panel-Anzeige ist nur die verkürzte „Voreinstellungs"-Darstellung. Der Zwilling nimmt den dig-Wert.
- DMARC bleibt `p=reject` → DKIM (k1, resend, litesrv) zeichengenau übernehmen.

Der Zwilling wird aus DIESER Liste gebaut, nicht aus der vom 26.07.

### Zwilling als Datei + Morgen-Checkliste (02.09. 02:50, Nachtlauf)

`aivantum.com.zwilling.zone` (neben diesem Runbook) ist die komplette Zone im BIND-Format, DKIM/DMARC gegen den autoritativen Nameserver gegengeprüft. **Squarespace-Nameserver-Seite gemessen:** Domains → aivantum.com → DNS → Domain-Nameserver → Schalter „Benutzerdefinierte Nameserver verwenden" — dort werden die Nameserver des neuen Anbieters eingetragen. Das ist der eine Klick des Umzugs.

**Jays drei Handgriffe am Wachtag (Empfehlung Cloudflare):**
1. Cloudflare in Chrome einloggen, „Add a site" → aivantum.com → Free-Plan. Cloudflare scannt die Zone selbst; ich vergleiche den Scan mit der Datei und importiere fehlende Einträge (Zwilling steht, nach außen passiert nichts). Alternative: DomainFactory „Externe Domain freischalten" (kostenpflichtig, 4-Schritt-Bestellung).
2. Ich messe den Zwilling gegen die neuen Nameserver (alle 20 CNAMEs, MX, 3× DKIM, DMARC, SPF). Erst bei „identisch" geht es weiter.
3. Jay trägt bei Squarespace die zwei neuen Nameserver ein (Schalter oben). Danach messe ich gegen 1.1.1.1 und 8.8.8.8, dann Schritt 5 (Apex/www auf GitHub Pages + CNAME-Datei im Repo).
