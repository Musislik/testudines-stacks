# Future Features & Plánovaná vylepšení (testudines-stacks)

Tento dokument slouží k evidenci plánovaných rozšíření a vylepšení správy aplikačních stacků.

---

## 1. GitOps architektura: Náhrada Dockge, Git jako jediný zdroj pravdy a dedikovaná `deploy` větev

### Současný stav:
Aplikační stacky jsou spravovány nástrojem **Dockge** (`/opt/dockge`), který poskytuje webové GUI pro přímou editaci `compose.yaml` na serveru. Repozitář stacků je perzistentně naklonován v `/opt/testudines-stacks` (se symlinkem `/opt/stacks -> /opt/testudines-stacks/stacks`).

**Problém současného řešení:**
- Dockge je koncepčně editor na serveru – úpravy prováděné v GUI vytvářejí lokální necommitnuté změny na disku.
- Dochází k rozcházení (driftu) mezi stavem na serveru a stavem v Git repozitáři.
- Klasický GitOps (automatický pull z Gitu) nelze s Dockge bezpečně provozovat bez rizika nechtěného přepsání lokální práce nebo kolizí při `git pull`.

---

### Cílový stav & Návrh řešení:
Přechod na čistý **GitOps model**, kde je Git jediným zdrojem pravdy (Single Source of Truth), server funguje v konzumním (read-only) režimu a Dockge je nahrazen vhodnějším nástrojem.

#### 1. Dedikovaná větev pro nasazení (`deploy` / `production`)
- Běžný vývoj, ladění a úpravy `compose.yaml` probíhají ve vývojové větvi (např. `master` / `main` nebo feature branches) mimo produkční server.
- Server má trvale checkoutnutou speciální větev (např. `deploy`).
- Do větve `deploy` se změny dostávají řízeně (např. merge / fast-forward po otestování). Tím je zaručeno, že se na server nikdy nedostanou nechtěné nebo rozpracované úpravy.

#### 2. Náhrada nástroje Dockge
Jelikož Dockge neslouží jako GitOps agent, bude nahrazen jedním z následujících řešení:
- **Kandidát A: [Komodo](https://komo.do/) (Doporučeno pro zachování GUI)**
  - Specializovaný, moderní GitOps správce pro Docker Compose.
  - Disponuje přehledným webovým GUI pro sledování stavu kontejnerů, logů a historie nasazení.
  - Nativně se propojuje s Git repozitářem a konkrétní větví (`deploy`), sleduje commity a automaticky nasazuje bez nutnosti ručních zásahů.
- **Kandidát B: Portainer (Git-backed stacks)**
  - Tradiční řešení, kde lze stacky vytvořit s přímým odkazem na Git repozitář a větev s podporou pollingu nebo webhooků.
- **Kandidát C: Headless GitOps + lehký read-only dashboard**
  - Synchronizaci řídí lehký systémový démon (systemd timer / cron nebo GitHub Action přes SSH) přímo na hostiteli.
  - Pro monitoring kontejnerů a prohlížení logů přes web se nasadí pouze read-only nástroje (např. **Dozzle** pro živé logy), čímž zcela odpadne riziko nechtěných zásahů do konfigurace na serveru.

#### 3. Logika automatického stahování a reloadu změn (Pull & Deploy)
Server v pravidelných intervalech (nebo po přijetí webhooku / GitHub Actions triggeru) provádí bezpečný fast-forward pull a nasazení dotčených stacků:

```bash
cd /opt/testudines-stacks
git fetch origin deploy

# Ověření, že na serveru nevznikly nechtěné lokální změny:
if [ -z "$(git status --porcelain)" ]; then
  PREV_REV=$(git rev-parse HEAD)
  git pull --ff-only origin deploy
  CURR_REV=$(git rev-parse HEAD)

  # Automatický reload pouze těch stacků, jejichž soubory se změnily:
  if [ "$PREV_REV" != "$CURR_REV" ]; then
    CHANGED_STACKS=$(git diff --name-only "$PREV_REV" "$CURR_REV" | grep '^stacks/' | cut -d/ -f2 | sort -u)
    for STACK in $CHANGED_STACKS; do
      if [ -f "/opt/stacks/$STACK/compose.yaml" ]; then
        echo "Aktualizuji stack: $STACK"
        docker compose -f "/opt/stacks/$STACK/compose.yaml" up -d --remove-orphans
      fi
    done
  fi
else
  echo "Varování: Na serveru byly detekovány lokální změny. Pull pozastaven pro zamezení konfliktů."
fi
```

#### 4. Metody spouštění synchronizace
- **Systemd Timer / Cron:** Periodická kontrola přímo na hostiteli (např. každých 5 minut). Jednoduché, robustní, nevyžaduje otevírání portů.
- **Webhook / CI/CD (GitHub Actions):** Push/merge do větve `deploy` automaticky přes SSH nebo webhook notifikuje server a nasadí novou verzi okamžitě.


