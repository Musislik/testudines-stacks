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

---

## 4. Nextcloud Core – Správa verzí MariaDB a Nextcloud Core (`stacks/nextcloud-core`)

### A) MariaDB: Vazba na větev 11.8 a přechod na budoucí LTS
- **Popis problému:** Stávající obnovená databáze byla vytvořena na verzi **MariaDB 11.8.2** (rolling / short-term větev). Tag `mariadb:lts` odkazuje na verzi **11.4 LTS**. V MariaDB nelze provést in-place downgrade na raw souborech databáze. Pokus o spuštění starší MariaDB 11.4 nad daty z 11.8 selže s chybou `[ERROR] Bad magic header in tc log` a databáze nenastartuje.
- **Aktuální stav:** Obraz databáze je zafixován na `image: mariadb:11.8`.
- **Kdy vyjde další LTS:** MariaDB vydává novou LTS verzi přibližně každé 2 roky (poslední LTS 11.4 vyšla v květnu 2024, předchozí 10.11 v únoru 2023). Další plnohodnotná LTS verze (např. větev 12.x LTS) se očekává na **přelomu let 2026/2027**.
- **Doporučený plán přechodu na LTS:**
  - Ponechat databázi běžet na `mariadb:11.8`.
  - Jakmile vyjde nová LTS verze (bude mít vyšší verzi než 11.8), provést přímý in-place upgrade pouhou změnou obrazu na tuto novou LTS verzi bez nutnosti mezikroků a ručních exportů/importů.
  - *Poznámka:* Pokud by došlo k chybě `Bad magic header in tc log`, odstraní se dočasný soubor: `sudo rm -f /mnt/data-sync/nextcloud-db/tc.log`.

### B) Nextcloud: Zákaz přeskakování hlavních verzí (Major Version Upgrade)
- **Popis problému:** Nextcloud oficiálně nepodporuje přímý upgrade přes více hlavních verzí najednou (např. z verze 32 přímo na 34/35). Pokus o start s generickým tagem `nextcloud:apache` (který stahuje nejnovější verzi) skončí chybou:
  `It is only possible to upgrade one major version at a time.`
- **Aktuální stav:** Obnovená záloha běží na **Nextcloud 32.0.3**, proto jsou kontejnery `app` a `cron` zafixovány na obraz `nextcloud:32-apache`.
- **Postup budoucího upgradu:** Upgrady je nutné provádět postupně o jednu hlavní verzi (32 ➔ 33 ➔ 34 ➔ ...):
  1. V `compose.yaml` změnit tag na `nextcloud:33-apache` a spustit `docker compose up -d`.
  2. Spustit migraci: `docker exec -u 33 nextcloud php occ upgrade`.
  3. Až po úspěšném dokončení zopakovat proces pro verzi 34 atd.


