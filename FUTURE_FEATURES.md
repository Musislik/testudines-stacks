# Future Features & Plánovaná vylepšení (testudines-stacks)

Tento dokument slouží k evidenci plánovaných rozšíření a vylepšení správy aplikačních stacků.

---

## 1. Automatická synchronizace Git repozitáře (Auto-Sync)

### Současný stav:
Synchronizace stacků z repozitáře `testudines-stacks` do produkčního adresáře `/opt/stacks/` probíhá pouze na vyžádání při ručním spuštění Ansible playbooku (`site.yml` / `bootstrap.sh`), případně přímou editací v rozhraní Dockge.

### Návrh řešení:
Implementovat automatické stahování a aplikování změn bez nutnosti ručního spouštění Ansible:

1. **Varianta A: Periodická kontrola (Cron / Systemd Timer)**
   - Vytvořit lehký skript nebo systemd timer uvnitř virtuálního stroje, který např. každou hodinu (nebo každých 15 minut) provede:
     ```bash
     cd /tmp/stacks-repo && git pull
     rsync -av --delete /tmp/stacks-repo/stacks/ /opt/stacks/
     # Volitelně detekovat změny v compose.yaml a provést docker compose up -d
     ```
2. **Varianta B: Webhook (Okamžitá reakce na Git Push)**
   - Nasadit lehký webhook listener (např. `adnanh/webhook` v Dockeru nebo na hostiteli), který přijme GitHub/Gitea webhook po provedení `git push` do větve `main`.
   - Zabezpečit webhook sdíleným tajným tokenem (secret).
   - Při příchozím webhooku automaticky provést `git pull` a zaktualizovat dotčený stack v `/opt/stacks/`.
