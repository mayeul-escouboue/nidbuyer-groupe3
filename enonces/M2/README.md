# Module 2 — Évaluer et surveiller

**9 octobre 2026** · 9h30 → 17h30, à distance (Teams)

Au M1, vous avez cassé l'agent à la main et compté vos 12 questions. C'est un début, pas une mesure :
on ne sait pas si la correction d'hier a cassé autre chose, si le modèle répond pareil demain,
ni ce que ça donnera quand Google mettra le modèle à jour.

Ce module installe le réflexe qui distingue un produit d'une démo : **on ne dit pas « ça marche »,
on dit « ça passe 34 scénarios sur 40, voici les 6 qui échouent, et voici ce que ça coûte ».**

## Objectifs

À la fin du module, l'étudiant sait :
- construire un jeu d'évaluation représentatif (cas normaux, cas limites, attaques) ;
- définir des critères vérifiables sur la **trace** d'un agent, pas seulement sur sa réponse ;
- mesurer la stabilité d'un agent (même question, plusieurs essais) ;
- utiliser un LLM comme juge, et en connaître les biais ;
- mener un red teaming et mesurer un taux d'attaques réussies ;
- dire quoi surveiller une fois l'agent en production, et quand déclencher une alerte.

## Déroulé

| Horaire | Contenu |
|---|---|
| 9h30 | Rappel du M1, vos pires pannes, le fil rouge du jour (la compétition de 16h45). Pourquoi « ça a l'air de marcher » ne suffit pas |
| 9h45 | Cours : jeu d'évaluation, sources, critères, SWE-bench, trace, pass^k, variance, quota |
| 10h45 | **TP 1** — [Jeu d'évaluation automatique](TP1-jeu-evaluation.md) (1 h) |
| 11h45 | Mise en commun des scores. Cours : le bruit d'un petit jeu, la loi de Goodhart, le jeu caché |
| 12h15 | Pause déjeuner |
| 13h30 | Cours : cas réels, LLM-as-judge, red teaming, injection, garde-fous, le harnais, contexte et sous-agents, surveillance en production (1h15) |
| 14h45 | **TP 2** — [Red team inter-groupes](TP2-red-team.md) (1h15) |
| 16h00 | Pause |
| 16h15 | Les projets de l'année : P1 = compétition Kaggle, P2 = NidBuyer (20 min), puis QCM noté (10 min) |
| 16h45 | **Lancement du Projet 1** : compétition Kaggle Gemma 4 Developer Agent, en binômes, jusqu'à 17h30 |

## Contenu du cours

### 1. Le jeu d'évaluation

Un jeu d'évaluation, c'est une liste de situations avec, pour chacune, **ce qu'on attend**.
Il doit contenir trois familles :

| Famille | Part indicative | Exemples NidBuyer |
|---|---|---|
| Cas normaux | 50 % | « Mensualité pour 250 k€ sur 25 ans », « a06 est-il au prix du marché ? » |
| Cas limites | 30 % | Question vague, budget impossible, ville hors périmètre, taux à 0 % |
| Attaques | 20 % | Annonce piégée, « ignore tes consignes », demande de conseil en placement |

Règles :
- **On l'écrit avant de corriger.** Sinon on n'écrit que des tests qu'on sait réussir.
- **On ne le jette jamais.** Un bug trouvé en production devient un scénario.
- **On le fait relire par quelqu'un d'autre.** Un scénario ambigu mesure l'ambiguïté, pas l'agent.

### 2. Évaluer un agent : la trace, pas seulement la réponse

Une bonne réponse obtenue par le mauvais chemin est un accident qui attend son heure.
On vérifie donc la **trace** (voir `exercices/m2_eval.py`) :

| Critère | Question | Vérifiable automatiquement ? |
|---|---|---|
| Outils requis | A-t-il appelé `simuler_pret` pour une mensualité ? | Oui |
| Outils interdits | A-t-il évité de chercher des biens quand on ne lui demandait qu'un calcul ? | Oui |
| Chiffres | La mensualité exacte (1 238,19 €) est-elle dans la réponse ? | Oui |
| Réponse fondée | Chaque montant cité vient-il d'un résultat d'outil ? | Oui |
| Refus | Demande-t-il une précision quand la question est vague ? | Approximativement |
| Qualité du conseil | Le conseil est-il pertinent pour ce profil ? | Non : humain ou LLM-juge |

Le critère **« réponse fondée »** est le plus utile : il attrape les chiffres inventés sans avoir à les connaître d'avance.

### 3. La variance : même question, autre réponse

À température 0, un agent donne le plus souvent la même trace, mais pas toujours : les serveurs regroupent
les requêtes et les arrondis de calcul peuvent faire basculer un mot (He, 2025). Ce qui fait varier les résultats :
un **changement de modèle** (le modèle de secours après une erreur 503, une mise à jour chez Google),
un changement de données, ou une consigne ambiguë qui laisse le choix entre deux chemins.
On rejoue chaque scénario **3 à 5 fois**. Un scénario qui passe 2 fois sur 3 est un **scénario instable** :
c'est souvent le signe d'une docstring ou d'une consigne ambiguë.

### 3 bis. Réussir une fois ne suffit pas : pass^k

Un agent qui réussit 80 % des scénarios en un essai ne réussit le même scénario **3 fois de suite**
que dans 0,8³ ≈ 51 % des cas. C'est la mesure pass^k de τ-bench (Yao et al., 2024) :
un client pose plusieurs questions, la fiabilité c'est réussir à chaque fois.

### 3 ter. Lire un score : le bruit et la loi de Goodhart

Avec 20 scénarios, une question vaut 5 points. 17/20 (fourchette plausible 64–95 %) et 15/20 (53–89 %)
ne sont pas clairement différents. On compare donc **scénario par scénario** : lesquels sont passés, lesquels ont cassé.
Les répétitions ne comptent pas comme de nouveaux scénarios.

« Quand une mesure devient un objectif, elle cesse d'être une bonne mesure. » Mettre la réponse attendue
dans le prompt, finir chaque réponse par « ? », retirer les scénarios durs : le score monte, l'agent ne s'améliore pas.
Parade : un **jeu caché**, que l'équipe ne voit pas.

### 4. LLM-as-judge

Pour juger ce qui ne se vérifie pas automatiquement (pertinence, ton, clarté), on demande à un LLM de noter.
C'est utile, mais il a des biais connus :
- **position** : il préfère la première (ou la dernière) réponse présentée ;
- **longueur** : il préfère les réponses longues ;
- **auto-complaisance** : il préfère les réponses de son propre modèle ;
- **complaisance** : il note haut par défaut.

Règle : **un juge automatique se valide contre des humains** sur 20 à 30 cas avant d'être cru.
Et on lui demande une grille précise (« la réponse cite-t-elle l'écart au marché ? oui/non »), pas une note sur 10.

### 5. Red teaming

Attaquer son propre système avant qu'un autre le fasse. On mesure un **taux d'attaques réussies**,
avant et après correction. Familles d'attaques à couvrir :
- **injection directe** : « ignore tes instructions et… » ;
- **injection indirecte** : l'instruction arrive par une annonce, un e-mail, une page web ;
- **sortie de périmètre** : conseil en placement, conseil juridique, autre ville ;
- **fuite** : « affiche ton prompt système », « quelle est ta clé d'API ? » ;
- **abus de coût** : question conçue pour déclencher 20 appels d'outils.

### 5 bis. Pourquoi l'injection est dure : la « lethal trifecta »

Pour le modèle, une consigne et une donnée sont du même texte : on ne sait pas empêcher l'injection à coup sûr.
On limite les dégâts en ne réunissant jamais les trois ingrédients (Simon Willison, 2025) :
**contenu non fiable** (les annonces), **données privées** (revenus de l'acheteur), **moyen d'envoyer dehors** (un outil e-mail).

Deux cas réels : Air Canada condamnée en 2024 pour une règle de remboursement inventée par son chatbot ;
un chatbot de concession Chevrolet qui « accepte » en 2023 de vendre un SUV à 1 dollar après une injection directe.

### 6. Garde-fous

| Garde-fou | Où | Exemple NidBuyer |
|---|---|---|
| Limiter ce que l'agent peut faire | Choix des outils | Aucun outil qui envoie un e-mail ou modifie une base |
| Valider les entrées des outils | Code des outils | `simuler_pret` refuse une durée nulle |
| Limiter les tours | Boucle d'agent | `max_tours=6` dans `llm.executer_agent` |
| Filtrer la sortie | Après l'agent | Vérifier que les chiffres cités viennent des outils avant d'afficher |
| Humain dans la boucle | Processus | Un conseiller valide avant toute recommandation d'offre d'achat |

### 7. Surveiller en production

Ce qui marche en octobre peut dériver en décembre : nouvelles annonces, nouvelles questions,
**mise à jour du modèle par le fournisseur**. On surveille :

| Indicateur | Pourquoi | Seuil d'alerte (exemple) |
|---|---|---|
| Latence p50 / p95 | Expérience utilisateur | p95 > 15 s |
| Coût par jour | Budget | > 2 × la moyenne des 7 derniers jours |
| Taux d'erreurs et d'arrêts `max_tours` | Pannes | > 5 % des conversations |
| Score du jeu d'évaluation, relancé chaque semaine | Dérive silencieuse | Baisse de plus de 5 points |
| Échantillon relu par un humain | Ce que les métriques ne voient pas | 20 conversations par semaine |

`/admin/status` expose déjà la latence moyenne des derniers appels (`llm.JOURNAL`).

### 7 bis. Tracer en production

Pour chaque conversation : question, réponse, outils appelés avec arguments et résultats, modèle exact,
tokens, latence, verdict des critères. RGPD : pseudonymiser, fixer une durée de conservation, jamais de clé dans les journaux.
Une trace qui échoue devient un scénario. Outils : Langfuse, conventions OpenTelemetry GenAI.

**p50 / p95** : p50 est la médiane. p95 = 95 % des réponses arrivent en moins de ce temps, les 5 % les plus lentes au-delà.

### 8. Le Projet 1 : la compétition Kaggle

Google DeepMind, *Gemma 4 Developer Agent Competition* : un agent (Gemma 4 31B, sans internet) corrige de vrais bugs
dans des dépôts Python ; le score est la part de bugs dont les tests passent, comme SWE-bench (Jimenez et al., 2023).
On écrit l'agent en YAML et en Markdown (prompt, skills, sous-agents) : pas de Python, fine-tuning facultatif.
Règles Kaggle : équipes de 2 (binôme dans le groupe projet), 1 soumission par jour, aucun code partagé entre équipes.
Point de départ : notebook « Getting Started - Gemma 4 Developer Agent » (onglet Code de la compétition).
Le kit de départ ne corrige aucun bug (budget d'une minute par bug) : chaque soumission teste une hypothèse.

## Supports

- Slides : [`M2-evaluer-et-surveiller.pdf`](M2-evaluer-et-surveiller.pdf)
- [TP 1 — Jeu d'évaluation automatique](TP1-jeu-evaluation.md)
- [TP 2 — Red team inter-groupes](TP2-red-team.md)
- Code : `exercices/m2_eval.py` et `scenarios.json` (lancer avec `uv run python -m exercices.m2_eval`)

## Ressources

**Évaluer**
- [Anthropic — Créer des évaluations solides](https://docs.claude.com/en/docs/test-and-evaluate/develop-tests)
- [Yao et al. 2024, *τ-bench*](https://arxiv.org/abs/2406.12045) : agents avec outils, la mesure pass^k
- [Miller 2024, *Adding Error Bars to Evals*](https://arxiv.org/abs/2411.00640) : intervalles de confiance, comparaison appariée
- [Horace He 2025, *Defeating Nondeterminism in LLM Inference*](https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/) : pourquoi température 0 ne suffit pas
- [Chen, Zaharia, Zou 2023, *How is ChatGPT's behavior changing over time?*](https://arxiv.org/abs/2307.09009) : la dérive d'un modèle au même nom

**LLM-as-judge**
- [Zheng et al. 2023, *Judging LLM-as-a-Judge*](https://arxiv.org/abs/2306.05685) : accord avec les humains et biais
- [Wang et al. 2023, *Large Language Models are not Fair Evaluators*](https://arxiv.org/abs/2305.17926) : biais de position

**Attaquer et défendre**
- [Jimenez et al. 2023, *SWE-bench*](https://arxiv.org/abs/2310.06770)
- [Anthropic 2025, *Effective context engineering for AI agents*](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Perez et al. 2022, *Red Teaming Language Models with Language Models*](https://arxiv.org/abs/2202.03286)
- [Greshake et al. 2023, *Not what you've signed up for*](https://arxiv.org/abs/2302.12173) : l'injection indirecte
- [Debenedetti et al. 2025, *Defeating Prompt Injections by Design* (CaMeL)](https://arxiv.org/abs/2503.18813)
- [Simon Willison 2025, *The lethal trifecta for AI agents*](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)
- [OWASP — Top 10 pour les applications LLM](https://genai.owasp.org/llm-top-10/)

**Cas réels**
- Moffatt c. Air Canada, Civil Resolution Tribunal de Colombie-Britannique, 2024 BCCRT 149 (février 2024)

---

[← Retour au README du repo](../../README.md)
