# Projet 1 — Un agent de code évalué par Google

**Lancement 9 oct. · séances 30 oct., 13 nov., 27 nov. (soutenance) · rendu Kaggle 2 déc. 2026**

---

## La situation

Les meilleurs agents de code tournent sur d'énormes modèles dans le cloud. Google DeepMind veut savoir
si un modèle ouvert, sur une seule machine et sans internet, peut faire aussi bien. Pour le savoir,
il organise une compétition : la **Gemma 4 Developer Agent Competition**, sur Kaggle.

Votre agent reçoit un vrai ticket de bug, dans un vrai dépôt de code Python. Il lit le code, le modifie
et propose un correctif. Google applique ce correctif dans un conteneur neuf et lance les tests du projet.
Ils passent ou non. Le score est la part de bugs corrigés, sur un **jeu caché** que personne ne voit.

C'est exactement ce qu'on a fait en M1 et M2 : une boucle d'agent, des outils, une consigne, et une
évaluation automatique sur un jeu caché. À une différence près : ici, c'est Google qui évalue.

> Page de la compétition : https://www.kaggle.com/competitions/gemma-4-developer-agent

---

## Ce que vous écrivez

Pas de Python : l'agent se décrit en YAML et en Markdown, et il est zippé dans `submission.zip`.

| Fichier | Rôle | Lien avec le cours |
|---|---|---|
| `agent.yaml` | L'agent, ses outils, ses réglages | La boucle d'agent du M1 |
| `prompts/system.md` | Le prompt système | Le prompt de NidBuyer |
| `eval_config.yaml` | Le budget par bug : temps, appels d'outils, tours | `max_tours`, le harnais du M2 |
| `configs/sampling.yaml` | Température, budget de réflexion | La variance du M2 |
| `sub_agents/*.yaml` | Des assistants (par exemple un explorateur de code) | Le sous-agent, la fenêtre de contexte |
| `skills/*/SKILL.md` | Des fiches méthode que l'agent lit au besoin | Les docstrings du M1 |
| `adapters/` | Un fine-tuning LoRA | Facultatif |

Ce qui est imposé : le modèle (Gemma 4 31B), les outils (`read_file`, `edit_file`, `run_command`,
`submit_patch`…), 12 h au total pour tous les bugs, **une soumission par jour et par équipe**.

---

## Démarrer (9 octobre, 16h45)

1. Créez un compte Kaggle et **vérifiez votre téléphone** (sans ça, pas d'accès aux GPU).
2. Rejoignez la compétition (*Join Competition*) et acceptez le règlement.
3. Formez votre **binôme** dans votre groupe projet : l'un crée l'équipe (onglet *Team*), l'autre la rejoint.
4. Dans l'onglet *Code*, ouvrez le notebook **« Getting Started - Gemma 4 Developer Agent »** et cliquez sur *Copy & Edit*.
5. Faites **un seul changement** (votre première hypothèse), puis *Save Version* et soumettez le `submission.zip` produit.

Le kit de départ ne corrige aucun bug : son budget d'une minute par bug coupe l'agent avant qu'il
ait modifié quoi que ce soit. Vous partez donc de zéro, comme tout le monde.

---

## La méthode : une hypothèse par soumission

Vous n'avez qu'une soumission par jour. Chaque soumission est donc une **expérience** : une hypothèse,
un seul changement, un score, une conclusion. Si vous changez trois choses à la fois, vous ne saurez
pas laquelle a compté (M2).

Pistes de départ, dans l'ordre où nous les testerions :

1. Une minute ne suffit pas : relever le budget dans `eval_config.yaml`, **dans la limite des 12 h** au total.
2. Le prompt oublie de dire « scripts temporaires dans `/tmp` » : ils polluent le correctif.
3. Le prompt ne dit jamais quand appeler le sous-agent `code_analyzer_agent`.
4. Trop de réflexion coupe les appels d'outils : régler `thinking_budget`.

Lisez aussi la section 10 du `HARNESS_README.md` (onglet *Data*) : ce sont les pièges connus.

---

## Le journal d'expériences

Un fichier `journal.md` par équipe, une entrée par soumission, **signée** par la personne qui l'a menée :

```markdown
## Expérience 3 — 14 octobre — menée par : Prénom N.

**Hypothèse** : avec 4 minutes par bug au lieu de 1, l'agent a le temps de modifier le code.
**Changement** : eval_config.yaml, max_time_minutes 1 → 4, max_tool_calls 10 → 40. Rien d'autre.
**Résultat** : score public 0,00 → 0,03 (soumission n° 3).
**Conclusion** : le budget était bien le premier verrou. Prochaine hypothèse : le prompt.
```

Un résultat qui baisse est aussi une conclusion. Ce qui compte, c'est qu'on puisse la relire.

---

## Règles Kaggle à respecter

- **Une soumission par jour et par équipe.**
- **Aucun code partagé en privé avec une autre équipe**, y compris l'autre binôme de votre groupe.
  Les idées peuvent circuler, pas le code. Sanction : disqualification des deux équipes.
- Un seul compte Kaggle par personne.
- Les notebooks publics de la compétition (onglet *Code*) peuvent être lus et réutilisés : c'est autorisé.
- Organisation des équipes à partir du 30 octobre : annoncée en séance.

---

## Évaluation — 20 points

| Points | Critère | Échéance |
|---|---|---|
| 2 + 2 | Binôme inscrit · première soumission valide | 16 oct. |
| 6 | Journal d'expériences : 5 soumissions ou plus, avec hypothèse, mesure, conclusion | 2 déc. |
| 2 | Contribution individuelle : chacun signe au moins une expérience | 2 déc. |
| 4 | Soutenance : chacun présente son expérience et répond à une question | 27 nov. |
| 2 + 2 | Battre la référence (le kit de départ soumis tel quel) · meilleure équipe de la classe (ex æquo si l'écart est faible) | 2 déc. |

Le classement ne compte que pour 4 points sur 20 : être en bas du classement coûte peu,
ne rien documenter coûte cher.

**Facultatif** : le *Paper Track* de la compétition accepte un article qui documente votre démarche
(3 000 mots maximum, avant le 12 novembre). Votre journal en est la matière première.

---

## Calendrier

| Date | Attendu |
|---|---|
| 9 oct. | Comptes, binôme, première soumission si possible |
| 16 oct. | Binôme inscrit et première soumission valide (début de séance en autonomie) |
| 30 oct. | Séance 1 : bilan des premières expériences, plan d'expériences |
| 13 nov. | Séance 2 : revue du journal |
| 27 nov. | Soutenance, sur le classement public |
| 2 déc. | Rendu final Kaggle ; points de classement sur le classement privé |

---

*On ne dit pas « notre agent marche ». On dit : « il corrige 3 % des bugs, voici ce qu'on a essayé, et voici ce qui a compté ».*
