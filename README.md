# SOC Lab — Wazuh (indexer & dashboard, base OpenSearch)

> Laboratoire de supervision de sécurité monté de bout en bout sur un réseau
> virtualisé : déploiement du SIEM, enrôlement d'agents multi-OS, simulation
> d'attaques depuis une machine offensive, analyse et tri des alertes.
>
> **Fiat KPONOU** — Mastère 1 Expert IT & Cybersécurité, Rouen
> [LinkedIn](https://linkedin.com/in/fiat-kponou) · [GitHub](https://github.com/MichelFiatinium)

---

## Pourquoi ce lab

J'avais participé à la mise en place d'un SOC lors d'un projet d'école, en équipe.
J'ai voulu le refaire seul, de zéro, pour maîtriser réellement chaque brique — du
déploiement du SIEM jusqu'au tri des alertes. Mon objectif : comprendre comment une
alerte remonte concrètement depuis une machine attaquée jusqu'au tableau de bord, et
surtout pourquoi elle ne remonte parfois pas (le dépannage de la chaîne de collecte
a été, de loin, la partie la plus formatrice — voir [`docs/RUNBOOK.md`](docs/RUNBOOK.md)).

> *Tu peux réécrire ce paragraphe avec tes propres mots si tu veux l'affiner.*

---

## Architecture

Quatre machines sur un réseau privé hôte VirtualBox (`192.168.154.0/24`) :

```
   ┌──────────────────────────┐        ┌──────────────────────────┐
   │  VM-KALI (attaquant)     │        │  Windows (hôte)          │
   │  192.168.154.30          │        │  192.168.154.1 · agent   │
   └───────────┬──────────────┘        └───────────┬──────────────┘
               │ attaques                           │ logs
               ▼                                     ▼
   ┌──────────────────────────┐        ┌──────────────────────────┐
   │  VM-AGENT-UBUNTU         │ logs   │        VM-SIEM            │
   │  192.168.154.20 · agent  │──────► │     192.168.154.10        │
   └──────────────────────────┘        │  Wazuh manager + indexer  │
                                        │  + dashboard (Docker)     │
                                        └──────────────────────────┘
```

| Machine | Rôle | IP | OS |
|---|---|---|---|
| VM-SIEM | Manager + Indexer + Dashboard (Docker, single-node) | `192.168.154.10` | Ubuntu Server |
| VM-AGENT-UBUNTU | Agent supervisé | `192.168.154.20` | Ubuntu |
| VM-KALI | Machine offensive | `192.168.154.30` | Kali Linux |
| Windows (hôte) | Agent supervisé (poste de travail) | `192.168.154.1` | Windows |

**Stack :** Wazuh 4.14.9, déployé via [`wazuh/wazuh-docker`](https://github.com/wazuh/wazuh-docker)
(single-node). Le Wazuh Indexer et le Wazuh Dashboard sont des composants basés
sur **OpenSearch** — Wazuh a quitté la stack Elastic depuis la version 4.x.

---

## Compétences mises en œuvre

- **Déploiement SIEM** : conteneurisation Docker, génération des certificats, paramétrage noyau (`vm.max_map_count`).
- **Réseau** : conception d'un plan d'adressage, cartes NAT + hôte-uniquement, IP fixes via netplan.
- **Enrôlement d'agents** : Linux et Windows, enregistrement auprès du manager.
- **Dépannage opérationnel** : diagnostic d'une chaîne de collecte en panne (voir [`docs/RUNBOOK.md`](docs/RUNBOOK.md)).
- **Simulation d'attaque** : force brute SSH, scan de ports, intégrité de fichiers, création de compte.
- **Analyse** : tri des alertes, écriture et test de règles (`wazuh-logtest`), mapping MITRE ATT&CK.

---

## Scénarios de détection

Le détail est dans [`docs/DETECTIONS.md`](docs/DETECTIONS.md).

| # | Scénario | MITRE | Détecté | Règle(s) |
|---|---|---|---|---|
| 1 | Force brute SSH | T1110.001 | ✅ | 5710 / 5712 |
| 2 | Création de compte Linux | T1136 | ✅ | 5902 |
| 3 | Abus de privilèges (sudo) | T1548.003 | ✅ | 5402 / 5403 |
| 4 | Cycle de vie compte Windows | T1136.001 | ✅ | 60109 / **60154 (niv. 12)** / 60111 |
| 5 | Scan de ports | T1046 | ❌ *(angle mort analysé)* | — |

**4 détections** (Linux, Windows, élévation de privilèges, brute force) **+ 1 angle
mort documenté** (un SIEM host-based ne voit pas un scan réseau — d'où l'intérêt d'une
sonde type Suricata).

---

## Résultats & réduction du bruit

Ligne de base mesurée : **331 alertes sur 24 h** (dont 32 échecs d'authentification).
Analyse détaillée et pistes de tuning dans [`docs/RESULTATS.md`](docs/RESULTATS.md).

---

## Limites de ce lab

- Faible volumétrie (quatre machines, pas un parc réel).
- Pas de vrais utilisateurs générant de l'activité métier.
- Pas de corrélation multi-sources avancée, pas de Threat Intelligence, pas de SOAR.
- Environnement de laboratoire : toutes les attaques sont jouées contre mes propres machines.

> Ces limites sont assumées : l'objectif était de maîtriser la chaîne de détection
> de bout en bout, pas de reproduire un SOC de production.
