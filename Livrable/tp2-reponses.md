# TP2 — Conception d'un agent cyber : CyberOps Copilot

---

## Rappel du scénario

**CyberOps Copilot** est un agent IA mis en production pour assister l'équipe cyber dans les tâches suivantes :
1. Lire des alertes SIEM
2. Lire des logs applicatifs
3. Chercher dans des runbooks internes
4. Proposer une analyse
5. Créer un ticket Jira
6. Proposer une commande Linux
7. Exécuter certaines commandes de diagnostic
8. Envoyer un résumé Slack

**Contrainte absolue :** L'agent ne doit jamais pouvoir modifier la production sans validation humaine.

---

## Livrable 1 — Matrice de permissions

La matrice ci-dessous liste chaque capacité de l'agent et le niveau d'accès accordé, selon trois principes :
- **Lecture seule** : l'agent consulte sans écrire.
- **Écriture contrôlée** : l'agent peut agir, mais dans un périmètre limité et tracé.
- **Interdit** : l'agent ne peut pas réaliser l'action, même si on le lui demande.

| Capacité | Ressource cible | Type d'accès | Qui peut autoriser | Remarque |
|---|---|---|---|---|
| Lire des alertes SIEM | SIEM (alertes) | 🟢 Lecture seule | Automatique | API read-only, scope limité aux alertes |
| Lire des logs applicatifs | Serveurs de logs (ELK, Splunk…) | 🟢 Lecture seule | Automatique | Pas d'accès aux logs de sécurité RH ou légaux |
| Chercher dans les runbooks | Base de connaissance interne (RAG) | 🟢 Lecture seule | Automatique | Documents indexés et contrôlés par l'équipe SOC |
| Proposer une analyse | Aucune (sortie texte) | 🟢 Aucun accès système | Automatique | Génération de texte uniquement |
| Créer un ticket Jira | Jira (projet SOC) | 🟡 Écriture contrôlée | Automatique (scope restreint) | Uniquement dans le projet `SOC-INCIDENTS`, pas en tant qu'admin |
| Proposer une commande Linux | Aucune (sortie texte) | 🟢 Aucun accès système | Automatique | L'agent propose, n'exécute pas |
| Exécuter des commandes de diagnostic | Serveurs cibles (via sandbox) | 🟡 Écriture contrôlée | Validation humaine requise | Commandes pré-approuvées uniquement (liste blanche) |
| Envoyer un résumé Slack | Canal Slack `#soc-alerts` | 🟡 Écriture contrôlée | Automatique (canal dédié) | Lecture uniquement du canal, écriture dans `#soc-alerts` uniquement |
| Modifier la configuration de production | Infra, pare-feu, serveurs, BDD | 🔴 **INTERDIT** | Jamais | Contrainte fondamentale du TP |
| Supprimer des tickets ou des logs | Jira, SIEM, ELK | 🔴 **INTERDIT** | Jamais | Principe de non-répudiation |
| Accéder aux secrets / credentials | Vault, variables d'env, .env | 🔴 **INTERDIT** | Jamais | Isolation totale des secrets |
| Escalader des privilèges | Système d'exploitation | 🔴 **INTERDIT** | Jamais | L'agent s'exécute dans un compte dédié sans sudo |

---

## Livrable 2 — Architecture logique

L'architecture suit le modèle **"Read-Think-Propose-Act"** avec un point de contrôle humain avant toute action à effet de bord sur la production.

