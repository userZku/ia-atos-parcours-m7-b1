# Rapport d'audit — prédicteur de séjour prolongé v1 (MediVox)

> 2 lectorats : 👩‍💻 Hélène (technique) · ⚖️ Marc (DPO). Ce document vise à informer décision et gouvernance sans proposer la solution technique.

## 1. Synthèse exécutive

Le prédicteur DMS hérité de MediVox présente un risque significatif sur trois dimensions : éthique, technique et réglementaire. Le signal le plus grave concerne le biais observé par sexe. Le disparate impact sur les prédictions est de 0,291, soit bien en dessous du seuil conventionnel de 0,80. Les femmes sont moins souvent identifiées comme à risque de séjour prolongé que les hommes, alors que la durée réelle de séjour est pratiquement identique entre les groupes. Le modèle semble donc reproduire et amplifier un biais historique déjà présent dans les données. En l’absence de justification clinique et de revue humaine explicite, ce point constitue un risque majeur pour la conformité et l’acceptabilité du système.

Sur le plan technique, le code hérité présente des faiblesses évidentes : secret en clair dans le code, validation des entrées absente, pas d’archivage reproductible du modèle, pas de logs ni de traçabilité, et dépendance à une machine locale unique. La persistance du modèle sur un disque local sans versionnement est une faille de robustesse et de continuité de service. En cas de panne ou de perte du fichier, le service peut être interrompu sans mécanisme de reprise.

Sur le plan réglementaire, il faut traiter les données comme des données de santé au sens du RGPD art. 9. Le score est utilisé sans revue humaine explicite et sans preuve de finalité claire et de base légale justifiée. Il convient donc d’examiner, avec le client et le DPO, si le score est un simple indicateur ou s’il pilote effectivement une décision avec impact significatif sur les patients. La qualification AI Act ne doit pas être présumée, mais le risque de profilage et de décision automatisée ne peut pas être exclu à l’état actuel des éléments.

En conclusion, le sujet ne se résume pas à une simple question de performance. Le point critique est qu’un système de signalisation de risque de séjour prolongé, s’il est utilisé comme décision automatique ou comme signal d’orientation non contesté, peut produire des effets inégaux selon le sexe et sans traçabilité suffisante. L’audit ne conclut pas à la mise hors service du système, mais met en avant la nécessité d’une clarification métier, juridique et technique avant toute évolution.

## 2. Contexte et périmètre

Le système audité est un modèle historique nommé “prédicteur DMS”, entraîné sur un jeu de données de séjours hospitaliers. Il a été fourni dans le dépôt sous forme de scripts hérité : `legacy/train.py` pour la génération du modèle et `legacy/predict.py` pour la prédiction. Le dataset de référence est `data/dms_dataset.csv` et contient des variables de santé et administratives, notamment `age`, `sexe`, `nb_comorbidites`, `imc`, `dms_jours` et `sejour_prolonge`.

Le périmètre de l’audit couvre :
- le code de training et de prédiction historique ;
- le dataset fourni ;
- la logique observée de décision et d’usage réel du score ;
- les risques éthiques, techniques et de ressources associés au modèle existant.

Le périmètre exclut :
- tout correctif ou refonte du système ;
- toute AIPD complète ou validation juridique définitive ;
- tout pentest ou sécurité offensive détaillée ;
- toute proposition d’architecture cible.

La finalité de ce rapport est d’informer Hélène et Marc sur les risques réels, leur gravité et les questions à clarifier avant toute décision d’évolution.

## 3. Volet éthique ⚖️

### 3.1 Variable sensible et signal d’alerte
Le script de formation intègre explicitement la variable `sexe` sous forme binaire dans le modèle (`sexe_bin`), sans justification clinique ou métier. Cela constitue un point d’attention majeur dans un contexte de données de santé. Le calcul de disparate impact montre un écart substantiel selon le sexe :
- DI sur les étiquettes : 0,653 ;
- DI sur les prédictions : 0,291.

Le seuil conventionnel de 0,80 est dépassé dans les deux cas, avec un écart particulièrement fort sur les prédictions. Le modèle semble donc traiter les groupes de manière inégale.

