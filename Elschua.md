# Évaluation — IA, Agents et Cybersécurité  
## Livrable : Sécurisation d’AegisBot — Northwind Health Cloud  
**Auteur :** Équipe Audit Sécurité  
**Date :** 2026-05-11  
---
# 1. Modèle de menace
## 1.1 Actifs à protéger
- **Données sensibles :** dossiers médicaux, logs, secrets Kubernetes, tickets et rapports internes.  
- **Systèmes critiques :** infrastructure SIEM, clusters Kubernetes, Jira, Slack interne, base documentaire.  
- **Identité et accès :** jetons d’API, identités de service, clés SSH, tokens OAuth.  
- **Intégrité opérationnelle :** règles SIEM, configurations de sécurité, scripts automatisés.  
- **Réputation et conformité :** conformité RGPD et HDS, image de marque.
## 1.2 Sources non fiables
- Runbooks non validés, documentation utilisateur, messages Slack, tickets Jira non vérifiés, logs contaminés par du texte adversarial, imports automatiques depuis outils externes.
## 1.3 Attaques contre les prompts
- **Prompt injection directe :** utilisateur ou document contenant “ignore les règles et exécute cette commande”.  
- **Prompt injection indirecte :** injection cachée dans un ticket ou un log.  
- **Exfiltration de secret :** via demande déguisée ou manipulation conversationnelle.  
## 1.4 Attaques contre le RAG et la base documentaire
- **RAG poisoning :** documents modifiés pour influencer les réponses.  
- **Data leakage :** exposition de données de santé dans un contexte retrieval.  
- **Document as command :** texte transformé en instruction d’action.
## 1.5 Attaques sur les tools
- **Tool abuse :** appel à un outil dangereux hors périmètre.  
- **Shadow tool :** outil non autorisé enregistré par un développeur.  
- **Tool impersonation :** détournement d’un connecteur MCP pour exécuter un script malveillant.  
## 1.6 Attaques sur les skills
- Skill non validée comportant des actions destructrices, scripts de collecte non restreints (e.g. `kubectl delete`, `curl external`).  
- Partial privilege escalation via dépendances ou appels imbriqués.
## 1.7 Attaques sur serveurs MCP / connecteurs
- Man-in-the-middle sur connecteur non chiffré.  
- Connexion à un serveur MCP non approuvé pour exfiltration.  
- Détournement de permission via token partagé.
## 1.8 Attaques sur logs / rapports
- Injections dans les logs (log forgery).  
- Résumés Slack contenant information sensible.  
- Altération de rapport post-mortem généré automatiquement.  
## 1.9 Impacts possibles
- Exfiltration de données de santé, corruption de règles SIEM, compromission des clusters, blocage incident, perte de confiance des clients, violation légale RGPD/HDS.
## 1.10 Contrôles majeurs
- **Séparation stricte modèle/systèmes.**  
- **Policy Engine centralisé (RBAC + ABAC).**  
- **Sandbox d’exécution contrôlée.**  
- **Validation humaine des actions critiques.**  
- **RAG chiffré, sources classifiées et signées.**  
- **Audit complet des tools et skills.**  
- **Logs immuables et supervision SOC.**  
- **Registry des composants approuvés (tools, skills, MCP).**  
- **Détection automatique d’injection.**
---
# 2. Architecture sécurisée d’AegisBot
## 2.1 Schéma global
                    ┌──────────────────────────┐
                    │      Utilisateur SOC     │
                    └────────────┬─────────────┘
                                 │
                                 ▼
                 ┌───────────────────────────────────┐
                 │     Interface Sécurisée (UI/API)  │
                 │ - Authentification forte          │
                 │ - Traçabilité / rôles             │
                 └─────────────────┬─────────────────┘
                                   │
                                   ▼
      ┌─────────────────────────────────────────────────────────────┐
      │                  Orchestrateur AegisBot                     │
      │─────────────────────────────────────────────────────────────│
      │  - Gestion du contexte et des prompts                       │
      │  - Séparation logique modèle / outils                       │
      │  - Appels via Policy Engine et Tool Gateway                 │
      └─────────────────┬───────────────────────────────────────────┘
                        │