```
┌─────────────────────────────────────────────────────────────────┐
│                        SOURCES DE DONNÉES                       │
│                         (Lecture seule)                         │
│                                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────────────┐    │
│  │   SIEM   │   │  Logs    │   │  Runbooks internes (RAG) │    │
│  │ (alertes)│   │  apps    │   │  (base vectorielle)      │    │
│  └────┬─────┘   └────┬─────┘   └────────────┬─────────────┘    │
└───────┼──────────────┼──────────────────────┼─────────────────-┘
        │              │                       │
        └──────────────┴───────────────────────┘
                               │
                               ▼
        ┌──────────────────────────────────────┐
        │         CYBEROPS COPILOT (LLM)       │
        │                                      │
        │  System Prompt sécurisé :            │
        │  - Séparer faits / hypothèses        │
        │  - Données externes = non fiables    │
        │  - Ne jamais modifier la prod seul   │
        └──────────────────────┬───────────────┘
                               │
               ┌───────────────┼───────────────┐
               │               │               │
               ▼               ▼               ▼
     ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐
     │  ANALYSE &  │  │   ACTIONS    │  │  ACTIONS À EFFET │
     │  PROPOSITION│  │  AUTONOMES   │  │  DE BORD         │
     │  (texte)    │  │  (contrôlées)│  │  (bloquées)      │
     │             │  │              │  │                  │
     │ • Analyse   │  │ • Créer      │  │ • Modifier config│
     │ • Commande  │  │   ticket Jira│  │   production     │
     │   Linux     │  │ • Envoyer    │  │ • Supprimer logs │
     │   proposée  │  │   résumé     │  │ • Accéder secrets│
     └─────────────┘  │   Slack      │  └──────┬───────────┘
                      │ • Exécuter   │         │
                      │   diag       │         │
                      └──────┬───────┘         │
                             │                 │
                             ▼                 ▼
              ┌──────────────────────────────────────────┐
              │       HUMAN-IN-THE-LOOP (ANALYSTE SOC)   │
              │                                          │
              │  Pour les commandes de diagnostic :      │
              │  ✅ L'analyste valide avant exécution    │
              │                                          │
              │  Pour toute action de production :       │
              │  🔴 Validation obligatoire + traçabilité │
              └──────────────────────────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────────────────┐
              │            AUDIT & LOGS DE L'AGENT       │
              │   (Toutes les actions tracées et signées) │
              └──────────────────────────────────────────┘
```

### Composants clés

| Composant | Rôle | Technologie possible |
|-----------|------|----------------------|
| **LLM Core** | Raisonnement, analyse, génération de réponses | GPT-4o, Claude, Gemini |
| **Orchestrateur** | Gestion des outils et du flux d'exécution | LangGraph, AutoGen |
| **RAG Engine** | Recherche sémantique dans les runbooks | ChromaDB, Pinecone + embeddings |
| **Tool Layer** | Appels API vers SIEM, Jira, Slack, logs | SDK Python + API REST |
| **Human-in-the-loop** | Blocage + notification avant actions sensibles | Interface web ou Slack interactif |
| **Audit Logger** | Traçabilité de toutes les actions de l'agent | ELK, Datadog, ou fichier signé |
| **Sandbox d'exécution** | Environnement isolé pour les commandes de diagnostic | Docker container read-only, netns isolé |

---

## Livrable 3 — Liste des logs nécessaires

Pour garantir la traçabilité, la détection d'abus et l'auditabilité de l'agent, les logs suivants sont indispensables.

### Logs de l'agent lui-même

| Log | Contenu | Pourquoi ? |
|-----|---------|------------|
| **Log d'entrée** | Prompt utilisateur, timestamp, identifiant de session | Savoir qui a déclenché quoi et quand |
| **Log de raisonnement** | Étapes de raisonnement de l'agent (chain-of-thought si disponible) | Auditabilité et débogage |
| **Log d'appel d'outil** | Outil appelé, paramètres, résultat, timestamp | Tracer chaque action de l'agent |
| **Log de sortie** | Réponse finale de l'agent, recommandation produite | Archivage des analyses |
| **Log de validation humaine** | Décision de l'analyste (approuvé/refusé), motif, timestamp, identité | Traçabilité des décisions humaines |
| **Log d'erreur / refus** | Tentative d'action interdite, message d'erreur, contexte | Détecter les abus et injections de prompt |

