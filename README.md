# Tri des mails Gmail avec Claude (n8n)

Workflow n8n Cloud qui lit les nouveaux mails Gmail chaque heure, demande à Claude
de les classer et de les résumer, puis ajoute une ligne par mail dans un Google Sheet.

## Fichiers

| Fichier | Rôle |
|---|---|
| `workflow-principal-v2.json` | **Version à importer** : workflow principal durci (branche erreur + reprise toutes les 6 h). En production depuis le 2026-10-08. |
| `workflow-principal.json` | Première version du workflow principal (étape 2), gardée pour l'historique. |
| `workflow-alerte.json` | Workflow d'alerte par mail quand le workflow principal échoue. |
| `prompt-claude.md` | Le prompt envoyé à Claude et le choix des paramètres. |
| `guide-import.md` | Étapes pour importer et brancher les workflows dans n8n. |
| `workflow-gmail-claude-sheets.md` | Document de conception. |

## Mettre à jour le dépôt après une modification dans n8n

1. Dans n8n, ouvrir le workflow, menu `...` puis **Download**.
2. Retirer les données épinglées (pinned data) avant l'export : elles peuvent contenir de vrais mails.
3. Remplacer le fichier `.json` correspondant dans ce dossier.
4. `git add -A` puis `git commit -m "Ce qui a changé"`.

Les clés API restent dans les credentials de n8n : les fichiers exportés ne contiennent
que l'identifiant et le nom du credential, jamais la clé elle-même.
