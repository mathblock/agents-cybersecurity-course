# Livrable 3 — Matrice de permissions (20 points)

**Entreprise :** Northwind Health Cloud | **Agent :** AegisBot

---

## Principe directeur : moindre privilège

> Un **tool est une permission**. Accorder un tool à AegisBot, c'est lui accorder un accès réel à un système. Chaque permission doit être justifiée, minimale et tracée.

**Niveaux d'accès utilisés dans cette matrice :**

| Symbole | Niveau | Description |
|---|---|---|
| ✅ AUTO | Autonome | L'agent peut agir sans validation humaine |
| 🟡 L1 | Validation simple | Un analyste SOC doit approuver |
| 🟠 L2 | Validation renforcée | Un analyste senior doit approuver |
| 🔴 L3 | Double validation | Senior + RSSI doivent approuver |
| ⛔ INTERDIT | Bloqué techniquement | Impossible, même sur instruction explicite |

---

## Matrice complète

### Lecture et consultation

| # | Action | Niveau | Scope technique | Justification |
|---|---|---|---|---|
| 1 | Lire une alerte SIEM | ✅ AUTO | `siem:read` (alertes uniquement) | Lecture pure, indispensable à la mission de l'agent |
| 2 | Lire des logs Kubernetes filtrés | ✅ AUTO | `k8s:logs:read` (namespaces autorisés, sans secrets) | Lecture filtrée, pas d'accès aux namespaces système |
| 3 | Lire des secrets Kubernetes | ⛔ INTERDIT | — | Les secrets ne doivent jamais transiter par le LLM ; utiliser Vault |
| 4 | Consulter un runbook validé | ✅ AUTO | RAG read-only (documents fiables uniquement) | Source interne de confiance — lecture pure |
| 5 | Consulter un ticket d'incident | ✅ AUTO | `jira:read` (projet SOC uniquement) | Données traitées comme non fiables ; lecture uniquement |
| 6 | Lire les messages Slack du canal SOC | ✅ AUTO | `slack:channels:history` (canal `#soc` uniquement) | Données non fiables ; lecture uniquement |
| 7 | Consulter les règles Sigma existantes | ✅ AUTO | `siem:rules:read` | Lecture pure des règles en place, utile pour proposer des améliorations |
| 8 | Consulter un rapport post-mortem | ✅ AUTO | RAG read-only (classé non fiable) | Données historiques, traitées comme non fiables |
| 9 | Accéder à des données de santé patient | ⛔ INTERDIT | — | Hors périmètre absolu d'AegisBot ; violation RGPD/HDS |
| 10 | Accéder aux credentials / fichiers `.env` | ⛔ INTERDIT | — | Risque de compromission totale de l'infrastructure |

---

### Analyse et génération (sortie texte uniquement)

| # | Action | Niveau | Scope technique | Justification |
|---|---|---|---|---|
| 11 | Proposer une analyse d'incident | ✅ AUTO | Aucun (sortie texte) | Aucun effet de bord ; l'agent formule, l'humain décide |
| 12 | Générer une commande Linux de diagnostic | ✅ AUTO | Aucun (sortie texte) | L'agent propose, n'exécute pas ; aucun accès système |
| 13 | Proposer une règle Sigma | ✅ AUTO | Aucun (sortie texte) | Génération de texte, la règle doit être validée avant déploiement |
| 14 | Aider à rédiger un rapport d'incident | ✅ AUTO | Aucun (sortie texte) | Sortie texte, relecture humaine obligatoire avant publication |

---

### Création et écriture contrôlée

| # | Action | Niveau | Scope technique | Justification |
|---|---|---|---|---|
| 15 | Créer un ticket dans `SOC-INCIDENTS` | ✅ AUTO | `jira:issues:create` (projet SOC uniquement) | Action limitée à un projet dédié, réversible, tracée |
| 16 | Modifier un ticket existant | 🟡 L1 | `jira:issues:edit` (projet SOC uniquement) | Modification d'un ticket = risque de perte d'information ; validation analyste requise |
| 17 | Fermer un ticket | 🟠 L2 | `jira:issues:transition` | Fermer un incident = décision significative ; risque de masquer une vraie menace |
| 18 | Envoyer un résumé dans `#soc-alerts` | ✅ AUTO | `slack:chat:write` (canal `#soc-alerts` uniquement) | Canal dédié SOC, contenu généré contrôlé, pas de données sensibles |
| 19 | Envoyer un message dans un autre canal Slack | ⛔ INTERDIT | — | Risque d'exfiltration de données vers un canal non sécurisé |
| 20 | Envoyer des informations à un destinataire externe | ⛔ INTERDIT | — | Risque d'exfiltration de données hors périmètre |

---

### Exécution de commandes

