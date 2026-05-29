# Deploy Plan — whatch-this-radar

**Platform**: Cloudflare Workers + Pages  
**Stack**: Astro 6 SSR · `@astrojs/cloudflare` v13.5.0 · Wrangler 4.90.0 · Supabase (external) · GitHub Actions  
**Source**: `context/foundation/infrastructure.md` + `context/foundation/tech-stack.md`

---

## Stan wyjściowy (co jest, a czego brakuje)

| Element | Stan | Uwaga |
|---|---|---|
| `wrangler.jsonc` | ✅ present | `name` = `10x-astro-starter` → trzeba zmienić |
| `@astrojs/cloudflare` adapter | ✅ v13.5.0 | poprawnie ustawiony w `astro.config.mjs` |
| `compatibility_flags: nodejs_compat` | ✅ set | wymagane |
| `assets.directory: ./dist` | ✅ set | wymagane |
| `SUPABASE_URL` / `SUPABASE_KEY` w `astro:env/server` | ✅ schema w `astro.config.mjs` | tylko jako `optional: true` — do weryfikacji |
| `.env.example` | ✅ present | |
| `.dev.vars` | ❌ brak | wymagany dla lokalnego `npm run dev` |
| `deploy` script w `package.json` | ❌ brak | |
| GitHub Actions — CI workflow | ✅ `.github/workflows/ci.yml` | lint + build, bez deploy |
| GitHub Actions — deploy workflow | ❌ brak | |
| Workers Secrets w Cloudflare | ❌ brak | SUPABASE_URL + SUPABASE_KEY |
| GitHub Secrets | ⚠️ częściowy | SUPABASE_URL/KEY już ref. w CI, brak CLOUDFLARE_API_TOKEN |

---

## Faza 1 — Poprawki lokalne (repo)

### 1.1 Zmień nazwę projektu w `wrangler.jsonc`

`wrangler.jsonc` → pole `name` z `10x-astro-starter` na `whatch-this-radar`. Ta wartość staje się subdomeną Workers: `https://whatch-this-radar.<account>.workers.dev`.

```jsonc
// wrangler.jsonc
{
  "name": "whatch-this-radar",
  ...
}
```

### 1.2 Uaktualnij nazwę w `package.json`

```json
// package.json
{
  "name": "whatch-this-radar",
  ...
}
```

### 1.3 Dodaj skrypt `deploy` do `package.json`

```json
"scripts": {
  "deploy": "astro build && wrangler deploy",
  ...
}
```

### 1.4 Utwórz `.dev.vars` dla lokalnego devu

Skopiuj `.env.example` → `.dev.vars` i uzupełnij prawdziwymi wartościami. Wrangler 4 czyta `.dev.vars` jako Workers bindings podczas `npm run dev`.

```
SUPABASE_URL=https://<project-ref>.supabase.co
SUPABASE_KEY=<anon-key>
```

`.dev.vars` jest już w `.gitignore` (zweryfikować — jeśli nie, dodać).

### 1.5 Zweryfikuj `SUPABASE_URL` / `SUPABASE_KEY` jako nieobowiązkowe tylko w CI build

W `astro.config.mjs` oba pola mają `optional: true` — dzięki temu `npm run build` w CI przechodzi bez sekretów (tylko buduje artefakt). W runtime Workers Secrets muszą być ustawione.

---

## Faza 2 — Konto Cloudflare i pierwsze uwierzytelnienie

### 2.1 Zaloguj się przez Wrangler

```bash
npx wrangler login
```

Otwiera przeglądarkę z OAuth. Credentials zapisywane w `~/.config/.wrangler/`. Weryfikacja:

```bash
npx wrangler whoami
```

### 2.2 Zweryfikuj wersję Wranglera

```bash
npx wrangler --version
# oczekiwane: 4.90.0 lub nowsze
```

---

## Faza 3 — Konfiguracja Supabase

### 3.1 Utwórz projekt Supabase

