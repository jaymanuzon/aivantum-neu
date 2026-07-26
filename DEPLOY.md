# Deploy-Runbook — aivantum-site (Haupt-Landingpage)

**Stand:** 2026-07-26 · **Status:** 🟡 GEBAUT + DEPLOY-BEREIT — es fehlt genau ein Go für zwei ausgehende Aktionen (Pages-Push, Worker-Deploy)
**Muster:** wie `coachjay-site` (GitHub Pages + DNS beim Registrar) und die `*.aivantum.com`-Subdomains
**Design-Brief / Begründung:** [`02_Documents_produced/Design-Briefs_2026-07-26/aivantum-landingpage-brief.md`](../../../02_Documents_produced/Design-Briefs_2026-07-26/aivantum-landingpage-brief.md)

> **Nichts hiervon ist vollzogen.** Publizieren und Worker-Deploy sind ausgehende Aktionen und damit hartes Tor — sie brauchen Jays Go pro Vorgang, auch wenn inhaltlich alles entschieden ist. Alles andere ist vorbereitet.

---

## Was lokal fertig liegt (gemessen 26.07.)

| Stück | Zustand |
| :-- | :-- |
| `index.html` | Landingpage, 11 Sektionen, ~55 KB · Render-Gate exit 0 · Mobile-Gate `scrollW=390 bad=[]` |
| `impressum.html` · `datenschutz.html` | gegen die **tatsächliche** Funktion geschrieben: Formular ja, Google-Kalender als Link (nicht eingebettet), Schriften lokal, kein Tracking, kein Cookie, keine KI auf der Seite |
| `assets/portrait-jay.jpg` | 1050×1400, aus der coachjay-Site übernommen |
| `assets/aivantum-lockup-creme.png` · `-tinte.png` | Wortbildmarke, aus Jays Gemini-Quelle freigestellt (Monogramm + Wortmarke als horizontale Sperrmarke). Zwei Farbfassungen, weil die Kopfleiste beim Scrollen von Dunkel auf Creme kippt — Umblendung per Deckkraft, ohne JavaScript |
| `assets/favicon.png` | aus demselben Monogramm, auf Creme-Plättchen |
| `assets/aivantum-logo-horizontal.png` · `-icon-monogram.png` | **derzeit ungenutzt** (Vorgänger des Lockups). Bewusst nicht gelöscht — Löschen ist Jays Tor; können vor dem Deploy raus |
| `fonts/` | 6 × woff2 (Cormorant Garamond + Hanken Grotesk), SIL OFL 1.1, lokal — **keine Verbindung zu Google Fonts** |
| `.nojekyll` | gesetzt |
| `CNAME` | `neu.aivantum.com` — Weg A gesetzt, folgt aus dem Squarespace-Entscheid vom 25.07. |
| Externe Hosts | genau drei: `calendar.app.google` (Link), `aivantum-check.aivantum.workers.dev` (Formular), `ai-act.aivantum.com` (Verweis) |
| No-JS | 12.368 Zeichen Kerninhalt ohne JavaScript lesbar |
| `_patch/worker-aivantum.patch.md` | fertiger Worker-Diff — **ohne ihn kommt keine Formular-Anfrage an** |

---

## Entscheidung 1 (vor allem anderen): Wohin?

| Weg | Was es bedeutet | Risiko |
| :-- | :-- | :-- |
| **A · Zweitfläche** `neu.aivantum.com` (Empfehlung) | Neue Subdomain per CNAME, Squarespace bleibt unberührt auf dem Apex. Du kannst beide nebeneinander sehen, Leute draufschicken, Feedback sammeln. Umschalten später ist ein DNS-Handgriff. | praktisch keins — Rückweg = Subdomain löschen |
| **B · Apex ersetzen** `aivantum.com` | A-Records vom Squarespace-Server auf GitHub Pages. Squarespace ist damit faktisch offline, obwohl bis 04.05. bezahlt. | Rückweg dauert eine TTL-Runde; und der Squarespace-Shop/Kalender wären weg, bevor du sie ersetzt hast |

