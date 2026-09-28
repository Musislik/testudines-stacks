# Future Features & Plánovaná vylepšení (testudines-stacks)

Tento dokument slouží k evidenci plánovaných rozšíření a vylepšení správy aplikačních stacků.

---

## 1. Automatická synchronizace Git repozitáře (Auto-Sync)

### Současný stav:
Repozitář stacků je perzistentně naklonován v `/opt/testudines-stacks` (se symlinkem `/opt/stacks -> /opt/testudines-stacks/stacks`). Synchronizace probíhá ručně (`git pull` nebo `git push`), případně při běhu Ansible playbooku (`site.yml`), pokud je pracovní strom čistý.

### Návrh řešení:
Implementovat automatické stahování a aplikování změn bez nutnosti ručního spouštění Ansible:

1. **Varianta A: Periodická kontrola (Cron / Systemd Timer)**
   - Vytvořit lehký skript nebo systemd timer uvnitř virtuálního stroje, který např. každou hodinu (nebo každých 15 minut) provede:
     ```bash
     cd /opt/testudines-stacks
     # Pokud je pracovní strom čistý, stáhnout změny
     [ -z "$(git status --porcelain)" ] && git pull origin master
     # Volitelně detekovat změny v compose.yaml a provést docker compose up -d
     ```
2. **Varianta B: Webhook (Okamžitá reakce na Git Push)**
   - Nasadit lehký webhook listener (např. `adnanh/webhook` v Dockeru nebo na hostiteli), který přijme GitHub webhook po provedení `git push` do větve `master`.
   - Zabezpečit webhook sdíleným tajným tokenem (secret).
   - Při příchozím webhooku provést `cd /opt/testudines-stacks && git pull` a zaktualizovat dotčený stack v `/opt/stacks/`.
