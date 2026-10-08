# Conception : workflow Gmail → Claude → Google Sheets

Étape 1 sur 3 (conception), validée par Farid le 2026-10-07. Le JSON importable et les tests viendront dans les étapes 2 et 3.
Hébergement : n8n Cloud (plan Starter).

## Vue d'ensemble

```
Workflow principal
  [1] Gmail Trigger → [2] Edit Fields → [3] Claude → [4] Parser/validation → [5] Google Sheets

Workflow d'alerte (séparé)
  [6] Error Trigger → Gmail (envoi d'une alerte à toi-même)
```

Un mail = un item qui traverse la chaîne. Si plusieurs mails arrivent entre deux vérifications, n8n les traite en lot, chacun comme un item.

---

## Étape [1] Gmail Trigger

**Ce qu'il fait :** n8n interroge ta boîte Gmail à intervalle fixe (polling) et récupère les mails arrivés depuis la dernière vérification.

**Pourquoi le polling :** Gmail n'envoie pas de webhook simple à n8n. Le polling est la méthode native du nœud.

**Réglages proposés :**
| Réglage | Valeur par défaut | Pourquoi |
|---|---|---|
| Fréquence | **Toutes les 60 minutes** (validé) | Un mail est traité au plus 1 h après réception. Tous les mails de l'heure passent dans une seule exécution, ce qui ménage le quota. |
| Simplify | Désactivé | On a besoin du corps complet du mail, pas seulement de l'extrait (snippet). |
| Filtre | **Recherche `in:inbox -category:promotions -category:social`** (validé) | Boîte de réception seulement, sans les onglets Promotions et Réseaux sociaux. Ignore aussi les envoyés, brouillons et spam. |
| Pièces jointes | Non téléchargées | Inutiles pour classer, et alourdissent l'exécution. |

**Identifiants :** OAuth2 Google (tu les crées toi-même dans n8n). Sur n8n Cloud, la connexion Google se fait en quelques clics.

---

## Étape [2] Edit Fields (Set)

**Ce qu'il fait :** il ne garde que les champs utiles et prépare le texte envoyé à Claude.

**Pourquoi :** un mail brut contient beaucoup de données (en-têtes, HTML, MIME). Envoyer moins de texte = appel API moins cher et plus rapide.

**Champs conservés :**
- `message_id` : identifiant unique Gmail (servira de clé anti-doublon)
- `thread_id` : identifiant de la conversation
- `date` : date de réception
- `from_name`, `from_email` : expéditeur
- `subject` : objet
- `body` : texte brut du mail, **tronqué à 4 000 caractères**

Les noms exacts des champs en sortie du Gmail Trigger seront vérifiés à l'étape 2 (ils varient selon l'option Simplify et la version du nœud).

---

## Étape [3] Claude (nœud Anthropic)

**Ce qu'il fait :** il envoie l'expéditeur, l'objet et le texte à Claude, qui renvoie un JSON avec la catégorie, la priorité et un résumé.

**Modèle :** un modèle Haiku (le moins cher, largement suffisant pour classer et résumer). Le coût exact par mail dépend du tarif Anthropic en vigueur, à vérifier sur la page de prix.

**Sortie attendue :**
```json
{
  "categorie": "Client / Mission",
  "priorite": "haute",
  "resume": "Le client demande un devis pour une refonte Next.js avant vendredi.",
  "action": "Envoyer le devis avant vendredi"
}
```

**Règles imposées dans le prompt :**
- choisir **une seule** catégorie dans la liste fixe (voir plus bas)
- résumé en français, 1 à 2 phrases, 200 caractères max
- `action` vide si aucune action n'est attendue
- répondre **uniquement** en JSON, sans texte autour
- température basse (0 à 0.2) pour des réponses stables

**Selon ta version de n8n :** soit le nœud « Anthropic » (action « Message a model »), soit « Basic LLM Chain » + « Anthropic Chat Model ». Les deux conviennent.

**Identifiants :** clé API Anthropic (tu la saisis toi-même dans n8n).

---

## Étape [4] Parser / validation (nœud Code)

**Ce qu'il fait :** il lit la réponse de Claude, la transforme en vrai objet JSON et vérifie les valeurs.

