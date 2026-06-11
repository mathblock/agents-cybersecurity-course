# 2. Architecture sécurisée d’AegisBot

## 2.1 Principe d’architecture

AegisBot ne doit jamais appeler directement les systèmes critiques. Le modèle propose, l’orchestrateur décide, la policy autorise ou refuse, la tool gateway exécute uniquement les actions permises, et l’humain valide les actions sensibles.

```txt
Utilisateur SOC
    |
    v
Interface AegisBot
    |
    v
Orchestrateur agentique
    |
    +--> Policy layer / Risk engine
    |        - classification action
    |        - contrôle identité/scopes
    |        - règles human approval
    |        - DLP entrée/sortie
    |
    +--> RAG sécurisé
    |        - documents signés/versionnés
    |        - score de confiance
    |        - séparation données/instructions
    |
    +--> Tool gateway
             |
             +--> SIEM read-only
             +--> Logs Kubernetes filtrés read-only
             +--> Jira création limitée
             +--> Slack brouillon / envoi contrôlé
             +--> Sandbox diagnostic read-only
             +--> Registry MCP approuvé
```

## 2.2 Séparation modèle / systèmes critiques

- Le LLM ne reçoit aucun secret brut.
- Le LLM ne détient aucun token de production.
- Les tools sont appelés via une gateway, jamais directement depuis le modèle.
- Les commandes sont représentées par des intentions structurées, pas par un shell libre.
- Les actions critiques sont bloquées ou mises en attente de validation humaine.

## 2.3 Composants attendus

| Composant | Rôle sécurité |
|---|---|
| Interface utilisateur | authentification, contexte utilisateur, avertissements |
| Orchestrateur | contrôle du cycle agentique, séparation faits/hypothèses/actions |
| Policy layer | décision autoriser/refuser/demander validation |
| Tool gateway | point unique d’exécution contrôlée |
| Sandbox | diagnostics isolés, sans privilèges prod |
| RAG sécurisé | documents gouvernés, provenance et confiance |
| IAM | identité agent dédiée, RBAC minimal |
| Secret manager | secrets hors prompt, injection minimale côté tool si nécessaire |
| Human approval | frein d’urgence et décision métier |
| Audit logs | traçabilité complète et immuable |
| Monitoring | détection d’abus, dérive, erreurs |
| Skill registry | validation, version, signature des skills |
| MCP registry | connecteurs approuvés, scopes, versioning |

## 2.4 Gestion des permissions

- Scopes séparés : lecture, écriture, exécution, diffusion.
- En production : lecture fortement filtrée, écriture interdite par défaut.
- En sandbox : exécution read-only limitée possible.
- Secrets : accès interdit au LLM ; seuls certains tools backend peuvent y accéder, avec masquage en sortie.

## 2.5 Validation des tools

Chaque tool doit déclarer :

- nom stable ;
- finalité métier ;
- schéma strict des arguments ;
- scopes nécessaires ;
- environnements autorisés ;
- niveau de criticité ;
- politique de logs ;
- exemples autorisés et interdits ;
- propriétaire technique ;
- version.

Tout tool inconnu ou modifié sans validation est refusé.

## 2.6 Exécution de commandes

AegisBot peut proposer des commandes, mais ne peut exécuter automatiquement que des diagnostics read-only approuvés dans une sandbox.

Les diagnostics Kubernetes autorisés doivent cibler uniquement un `<namespace-autorisé>` issu d’une allowlist validée par la policy ; la sandbox, le filtrage/masquage des résultats et le RBAC doivent empêcher tout accès aux namespaces sensibles.

Autorisés en sandbox :

```txt
kubectl get pods -n <namespace-autorisé>
kubectl describe pod <pod> -n <namespace-autorisé>
kubectl logs <pod> -n <namespace-autorisé> --tail=200
kubectl get events -n <namespace-autorisé>
```

Interdits automatiquement :

```txt
kubectl delete
kubectl apply
kubectl patch
kubectl edit
kubectl exec
helm upgrade
systemctl restart
rm -rf
curl | sh
```

## 2.7 RAG sécurisé

- Index séparés par sensibilité.
- Documents avec propriétaire, date, version et statut de validation.
- Score de confiance affiché dans la réponse.
- Citations obligatoires pour les recommandations.
- Conflits documentaires remontés explicitement.
- Instructions trouvées dans documents traitées comme contenu non fiable.

## 2.8 Logs d’audit

Les logs doivent être immuables et contenir : utilisateur, agent, modèle, prompt système, objectif, sources consultées, score de confiance, tools appelés, arguments, résultats filtrés, décision policy, action proposée, action exécutée, validation humaine, erreurs/refus.

## 2.9 Actions sensibles

Validation humaine obligatoire pour :

- toute modification production ;
- toute exécution en production ;
- accès ou manipulation de secrets ;
- envoi Slack contenant données sensibles ;
- fermeture d’incident ;
- désactivation ou modification SIEM ;
- appel réseau externe non approuvé ;
- activation d’une skill ou MCP non encore validé.

## 2.10 Shadow tools / shadow MCP

- Registry central obligatoire.
- Allowlist par environnement.
- Signature et hash des connecteurs.
- Inventaire continu des serveurs MCP.
- Blocage des serveurs locaux ou externes non déclarés.
- Alerting si un tool nouveau apparaît ou si un scope change.