| # | Action | Niveau | Scope technique | Justification |
|---|---|---|---|---|
| 21 | Exécuter une commande de diagnostic (liste blanche) | 🟡 L1 | Sandbox isolée, liste blanche stricte | Validation humaine avant exécution ; sandbox limite le rayon d'explosion |
| 22 | Exécuter une commande non présente dans la liste blanche | ⛔ INTERDIT | — | Risque d'exécution de code arbitraire (RCE) |
| 23 | Exécuter une commande en production directement | ⛔ INTERDIT | — | Contrainte fondamentale ; toujours via sandbox + validation |
| 24 | Installer une dépendance | ⛔ INTERDIT | — | Risque de supply chain attack ; aucun accès package manager |
| 25 | Modifier une configuration Kubernetes | 🔴 L3 | `k8s:apply` (namespaces non-prod uniquement) | Double validation obligatoire ; interdit en production |
| 26 | Modifier une configuration Kubernetes en production | ⛔ INTERDIT | — | Contrainte absolue ; risque d'interruption de service |
| 27 | Escalader ses privilèges (sudo, SUID) | ⛔ INTERDIT | — | L'agent s'exécute sous un compte dédié sans droits d'administration |

---

### Gestion des règles de sécurité

| # | Action | Niveau | Scope technique | Justification |
|---|---|---|---|---|
| 28 | Proposer une modification de règle SIEM | ✅ AUTO | Aucun (sortie texte) | Proposition uniquement, pas d'application directe |
| 29 | Appliquer une modification de règle SIEM | 🔴 L3 | `siem:rules:write` | Toute modification de règle = angle mort potentiel ; double validation requise |
| 30 | Désactiver une règle SIEM | ⛔ INTERDIT | — | Désactiver une règle = aveugler la détection ; jamais autorisé de manière autonome, même avec L3 |
| 31 | Désactiver une alerte | ⛔ INTERDIT | — | Même risque que désactiver une règle SIEM |

---

### Gouvernance et connecteurs

| # | Action | Niveau | Scope technique | Justification |
|---|---|---|---|---|
| 32 | Utiliser une skill validée du registry | ✅ AUTO | Registry signé uniquement | Les skills validées sont auditées et sécurisées |
| 33 | Utiliser une skill non validée | ⛔ INTERDIT | — | Une skill est du logiciel ; une skill non validée est un risque supply chain |
| 34 | Appeler un serveur MCP approuvé | ✅ AUTO | Allowlist de connecteurs signés | Connecteurs validés, scopes minimaux, monitoring actif |
| 35 | Appeler un serveur MCP non approuvé | ⛔ INTERDIT | — | Shadow MCP = vecteur d'exfiltration ou d'escalade de privilèges |
| 36 | Créer ou modifier un compte utilisateur | ⛔ INTERDIT | — | Risque de backdoor et d'élévation de privilèges |
| 37 | Supprimer des logs ou des tickets | ⛔ INTERDIT | — | Destruction de preuves, violation de non-répudiation |

---

## Synthèse visuelle

```
                    LECTURE    ÉCRITURE    EXÉCUTION    DESTRUCTIF
                    ────────   ────────    ─────────    ──────────
SIEM alertes         ✅ AUTO    ⛔ INTERDIT  ⛔ INTERDIT   ⛔ INTERDIT
SIEM règles          ✅ AUTO    🔴 L3        —            ⛔ INTERDIT
K8s logs filtrés     ✅ AUTO    ⛔ INTERDIT  ⛔ INTERDIT   ⛔ INTERDIT
K8s config           ⛔ INTERDIT 🔴 L3(non-prod) —        ⛔ INTERDIT
K8s secrets          ⛔ INTERDIT ⛔ INTERDIT  —           ⛔ INTERDIT
Jira (SOC project)   ✅ AUTO    🟡 L1        —            ⛔ INTERDIT
Jira (fermeture)     ✅ AUTO    🟠 L2        —            ⛔ INTERDIT
Slack (#soc-alerts)  ✅ AUTO    ✅ AUTO      —            —
Slack (autres)       ⛔ INTERDIT ⛔ INTERDIT  —           —
Sandbox (diag.)      —         —            🟡 L1         ⛔ INTERDIT
Sandbox (prod)       ⛔ INTERDIT ⛔ INTERDIT  ⛔ INTERDIT  ⛔ INTERDIT
Skills (validées)    ✅ AUTO    —            ✅ AUTO       ⛔ INTERDIT
Skills (non validées) ⛔ INTERDIT —          ⛔ INTERDIT  ⛔ INTERDIT
MCP (approuvé)       ✅ AUTO    selon action  selon action ⛔ INTERDIT
MCP (non approuvé)   ⛔ INTERDIT ⛔ INTERDIT  ⛔ INTERDIT  ⛔ INTERDIT
Secrets / Vault      ⛔ INTERDIT ⛔ INTERDIT  —           ⛔ INTERDIT
Données de santé     ⛔ INTERDIT ⛔ INTERDIT  —           ⛔ INTERDIT
```

---

## Note sur l'application technique

Les niveaux ⛔ INTERDIT ne sont **pas** uniquement interdits par le prompt système. Ils sont **bloqués techniquement** :
- par la gateway de tools (absence du scope dans l'allowlist) ;
- par les permissions IAM du compte de service d'AegisBot ;
- par les règles réseau (blocage des endpoints non autorisés) ;
- par la sandbox (seccomp, AppArmor, pas d'accès réseau externe).

Le prompt système est le dernier recours, pas le premier. La sécurité est côté architecture.