### 3.2 Qui est désavantagé ?
Le préjudice observé est le suivant : les femmes sont moins souvent identifiées comme “séjour prolongé”, tandis que les hommes sont plus souvent signalés à tort. Les taux d’erreur corroborent cette asymétrie :
- FNR femmes : 0,201 ;
- FNR hommes : 0,119 ;
- FPR femmes : 0,022 ;
- FPR hommes : 0,113.

Le modèle sous-estime davantage les femmes en cas de séjour réel prolongé, alors qu’il sur-signale davantage les hommes. Cela peut entraîner un traitement différencié non justifié si le score est utilisé dans des décisions opérationnelles.

### 3.3 Étiquette historique vs réalité observée
La durée réelle de séjour est quasiment identique entre les groupes : 5,62 jours pour les femmes et 5,59 jours pour les hommes. Cela signifie que la différence d’étiquetage n’est pas expliquée par une différence claire de réalité clinique. En conséquence, il faut considérer la possibilité d’un biais historique reproduit par le modèle, et non un simple reflet d’une différence de risque de santé déjà avérée.

### 3.4 RGPD santé et art. 9
Les données manipulées contiennent des variables de santé : durée de séjour, comorbidités, IMC, type d’admission, service hospitalier, etc. Le traitement de ces données relève du RGPD art. 9. Il n’existe pas d’élément dans le code permettant de justifier la base légale, la minimisation ou la conservation du traitement. Cette absence de documentation est un risque important pour le DPO.

### 3.5 Usage réel et art. 22
Le script de prédiction affiche un résultat binaire (`RISQUE_SEJOUR_PROLONGE` ou `SEJOUR_STANDARD`) selon un seuil de 0,5, sans revue humaine ni journalisation. L’usage réel paraît être une prise de décision automatisée de signalement d’un risque de prolongation de séjour. L’article 22 du RGPD ne peut être exclu d’emblée, car les conditions cumulatives doivent être vérifiées : décision exclusivement automatisée et effet juridiquement ou significativement similaire sur la personne.

### 3.6 AI Act et qualification raisonnée
Le secteur de santé ne suffit pas à qualifier automatiquement le système comme “haut risque”. L’usage réel du score doit être examiné : est-il un simple indicateur de pilotage ou un outil décisionnel avec effet sur la prise en charge ? À l’état actuel des éléments, le système apparaît comme un outil de profilage lié à la santé avec un risque réel de dérive décisionnelle. Une qualification prudente et un examen avec le client et le DPO sont nécessaires avant toute conclusion définitive.

## 4. Volet technique 👩‍💻

### 4.1 Architecture et modularité
Le système est fortement monolithique. Le modèle est entraîné dans `legacy/train.py` puis sérialisé localement dans `legacy/dms_predictor_v1.joblib`. La prédiction est ensuite effectuée dans un autre script sans validation, sans pipeline, sans schéma de données et sans séparation claire entre entraînement, persistance et inférence.

Cette architecture ne présente ni modularité ni gouvernance de version pour les artefacts de production.

### 4.2 Sécurité immédiate
Des faiblesses évidentes ressortent immédiatement :
- secret `DB_PASSWORD` stocké en clair dans le code ;
- conversion brute des entrées sans validation de plage ni de cohérence ;
- modèle persistant sur disque local sans contrôle d’intégrité ;
- absence de journalisation et d’historique exploitables.

Ces éléments augmentent le risque de fuite de secrets, de mauvaise donnée, de mauvaise décision et de manque de traçabilité.

### 4.3 Résilience et point de rupture
Le modèle dépend d’un fichier local unique et d’une machine unique. Il n’existe ni service partagé, ni environnement reproductible, ni mécanisme de reprise. Tout incident sur la machine hébergeant le modèle ou sur le fichier sauvegardé peut provoquer l’arrêt du service. C’est un point de rupture unique (SPOF) clairement identifié.

### 4.4 Validation et gouvernance de production
Le code ne fournit aucune preuve de validation hors échantillon ni de test sur données non vues. L’absence de journalisation, de versioning du modèle et de métadonnées rend impossible la traçabilité des décisions. Le risque opérationnel est donc élevé, même si le modèle reste relativement léger en calcul.

