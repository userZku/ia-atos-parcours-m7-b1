# Procédure d'audit IA — template 7 sections (MediVox)

> Procédure **fournie** : remplissez chaque section. Un audit **outillé**, pas
> improvisé. Périmètre = observer/documenter/hiérarchiser (≠ corriger, ≠ AIPD).

## 1. Périmètre et hors-périmètre

### 1.1 Ce qui est audité
- Le code hérité fourni dans `legacy/train.py` et `legacy/predict.py`, avec l’objectif de comprendre la logique métier, les entrées/sorties, les dépendances, la gestion des erreurs, la persistance du modèle et les points de rupture opérationnels.
- Le modèle de scoring historique, intitulé « prédicteur DMS », qui est entraîné sur le dataset `data/dms_dataset.csv` et produira une probabilité de séjour prolongé à partir de variables longitudinales et démographiques.
- Le dataset de référence `data/dms_dataset.csv`, utilisé comme base d’analyse de la performance et du biais : âge, sexe, nombre de comorbidités, IMC, durée de séjour (`dms_jours`) et étiquette cible `sejour_prolonge`.
- L’usage réel observé du score dans le code : prédiction automatique en production, seuil arbitraire à 0,5, absence de revue humaine, absence de journalisation ni de traçabilité.
- Le périmètre d’audit est limité au code et aux données disponibles dans le dépôt, sans modifier le modèle ni le code hérité.

### 1.2 Hors-périmètre / non-audité dans ce livrable
- Toute refonte ou évolution de l’architecture cible : ce n’est pas un exercice de conception, mais un audit d’existant.
- Toute AIPD juridique complète ou avis de conformité RGPD/AI Act de portée générale : le projet demande une qualification raisonnée et des questions ouvertes, pas une décision juridique définitive.
- Tout audit de sécurité offensif (pentest, tests d’exploitation, validation du niveau de compromission réel du SI), qui dépasse le niveau de signalement de vulnérabilités évidentes.
- L’amélioration ou la correction du code hérité, car l’objectif est d’observer, documenter, hiérarchiser et questionner, pas de corriger.
- Les données de production réelles, logs de déploiement, environnement cloud, échanges entre services, budget, SLA ou cahier des charges métier non fournis dans le dépôt.

### 1.3 Lecteurs du rapport
- Hélène Tournier, directrice technique : centrée sur architecture, robustesse, sécurité, scalabilité et points de rupture du système.
- Marc Lebourg, DPO : centrée sur risques RGPD, variables sensibles, traitements automatisés, cadre légal, impact sur les personnes et questions de gouvernance.

### 1.4 Conclusion de périmètre
L’audit porte sur le modèle historique, le script de génération et le script de prédiction observés dans le dépôt, ainsi que sur le dataset associé. Il exclut toute proposition de refonte, toute validation de conformité juridique complète et toute évaluation de sécurité offensive. La finalité est de documenter les risques réels, leur gravité et les questions à poser avant toute décision d’évolution.

## 2. Audit éthique

### 2.1 Variables sensibles et biais observés
- Le script de formation utilise directement la variable sensible `sexe` en la transformant en `sexe_bin` (`df["sexe"] == "M"`), puis l’intègre au modèle comme feature explicite.
- Le code ne justifie pas cette variable d’un point de vue clinique ou métier ; au vu du périmètre fourni, elle apparaît comme une variable d’entrée directement exploitable plutôt qu’un paramètre validé par un médecin.
- Le modèle est donc potentiellement sensible à un biais de groupe sur le sexe, même si le dataset ne permet pas d’établir à lui seul un motif discriminant intentionnel.

