# 1. Modèle de menace — AegisBot

## 1.1 Actifs à protéger

| Actif | Sensibilité | Risque principal |
|---|---:|---|
| Données de santé hébergées | Très élevée | fuite, non-conformité, atteinte patient/client |
| Données clients | Très élevée | fuite contractuelle ou réglementaire |
| Logs Kubernetes et applicatifs | Élevée | secrets, tokens, IP internes, chemins, erreurs sensibles |
| Tickets d’incident | Élevée | informations d’attaque, noms d’actifs, données personnelles |
| Documents internes et runbooks | Élevée | architecture, procédures, contournements possibles |
| Secrets techniques | Critique | compromission directe d’environnements |
| Configurations Kubernetes | Critique | modification ou destruction de ressources |
| Règles SIEM / Sigma | Élevée | désactivation de détection ou bruit opérationnel |
| Rapports d’incident | Élevée | fuite d’informations sensibles et réputationnelles |
| Identité technique d’AegisBot | Critique | abus des permissions de l’agent |
| Prompts, skills, tools, serveurs MCP | Critique | altération du comportement agentique |
| Logs d’audit de l’agent | Critique | perte de traçabilité, dissimulation d’abus |

## 1.2 Sources non fiables ou partiellement fiables

| Source | Niveau de confiance | Justification |
|---|---:|---|
| Prompt utilisateur | Faible à moyen | peut contenir demandes ambiguës ou malveillantes |
| Tickets d’incident | Faible | peuvent contenir instructions cachées ou commentaires hostiles |
| Messages Slack SOC | Faible à moyen | bruit, opinions, données sensibles, injections indirectes |
| Logs applicatifs/Kubernetes | Moyen | faits techniques utiles mais peuvent contenir secrets ou chaînes contrôlées par attaquant |
| Runbooks internes validés | Moyen à élevé | utiles mais peuvent être obsolètes ou empoisonnés |
| Documentation importée automatiquement | Faible à moyen | provenance et fraîcheur variables |
| Historique d’incidents | Moyen | utile mais contexte potentiellement dépassé |
| Règles Sigma existantes | Moyen à élevé | contrôles existants mais non forcément adaptés au cas |
| Serveurs MCP/connecteurs | Variable | dépend de l’approbation, version, scopes et audit |
| Skills | Variable | doivent être auditées comme du logiciel exécutable |

Règle centrale : une source documentaire est une **donnée**, pas une instruction. Seuls le prompt système validé, la politique de sécurité et l’orchestrateur de contrôle peuvent définir le comportement autorisé.

## 1.3 Menaces sur les prompts

| Menace | Exemple | Impact | Contrôles |
|---|---|---|---|
| Prompt injection directe | utilisateur demande d’ignorer les règles | fuite ou action interdite | hiérarchie d’instructions, policy engine, refus explicite |
| Prompt injection indirecte | ticket contenant `ignore les alertes` | mauvaise analyse SOC | classification des sources, isolation données/instructions |
| Injection multimodale | texte caché dans image ou PDF | contournement des règles | OCR contrôlé, étiquetage source, sandbox de parsing |
| Jailbreak métier | reformulation d’une action interdite | exécution non autorisée | validation d’intention, allowlist d’actions |
| Persistance dans mémoire | instruction malveillante mémorisée | compromission durable | mémoire limitée, expiration, revue, purge |

## 1.4 Menaces sur RAG et base documentaire

| Menace | Description | Impact | Contrôles |
|---|---|---|---|
| RAG poisoning | document interne empoisonné | procédure dangereuse suivie par l’agent | validation documentaire, signatures, scoring de confiance |
| Surpartage documentaire | documents trop sensibles indexés | fuite dans les réponses | DLP, filtrage par identité, minimisation |
| Documents obsolètes | runbooks dépassés | mauvaise recommandation | versioning, date de validation, propriétaire |
| Sources contradictoires | deux procédures incompatibles | décision non fiable | signalement de conflit, humain dans la boucle |
| Confusion donnée/instruction | runbook donne ordre au modèle | bypass du prompt système | wrapper RAG : “contenu non normatif” |

## 1.5 Menaces sur tools, skills et MCP

| Surface | Menace | Impact | Contrôles |
|---|---|---|---|
| Tool terminal | commande destructive | indisponibilité prod | sandbox, allowlist read-only, approbation humaine |
| Tool Jira | fermeture ou modification abusive | perte de suivi incident | scopes séparés créer/modifier/fermer, approbation |
| Tool Slack | fuite externe ou interne large | non-conformité | DLP, canaux autorisés, validation avant envoi |
| Tool SIEM | désactivation d’alerte | perte de détection | interdiction ou validation forte |
| Kubernetes API | modification de ressource | impact production | identité read-only, séparation prod/sandbox |
| Skill | script malveillant | exfiltration/action cachée | revue code, signature, registre approuvé |
| Serveur MCP | tool poisoning/shadow server | backdoor ou scope creep | registry, allowlist, attestation, audit logs |
| Arguments de tool | command injection | exécution arbitraire | schémas stricts, validation, échappement, refus shell libre |

## 1.6 Menaces sur logs et rapports générés

- Inclusion de secrets non masqués.
- Hallucination d’une preuve inexistante.
- Rapport trop confiant sans sources.
- Omission d’une instruction malveillante détectée.
- Falsification ou suppression de logs d’audit.
- Envoi automatique à une audience trop large.

Contrôles : logs immuables, masquage des secrets, séparation faits/hypothèses/recommandations, citations de sources, validation humaine avant diffusion externe ou sensible.

## 1.7 Impacts possibles

- Fuite de données de santé, données clients ou secrets.
- Exécution de commandes destructives en production.
- Désactivation ou affaiblissement de la détection SIEM.
- Mauvaise qualification d’un incident réel.
- Création, fermeture ou modification abusive de tickets.
- Propagation de procédures empoisonnées.
- Perte d’auditabilité en cas d’incident causé par l’agent.

## 1.8 Contrôles prioritaires

1. Identité dédiée AegisBot avec droits minimaux.
2. Policy layer obligatoire avant tout appel de tool.
3. Tool gateway avec allowlist, schémas d’arguments et journalisation.
4. Sandbox pour diagnostics, sans accès secrets et sans écriture production.
5. RAG gouverné : provenance, version, confiance, validation.
6. Human approval pour toute action critique ou diffusion sensible.
7. DLP et masquage de secrets sur entrées/sorties.
8. Registry approuvé de tools, skills et MCP.
9. Logs immuables et corrélables.
10. Red teaming régulier des prompts, RAG, tools, skills et connecteurs.