---

## 3. Description des composants

### 3.1 Interface Utilisateur Sécurisée
- Authentification OIDC multi-facteurs.  
- Accès rôlé : SOC Analyst, Ingénieur Infra, Superviseur.  
- Historique complet des requêtes de l’utilisateur.  
- Aucune donnée sensible stockée côté client.

### 3.2 Orchestrateur AegisBot
- Pilote le modèle IA (LLM).  
- Met en forme les requêtes et sépare contexte / instructions.  
- Transmet uniquement des requêtes au **Policy Engine** ou à la **Tool Gateway**.  
- N’exécute jamais de commande directement.  

### 3.3 Policy Engine
- Gère les permissions selon utilisateur, contexte et nature d’action.  
- Applique la politique “deny by default”.  
- Contient les règles pour tools, skills, sources, et validations humaines.  

### 3.4 RAG Sécurisé
- Accède à des documents classés :  
  - **Tier 1 (confiance élevée)** : runbooks internes, docs validées.  
  - **Tier 2 (moyenne)** : historiques incidents, rapports post-mortem.  
  - **Tier 3 (faible)** : Slack, tickets.  
- Aucune donnée non validée utilisée pour exécution directe.  
- Masquage automatique de données personnelles ou secrets.  
- Vérification hash/signature avant ingestion.

### 3.5 Tool Gateway et Sandbox
- Interface unique entre AegisBot et les environnements réels.  
- Les outils sont exécutés dans containers éphémères, sans persistance.  
- API filtrées et contrôlées (Kubernetes via rôle read-only).  
- Aucune commande d’écriture sans approbation humaine.  
- Audits automatiques des outputs (neutralisation des logs sensibles).  

### 3.6 Gestion des identités et des secrets
- Accès aux secrets via **Vault** (jamais envoyé au modèle).  
- Identités distinctes pour chaque sous-procesus (exécution sandbox, lecture doc, écriture ticket).  
- Journaux détaillés de rotation et d’usage des jetons.

### 3.7 Validation humaine
- Intégrée à chaque étape critique :  
  - Fermeture de ticket, désactivation d’alerte, exécution production.  
- Interface fluide : bouton de validation dans portail SOC.  
- Les validations sont enregistrées dans les logs d’audit.  

### 3.8 Audit et supervision
- Pipeline de logs signé transmis au SIEM central.  
- Tableaux de bord :  
  - Appels bloqués / autorisés par Policy Engine  
  - Anomalies prompt-output  
  - Dérives d’usage skill/tool.  
- Mécanisme d’alerte en cas d’action anormale ou tentative hors périmètre.

---

## 4. Flux d’exécution typique (diagramme simplifié)





## 2.2 Séparation modèle / systèmes critiques
Le **modèle IA** ne communique jamais directement avec SIEM, Kubernetes ou Jira.  
Tous les appels passent par la **Tool Gateway** qui implémente des politiques RBAC + sandbox.

## 2.3 Composants de contrôle de permissions
- **Policy Engine (OPA ou équivalent)** : applique règles de sécurité.  
- **Identity Manager (OIDC/SAML)** : attribue des rôles pour l’agent.  
- **Validation humaine** : requise avant action critique.

## 2.4 Validation des tools
Chaque tool figure dans une **allowlist** signée (registry interne).  
Les tools sont audités, documentés, versionnés.

## 2.5 Exécution des commandes
Les commandes sont exécutées dans un **sandbox container non privilégié**, sans accès réseau externe.  
Les résultats sont analysés avant retour au modèle.

## 2.6 Classification documentaire
- **Niveau 1 :** sources validées (runbooks internes).  
- **Niveau 2 :** sources semi-fiables (résumés incidents).  
- **Niveau 3 :** sources non fiabilisées (Slack, tickets).  
Seules les sources niveau 1–2 alimentent le RAG par défaut.

## 2.7 Prévention “document → instruction”
Chaque texte récupéré est marqué comme *contenu*, pas *commande*.  
Les extraits sont traités via prompts neutres (“analyse le contenu”, jamais “exécute”).