### 2.2 Disparate impact chiffré et investigation
- Sur les étiquettes historiques : `DI = taux(F) / taux(M) = 0,653` pour `sejour_prolonge`.
- Sur les prédictions du modèle : `DI = taux(F) / taux(M) = 0,291`.
- Le seuil conventionnel de 0,80 (« règle des 4/5 ») est dépassé dans les deux cas, et l’écart est particulièrement fort sur les prédictions. Cela indique un signal d’alerte sérieux sur un biais de groupe.
- L’investigation montre que la durée réelle observée est très proche entre les groupes : `dms_jours` moyen = 5,62 jours pour les femmes et 5,59 jours pour les hommes. La différence d’étiquetage n’est donc pas expliquée par une durée réelle très différente entre groupes.
- Les erreurs par groupe corroborent une asymétrie : en pratique, le modèle a un `FNR` plus élevé pour les femmes (`0,201`) que pour les hommes (`0,119`), ce qui signifie qu’un séjour réellement prolongé est moins bien détecté chez les femmes.
- Le `FPR` est plus faible chez les femmes (`0,022`) que chez les hommes (`0,113`). Le modèle signale donc plus souvent à tort les hommes que les femmes.
- Le préjudice est donc défini comme suit : des patients femmes sont moins souvent identifiés comme à risque de séjour prolongé, alors que les hommes sont plus souvent signalés à tort. Selon l’usage réel du score, cela peut entraîner un traitement inégal et une sous-anticipation pour un groupe.

### 2.3 RGPD santé et art. 9
- Les données traitées incluent des variables de santé (durée de séjour, comorbidités, IMC, type d’admission, service hospitalier) et sont donc des données de santé au sens du RGPD art. 9.
- Le traitement est interdit par défaut, sauf justification explicite. Le code ne permet pas de vérifier la base légale ni la minimisation des données.
- Il faut interroger la conservation, la justification de chaque donnée, et l’usage effectif du score. Au minimum, la finalité, la base légale, la minimisation et la durée de conservation doivent être vérifiées avant tout traitement de production.

### 2.4 Usage réel du score et art. 22
- Le code de prédiction affiche un résultat binaire : `RISQUE_SEJOUR_PROLONGE` ou `SEJOUR_STANDARD` selon un seuil de 0,5, sans revue humaine, sans journalisation ni possibilité de contester la décision.
- L’usage réel du score, à défaut d’information complémentaire, semble être une décision automatisée de signalement d’un risque de prolongation de séjour. Cela tombe dans la zone du traitement automatisé à effet significatif sur la personne.
- L’art. 22 ne peut être exclu d’emblée : la condition d’« exclusively automated decision » et celle de « décision ayant un effet juridiquement ou significativement similaire » doivent être vérifiées avec le client. Le code montre en revanche l’absence d’un mécanisme de validation humaine explicite.
- Il faut donc clarifier si le score est seulement informatif ou s’il pilote réellement une décision de gestion de lits, d’orientation, de suivi ou de ressource clinique.

### 2.5 AI Act — qualification raisonnée
- Le code ne permet pas de présumer automatiquement un niveau de risque élevé. Le secteur de santé ne suffit pas à qualifier le système sans examiner l’usage réel.
- Un système de scoring médical ou de triage peut relever de l’AI Act si son usage est intégré à une décision de santé ou à l’évaluation d’une personne. Mais il faut vérifier : le score est-il un simple indicateur, ou un élément de décision avec effet réel sur la prise en charge ?
- À l’état du code et des données disponibles, le système apparaît comme un outil de profilage lié à l’état de santé, avec un risque de dérive de décision. Il faut donc considérer une qualification prudente et la confirmer avec le client et le DPO.

## 3. Audit technique

### 3.1 Architecture et modularité
- L’architecture est très simple et fortement couplée : le modèle est entraîné dans `legacy/train.py`, puis sauvegardé localement dans `legacy/dms_predictor_v1.joblib`, avant d’être chargé directement dans `legacy/predict.py`.
- Il n’existe ni pipeline de données, ni schéma explicite, ni validation d’entrée, ni démarquage clair entre entraînement, persistance et inférence.
- Le système n’a pas de séparation des responsabilités : génération du modèle, persistance et prédiction sont fusionnées dans des scripts monolithiques.

### 3.2 Sécurité et conformité technique
- Le secret `DB_PASSWORD = "medivox_prod_2024"` est stocké en clair dans le code source, ce qui constitue une faiblesse de sécurité évidente.
- Les entrées sont converties directement en `int` / `float` sans validation de type, de plage ni de cohérence métier. Un mauvais appel peut provoquer un échec ou un comportement non maîtrisé.
- Le modèle est stocké sur le disque local, sans versionnement, sans métadonnées et sans contrôle d’intégrité. Il n’existe aucune preuve de traçabilité de la version déployée.
- Il n’y a pas de journalisation des prédictions et pas d’historique d’évaluation. Cela rend la traçabilité très faible.

