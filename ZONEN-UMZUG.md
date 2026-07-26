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