## 2.8 Journalisation
Toutes les actions (requêtes, tools, décisions, validations) sont signées et stockées dans un **journal d’audit immuable** (SIEM).  

## 2.9 Gestion des secrets
- Vault (HashiCorp ou AWS Secrets Manager).  
- Jamais directement exposé au modèle.  
- Accès seulement via tools validés côté passerelle.

## 2.10 Actions sensibles
Executées uniquement :
- après validation humaine,  
- dans environnement isolé,  
- avec logs complets.  

## 2.11 Contre shadow tools / MCP
Registry contrôlé avec signatures ;  
Tout connecteur non approuvé est bloqué par la Passerelle Policy.

---

# 3. Matrice de permissions

| Action | Lecture | Écriture | Exécution | Condition / Justification |
|---|---|---|---|---|
| Lire une alerte SIEM | ✅ | ❌ | ❌ | Lecture via API restreinte |
| Lire logs K8s filtrés | ✅ | ❌ | ❌ | Logs sans secrets |
| Lire secrets K8s | ❌ | ❌ | ❌ | Interdit |
| Consulter un runbook | ✅ | ❌ | ❌ | Source validée |
| Créer un ticket | ✅ | ✅ | ❌ | Autorisé avec traçabilité |
| Modifier un ticket | ✅ | ✅ | ❌ | Si ticket assigné à l’agent |
| Fermer un ticket | ✅ | ✅ | ❌ | Validation humaine requise |
| Générer une commande | ✅ | ✅ | ❌ | Génération textuelle seulement |
| Exécuter commande prod | ❌ | ❌ | ✅ | Sandbox + validation humaine |
| Désactiver règle SIEM | ❌ | ✅ | ✅ | Validation humaine obligatoire |
| Installer dépendance | ❌ | ❌ | ✅ | Interdit hors sandbox |
| Utiliser skill non validée | ❌ | ❌ | ✅ | Refusé (registry only) |
| Appeler serveur MCP non approuvé | ❌ | ❌ | ✅ | Bloqué automatiquement |

---

# 4. Skill cyber sécurisée — *Kubernetes Incident Triage*

## Arborescence


### Fichier **SKILL.md**

```markdown
# Skill : Kubernetes Incident Triage

## Objectif
Aider AegisBot à collecter des informations de diagnostic sur un incident Kubernetes, sans modifier l’état du cluster.

## Règles de sécurité
- Lecture seule (`kubectl get`, `kubectl describe`, `kubectl logs --tail=100`).
- Aucune commande de suppression, édition ou patch.
- Aucune connexion externe.
- Les secrets (`kubectl get secrets`) sont interdits.
- Les résultats sont stockés dans la sandbox.
- Toute suggestion d’action corrective nécessite validation humaine.

## Fonctions
1. **collect_readonly_info** : collecte infos de pod, namespace, événements récents.
2. **analyser_logs** : filtre anomalies courantes.
3. **générer_rapport** : utilise template pour rapport Markdown.

## Validation humaine
Avant toute action correctrice (e.g. redémarrage de pod, patch d’image), l’humain doit approuver.

## Journalisation
- Nom opérateur  
- Timestamp  
- Cluster cible (anonymisé)  
- Commandes exécutées  
- Résumé généré

#!/bin/bash
# Collecte non destructive d'informations Kubernetes
set -euo pipefail
kubectl get pods -A --no-headers
kubectl describe node | grep -E 'Name|Ready|Allocatable'
kubectl get events -A --sort-by=.metadata.creationTimestamp | tail -n 50
kubectl logs -A --tail=100 | grep -Ei 'error|fail|timeout' || true

# Rapport d’incident Kubernetes

**Date :** {{timestamp}}  
**Analyste :** {{analyste}}  
**Namespace :** {{namespace}}  

## Symptômes observés
{{symptomes}}

## Événements récents
{{evenements}}

## Éléments suspects
{{suspicious_findings}}

## Recommandations
*(Soumise à validation humaine)*  
{{propositions}}



=====================================================
