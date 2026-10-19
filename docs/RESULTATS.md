# Résultats & analyse du bruit

## Volume d'alertes (ligne de base, 24 h)

Après enrôlement des agents et exécution des scénarios d'attaque, sur une fenêtre
de 24 heures :

| Indicateur | Valeur |
|---|---|
| Alertes totales | **331** |
| Échecs d'authentification | **32** |
| Authentifications réussies | **6** |
| Alertes de niveau 12+ | 0 |

📸 `screenshots/05-dashboard-overview.png`

**Répartition par niveau observée :** majorité de niveau 3 (événements système de
routine), puis des niveaux 4, 5, 7, 8 et quelques niveaux 10 (brute force).

**Top MITRE ATT&CK :** Password Guessing, SSH, Brute Force, Valid Accounts,
Sudo and Sudo Caching, Create Account.

---

## Ce que cette ligne de base apprend

Sur 331 alertes, seules quelques-unes correspondent à mes attaques volontaires.
La grande majorité est du **bruit de fond** : événements système de niveau 3,
usages sudo légitimes, authentifications normales. C'est exactement le problème
central d'un SOC : **distinguer le signal de l'ambiance**.

Mes 6 authentifications réussies sont mes propres connexions SSH d'administration —
du bruit légitime, mais qui, mélangé à une vraie attaque, peut masquer l'essentiel.

---

## Prochaine étape : le tuning (non encore réalisé)

> ⚠️ Honnêteté : à ce stade, aucune règle de réduction de bruit n'a encore été
> écrite. Le chiffre ci-dessus est une **ligne de base avant tuning**, pas un
> résultat d'optimisation.

Pistes de tuning identifiées pour la suite :
- Réduire ou regrouper les événements système de niveau 3 non pertinents.
- Créer une règle distinguant mes connexions d'admin (sources connues) du reste.
- Écrire une règle « succès d'authentification juste après une rafale d'échecs »
  (brute force réussi) — bien plus critique que les échecs seuls.

Le volume « après tuning » sera mesuré lors de cette prochaine itération, pour
chiffrer la réduction de bruit obtenue.