1. Dashboard: [supabase.com/dashboard](https://supabase.com/dashboard) → New Project.
2. Region: `eu-central-1` (Frankfurt) — najbliższy Polsce.
3. Zapisz hasło do bazy — nie jest dostępne później.

### 3.2 Pobierz klucze

Project Settings → API:
- `SUPABASE_URL` = `https://<project-ref>.supabase.co`
- `SUPABASE_KEY` = `anon` public key (nie `service_role`)

### 3.3 Uzupełnij `.dev.vars` (Faza 1.4)

---

## Faza 4 — Pierwsze ręczne wdrożenie

Wykonaj lokalnie — to rejestruje projekt w Cloudflare i weryfikuje cały pipeline przed podpięciem CI.

### 4.1 Build

```bash
npm run build
```

Oczekiwany wynik: katalog `dist/` z `dist/_worker.js/` i `dist/` dla statycznych assetów. Brak błędów TypeScript.

> **Uwaga z weryfikacji build (2026-05-29):** `@astrojs/cloudflare` automatycznie włącza dwa bindingi widoczne w logach build:
> - `[@astrojs/cloudflare] Enabling sessions with Cloudflare KV with the "SESSION" KV binding.`
> - `[@astrojs/cloudflare] Enabling image processing with Cloudflare Images for production with the "IMAGES" Images binding.`
>
> Lokalnie (`npm run dev` / build) działają na proxy bez konfiguracji. W produkcji `SESSION` wymaga realnego KV namespace (auth `@supabase/ssr` korzysta z sesji) — patrz krok 4.2. `IMAGES` używa platformowego bindingu i zwykle nie wymaga konfiguracji.

### 4.2 Utwórz KV namespace dla sesji (`SESSION`)

Astro sesje na Cloudflare wymagają KV namespace zbindowanego jako `SESSION`. Utwórz go:

```bash
npx wrangler kv namespace create SESSION
```

Output zwróci `id` namespace'u. Dodaj binding do `wrangler.jsonc`:

```jsonc
// wrangler.jsonc
{
  "name": "whatch-this-radar",
  ...
  "kv_namespaces": [
    { "binding": "SESSION", "id": "<id-z-komendy-powyzej>" }
  ],
}
```

Opcjonalnie utwórz osobny namespace dla preview (`--preview` flaga) jeśli używasz środowisk preview.

> Jeśli aplikacja nie polega na server-side sesjach Astro (bo `@supabase/ssr` trzyma sesję w cookies), `wrangler deploy` może przejść bez tego bindingu — ale brak `SESSION` KV ujawni się dopiero przy pierwszym użyciu API sesji w runtime. Bezpieczniej utworzyć namespace teraz.

### 4.3 Ustaw Workers Secrets

Przed deployem ustaw sekrety w projekcie Cloudflare:

```bash
npx wrangler secret put SUPABASE_URL
# paste: https://<project-ref>.supabase.co

npx wrangler secret put SUPABASE_KEY
# paste: <anon-key>
```

Sekrety są szyfrowane po stronie Cloudflare i nigdy nie trafiają do logów.

### 4.4 Deploy

```bash
npx wrangler deploy
```

Pierwsze wywołanie tworzy projekt Workers w Cloudflare Dashboard automatycznie (nie trzeba konfigurować ręcznie). Oczekiwany output:

```
Uploaded whatch-this-radar (X.XX sec)
Deployed whatch-this-radar triggers (X.XX sec)
  https://whatch-this-radar.<account>.workers.dev
```

### 4.5 Weryfikacja

1. Otwórz `https://whatch-this-radar.<account>.workers.dev` — aplikacja powinna się załadować.
2. Sprawdź logi w czasie rzeczywistym: `npx wrangler tail`
3. Zweryfikuj zapis wersji: `npx wrangler versions list` — zapisz `VERSION_ID` pierwszego deployu.

### 4.6 Test rollback (raz, na start)

```bash
npx wrangler rollback
# wraca do poprzedniej wersji (lub podaj VERSION_ID)
npx wrangler deploy  # przywróć aktualną wersję po teście
```

---

## Faza 5 — GitHub Actions: workflow deploy

### 5.1 Utwórz Cloudflare API Token

Dashboard → My Profile → API Tokens → Create Token:
- Template: **Edit Cloudflare Workers**
- Permissions: `Account > Workers Scripts > Edit`, `Zone > Workers Routes > Edit` (jeśli custom domain)
- Account Resources: specific account

Skopiuj token — widoczny tylko raz.

### 5.2 Pobierz Cloudflare Account ID

Dashboard → Workers & Pages → Overview — Account ID w prawym panelu.

### 5.3 Dodaj GitHub Secrets

Repo → Settings → Secrets and variables → Actions → New repository secret:

| Name | Value |
|---|---|
| `CLOUDFLARE_API_TOKEN` | token z 5.1 |
| `CLOUDFLARE_ACCOUNT_ID` | ID z 5.2 |
| `SUPABASE_URL` | (już powinien być z CI) |
| `SUPABASE_KEY` | (już powinien być z CI) |

### 5.4 Utwórz `.github/workflows/deploy.yml`

```yaml
name: Deploy

on:
  push:
    branches: [master]

jobs:
  deploy:
    runs-on: ubuntu-latest
    needs: []
    # Uruchamia się tylko gdy CI (ci.yml) przeszło — opcja z workflow_run:
    # on:
    #   workflow_run:
    #     workflows: [CI]
    #     types: [completed]
    #     branches: [master]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run build
        env:
          SUPABASE_URL: ${{ secrets.SUPABASE_URL }}
          SUPABASE_KEY: ${{ secrets.SUPABASE_KEY }}
      - name: Deploy to Cloudflare Workers
        run: npx wrangler deploy
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
```

> **Uwaga**: Aktualny `ci.yml` triggeruje również na push do `master`. Możesz scalić CI i deploy w jeden plik lub zostawić dwa oddzielne — oddzielne dają lepszą widoczność w GitHub Actions UI. Jeśli zostawiasz dwa, rozważ `workflow_run` trigger w deploy.yml żeby deploy nie wystartował zanim CI nie przejdzie.

---

## Faza 6 — Weryfikacja end-to-end

1. Stwórz gałąź `chore/rename-project`, wprowadź zmiany z Fazy 1, push PR.
2. Sprawdź że CI (lint + build) zielenią się w PR.
3. Merge do `master`.
4. Sprawdź zakładkę Actions → Deploy workflow — powinien uruchomić się automatycznie.
5. Po zakończeniu deployu: otwórz produkcyjny URL, zweryfikuj że app działa.
6. `npx wrangler versions list` — potwierdź nową wersję.

---

## Faza 7 — Custom domain (opcjonalnie, post-MVP)

Jeśli potrzebna własna domena (nie `*.workers.dev`):

1. Cloudflare Dashboard → Workers & Pages → `whatch-this-radar` → Settings → Domains & Routes → Add Custom Domain.
2. Domena musi być zarządzana przez Cloudflare DNS (lub dodaj rekord CNAME ręcznie).
3. TLS jest automatyczny.

Nie jest wymagane do MVP — `workers.dev` subdomena w zupełności wystarczy.

---

## Checklist gotowości do pierwszego deploy

- [ ] `wrangler.jsonc` → `name: "whatch-this-radar"`
- [ ] `package.json` → `name: "whatch-this-radar"` + skrypt `deploy`
- [ ] `.dev.vars` utworzony z prawdziwymi wartościami (lokalny dev działa)
- [ ] `npx wrangler login` — jesteś zalogowany
- [ ] Supabase projekt założony, klucze pobrane
- [ ] `npx wrangler kv namespace create SESSION` — wykonane, binding dodany do `wrangler.jsonc`
- [ ] `npx wrangler secret put SUPABASE_URL` — wykonane
- [ ] `npx wrangler secret put SUPABASE_KEY` — wykonane
- [ ] `npx wrangler deploy` — pierwsze ręczne wdrożenie przeszło
- [ ] Aplikacja otwiera się pod `https://whatch-this-radar.<account>.workers.dev`
- [ ] `npx wrangler versions list` — wersja widoczna, VERSION_ID zapisany
- [ ] GitHub Secrets: `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID` dodane
- [ ] `.github/workflows/deploy.yml` — plik stworzony i push wykonany
- [ ] Deploy workflow w GitHub Actions — przeszedł na `master`

---

## Zmiany repo do wykonania przed Fazą 4

Poniższe zmiany plików są wymagane (możesz je wykonać teraz lub zlecić agentowi):

1. `wrangler.jsonc` — `"name": "whatch-this-radar"`
2. `package.json` — `"name": "whatch-this-radar"` + skrypt `"deploy": "astro build && wrangler deploy"`
3. `.dev.vars` — nowy plik (gitignored), uzupełniony ręcznie
4. `.github/workflows/deploy.yml` — nowy workflow (treść w Fazie 5.4)
