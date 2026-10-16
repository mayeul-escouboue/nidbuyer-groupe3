# TP 0 — La carte des données de NidBuyer

Mme Martin est un personnage fictif. Aucune donnée personnelle réelle n'est utilisée dans ce document.

## Comment lire cette carte

Une donnée peut être utilisée temporairement en mémoire, enregistrée dans un fichier ou une base, ou envoyée à un autre service. Ces situations ne sont pas équivalentes.

Cette carte décrit le code du dépôt, sans vérifier un déploiement ni les réglages des fournisseurs. « Durée non définie » signifie qu'aucune expiration automatique n'est prévue dans le code, pas qu'une conservation illimitée a été décidée.

**Attention aux TODO :** `POST /chat` et `POST /alerte` répondent actuellement 501 (« non implémenté »). Le frontend n'envoie pas encore les formulaires à l'API. Les sources automatiques sont désactivées (`SOURCES_ACTIVES = []`), et les collecteurs Bien'ici/LeBonCoin restent à implémenter. Les flux ci-dessous précisent donc leurs conditions de fonctionnement.

## 1. Où passent les données ?

| Donnée | Entre par | Passe par | Écrite où ? | Combien de temps ? | Qui d'autre la voit ? |
|---|---|---|---|---|---|
| Question et profil acheteur : intention, budget, quartiers, surface, pièces et texte libre | Formulaire Streamlit ; modèle `Question` de `POST /chat` | `ProfilAcheteur`, validation FastAPI si l'API est appelée directement | Saisie dans la session de l'interface ; objet de requête en mémoire côté API. Aucun historique de chat persistant dans cette route : TODO, réponse 501 | Aucune durée de session définie ici ; pas de stockage durable démontré par cette route | Utilisatrice ; serveur de l'interface ; serveur API si appelé. Aucun envoi au LLM depuis cette route actuellement |
| Texte libre du profil et critères de recherche | `POST /rechercher`, déjà implémenté | `search_similar(description_libre ou "appartement")` ; filtre du budget ; calcul des scores | Texte encodé en vecteur pour interroger ChromaDB ; aucun ajout du profil à la collection. L'analyse LLM reçoit l'intention et les faits sur les biens, pas le texte libre dans cette route | Traitement en mémoire ; aucune persistance du profil montrée ici | Backend et moteur local d'embedding ; fournisseur LLM pour l'intention et les faits effectivement envoyés |
| Texte transmis au modèle : questions, contexte, résultats d'outils, éventuellement données personnelles présentes dedans | `llm.generer()` ou `executer_agent()` ; notamment le harnais d'évaluation | Client Google pour `gemini`/`vertex` ; requête HTTP vers `OLLAMA_URL` pour `ollama` ; échanges successifs de l'agent | Mémoire du processus et traitement par le service choisi. Le code ne permet pas de connaître les éventuels journaux du fournisseur | Durée du traitement local ; conservation chez le fournisseur non déterminable à partir du dépôt | Google si Gemini/Vertex ; exploitant du serveur Ollama si choisi (localhost par défaut). `fake` fonctionne sans envoi réseau au modèle |
| Métadonnées des appels LLM : fournisseur, modèle, succès, latence, nombres de tokens | Chaque appel journalisé dans `backend/llm.py` | `_journaliser()` puis `JOURNAL` | Liste en mémoire uniquement. **Pas de question, de profil, d'e-mail ou de réponse dans ce journal** | Au plus 1 000 dernières entrées par processus, perdues à son arrêt ; aucune durée en jours | Processus et personnes pouvant l'inspecter. `/admin/status` expose des statistiques sur les 50 derniers appels, pas les textes ni la liste brute |
| E-mail et profil complet d'alerte, donc potentiellement texte intime | Modèle `AlerteProfil` de `POST /alerte` ; fonction `sauvegarder_profil(email, profil)` | `charger_profils()` ; ajout du couple e-mail/profil | La fonction écrit dans `data/alertes.json`, sans chiffrement applicatif. **La route ne l'appelle pas encore** ; son TODO prévoit à la place une table Supabase `alertes` | Aucune expiration ni fonction de suppression ; fichier conservé tant qu'il n'est pas supprimé (stockage local éphémère sur Cloud Run) | Personnes ayant accès au fichier ; hébergeur du disque. Accès à la future table Supabase à définir |
| E-mail de destination et biens correspondant au profil | `sync()` charge les profils d'alerte | `_filtrer_pour_profil()` puis `notifier_email(email, biens_matching)` | Envoi SMTP et copies dans les boîtes mail envisagés, **notification encore TODO**. Slack est également un TODO et n'est pas branché au pipeline ; le profil complet n'est pas passé à ces fonctions de notification | Aucune durée définie pour ces futurs services | Si implémenté : prestataire mail, destinataire ; pour Slack, membres du canal et service Slack selon le raccordement retenu |
| Questions réelles copiées pour évaluer l'agent | Fichier choisi par `m2_eval.py --scenarios` (défaut : `exercices/scenarios.json`) | `rejouer()` → `executer_agent()` → fournisseur LLM configuré | Fichier de scénarios : une copie manuelle peut y introduire une donnée personnelle. Le harnais lit ce fichier, sans anonymisation automatique | Aucune expiration ; si le fichier est versionné, copies dans l'historique Git et les clones | Équipe ; personnes ayant accès au dépôt si versionné ; service LLM selon configuration |
| Réponses d'évaluation pouvant reprendre la question ou des informations personnelles | Résultat de chaque scénario | `m2_eval.py`, construction de `lignes` | `eval_resultats.json` par défaut, ou chemin `--sortie` : id, répétition, réussite, verdicts, tours, durée, noms d'outils et **réponse complète**. **Ni question brute ni arguments/résultats complets des outils dans ce fichier** | Écrasé lors d'une exécution au même chemin ; sinon aucune expiration. Des variantes et copies peuvent subsister | Équipe et personnes ayant accès aux fichiers. Le motif `eval_resultats*.json` est ignoré par Git, ce qui ne supprime pas les fichiers locaux |
| Traces détaillées de l'évaluation : arguments, résultats, erreurs et réponse | Option `m2_eval.py --id ...` | `print(json.dumps(res["appels"]))` et affichage de la réponse | Sortie du terminal ; fichier ou journaux CI seulement si cette sortie est capturée | Selon l'historique du terminal ou la conservation des journaux, non définie dans le script | Personne lançant le test ; lecteurs des journaux s'ils sont conservés |
| Informations de vendeurs éventuellement présentes dans une annonce : coordonnées dans le texte, éléments identifiants dans les photos ou leurs URL | Sources Bien'ici/LeBonCoin (collecte TODO) ou source générique appelée manuellement | `_parse()` / `normalize()` ; `SourceGenerique.fetch_from_text()` envoie le texte brut au LLM (HTML limité aux 8 000 premiers caractères pour `fetch_from_urls()`) | Objets en mémoire avant ingestion. Aucun retrait automatique des coordonnées ; les parseurs conservent description et URL de photos. La source générique extrait une description, sans champ photos dédié | Traitement temporaire, puis durée des stockages ci-dessous si ingestion ; éventuelle conservation LLM inconnue | Site d'origine ; backend ; fournisseur LLM pour la source générique. Aucune collecte automatique active actuellement |
| Annonces normalisées : description, prix, surface, quartier, ville, URL source, photos, etc. | `sync()` après collecte et déduplication | `db.enregistrer_annonces()` via API REST Supabase, si configurée | Table Supabase `annonces` : colonnes autorisées par `COLONNES`, dont description et liste d'URL de photos ; `vu_le` par défaut dans le schéma proposé | Aucune purge ni expiration dans le code ; un même id est mis à jour | Backend, hébergeur Supabase et personnes disposant d'accès à la base ; politiques d'accès effectives non vérifiées |
| Copie des annonces sous forme de texte, vecteurs et métadonnées | `indexer_annonces()` ; ingestion ou reconstruction depuis Supabase au démarrage | `texte_a_encoder()` assemble type, surface, quartier et **description complète** ; modèle d'embedding local | ChromaDB, collection `annonces_toulon`, dossier `CHROMA_PATH` (défaut `./chroma_db`). Métadonnées : tous les champs scalaires, donc aussi description/URL ; les listes de photos sont exclues | Pas de purge prévue ; persiste localement. Sur Cloud Run, cache perdu au redémarrage et reconstruit depuis Supabase | Backend et personnes ayant accès au stockage ; annonces retournées aux utilisateurs selon les routes implémentées. Le profil acheteur n'est pas indexé par cette fonction |
| Éventuellement e-mail dans un message d'erreur d'alerte ; détails d'erreurs de fournisseurs/extraction | Gestion des erreurs dans `ingestion.py`, `llm.py` et `sources/generique.py` | Rapport de synchronisation, `logger`, `print` ; une erreur d'alerte ajoute `Alerte {email} : ...` au rapport | Mémoire, puis logs si les erreurs/rapports sont affichés ou collectés. Certaines erreurs de validation peuvent reprendre une valeur ; ce n'est pas un journal systématique des conversations | Aucune durée de rétention définie dans ces fichiers ; réglages de l'hébergement à vérifier | Équipe et personnes autorisées à lire les logs ; hébergeur si collecte activée |