### Logs des systèmes cibles

| Système | Log requis | Pourquoi ? |
|---------|------------|------------|
| **SIEM** | Historique des alertes lues par l'agent | Confirmer que l'agent n'a accédé qu'aux alertes autorisées |
| **Logs applicatifs** | Liste des fichiers/index consultés par l'agent | Vérifier le périmètre d'accès |
| **Jira** | Tickets créés par l'agent (auteur = `cyberops-copilot`) | Distinguer les tickets humains des tickets IA |
| **Slack** | Messages postés par le bot dans `#soc-alerts` | Auditabilité des communications |
| **Sandbox Linux** | Commandes exécutées, output, code retour, durée | Détecter les commandes anormales |
| **RAG Engine** | Requêtes effectuées, documents retournés | Vérifier que seuls des runbooks autorisés sont consultés |

### Exigences générales sur les logs

- **Horodatage standardisé** (ISO 8601 / UTC)
- **Identifiant de session unique** pour corréler tous les logs d'une même interaction
- **Intégrité** : logs signés ou stockés dans un système immuable (ex: WORM storage)
- **Rétention minimale** : 1 an (ou selon la politique de l'entreprise)
- **Accès aux logs de l'agent restreint** : ni l'agent ni les utilisateurs ordinaires ne doivent pouvoir modifier ces logs

---

## Livrable 4 — Actions autorisées et actions interdites

### ✅ Actions autorisées (liste blanche)

Ces actions peuvent être réalisées par l'agent, de façon autonome ou avec validation humaine selon le niveau de risque.

#### Autonomes (sans validation humaine)

| Action | Conditions |
|--------|-----------|
| Lire une alerte SIEM | Via API read-only, scope `soc:read` uniquement |
| Lire des logs applicatifs | Dans les index autorisés (liste blanche d'index) |
| Effectuer une recherche sémantique dans les runbooks | Sur les documents indexés et validés par l'équipe SOC |
| Produire une analyse textuelle | Sortie texte uniquement, pas d'effet de bord |
| Suggérer une commande Linux | Texte uniquement, l'agent ne l'exécute pas lui-même |
| Créer un ticket Jira dans le projet `SOC-INCIDENTS` | Avec le compte de service dédié, champs obligatoires renseignés |
| Envoyer un résumé dans `#soc-alerts` | Via le bot Slack dédié, canal unique autorisé |

#### Avec validation humaine obligatoire

| Action | Condition de validation |
|--------|------------------------|
| Exécuter une commande de diagnostic (liste blanche) | L'analyste SOC approuve via l'interface human-in-the-loop avant exécution |
| Bloquer une IP au niveau d'un WAF ou pare-feu | Validation d'un analyste senior + double validation si impacte la prod |
| Modifier une règle de détection SIEM | Validation du responsable SOC |

---

### 🔴 Actions interdites (liste noire)

Ces actions sont **bloquées techniquement** et ne peuvent être réalisées même si un utilisateur ou un document l'ordonne.

| Action interdite | Risque associé |
|-----------------|----------------|
| Modifier la configuration d'un serveur de production | Interruption de service, sabotage |
| Supprimer ou modifier des logs système | Destruction de preuves, non-conformité |
| Accéder aux fichiers de secrets (`.env`, Vault, SSH keys) | Vol de credentials, compromission de l'infrastructure |
| Exécuter des commandes non présentes dans la liste blanche | Exécution de code arbitraire, RCE |
| Escalader ses propres privilèges (sudo, SUID) | Prise de contrôle du système |
| Répondre à des instructions trouvées dans des données non fiables | Prompt injection (tickets, logs, alertes) |
| Marquer un incident comme faux positif sans preuve | Dissimulation d'une vraie menace |
| Agir sur la production sans validation humaine | Principe fondamental de la contrainte du projet |
| Créer, modifier ou supprimer des comptes utilisateurs | Risque de backdoor ou d'élévation de privilèges |
| Envoyer des données sensibles en dehors du périmètre autorisé | Exfiltration de données |

---

## Livrable 5 — Politique de validation humaine

### Principe général

> **Toute action à effet de bord irréversible ou impactant la production nécessite une validation humaine explicite avant exécution.**

L'agent ne peut jamais auto-approuver une action sensible, même si l'utilisateur le lui demande explicitement.

---

### Niveaux de validation

| Niveau | Déclencheur | Validateur requis | Délai max | Mécanisme |
|--------|-------------|-------------------|-----------|-----------|
| **L0 — Aucune validation** | Actions de lecture pure, génération de texte | — | — | Automatique |
| **L1 — Validation simple** | Exécution d'une commande de diagnostic, création de ticket | Analyste SOC (n'importe lequel) | 15 min | Bouton Approuver/Refuser dans l'interface ou Slack |
| **L2 — Validation renforcée** | Blocage d'IP, modification d'une règle SIEM | Analyste SOC senior | 30 min | Interface dédiée + justification obligatoire |
| **L3 — Double validation** | Toute action touchant la production (infrastructure, firewall, BDD) | Analyste senior + responsable SOC | 1h | Workflow d'approbation avec audit trail |

---

### Processus de validation (L1 à L3)

```
L'agent propose une action sensible
           │
           ▼
  L'agent génère une fiche de demande :
  - Action demandée (lisible par un humain)
  - Justification (pourquoi l'agent veut faire ça)
  - Risques identifiés
  - Systèmes impactés
           │
           ▼
  Notification envoyée au(x) validateur(s)
  (Slack @mention + email + interface web)
           │
           ▼
    L'analyste examine et décide :
    ┌──── APPROUVE ────┐     ┌──── REFUSE ────┐
    │                  │     │                │
    ▼                  │     ▼                │
L'agent exécute        │  L'agent ne fait rien│
l'action              │  et log le refus     │
Log de l'action +     │  Notification à      │
résultat              │  l'équipe SOC        │
    └──────────────────┘     └────────────────┘
```

---

### Règles complémentaires

| Règle | Détail |
|-------|--------|
| **Expiration automatique** | Si aucune validation n'est reçue avant le délai max, l'action est annulée et l'équipe est notifiée. |
| **Refus par défaut** | En cas de doute ou d'erreur technique, l'agent n'agit pas ("fail-safe"). |
| **Traçabilité des validations** | Chaque décision (approuvé/refusé) est enregistrée avec l'identité du validateur, la date et le motif. |
| **Impossibilité d'auto-validation** | L'utilisateur qui a initié la demande ne peut pas valider sa propre requête. |
| **Résistance aux injections** | Une instruction trouvée dans un ticket, un log ou une alerte ne peut pas déclencher une approbation automatique. |
| **Révocabilité** | Un analyste peut annuler une action approuvée tant qu'elle n'a pas été exécutée. |

---

### Illustration : cas concret d'une demande de blocage d'IP

```
Analyste : "Bloque l'IP 185.199.110.153"
      │
      ▼
Agent [L2] : "Action demandée : bloquer 185.199.110.153 au niveau du pare-feu.
              Justification : 287 tentatives SSH échouées détectées.
              Risque : si IP légitime (CDN, VPN d'entreprise), cela peut
              bloquer des utilisateurs valides.
              Validation requise : analyste senior."
      │
      ▼
[Analyste senior reçoit la notification Slack]
[Vérifie l'IP sur AbuseIPDB → score 98/100, confirmée malveillante]
[Approuve]
      │
      ▼
Agent exécute : règle de blocage ajoutée au pare-feu
Agent log : timestamp, action, IP, validateur, motif
Agent notifie #soc-alerts : "IP 185.199.110.153 bloquée. Validé par [analyste senior]."
```