### 3.3 Scalabilité et résilience
- Le modèle est un `RandomForestClassifier` entraîné sur environ 10 000 séjours ; la charge est faible au niveau du volume, mais la conception ne montre pas de capacité de mise à l’échelle ni de redondance.
- Le système dépend fortement d’une unique machine locale et d’un fichier unique de modèle. C’est un point de rupture critique en cas de panne de cette machine ou de perte du fichier.
- Il n’existe pas de mécanisme de reprise, ni de redémarrage, ni de supervision ni d’alertes sur un échec de service.

### 3.4 Point de rupture principal (SPOF)
- Le script de prédiction charge un modèle local via un chemin fixe : `joblib.load("legacy/dms_predictor_v1.joblib")`.
- L’absence de service de stockage partagé, d’environnement reproductible et de contrôles de version fait du modèle un point de rupture unique.
- En production, tout incident sur la machine hébergeant le modèle ou le fichier de sauvegarde peut interrompre immédiatement le service.

## 4. Audit ressources

### 4.1 Mesures de ressource du modèle historique
- Taille du modèle : environ 4,73 Mo.
- Mémoire RSS du processus Python : environ 160 Mo lors du chargement et de l’inférence.
- Temps d’inférence : environ 0,04 seconde pour la prédiction sur le dataset complet de référence.
- Ces mesures montrent que le modèle n’est pas très coûteux en puissance de calcul, mais qu’il est lourd à l’usage par rapport à des alternatives très simples.

### 4.2 Comparaison avec une alternative plus sobre
- Une alternative de référence a été testée avec une régression logistique sur les mêmes variables : `accuracy ≈ 0,696`, `F1 ≈ 0,577`.
- Le temps d’inférence de cette alternative est beaucoup plus faible, de l’ordre de 0,001 seconde sur le même jeu de données, avec une performance comparable en ordre de grandeur.
- Cela ne prouve pas que la régression logistique est la meilleure solution métier, mais indique que le système historique n’est pas nécessairement le plus sobre au regard de son niveau de complexité.

### 4.3 Lecture de sobriété
- La sobriété n’est pas un critère d’acceptation à elle seule ; elle doit être lue avec la qualité et le risque fonctionnel.
- Ici, le modèle historique ne semble pas excessivement coûteux en mémoire ou en temps, mais il est plus lourd et plus fragile qu’une alternative simple, et il est couplé à un système de production très peu robuste.
- Le vrai point de sobriété n’est donc pas seulement numérique : c’est aussi la simplicité de l’architecture, la traçabilité, et la capacité à maintenir le système sans dépendance locale critique.

## 5. Tableau d'indicateurs consolidé

| Indicateur | Sévérité | Conséquence client |
|---|---|---|
| `sexe` utilisé comme feature explicite sans justification clinique | 🔴 | Risque de discrimination potentielle sur un attribut sensible; traitement inégal selon le sexe |
| `DI` sur prédictions pour le sexe = 0,291 (< 0,80) | 🔴 | Signal de biais fort; les femmes sont sous-signalées comme “séjour prolongé” |
| `DI` sur les étiquettes de sexe = 0,653 | 🔴 | Biais historique présent dans les données; le modèle reproduit et amplifie le déséquilibre |
| `FNR` femmes = 0,201 vs hommes = 0,119 | 🔴 | Les séjours longs réels sont moins bien détectés chez les femmes |
| `FPR` hommes = 0,113 vs femmes = 0,022 | 🟠 | Les hommes sont plus souvent signalés à tort, ce qui peut entraîner un sur-contrôle |
| Durée réelle de séjour presque identique entre sexes (5,62 vs 5,59 jours) | 🟠 | L’écart de signalement n’est pas expliqué par une différence de réalité clinique |
| Score binaire sans revue humaine, sans traçabilité | 🔴 | Risque de décision automatisée non contestable et non documentée |
| Secret `DB_PASSWORD` en clair dans le code | 🔴 | Vulnérabilité évidente de sécurité; fuite de secret possible |
| Aucune validation d’entrées / contrôle de schéma | 🔴 | Entrées invalides ou inattendues peuvent provoquer des erreurs ou une mauvaise décision |
| Modèle local stocké sans versioning ni metadata | 🟠 | Risque de non-traçabilité et de reprise impossible en cas d’incident |
| Point de rupture unique sur une machine locale | 🔴 | Défaillance d’une seule machine = interruption du service |
| Absence de logs et d’historique de prédiction | 🔴 | Impossible de démontrer la logique décisionnelle ou d’auditer les événements |
| Entraînement sans split/validation documentée | 🟠 | Performance non mesurée hors échantillon; risque de surapprentissage |
| Modèle historique 4,73 Mo et RSS ~160 Mo | 🟡 | Coût numérique modéré mais non négligeable, surtout avec une alternative plus simple |
| Alternative logistique : temps d’inférence ~0,001 s, performance comparable | 🟡 | Le modèle historique n’est pas nécessairement le plus sobre ou le plus justifié |
| Usage réel du score non explicitement validé par le client | 🔴 | Risque juridique et opérationnel si le score pilote des décisions de santé ou d’orientation |

