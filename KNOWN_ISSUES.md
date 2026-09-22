# Known Issues (testudines-stacks)

Tento dokument eviduje známé limity, chování a specifika jednotlivých aplikačních stacků.

---

## 1. OpenTTD Server (`stacks/openttd-server`)

### Popis chování:
- Kontejner používá obraz `bateau/openttd:14.1`.
- Tento obraz **nečte ani nepodporuje proměnnou prostředí `OPENTTD_SERVER_NAME`** ze souboru `.env`.
- Veškerou síťovou i herní konfiguraci (včetně jména serveru `server_name`, hesel a limitů hráčů) OpenTTD načítá a ukládá výhradně přímo do souboru `openttd.cfg` na persistentním svazku:
  `/mnt/data-sync/openttd-server/openttd.cfg` (uvnitř kontejneru `/home/openttd/.local/share/openttd/openttd.cfg`).

### Jak správně upravit nastavení serveru:
1. Zastavit kontejner (např. přes Dockge nebo `docker compose down`).
2. Upravit soubor `/mnt/data-sync/openttd-server/openttd.cfg` na serveru:
   ```ini
   [network]
   server_name = "Moje Jméno Serveru"
   server_password = ""
   max_clients = 15
   max_companies = 15
   min_active_clients = 1
   ```
3. Znovu spustit kontejner (`docker compose up -d`).

---

## 2. OpenTTD Server – Chování při prvním startu a vyžadování existující pozice (`loadgame=last-autosave`)

### Popis problému:
- V souboru `stacks/openttd-server/compose.yaml` je nastavena proměnná prostředí `loadgame=last-autosave`.
- Obraz `bateau/openttd` se při startu snaží automaticky najít a načíst nejnovější uloženou pozici v adresáři `/home/openttd/.local/share/openttd/save/autosave/`.
- Při čisté instalaci na nový server je adresář `/mnt/data-sync/openttd-server` zcela prázdný a žádný autosave soubor v něm neexistuje.
- **Důsledek:** Server při pokusu o načtení neexistujícího autosavu selže a kontejner spadne v restartovací smyčce (`CrashLoopBackOff`).

### Postup řešení:

#### Varianta A: Migrace stávající herní pozice (Doporučeno pro pokračování v rozehrané hře)
Před spuštěním kontejneru přeneste zálohu původní hry (např. z archivní složky `disabled/openttd_saves` a `disabled/openttd_backup`) na server:
```bash
# Vytvoření cílové složky
mkdir -p /mnt/data-sync/openttd-server/save/autosave

# Zkopírování autosavu (např. CZ-1.sav) a konfigurace
cp disabled/openttd_saves/save/autosave/*.sav /mnt/data-sync/openttd-server/save/autosave/
cp disabled/openttd_backup/openttd_data/openttd.cfg /mnt/data-sync/openttd-server/openttd.cfg

# Nastavení práv pro kontejner
chown -R 1000:1000 /mnt/data-sync/openttd-server
```
Poté spusťte kontejner přes Dockge nebo `docker compose up -d`. OpenTTD automaticky naváže na poslední autosave.

#### Varianta B: Spuštění nové čisté hry
Pokud chcete začít hrát zcela novou hru od začátku:
1. Dočasně v `stacks/openttd-server/compose.yaml` změňte:
   ```yaml
   environment:
     - loadgame=false
   ```
2. Spusťte kontejner: `docker compose up -d`.
3. Počkejte, až server vygeneruje nový svět a proběhne první automatické uložení do složky `save/autosave/`.
4. Následně můžete vrátit hodnotu zpět na `loadgame=last-autosave`, aby server po budoucích restartech vždy navazoval na rozehranou hru.

---

## 3. Oprávnění ke sdílenému disku `/mnt/data-nosync/avdb` (JDownloader vs Nextcloud)

### Popis chování:
- Adresář `/mnt/data-nosync/avdb` je sdílen mezi kontejnerem **JDownloader** (`/output`) a **Nextcloud** (`/avdb`).
- JDownloader běží pod výchozím uživatelem s UID `1000`, zatímco Nextcloud (Apache) běží pod systémovým uživatelem `www-data` s UID `33`.
- Pokud by JDownloader vytvářel nové soubory a složky se standardní umaskou `022` nebo `027`, Nextcloud by neměl práva tyto soubory přes webové rozhraní přejmenovávat, přesouvat nebo mazat.

### Řešení v konfiguraci:
V `stacks/jdownloader/compose.yaml` je nastavena proměnná prostředí:
```yaml
environment:
  - UMASK=000
```
Tím JDownloader vytváří veškeré nově stažené soubory a složky s plnými právy pro čtení i zápis pro všechny uživatele (soubory `0666`, složky `0777`), což umožňuje Nextcloudu bezproblémovou plnou manipulaci se soubory.

