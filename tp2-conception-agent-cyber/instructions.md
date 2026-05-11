## Scénario

L’entreprise veut mettre en production **CyberOps Copilot**.

L’agent doit aider l’équipe cyber à :

```txt
1. Lire des alertes SIEM.
2. Lire des logs applicatifs.
3. Chercher dans des runbooks internes.
4. Proposer une analyse.
5. Créer un ticket Jira.
6. Proposer une commande Linux.
7. Exécuter certaines commandes de diagnostic.
8. Envoyer un résumé Slack.
```

Contrainte :

```txt
L’agent ne doit jamais pouvoir modifier la production sans validation humaine.
```

---

## Travail demandé

Les étudiants doivent produire :

1. une matrice de permissions ;
2. une architecture logique ;
3. une liste de logs nécessaires ;
4. une liste d’actions autorisées et une liste d’actions interdites ;
5. une politique de validation humaine.