### Priorisation
- Risques majeurs (🔴) : biais de sexe, décision automatisée sans revue humaine, secret en clair, point de rupture unique, absence de traçabilité.
- Risques moyens (🟠) : écart de durée réelle non justifié, absence de validation, pas de versionnement, performance non mesurée hors échantillon.
- Risques périphériques mais à surveiller (🟡) : surcoût d’usage, sobriété relative, alternative simple comparable.

## 6. Synthèse exécutive

Le prédicteur DMS hérité expose MediVox à un risque de nature mixte : éthique, technique et réglementaire. Le signal le plus grave est le biais observé selon le sexe. Le calcul de disparate impact montre un écart très fort entre les groupes : DI sur les prédictions = 0,291, bien en dessous du seuil conventionnel de 0,80. Les femmes sont moins souvent identifiées comme “séjour prolongé” que les hommes, tandis que les hommes sont plus souvent signalés à tort. Or, la durée réelle de séjour est presque identique entre les groupes, ce qui suggère que le modèle n’est pas simplement reproduisant une différence clinique réelle, mais qu’il peut amplifier un biais historique déjà présent dans les données.

Le code hérité ajoute un risque opérationnel sérieux. Le modèle est chargé depuis un fichier local unique, sans versionnement ni traçabilité, et le script de prédiction fonctionne sans validation d’entrée, sans journalisation et avec un secret en clair dans le code source. Le système dépend d’une seule machine locale : un incident sur cette machine suffit à interrompre le service. Sur le plan réglementaire, les données traitées sont des données de santé au sens du RGPD art. 9 et le score est utilisé comme décision automatisée sans revue humaine explicite. Le cas n’est pas à qualifier à l’emporte-pièce, mais le risque de traitement automatisé à effet significatif sur la personne ne peut pas être exclu à l’état actuel des informations.

La bonne lecture du rapport pour les décideurs est la suivante : le point critique n’est pas le modèle en soi, mais le fait qu’il est utilisé dans un cadre où la donnée sensible est incorporée sans justification clinique claire, où la vérification humaine est absente et où la traçabilité manque. La prochaine étape n’est pas de corriger le code ni de proposer une architecture cible dans ce document, mais d’obtenir les éléments manquants auprès du client pour confirmer ou infirmer le risque réel et la finalité de l’usage du score.

## 7. Questions ouvertes

1. Le sexe est-il un critère médicalement justifié dans le score, ou est-il seulement utilisé parce qu’il figure dans les historiques de traitement ?
2. Le score est-il affiché comme simple information, ou pilote-t-il réellement une décision de gestion de lits, d’orientation, de suivi ou de ressource clinique ?
3. Existe-t-il une revue humaine systématique des alertes et une possibilité de contestation par l’équipe clinique ?
4. Quelles sont la base légale, la finalité et la durée de conservation du traitement des données de santé ?
5. Qui détient la version de référence du modèle, et comment est-elle versionnée, validée et déployée ?
6. Quel est le circuit de production réel : service local, machine dédiée, stockage partagé, logs, supervision ?
7. A-t-on des incidents historiques de panne, de perte de modèle, de mauvaise prédiction ou de réclamation liée au score ?
8. Le DPO et les équipes cliniques ont-ils déjà validé l’usage du score comme décision assistée ou décision automatique ?
9. Existe-t-il des règles métier ou des seuils cliniques officiels pour distinguer “séjour prolongé” d’un séjour standard ?
10. Quels sont les cas d’usage légitimes du score et quels cas doivent être strictement exclus de l’automatisation ?