**Pourquoi :** un LLM peut parfois renvoyer du texte autour du JSON ou une catégorie hors liste. Sans ce garde-fou, la ligne dans Sheets serait vide ou fausse.

**Règles :**
- JSON illisible → `categorie = "À vérifier"`, `priorite = "moyenne"`, `resume = objet du mail`
- catégorie hors liste → `"À vérifier"`
- priorité hors `haute / moyenne / basse` → `"moyenne"`

Ainsi, chaque mail produit toujours une ligne, et les cas douteux sont faciles à filtrer.

---

## Étape [5] Google Sheets (Append or Update Row)

**Ce qu'il fait :** il écrit une ligne par mail dans le tableau.

**Pourquoi « Append or Update » et pas « Append » :** la colonne `Message ID` sert de clé. Si le même mail est traité deux fois (redémarrage, relance manuelle), la ligne existante est mise à jour au lieu d'être dupliquée.

**Identifiants :** OAuth2 Google, avec l'API Sheets.

---

## Étape [6] Workflow d'alerte (Error Trigger)

**Ce qu'il fait :** un second petit workflow se déclenche quand le workflow principal échoue (clé API invalide, quota dépassé, Sheets inaccessible…) et t'envoie un mail avec le nom du nœud en erreur et le lien vers l'exécution.

**Pourquoi :** sans alerte, un workflow en panne passe inaperçu pendant des jours.

À relier dans les paramètres du workflow principal (« Error workflow »).

---

## Catégories proposées (modifiables)

| Catégorie | Exemples |
|---|---|
| Action requise | Demande directe, question qui attend ta réponse |
| Client / Mission | Freelance : prospects, clients, devis, projets |
| Travail | Mails liés à ton poste salarié |
| Facture / Paiement | Factures, reçus, relances de paiement |
| Administratif | Banque, impôts, assurances, contrats |
| Notification | Alertes de services (GitHub, Azure, sécurité, livraisons) |
| Newsletter | Contenus auxquels tu es abonné |
| Promo | Publicités, offres commerciales |
| Personnel | Famille, amis |
| À vérifier | Réservée au parser (réponse invalide ou ambiguë) |

**Priorité :** `haute` (réponse attendue sous 24 h), `moyenne`, `basse` (lecture facultative).

---

## Colonnes du tableau Google Sheets

| Col. | Nom | Rempli par | Note |
|---|---|---|---|
| A | Date | Gmail | Date de réception |
| B | Expéditeur | Gmail | Nom |
| C | Email | Gmail | Adresse de l'expéditeur |
| D | Objet | Gmail | |
| E | Catégorie | Claude | Liste fixe |
| F | Priorité | Claude | haute / moyenne / basse |
| G | Résumé | Claude | 1 à 2 phrases |
| H | Action | Claude | Vide si rien à faire |
| I | Lien | n8n | `https://mail.google.com/mail/u/0/#all/<message_id>` |
| J | Message ID | Gmail | **Clé anti-doublon** |
| K | Traité le | n8n | Horodatage du traitement |
| L | Statut | Toi | Vide par défaut, à remplir à la main (ex. « fait ») |

Astuce : une validation de données (liste déroulante) sur les colonnes E, F et L facilite le filtrage dans Sheets.

---

## Choix que tu voudras peut-être changer

1. **Liste des catégories** : à adapter à ton usage réel (ex. séparer « Recrutement »).
2. ~~Périmètre~~ : validé, boîte de réception sans Promotions ni Réseaux sociaux.
3. ~~Fréquence du polling~~ : validé, 60 min.
4. **Colonne Action** : utile, ou superflue pour toi.
5. **Confidentialité** : le texte des mails est envoyé à l'API Anthropic. Si certains mails sont sensibles, on peut les exclure par un filtre Gmail.

## Points d'attention

- **Quota n8n Cloud Starter** : vérifie le nombre d'exécutions incluses sur n8n.io. Exclure les promos réduit le volume.
- **Polling et redémarrages** : un mail peut être relu deux fois ; la clé Message ID règle ce cas.
- **Mails très longs ou HTML seul** : la troncature et l'extraction du texte brut seront testées à l'étape 3.