### Fichiers vérifiés

- [Modèles, routes et reconstruction au démarrage](../backend/main.py)
- [Interface et saisie du profil](../frontend/app.py)
- [Envois aux modèles, échanges d'outils et journal](../backend/llm.py)
- [Stockage local et notifications d'alerte](../backend/alert.py)
- [Scénarios, résultats et affichage des traces](../exercices/m2_eval.py)
- [Agent utilisé par l'évaluation](../exercices/m1_agent.py) et [outils : annonces fictives locales](../backend/outils.py)
- [Sources actives](../backend/sources/__init__.py), [format commun](../backend/sources/base.py), [Bien'ici](../backend/sources/bienici.py), [LeBonCoin](../backend/sources/leboncoin.py), [extraction générique](../backend/sources/generique.py)
- [Pipeline d'ingestion](../backend/ingestion.py), [base Supabase](../backend/db.py), [index ChromaDB](../backend/rag.py), [fichiers exclus de Git](../.gitignore)

## 2. Les trois questions

### 1. Quelle donnée est la plus sensible ? Pourquoi ?

L'information sur le handicap du fils : elle révèle un élément de santé concernant un enfant, qui n'est même pas l'utilisatrice du service. Le divorce et la situation familiale sont aussi des informations intimes, qu'un champ libre peut transmettre sans nécessité.

### 2. Quelle donnée est la plus inutile pour conseiller Mme Martin ?

Le fait qu'elle vient de divorcer n'aide pas à comparer les logements. Il suffit de demander ses besoins concrets : budget, nombre de chambres, proximité des écoles et accessibilité en fauteuil, sans conserver les raisons personnelles derrière ces besoins.

### 3. Combien d'endroits vérifier pour tout effacer ? Où serait-ce difficile ?

Pour ses données d'acheteuse, **six familles d'endroits sont à vérifier selon les usages réels** : (1) sessions/mémoire, (2) alertes locales ou future table Supabase, (3) scénarios copiés et éventuel historique Git, (4) fichiers de résultats et copies, (5) traces/logs et futures notifications, (6) fournisseur LLM. Ce n'est pas six copies déjà prouvées : les routes chat/alerte restent des TODO, et les variantes de fichiers peuvent multiplier les copies.

L'effacement chez le fournisseur LLM n'est pas réalisable depuis ce dépôt : aucune fonction ni politique de suppression n'y est fournie. Relier un scénario ou une réponse à Mme Martin est aussi difficile sans identifiant commun ; pour une donnée de vendeur, il faut en plus supprimer l'annonce dans **Supabase et ChromaDB**, sinon le cache peut être reconstruit depuis la base.

## 3. Explication très simple pour présenter le TP

Les données sont comme une feuille qu'on photocopie : effacer l'original ne détruit pas les copies. Ici, une phrase intime peut se retrouver chez le modèle, dans un scénario de test, puis dans une réponse sauvegardée.

Les points principaux à retenir :

- Le chat et l'inscription aux alertes ne fonctionnent pas encore ; leurs TODO représentent des choix à faire.
- Le journal LLM garde des compteurs, pas les conversations ; les résultats d'évaluation gardent les réponses.
- Supabase conserve les annonces et ChromaDB en garde une copie pour la recherche.
- Pour conseiller Mme Martin, on a besoin d'un logement accessible, pas de toute son histoire familiale.
- Avant de promettre « tout effacer », il faut retrouver chaque copie et vérifier ce qui est conservé par les services externes.
