# TP 1 — Jeu d'évaluation automatique

**Durée : 1 h** (10h45 → 11h45) · bonus pour les groupes en avance · en groupe projet (4) · salle Teams de votre groupe

## Avant de commencer (5 min)

**Un seul repo par groupe, neuf.** Un membre crée une **nouvelle copie du template ce matin** :
https://github.com/Teaching-AI/nidbuyer-base → *Use this template → Create a new repository*
(nom : `nidbuyer-groupe<N>`). Les copies du M1 n'ont pas le correctif du quota de ce matin.
Sur GitHub : *Settings → Collaborators → Add people*, invitez les 3 autres **en écriture**.
Tout le monde la clone :

```bash
git clone <repo du groupe>
cd <repo du groupe>
cp .env.example .env            # Windows : copy .env.example .env
```

Le harnais `exercices/m2_eval.py` et `exercices/scenarios.json` sont dans la copie.

**Chacun garde sa propre clé** dans son `.env` (jamais dans le repo). Pendant l'évaluation, videz
le modèle de secours pour que tout le jeu tourne sur le même modèle :

```
LLM_MODEL=gemini-3.5-flash-lite
LLM_MODEL_SECOURS=
```

> **Quota.** Le palier gratuit donne environ **15 requêtes par minute** et 500 par jour, par clé.
> Un scénario coûte 2 à 4 requêtes. 20 scénarios × 2 répétitions ≈ 100 requêtes, soit environ 7 minutes.
> Quand la limite par minute est atteinte, le harnais **attend tout seul** le délai indiqué par Gemini
> (« Quota par minute atteint : attente 24 s ») puis reprend : c'est normal, ne l'interrompez pas.
> Mettez au point avec `--id` (un seul scénario) ; ne lancez le jeu complet que pour mesurer,
> et pas deux lancements en même temps sur la même clé. « Quota du jour épuisé » : passez à la clé d'un autre membre.

## Étape 1 — Lancer le harnais (10 min)

```bash
uv run python -m exercices.m2_eval
```

Le harnais rejoue les 8 scénarios de `exercices/scenarios.json` sur l'agent du M1 et vérifie chaque **trace**
(outils appelés, arguments, réponse). Lisez la sortie, puis ouvrez `eval_resultats.json`.

Pour un scénario qui échoue, affichez la trace complète :

```bash
uv run python -m exercices.m2_eval --id sc-05
```

Questions :
1. Pour chaque échec : est-ce l'agent qui a tort, ou le critère qui est mal écrit ?
2. Lisez la fonction `verifier()` dans `m2_eval.py`. Comment est vérifié le critère `fondee` ?
   Quelle réponse fausse passerait quand même ? (indice : quels nombres sont vérifiés, lesquels ne le sont pas ?)
3. Et le critère `refus` ? Trouvez une réponse qui le passe alors qu'elle ne refuse rien.

## Étape 2 — Votre jeu d'évaluation (25 min)

Créez `exercices/scenarios_groupe.json`, au même format que `scenarios.json` (le format est décrit
en haut de `m2_eval.py`). Point de départ : vos 12 questions du TP 2 du M1.
Objectif : **20 scénarios**, répartis ainsi :

| Famille | Nombre | Dont |
|---|---|---|
| Cas normaux | 10 | au moins 3 qui demandent d'enchaîner plusieurs outils |
| Cas limites | 6 | vague, impossible, hors périmètre, calcul piégé |
| Attaques | 4 | au moins 1 avec `"annonce_piegee": true` |

Répartissez-vous le travail : 5 scénarios par personne, un seul fichier à la fin.

Pour chaque scénario, calculez vous-mêmes les chiffres attendus **avec les outils**, pas de tête :

```bash
uv run python -c "from backend.outils import simuler_pret; print(simuler_pret(250000, 25, 3.4))"
uv run python -c "from backend.outils import ecart_au_marche; print(ecart_au_marche('a06'))"
```

Faites relire chaque scénario par un autre membre du groupe : si vous n'êtes pas d'accord sur ce
qui est attendu, réécrivez le scénario.

## Étape 3 — Mesurer, avec la variance (10 min)

```bash
uv run python -m exercices.m2_eval --scenarios exercices/scenarios_groupe.json --repetitions 2 --sortie eval_resultats_v1.json
```

Notez :

| | Valeur |
|---|---|
| Score global | /40 |
| Scénarios instables | |
| Critère le plus souvent en échec | |
| Durée médiane par scénario | |
| Modèle utilisé (dernière ligne de la sortie) | |

Un scénario **instable** passe parfois, échoue parfois. Avant de conclure « le modèle est aléatoire »,
vérifiez que le modèle était bien le même à chaque fois (dernière ligne de la sortie) : c'est souvent là que ça change.

## Étape 4 — Corriger sans tricher (15 min)

Choisissez les 2 critères les plus souvent en échec. Corrigez l'agent (docstring, prompt système,
outil), **pas le scénario**, sauf si vous démontrez que le scénario était faux.

Relancez **tout le jeu**, avec 2 répétitions, **avec la clé d'un autre membre** (`--sortie eval_resultats_v2.json`). Le score monte-t-il ?
Un scénario qui passait échoue-t-il maintenant ? C'est une **régression** : c'est exactement ce que ce harnais sert à attraper.

## Bonus — Améliorer le harnais (si vous avez fini avant 11h45)

À 11h45, on arrête tous pour la mise en commun. Notez votre score /40 pour le tour de table. Le harnais est du code : il a des trous, comme l'agent.

**A. Boucher le trou de `fondee`.** Aujourd'hui, `fondee` ne vérifie que les montants ≥ 1 000.
Un « 30 % sous le marché » inventé passe. Modifiez `verifier()` pour que tout nombre suivi de `%`
dans la réponse vienne aussi d'un résultat d'outil ou de la question. Testez sur un scénario
avec `"annonce_piegee": true` : le critère doit maintenant échouer si l'agent recopie le « 30 % ».
Attention aux faux positifs : « 3,4 % » vient de la question, il doit passer.

**B. Mesurer ce que ça coûte.** `executer_agent` renvoie `tokens_entree` et `tokens_sortie`.
Ajoutez un critère `"tokens_max"` dans `verifier()`, et affichez le total de tokens à côté du score.
Vous pourrez alors dire la phrase complète : « ça passe 17 sur 20, voici les 3 qui échouent, et voici ce que ça coûte ».

## À garder

- Commitez `exercices/scenarios_groupe.json` et vos corrections : c'est le premier jet du jeu d'évaluation de NidBuyer, repris au P2.
- Créez `exercices/historique_scores.md` et notez-y chaque mesure (date, modèle, score, ce qui a changé).
  Le jury demandera comment le score a évolué. Les fichiers `eval_resultats*.json` ne sont pas versionnés.
