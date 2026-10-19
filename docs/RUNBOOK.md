# Runbook — montage et exploitation du lab

Journal de bord réel. Les incidents rencontrés et résolus y figurent
volontairement : c'est la partie la plus instructive du projet.

---

## 1. Hôte et réseau

- Virtualisation : **VirtualBox** sur Windows (HP Victus, 32 Go RAM).
- Réseau du lab : réseau privé hôte VirtualBox, **`192.168.154.0/24`**, DHCP désactivé.
- Chaque VM a deux cartes : NAT (Internet) + hôte-uniquement (lab, IP fixe via netplan).
- Hôte Windows accessible sur le lab en `192.168.154.1` (il sert aussi d'agent).

---

## 2. Déploiement du SIEM (VM-SIEM, 192.168.154.10)

```bash
# paramètre noyau requis par l'indexer (OpenSearch)
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee /etc/sysctl.d/99-wazuh.conf
sudo sysctl --system

# Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER   # puis re-login

# stack Wazuh 4.14.9 (single-node)
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.9 --depth=1
cd wazuh-docker/single-node
docker compose -f generate-indexer-certs.yml run --rm generator
docker compose up -d
```

Dashboard : `https://192.168.154.10` (certificat auto-signé, avertissement normal).

---

## 3. INCIDENT N°1 — les agents ne remontaient aucune alerte

**Symptôme.** Après une attaque de force brute SSH depuis Kali, aucune alerte dans
le dashboard. Même les connexions SSH *réussies* n'apparaissaient pas.

**Diagnostic (chaîne de collecte, maillon par maillon) :**

1. **L'attaque atteint-elle la cible ?** `sudo tail /var/log/auth.log` sur l'agent →
   `Failed password for invalid user toto from 192.168.154.30` présent. ✅
2. **Le moteur de règles décode-t-il ?** `wazuh-logtest` sur le manager → règle 5710,
   *Alert to be generated*. ✅ Le ruleset fonctionne.
3. **Le manager reçoit-il les logs ?** Non → la rupture est entre l'agent et le manager.
4. **Logs de l'agent** (`/var/ossec/logs/ossec.log`) :
   `Unable to connect to enrollment service at '[192.168.56.10]:1515'`.

**Cause racine.** Les deux agents (Ubuntu **et** Windows) étaient configurés avec une
**mauvaise adresse de manager** (`192.168.56.10`, héritée d'un plan d'adressage
antérieur) au lieu de `192.168.154.10`. Cause commune, répercutée sur les deux agents.

**Correctif (Linux) :**
```bash
sudo sed -i 's/192\.168\.56\.10/192.168.154.10/g' /var/ossec/etc/ossec.conf
sudo rm -f /var/ossec/etc/client.keys
sudo systemctl restart wazuh-agent
```

**Correctif (Windows, PowerShell admin) :**
```powershell
(Get-Content "C:\Program Files (x86)\ossec-agent\ossec.conf") -replace '192\.168\.56\.10','192.168.154.10' | Set-Content "C:\Program Files (x86)\ossec-agent\ossec.conf"
Remove-Item "C:\Program Files (x86)\ossec-agent\client.keys" -ErrorAction SilentlyContinue
Restart-Service WazuhSvc
```

**Leçon.** Une alerte absente ne veut pas dire « pas d'attaque » : il faut remonter
toute la chaîne (collecte → transport → décodage → règle → indexation → affichage).
Ici les quatre premiers maillons étaient sains ; la panne était au **transport**.

---

## 4. INCIDENT N°2 — boucle d'enrôlement de l'agent Windows

**Symptôme.** Après correction de l'IP, l'agent Windows restait « disconnected » puis
« never connected », avec en boucle :
`ERROR: Duplicate agent name: DESKTOP-MB2IHI8. Unable to add agent (from manager)`.

**Cause racine.** La suppression de `client.keys` force une ré-inscription
automatique. Mais un enregistrement du même nom existait déjà côté manager → le
manager refuse le doublon → l'agent boucle sans jamais obtenir de clé valide.

**Correctif — provisioning manuel de la clé :**
```bash
# sur le manager : extraire la clé de l'agent existant
docker exec single-node-wazuh.manager-1 /var/ossec/bin/manage_agents -e <ID>
```
```powershell
# sur Windows : importer la clé (manage_agents.exe → I → coller → y → Q), puis
Restart-Service WazuhSvc
```

**Leçon.** J'ai pratiqué les **deux méthodes d'enrôlement** Wazuh — automatique
(auto-enrollment via authd) et manuelle (extraction/import de clé). Comprendre le
cycle de vie d'un agent (enregistrement → clé → communication) est indispensable
pour dépanner une flotte.

Résultat final : **deux agents Active** (VM-Agent-Ubuntu et DESKTOP-MB2IHI8).

---

## 5. Autres points

- **Décalage horaire** : les VM étant en UTC, les alertes apparaissaient « 2 h plus tôt »
  que l'heure locale. Toujours élargir la plage de temps du dashboard (« Last 24 hours »).
- **Pare-feu** : ports agents `1514/tcp` (communication) et `1515/tcp` (enrôlement) à
  autoriser côté manager ; règle de sortie ajoutée côté Windows par précaution.
