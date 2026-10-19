# Scénarios de détection

Chaque attaque est jouée **uniquement contre mes propres machines** du réseau
`192.168.154.0/24` (lab isolé). Rien n'est lancé en dehors de cet environnement.

Toutes les valeurs (ID de règle, niveau, description) sont celles réellement
observées dans le dashboard Wazuh.

| # | Scénario | MITRE | Détecté | Règle(s) · niveau |
|---|---|---|---|---|
| 1 | Force brute SSH | T1110.001 | ✅ | 5710 (5) · 5712 (10) |
| 2 | Création de compte Linux | T1136 | ✅ | 5902 (8) |
| 3 | Abus de privilèges (sudo) | T1548.003 | ✅ | 5402 (3) · 5403 (4) |
| 4 | Cycle de vie compte Windows | T1136.001 | ✅ | 60109 (8) · **60154 (12)** · 60110 · 60111 |
| 5 | Scan de ports | T1046 | ❌ *(angle mort documenté)* | — |

---

## Scénario 1 — Force brute SSH ✅

| | |
|---|---|
| **MITRE** | T1110.001 — Password Guessing (+ T1021.004 — SSH) |
| **Depuis** | VM-KALI (`192.168.154.30`) → **Cible** VM-AGENT-UBUNTU (`192.168.154.20`) |

**Commande :**
```bash
for i in $(seq 1 15); do
  ssh -o StrictHostKeyChecking=no -o ConnectTimeout=2 toto@192.168.154.20
done
```

**Résultat observé :**
- ~30 échecs d'authentification (règle **5710**, niveau 5 — *« sshd: Attempt to login using a non-existent user »*).
- **2 vagues agrégées** en alerte de brute force (règle **5712**, niveau 10 — *« sshd: brute force trying to get access to the system »*).
- Détection quasi temps réel.

**Validation du moteur de règles** (`wazuh-logtest`, en rejouant une ligne réelle) :
```
**Phase 1: Completed pre-decoding.
        program_name: 'sshd'
**Phase 2: Completed decoding.
        name: 'sshd'   srcip: '192.168.154.30'   srcuser: 'toto'
**Phase 3: Completed filtering (rules).
        id: '5710'   level: '5'
        description: 'sshd: Attempt to login using a non-existent user'
        mitre.id: ['T1110.001', 'T1021.004']
**Alert to be generated.
```

**Lecture d'analyste :** la règle 5712 est la vraie valeur du SIEM — elle **corrèle**
une rafale d'échecs en un seul événement « brute force » de niveau 10, au lieu de
noyer l'analyste sous des dizaines de 5710.

![Brute force SSH détecté — règle 5712](../screenshots/06-bruteforce-5712.png)

---

## Scénario 2 — Création de compte Linux ✅

| | |
|---|---|
| **MITRE** | T1136 — Create Account |
| **Sur** | VM-AGENT-UBUNTU (`192.168.154.20`) |

```bash
sudo useradd -m fauxcompte      # puis nettoyage : sudo userdel -r fauxcompte
```

**Résultat :** règle **5902**, niveau 8 — *« New user added to the system »*. Détection instantanée.

![Création de compte Linux — règle 5902](../screenshots/08-create-account-5902.png)

---

## Scénario 3 — Abus de privilèges (sudo) ✅

| | |
|---|---|
| **MITRE** | T1548.003 — Sudo and Sudo Caching |
| **Sur** | VM-AGENT-UBUNTU (`192.168.154.20`) |

**Résultat observé :**
- Règle **5403**, niveau 4 — *« First time user executed sudo »* (premier usage de sudo : signal d'escalade).
- Règle **5402**, niveau 3 — *« Successful sudo to ROOT executed »*.
- Contexte PAM associé : **5501** (session ouverte) / **5502** (session fermée).

**Lecture d'analyste :** le « first time sudo » (5403) est de la **détection comportementale** —
ce n'est pas l'action qui est suspecte, mais le fait qu'elle soit nouvelle pour ce compte.

![Abus de sudo — règles 5402 / 5403](../screenshots/09-sudo.png)

---

## Scénario 4 — Cycle de vie d'un compte Windows ✅

| | |
|---|---|
| **MITRE** | T1136.001 — Local Account |
| **Sur** | Windows (`192.168.154.1`), PowerShell admin |

```powershell
New-LocalUser -Name "soc_test2" -NoPassword
Add-LocalGroupMember -Group "Administrateurs" -Member "soc_test2"
Remove-LocalUser -Name "soc_test2"
```

**Résultat observé — toute la chaîne détectée :**

| Règle | Niveau | Description | Événement |
|---|---|---|---|
| 60109 | 8 | User account enabled or created | création (Event ID 4720) |
| **60154** | **12** | **Administrators Group Changed** | ajout au groupe Administrateurs |
| 60110 | 8 | User account changed | modification |
| 60160 | 5 | Domain Users Group Changed | — |
| 60111 | 8 | User account disabled or deleted | suppression |

**Lecture d'analyste :** l'ajout du compte au groupe Administrateurs déclenche la
règle **60154 de niveau 12** — la plus haute sévérité du lab. C'est logique : la
création d'un compte est notable, mais lui donner les droits admin est *critique*.
Le SIEM distingue bien les deux.

![Chaîne compte Windows — 60109 / 60154](../screenshots/10-windows-account.png)

---

## Scénario 5 — Scan de ports ❌ (angle mort documenté)

| | |
|---|---|
| **MITRE** | T1046 — Network Service Discovery |
| **Depuis** | VM-KALI (`192.168.154.30`) → `192.168.154.20` |

```bash
nmap -sS -p- 192.168.154.20     # scan SYN furtif
nmap -sV -p22 192.168.154.20    # scan en connexion / détection de version
```

**Résultat : aucune alerte.** Et c'est le **résultat attendu**, pas un échec.

Wazuh est un SIEM **basé sur l'hôte** : il analyse les logs des machines. Un scan de
ports est un événement de **couche réseau** qui ne génère pas d'action applicative
écrite dans un log sur la cible. Il est donc invisible pour un agent host-based.

**Recommandation :** pour couvrir ce cas, il faudrait une sonde réseau (NIDS) type
**Suricata**, dont les alertes peuvent ensuite être remontées dans Wazuh. C'est la
limite host-based vs network-based, et la raison d'être d'une défense en profondeur.

---

> Le dashboard Wazuh cartographie automatiquement ces détections sur MITRE ATT&CK
> (Password Guessing, SSH, Brute Force, Sudo and Sudo Caching, Create Account).
