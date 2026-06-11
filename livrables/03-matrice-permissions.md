# 3. Matrice de permissions d’AegisBot

| Action | Décision | Conditions | Justification |
|---|---|---|---|
| Lire une alerte SIEM | Autorisé | read-only, utilisateur SOC authentifié | nécessaire au triage |
| Lire logs Kubernetes filtrés | Autorisé conditionnel | namespace autorisé, masquage secrets, limite volume | utile mais sensible |
| Lire logs bruts non filtrés | Interdit par défaut | exception humaine documentée | risque de secrets |
| Lire secrets Kubernetes | Interdit | aucune exception côté LLM | secrets hors contexte modèle |
| Consulter runbook validé | Autorisé | afficher version/propriétaire/confiance | support décisionnel |
| Consulter document non validé | Autorisé conditionnel | marqué non fiable, pas traité comme instruction | prévention injection indirecte |
| Créer un ticket Jira | Autorisé conditionnel | brouillon ou création avec template validé | action réversible mais traçable |
| Modifier un ticket Jira | Validation humaine | préciser champ et raison | risque désorganisation |
| Fermer un ticket | Validation humaine obligatoire | preuve et approbateur | action critique |
| Générer une commande Linux | Autorisé | sortie en proposition, non exécutée | aide opérateur |
| Exécuter commande diagnostic read-only sandbox | Autorisé conditionnel | allowlist stricte, logs, pas de secrets | utile et borné |
| Exécuter commande en production | Validation humaine obligatoire | break-glass, journalisation renforcée | impact prod possible |
| `kubectl get/describe/logs` sandbox | Autorisé conditionnel | namespace autorisé, limite sortie | diagnostic limité |
| `kubectl delete/patch/apply/edit` | Interdit automatique | uniquement humain hors agent | destructif/modifiant |
| `kubectl exec` | Interdit automatique | exception manuelle hors agent | accès interactif risqué |
| Désactiver règle SIEM | Interdit automatique | CAB ou validation sécurité hors agent | perte de détection |
| Proposer règle Sigma | Autorisé | marquée brouillon, revue humaine | production de contenu non destructif |
| Déployer règle Sigma | Validation humaine obligatoire | test + approbation | risque faux positifs/négatifs |
| Envoyer résumé Slack canal SOC interne | Autorisé conditionnel | DLP, canal allowlist, pas de secret | communication contrôlée |
| Envoyer résumé externe | Validation humaine obligatoire | revue contenu/destinataire | risque fuite |
| Installer dépendance | Interdit automatique | pipeline approuvé uniquement | supply chain |
| Utiliser skill validée | Autorisé conditionnel | version signée, scopes compatibles | module audité |
| Utiliser skill non validée | Interdit | revue préalable | skill = logiciel |
| Appeler MCP approuvé | Autorisé conditionnel | scopes minimaux, logs | connecteur gouverné |
| Appeler serveur MCP non approuvé | Interdit | ajout au registry requis | shadow MCP |
| Accéder à Internet | Interdit par défaut | allowlist + validation | exfiltration |
| Révéler données santé/client | Interdit | masquage, synthèse minimale | conformité |
| Répondre sans preuve | Interdit fonctionnel | demander source ou signaler incertitude | overreliance |

## Règles transverses

1. Lecture ≠ écriture ≠ exécution : chaque niveau a un scope séparé.
2. Toute action modifiant production nécessite validation humaine.
3. Les secrets ne doivent jamais être transmis au LLM.
4. Les documents non fiables ne peuvent pas modifier les règles de l’agent.
5. Toute action doit produire une trace d’audit.
6. Les refus sont journalisés comme des événements de sécurité.