## 5. Volet ressources

### 5.1 Mesures du modèle historique
Les mesures effectuées sur le modèle historique montrent :
- taille du modèle : environ 4,73 Mo ;
- mémoire RSS du processus : environ 160 Mo ;
- temps d’inférence sur le dataset complet : ~0,04 s.

Le modèle n’est pas excessivement gourmand en temps de calcul, mais sa simplicité de conception ne compense pas ses lacunes de robustesse et de gouvernance.

### 5.2 Comparaison avec une alternative plus sobre
Une alternative basée sur une régression logistique a été testée sur les mêmes variables. Elle a montré une performance comparable en précision et un temps d’inférence nettement plus faible, de l’ordre de 0,001 s. La différence de coût en calcul n’est pas dramatique, mais elle montre que le système historique n’est pas nécessairement justifié par une nécessité de puissance de calcul.

### 5.3 Lecture de sobriété
La sobriété ne doit pas être lue comme un critère isolé. Ici, la question n’est pas seulement “combien de mémoire consomme le modèle ?”, mais “le modèle est-il le bon outil au regard de son risque, de sa complexité et de sa gouvernance ?”. Le modèle historique est plus fragile et moins traçable qu’une alternative plus simple, ce qui limite sa sobriété opérationnelle malgré une consommation modérée.

## 6. Tableau consolidé des risques

| Indicateur | Sévérité | Conséquence client |
|---|---|---|
| Sexe utilisé comme feature explicite sans justification clinique | 🔴 | Risque de discrimination sur un attribut sensible |
| DI sur prédictions = 0,291 | 🔴 | Femmes sous-signalées comme à risque de séjour prolongé |
| DI sur étiquettes = 0,653 | 🔴 | Biais historique reproduit par le modèle |
| FNR femmes = 0,201 vs hommes = 0,119 | 🔴 | Moins de séjours longs réels détectés chez les femmes |
| FPR hommes = 0,113 vs femmes = 0,022 | 🟠 | Hommes plus souvent signalés à tort |
| Durée réelle presque identique selon le sexe | 🟠 | Le biais n’est pas explicable par une différence de réalité clinique |
| Décision automatisée sans revue humaine | 🔴 | Risque de décision non contestable |
| Secret en clair dans le code | 🔴 | Risque de fuite de secret et de sécurité |
| Absence de validation d’entrées | 🔴 | Mauvaise donnée ou erreur non contrôlée |
| Modèle local unique sans versionnement | 🟠 | Traçabilité faible, reprise impossible |
| Point de rupture unique sur une machine locale | 🔴 | Défaillance du service possible |
| Absence de logs et de traçabilité | 🔴 | Audit et investigation difficiles |
| Alternative légère performante | 🟡 | Le modèle historique n’est pas nécessairement le plus sobre |

## 7. Questions ouvertes pour le client

1. Le sexe est-il un critère médicalement justifié dans le score, ou s’agit-il d’un biais historique reproduit par le modèle ?
2. Le score est-il un indicateur purement informatif, ou pilote-t-il réellement une décision de gestion de lits, d’orientation ou de ressource clinique ?
3. Existe-t-il une revue humaine des alertes et une possibilité de contestation par le personnel clinique ?
4. Quelle est la base légale du traitement de données de santé, la finalité exacte et la durée de conservation ?
5. Quelle est la version de référence du modèle et comment est-elle versionnée, testée et déployée ?
6. Quels sont les mécanismes de supervision et de journalisation en production réelle ?
7. Y a-t-il déjà eu des incidents, des erreurs de signalement ou des réclamations liées au score ?
8. Le DPO et les équipes cliniques ont-ils validé le système comme décision assistée ou comme décision automatisée ?
9. Existe-t-il un seuil clinique officiel pour distinguer “séjour prolongé” d’un séjour standard ?
10. Quelles situations doivent être strictement exclues de l’automatisation afin d’éviter un traitement discriminant ou non proportionné ?
