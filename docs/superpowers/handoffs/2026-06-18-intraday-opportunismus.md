# Handoff — 2026-06-18: Intraday-Opportunismus (B4) komplett + ausstehende Tasks

Dieser PR (**lmuenet/ape-signal#2**, Branch `feat/intraday-opportunismus-staffelung`) setzt
**Finding D / Backlog B4** in drei Stufen um. Code ist fertig, getestet (**433 grün**, `tsc`
sauber) und zweifach adversarial reviewt. Dieses Handoff beschreibt, **wie der PR temporär
getestet** wird, und listet die **ausstehenden Tasks** für eine eigene Folge-Session (die
wieder mit einem PR endet).

Beschluss & Begründung der Staffelung:
`docs/superpowers/brainstorms/2026-06-18-intraday-opportunismus.md`.

---

## Was dieser PR liefert

- **Stufe 1 — geduldige Order-Geometrie (null neue LLM-Calls):** Limit-Leiter (`rungGroup` +
  deterministisches Mutual-Cancel), Multi-Day-TTL (`ttlDays`→`expiresOn`), Prompt bevorzugt
  gestaffelte Limits statt Market. Löst Finding B (Late-Fill ~16:34) mit.
- **Stufe 2 — Setup-Radar (deterministisch, null LLM, kein Auto-Trade):** Kür seedet eine
  Tages-Watchlist aus nicht gehandelten Dossier-Kandidaten; jeder Monitor-Tick erkennt
  close-basierte Trigger (EMA10×EMA20-Cross, RSI-Extrem) und **meldet** sie per Telegram.
- **Stufe 3 — aktive Eröffnung (opt-in, Default OFF):** hinter `ENABLE_INTRADAY_OPPORTUNISM`
  darf ein Trigger **eine** Limit-Order setzen — eigenes Budget-Tier (`maxIntradayTrades=1`)
  plus Tagesdeckel, nur Limit, keine Dopplung, nie geraten.

**Datenkompatibilität:** Alle neuen Felder (`expiresOn`, `rungGroup`, `source` an Order/
Position/Historie) sind **optional** — bestehende `portfolio.json` lädt unverändert, **keine
Migration nötig**. `data/watchlist.json` wird zur Laufzeit von der Kür angelegt (gitignored).

---

## So testest du den PR (temporär, VPS)

**Deploy (klein):** Dieser PR ändert **keine Dependencies** und **nicht das UI** — reiner
tsx-Quellcode. Nach Merge auf `master`:

```bash
! /c/Windows/System32/OpenSSH/ssh.exe vps "cd /opt/ape-signal && git pull && systemctl restart ape-signal-listener && npm test 2>&1 | tail -3"
```

Die **Timer-Dienste** (`ape-signal-scan@PreUS`, `ape-signal-tick@Tick`) brauchen keinen
Neustart — sie starten `tsx` bei jedem Lauf neu und ziehen den neuen Code automatisch. Nur der
**Listener** (long-running) muss neu gestartet werden. Der UI-Container ist unberührt.

**Was auf Telegram zu beobachten ist (Stufe 1+2, ohne Flag):**
- **Kandidatenkür:** Order-Zeilen zeigen jetzt `Limit <level>`, ggf. `Leiter-Rung` und
  `gültig bis Handelsschluss <expiresOn>` (mehrtägig). Opus sollte Limits statt Market wählen.
- **Setup-Radar:** im Tagesverlauf `⚡ Setup TICKER @ <kurs>: EMA10×EMA20 ↑ · RSI … — <note>`.
  Prüfen, dass `data/watchlist.json` nach der Kür existiert (nicht gehandelte Kandidaten).
- **Mutual-Cancel:** füllt eine Rung, erscheint direkt danach eine „⏳ Order verfallen"-Zeile
  für die Geschwister-Rungs (Stake zurück).

**Stufe 3 erst nach Beobachtung von Stufe 1+2 scharf schalten (experimentell):**
```bash
# in /etc/ape-signal.env:
ENABLE_INTRADAY_OPPORTUNISM=1
# dann: systemctl restart ape-signal-listener (Timer ziehen es automatisch)
```
Erwartung: ein gefeuerter Trigger löst **höchstens einen** Intraday-Limit-Open aus
(`🦍 Mr Ape — Intraday-Chance …`), im eigenen Budget-Tier, nur Limit, nie auf gehaltenem Ticker.

**Lokal:** `npm test` (alles grün), `npm run doctor` (Config), `npm run gen-timers -- --out=/tmp/x`
(Timer-Generierung unberührt).

---

## Ausstehende Tasks (Folge-Session)

### A — Feinschliff direkt an dieser Arbeit
1. **Research/Debatte limit-aware (Constraint #6) — Voraussetzung für Stufe-3-Vertrauen.**
   Heute fangen die Sonnet-Research-/Debatte-Calls in `select.ts:82,106` Fehler **generisch**
   ab (kein `ClaudeError.kind`-Check). Bevor Intraday-LLM-Calls breiter genutzt werden, den
   Limit-/Timeout-Pfad dort wie beim Entscheider/Manager spezifisch melden.
2. **Kür-Journal zeigt TTL/Leiter (minor).** Der inline-Journaltext in `select.ts` (~Z. 189)
   weist `expiresOn`/`rungGroup` noch nicht aus; die Telegram-Kür (`formatKuer`→`orderLine`)
   schon. Für Konsistenz nachziehen.
3. **Optional: „Gruppe = 1 Trade"-Budget für Leitern.** Heute belegt eine N-Rung-Leiter N der
   3 Tages-Slots (ehrlich, einfach). Verfeinerung: eine Leiter als **ein** Trade zählen
   (`tradesPlacedToday` gruppen-bewusst). In der Stufe-1-Spec als möglicher Folge-Schritt notiert.
4. **Offene Frage Nautilus §5: Wake-/Setup-Trigger auf High/Low statt nur Close?** Bewusst
   close-basiert gelassen (Determinismus). Falls Intraday-Breakout-Trigger gewünscht: eigener
   Architektur-Beschluss nötig (Kritiker-Hinweis im Brainstorm beachten).

### B — Vertagte Beschlüsse (Nautilus-Brainstorm §4, Reihenfolge steht)
5. **Deterministischer Trailing-Stop** (`trailBy` an `Position`, in `applyTick` nachgezogen) —
   senkt LLM-Abhängigkeit, bildet die Exit-Seite der Order-Geometrie. Eigene Spec.
6. **Intent-Event-Stream** (typisierte `PlaceOrder`/`MoveStop`/`SetTP`/`Close` als JSONL;
   Telegram+UI rendern daraus; On-Ramp zu Real-Broker/Autonomie). Eigene Spec, nach Trailing-Stop.
7. **`/status`-Command + opt-in Heartbeat** (Pull-Lagebericht + bewusster Morgen-Blick).

### C — Roadmap (MASTERPLAN-Backlog, unverändert)
8. **B1 Residential-Proxy** (webshare/IPRoyal) — schaltet echte Candles (exaktes EMA 8),
   StockTwits/Reddit-Sentiment und C1-Embed frei. Transport-Layer, breiter Nutzen.
9. **Timing-Fix Finding B** — Scan vorziehen und/oder Kür entkoppeln (Limit-Orders aus Stufe 1
   mildern den Late-Fill bereits).
10. **B3 Trending abschalten + Refokus** („Report weg, Daten bleiben", `sendReport`-Flag).
11. **C1 Embed/Refresh** (braucht B1), **agent-reach**, **C2** Mr-Ape-Chat read-only,
    **C4** Session/Tick-UI, **C3** Setup-Assistent.

---

## Schlüsseldateien (Opportunismus)

| Bereich | Dateien |
|---|---|
| Order-Geometrie (Leiter, TTL, source, Budget) | `src/paper/engine.ts`, `src/paper/types.ts` |
| Setup-Erkennung (rein) | `src/paper/setupRadar.ts` |
| Watchlist-Persistenz | `src/paper/watchlist.ts` |
| Radar-Orchestrierung (Erkennung + Alert + gegateter Open) | `src/paper/radar.ts` |
| Intraday-Eröffnung (Stufe 3, gegated) | `src/paper/intraday.ts` |
| Prompts (Limit/Leiter/TTL, Intraday) | `src/paper/prompts.ts` |
| Kür-Seeding + Limit-Bevorzugung | `src/paper/select.ts` |
| Tick-Verdrahtung (Radar nach Monitor-Tick) | `src/paper/tick.ts` |
| Flag | `src/config/env.ts` (`ENABLE_INTRADAY_OPPORTUNISM`) |
| Spec/Plan/Brainstorm | `docs/superpowers/{brainstorms,specs,plans}/2026-06-18-intraday-opportunismus*` |

---

## Arbeitsweise (bewährt, beibehalten)

- Superpowers-Workflow strikt: brainstorming → spec → writing-plans → executing-plans (TDD
  red→green, **Commit pro Task**) → finishing-a-development-branch.
- Entwicklung **inline**. `npm test` + `npm run typecheck` grün vor jedem Commit.
- **PR-Route:** `git push fork <branch>` + `gh pr create --repo lmuenet/ape-signal
  --head lm-obs:<branch>` (Cross-Fork; `lm-obs` hat keine Push-Rechte auf `lmuenet`).
  **Merge & Squash macht Lars selbst.**
- Commit-Trailer: `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`.
- **SSH nur der Nutzer**, Stil: `! /c/Windows/System32/OpenSSH/ssh.exe vps "…"` (mehrere
  Schritte mit `;`/`&&` in EINEM Quote). UI-Deploy braucht `docker build` + Container-Neustart
  (ADR 0004) — für diesen PR aber **nicht** relevant (kein UI-Change).

## Wie weiter

Folge-Session: aus **A** (Opportunismus abrunden) oder **B/C** (Roadmap) wählen, pro Punkt
spec → plan → TDD, und wieder mit einem PR über die Cross-Fork-Route enden.