**Weg A ist gesetzt (26.07.).** Er folgt direkt aus deinem Entscheid vom 25.07. („Squarespace ausreizen, Umzug erst mit Kundenerfahrung") — die neue Seite laeuft daneben, Squarespace bleibt unberuehrt auf dem Apex. Die `CNAME`-Datei steht entsprechend auf `neu.aivantum.com`. Umschalten auf den Apex bleibt spaeter ein DNS-Handgriff; dafuer liegt Weg B unten weiter beschrieben.

---

## Die Schritte (wenn Jay Go sagt)

### 1. Repo + Push

```bash
gh repo create jaymanuzon/aivantum-site --public \
  --source="Vantum HQ/Vantum HQ Resources/aivantum-site" --push
```

Die `CNAME`-Datei liegt bereits mit `neu.aivantum.com` (Weg A). Nur falls du doch direkt auf den Apex willst:

```bash
echo "aivantum.com" > "Vantum HQ/Vantum HQ Resources/aivantum-site/CNAME"   # Weg B
```

### 2. GitHub Pages

Repo → **Settings → Pages** → Source `main` / `/ (root)` → Custom domain eintragen → speichern → **Enforce HTTPS**, sobald das Zertifikat steht.

### 3. DNS beim Registrar

**Weg A** (Subdomain — Apex bleibt bei Squarespace):

| Typ | Name | Wert |
| :-- | :-- | :-- |
| CNAME | `neu` | `jaymanuzon.github.io.` |

**Weg B** (Apex — Apex kann kein CNAME, es braucht A-Records):

| Typ | Name | Wert |
| :-- | :-- | :-- |
| A | `@` | `185.199.108.153` |
| A | `@` | `185.199.109.153` |
| A | `@` | `185.199.110.153` |
| A | `@` | `185.199.111.153` |
| CNAME | `www` | `jaymanuzon.github.io.` |

**MX nicht anfassen** — die steuern das Postfach. Bei Weg B müssen die alten Squarespace-A-Records vorher weg, sonst gewinnen sie.

### 4. Worker freischalten

`_patch/worker-aivantum.patch.md` anwenden, dann `npx wrangler deploy`. Ohne diesen Schritt läuft jede Formular-Anfrage in den CORS-Block — das Frontend fängt es ab und zeigt Mail und Kalender, aber es kommt nichts an.

### 5. Abnahme am Endpunkt, nicht am Gefühl

```bash
dig +short <domain>                                   # löst die Domain überhaupt auf?
dig +short <domain> @1.1.1.1                          # und bei einem fremden Resolver auch?
curl -sSI https://<domain> | head -3                  # HTTP 200, kein Redirect ins Leere
curl -s https://<domain> | grep -c "Anders arbeiten"  # Inhalt wirklich ausgeliefert
curl -sS -X OPTIONS https://aivantum-check.aivantum.workers.dev/kontakt \
  -H "Origin: https://<domain>" -H "Access-Control-Request-Method: POST" \
  -D- -o /dev/null | grep -i access-control-allow-origin
bash "00_Resources/mobile-390-probe.sh" <lokale-kopie>
```

> **`curl --resolve` ist hier verboten.** Es setzt die Ziel-IP von Hand und umgeht die DNS-Auflösung komplett — es beweist, dass GitHub ausliefert, nicht dass ein Besucher ankommt. Genau daran ist die coachjay-Abnahme am 25.07. vorbeigelaufen (Fritz!Box hielt den alten Record). Auflösen lassen, gegen einen fremden öffentlichen Resolver gegenprüfen, den lokalen Resolver getrennt ausweisen.

Danach im echten Browser: Portrait lädt, Schriften sind Cormorant/Hanken (nicht Georgia), Selbst-Check zeigt ein Ergebnis, Formular sendet wirklich — **Zustellbeweis nur mit fremdem Absender**, die eigene Adresse sagt nichts über die Spam-Einstufung.

### 6. Nachziehen

- Fläche in `00_Resources/flaechen-register.json` aufnehmen, dann `flaechen-register-build.py` (HTTP-Probe = die Landungs-Messung)
- Lagezentrum-Faden `go-live-paket-website-linkedin-offnen` + `squarespace-jetzt-ausreizen-umzug-mit-daten` nachziehen
- Externer-Endpunkt-Logbuch: was ist am Endpunkt jetzt anders (DNS, Pages, Worker)

---

## Bewusst offen gelassen

| Punkt | Warum |
| :-- | :-- |
| Anwaltliche Abnahme der Rechtstexte | Entwurfsstand mit Banner — läuft ohnehin gebündelt mit coachjay |
| Zweites Bild von Jay | braucht ein **echtes** Foto aus einem echten Termin — ein generiertes Arbeitsbild waere eine Behauptung ueber ihn. Begruendung im Bild-Prompt |
| Weiterleitungen alter Squarespace-URLs | erst relevant bei Weg B; die 14 alten Pfade müssten dann auf die Landingpage-Anker zeigen |
