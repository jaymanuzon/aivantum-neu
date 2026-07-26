# Worker-Patch — `art: "aivantum"` freischalten

**Datei:** `Productization HQ/Productization HQ Resources/check-aivantum-worker/`
**Status:** vorbereitet, NICHT deployt (Worker-Deploy = externer Endpunkt = Jays Go).
**Warum nötig:** Am 26.07. live gegen den Endpunkt gemessen — die Seite sendet `art:"aivantum"` von `aivantum.com`. Beides kennt der Worker heute nicht: der Preflight antwortet mit `Access-Control-Allow-Origin: https://check.aivantum.com`, die Anfrage wird von CORS blockiert (Konsole belegt). Das Frontend fängt das sauber ab und zeigt Mail + Kalender als Ausweg — aber es kommt keine Anfrage an.

## 1. `wrangler.toml` — Origin ergänzen

In `ALLOWED_ORIGIN` hinten anhängen (kommagetrennt, keine Leerzeichen):

```
,https://aivantum.com,https://www.aivantum.com
```

## 2. `src/worker.js` — drei Stellen

**(a) Zeile ~225, `art`-Allowlist:** `"aivantum"` in das Array aufnehmen.

```js
const art = ["tomodachi","tomo","oldschool","ciao","restart","shoshin","aiact","itr","coachjay","aivantum"].includes(body.art) ? body.art : null;
```

**(b) Zeile ~241, `source`-Default:** eine Zeile vor dem `coachjay`-Zweig.

```js
: art === "aivantum" ? "aivantum.com"
```

**(c) Zeile ~301, Handler:** neuer `else if`-Zweig neben `coachjay`.

```js
} else if (art === "aivantum") {
  // aivantum.com — Anfrage von der Haupt-Landingpage.
  // thema = Auswahl aus dem Dropdown (kann leer sein), firma = Organisation.
  // Der Freitext traegt das Thema bereits im Kopf (Frontend stellt es voran).
  const firma = String(body.firma || "").slice(0, 120);
  const thema = String(body.thema || "").slice(0, 80);
  const msg   = String(body.msg   || "").slice(0, 2000);
  confirmP = sendAivantumConfirm({ email, name }, env);
  leadP    = sendAivantumLead({ email, name, firma, thema, msg, source }, env);
}
```

**(d) Zwei Funktionen** — Duplikate von `sendCoachjayConfirm` / `sendCoachjayLead`, angepasst auf Du-Ansprache, Teal-Akzent `#1a6259` und deterministischen Betreff für den Gmail-Filter:

```js
/* aivantum: Bestaetigung an den Anfragenden (Du-Form, DSGVO-Hinweis) */
async function sendAivantumConfirm(p, env) {
  const vorname = p.name ? escapeHtml(p.name.split(" ")[0]) : "";
  const html = `
    <div style="font-family:Georgia,'Times New Roman',serif;color:#16233c;max-width:560px;margin:0 auto;line-height:1.65">
      <p style="font-family:Helvetica,Arial,sans-serif;font-size:.78rem;letter-spacing:.22em;text-transform:uppercase;color:#1a6259;font-weight:700">AIVantum</p>
      <h2 style="margin:.2em 0 .5em;font-weight:500;font-size:1.7rem">Deine Anfrage ist angekommen.</h2>
      <p>${vorname ? "Hallo " + vorname + ", " : "Hallo, "}danke fuer deine Nachricht. Ich melde mich persoenlich &mdash; in der Regel innerhalb von zwei Werktagen.</p>
      <p>Wenn es schneller gehen soll, kannst du dir auch direkt 30 Minuten in meinem Kalender nehmen: <a href="https://calendar.app.google/VPussJNynjUDuBJFA" style="color:#1a6259">Termin auswaehlen</a>. Kein Pitch, keine Folien &mdash; ich frage, du erzaehlst.</p>
      <p style="font-family:Helvetica,Arial,sans-serif;color:#5a6580;font-size:.84rem;margin-top:22px">Du erhaeltst diese Mail, weil auf aivantum.com das Anfrage-Formular mit deiner Adresse abgeschickt wurde. Deine Angaben nutze ich ausschliesslich fuer die Rueckmeldung und gebe sie nicht weiter. Fragen oder Widerruf jederzeit an <a href="mailto:jay@aivantum.com" style="color:#1a6259">jay@aivantum.com</a>.</p>
    </div>`;
  const resp = await fetch("https://api.resend.com/emails", {
    method: "POST",
    headers: { "content-type": "application/json", authorization: `Bearer ${env.RESEND_API_KEY}` },
    body: JSON.stringify({
      from: env.FROM_EMAIL || "onboarding@resend.dev",
      to: [p.email], reply_to: env.NOTIFY_EMAIL,
      subject: "Deine Anfrage bei AIVantum ist angekommen", html,
    }),
  });
  if (!resp.ok) throw new Error("resend_aivantum_user_" + resp.status);
  return true;
}

/* aivantum: Lead-Mail an Jay — Betreff deterministisch (Gmail-Filter: subject:AIVantum-Anfrage) */
async function sendAivantumLead(p, env) {
  const html = `
    <div style="font-family:Helvetica,Arial,sans-serif;color:#16233c">
      <h2>Neue Anfrage ueber aivantum.com</h2>
      <p><strong>Name:</strong> ${escapeHtml(p.name || "—")}<br>
         <strong>E-Mail:</strong> ${escapeHtml(p.email)}<br>
         <strong>Organisation:</strong> ${escapeHtml(p.firma || "—")}<br>
         <strong>Thema:</strong> ${escapeHtml(p.thema || "—")}<br>
         <strong>Quelle:</strong> ${escapeHtml(p.source)}</p>
      <p><strong>Anliegen:</strong></p>
      <div style="background:#f2ecdf;padding:14px;border-radius:8px;white-space:pre-wrap">${escapeHtml(p.msg || "—")}</div>
    </div>`;
  const resp = await fetch("https://api.resend.com/emails", {
    method: "POST",
    headers: { "content-type": "application/json", authorization: `Bearer ${env.RESEND_API_KEY}` },
    body: JSON.stringify({
      from: env.FROM_EMAIL || "onboarding@resend.dev",
      to: [env.NOTIFY_EMAIL], reply_to: p.email,
      subject: `AIVantum-Anfrage: ${p.thema || "allgemein"} — ${p.name || p.email}`, html,
    }),
  });
  if (!resp.ok) throw new Error("resend_aivantum_lead_" + resp.status);
  return true;
}
```

## 3. Deploy + Abnahme (nach Jays Go)

```bash
cd "Productization HQ/Productization HQ Resources/check-aivantum-worker"
npx wrangler deploy
```

Danach **am Endpunkt** messen, nicht am Gefuehl — und der Zustellbeweis nur mit **fremdem Absender** (eigene Adresse beweist nichts ueber Spam-Einstufung):

```bash
curl -sS -X OPTIONS https://aivantum-check.aivantum.workers.dev/kontakt \
  -H "Origin: https://aivantum.com" -H "Access-Control-Request-Method: POST" -D- -o /dev/null | grep -i access-control-allow-origin
```

Erwartung: der Header traegt `https://aivantum.com`. Erst danach einen echten Formular-Absendetest von der Live-Domain.
