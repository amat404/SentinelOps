# SentinelOps - Cahier des charges

> Document de travail - V1 en cours de conception  
> Dernière mise à jour : 26/09/2026

## 1. Objectif du projet

SentinelOps est un système de supervision d'infrastructure.

Son objectif est de centraliser la surveillance de plusieurs machines afin de :

- suivre leur état ;
- suivre certaines métriques comme le CPU et la RAM ;
- surveiller des services importants ;
- détecter automatiquement certaines anomalies ;
- créer et suivre des alertes ;
- conserver un historique permettant de comprendre les incidents.

Le principe général retenu est :

```text
Agent local
- observe
- collecte
- horodate
- transmet

SentinelOps Central
- reçoit
- conserve
- interprète
- décide
- affiche
```

L'agent local ne crée pas les alertes définitives. La décision reste centralisée.

## 2. Utilisateurs

### Administrateur

L'administrateur peut :

- visualiser les machines, leurs métriques et leur état ;
- visualiser et prendre en charge les alertes ;
- ajouter ou supprimer des machines ;
- gérer les utilisateurs ;
- gérer les rôles et les droits.

### Opérateur

L'opérateur peut :

- visualiser les machines et leur état ;
- consulter les alertes ;
- prendre en charge une alerte ;
- suivre un incident jusqu'à sa résolution.

Dans la V1, il n'agit pas directement sur les machines depuis SentinelOps.

### Viewer

Le Viewer possède uniquement des droits de consultation.

Il peut :

- visualiser les machines ;
- consulter leurs métriques et leur état ;
- consulter les alertes.

Il ne peut pas prendre en charge une alerte ni administrer SentinelOps.

## 3. Supervision prévue dans la V1

SentinelOps V1 supervise principalement :

- l'utilisation du CPU ;
- l'utilisation de la RAM ;
- l'état de certains services ;
- la disponibilité des machines ;
- la communication entre les agents et le central.

### 3.1 CPU

SentinelOps utilise une hystérésis :

- CPU > 82 % : entrée dans l'état `CPU élevé` ;
- CPU < 78 % : retour à l'état normal ;
- entre 78 % et 82 % : conservation de l'état précédent.

Une alerte est créée si l'état `CPU élevé` reste actif pendant 30 minutes consécutives.

Si le CPU repasse sous 78 % avant les 30 minutes :

- aucune alerte n'est créée ;
- la période d'observation est réinitialisée.

Si une alerte existe déjà et que le CPU repasse sous 78 %, l'alerte devient automatiquement `Résolue`.

Exemple :

```text
70 % -> normal
81 % -> reste normal
84 % -> CPU élevé
80 % -> reste CPU élevé
79 % -> reste CPU élevé
77 % -> retour à normal
```

### 3.2 RAM

Règle actuelle :

- RAM > 80 % pendant 30 minutes consécutives -> création d'une alerte ;
- si la RAM revient à 80 % ou moins avant 30 minutes -> aucune alerte ;
- si une alerte existe et que la RAM revient sous le seuil -> alerte `Résolue`.

L'application d'une hystérésis à la RAM reste à décider.

### 3.3 Services

SentinelOps doit pouvoir surveiller certains services importants, par exemple PostgreSQL.

Règle actuelle :

- vérification normale toutes les 15 secondes ;
- après un premier échec, trois nouvelles vérifications espacées de 5 secondes ;
- si l'une réussit -> aucune alerte et retour au rythme normal ;
- si les quatre vérifications échouent -> service considéré indisponible et création d'une alerte.

### 3.4 Disponibilité d'une machine

Hypothèse actuelle :

- test toutes les 2 secondes ;
- une machine est considérée indisponible après 8 échecs consécutifs ;
- un test réussi avant le huitième échec remet le compteur à zéro.

Ce mécanisme est distinct de la perte de communication entre l'agent et le central. Leur articulation reste à clarifier pour éviter des alertes redondantes.

### 3.5 Perte de communication

Si le central ne reçoit plus de données d'une machine pendant un délai défini, SentinelOps peut créer une alerte `Perte de communication`.

Cette alerte signifie uniquement que SentinelOps a perdu sa visibilité sur la machine.

Elle ne permet pas de conclure automatiquement que :

- la machine est éteinte ;
- le réseau est en panne ;
- l'agent est arrêté ;
- un service comme PostgreSQL est indisponible.

Lorsque la communication revient, cette alerte devient automatiquement `Résolue`.

Le délai exact de déclenchement reste à définir.

## 4. Gestion des alertes

Une alerte possède trois états.

### À résoudre

Créée automatiquement lorsqu'une règle confirme un problème actif.

### En cours

Un administrateur ou un opérateur clique sur `Prendre en charge`.

Cela signifie qu'un humain travaille sur l'incident, mais le problème technique peut encore être présent.

### Résolue

SentinelOps détecte que la condition ayant provoqué l'alerte n'est plus vraie.

Une alerte peut passer directement de :

```text
À résoudre -> Résolue
```

si le problème disparaît avant sa prise en charge.

Les alertes résolues sont conservées dans l'historique.

Les couleurs peuvent compléter l'état :

- rouge : À résoudre ;
- jaune : En cours ;
- vert : Résolue.

Le texte de l'état doit toujours rester visible.

### Unicité d'une alerte active

Un même incident actif ne doit produire qu'une seule alerte.

Les nouvelles mesures confirmant le même incident mettent à jour son suivi mais ne créent pas de doublon.

Une fois l'alerte résolue, une nouvelle occurrence crée une nouvelle alerte et doit satisfaire à nouveau sa règle complète de déclenchement.

## 5. Alertes découvertes après une coupure

Pendant une coupure réseau, l'agent continue à collecter et horodater les données.

Lors de la reconnexion :

- les données manquantes sont retransmises ;
- le central reconstruit la chronologie ;
- les règles sont réévaluées sur la période manquante.

Avant récupération des données, une période manquante est considérée comme inconnue. SentinelOps ne doit pas inventer les valeurs.

Une alerte possède aussi un `mode de détection` :

- `Temps réel` ;
- `Récupération`.

Si un incident est découvert après coup et qu'il est déjà terminé :

```text
État : Résolue
Mode de détection : Récupération
```

Si le problème est encore actif :

```text
État : À résoudre
Mode de détection : Récupération
```

Une alerte de perte de communication et une alerte technique comme `PostgreSQL indisponible` restent deux alertes différentes.

## 6. Architecture fonctionnelle

SentinelOps possède deux grandes parties :

```text
Machines surveillées
        |
        v
Agents locaux
        |
        v
SentinelOps Central
```

### 6.1 Agent local

Un agent est installé sur chaque machine surveillée.

Ses responsabilités sont :

- collecter les informations locales ;
- horodater les mesures ;
- conserver temporairement les données ;
- transmettre les données au central ;
- continuer à collecter pendant une coupure ;
- retransmettre les données non acquittées après reconnexion.

Le principe reste :

```text
Agent local = collecte et transmet
Central = interprète et décide
```

### 6.2 SentinelOps Central

Le central est découpé logiquement en quatre modules.

#### Module Données

Il est responsable de :

- recevoir les données ;
- conserver les métriques ;
- fournir l'historique ;
- constater l'absence de communication avec un agent.

#### Module Règles / Alertes

Il est responsable de :

- appliquer les règles de supervision ;
- créer les alertes ;
- gérer leur cycle de vie ;
- détecter les retours à la normale ;
- gérer la prise en charge ;
- décider si une absence de communication devient une alerte.

#### Module Affichage

Il est responsable de :

- présenter les machines ;
- présenter les métriques ;
- présenter l'historique ;
- présenter les alertes et leurs états ;
- adapter l'interface aux droits de l'utilisateur.

#### Module Administration

Il est responsable de :

- gérer les machines ;
- gérer les utilisateurs ;
- gérer les rôles ;
- gérer les droits.

## 7. Flux logiques principaux

### Données

```text
Agent
-> Module Données
-> Module Règles / Alertes
```

Le module Données possède les données brutes.

Le module Règles / Alertes les interprète.

### Affichage des métriques

```text
Module Affichage
-> Module Données
-> métriques / historique
```

### Affichage des alertes

```text
Module Affichage
-> Module Règles / Alertes
-> alertes / états
```

### Administration

Lorsqu'une machine est ajoutée :

```text
Administrateur
-> Module Administration
-> Module Données
```

### Autorisations

Masquer un bouton dans l'interface ne suffit pas à sécuriser une action.

Le module Affichage peut adapter l'interface, mais le module qui exécute réellement l'action doit vérifier l'autorisation.

Exemple :

```text
Utilisateur
-> Module Affichage
-> Module Règles / Alertes
-> vérification des droits auprès du Module Administration
-> action acceptée ou refusée
```

## 8. Architecture technique du central

Pour la V1, SentinelOps Central est une seule application organisée en modules.

```text
SentinelOps Central

|- Module Données
|- Module Règles / Alertes
|- Module Affichage
|- Module Administration
```

Ce choix permet :

- de simplifier le développement ;
- de simplifier le déploiement ;
- de simplifier la maintenance de la V1 ;
- de conserver malgré tout une séparation logique claire.

SentinelOps reste un système distribué dans son ensemble car plusieurs agents communiquent avec un central par le réseau.

Une séparation en services indépendants pourra être étudiée plus tard si elle devient nécessaire.

## 9. Communication agent -> central

Le modèle retenu est `PUSH`.

```text
Agent -> Central
```

L'agent décide quand envoyer ses données.

Ce choix facilite notamment :

- la retransmission après coupure ;
- le suivi des données non envoyées ;
- les connexions sortantes depuis les machines surveillées.

Le central doit cependant être protégé contre des pics de trafic lorsque beaucoup d'agents envoient simultanément.

## 10. Constitution des lots

Les métriques ne sont pas nécessairement envoyées une par une.

Un lot est créé lorsque l'une des deux conditions suivantes est satisfaite :

```text
nombre de métriques >= N
OU
ancienneté de la métrique la plus ancienne >= X
```

Les valeurs de `N` et `X` seront déterminées plus tard à partir de tests.

## 11. Stockage local de l'agent

Chaque agent possède un stockage structuré persistant local.

La technologie n'est pas encore choisie.

Le stockage contient deux ensembles :

```text
Stockage local

|- MetriqueTemporaire
|  -> métriques pas encore regroupées
|
|- LotLocal
   -> lots finalisés en attente d'ACK
```

Le stockage persistant permet de survivre à un crash ou à un redémarrage.

## 12. Structure d'une métrique temporaire

```text
MetriqueTemporaire
------------------
id
type
valeur
timestamp
```

### id

Identifiant technique local unique.

Il permet de distinguer deux enregistrements même s'ils ont le même contenu.

### type

Exemples :

```text
CPU
RAM
SERVICE_POSTGRESQL
```

### valeur

Valeur mesurée.

### timestamp

Instant réel auquel la mesure a été effectuée.

Le timestamp est créé au moment de la collecte, pas au moment de l'envoi ou de la réception.

Il sert notamment à :

- reconstruire la chronologie ;
- appliquer les règles basées sur une durée ;
- connaître l'ancienneté d'une métrique temporaire.

Le timestamp n'est pas un identifiant et n'est pas nécessairement unique.

## 13. Structure d'un lot local

```text
LotLocal
-------------------------
batchId
dateCreation
delaiBackoffActuel
prochainEnvoi
contenuMetriques
```

### batchId

UUID unique généré localement.

Il permet :

- d'identifier le lot ;
- de reconnaître une retransmission ;
- d'assurer l'idempotence côté central.

### dateCreation

Date de création du lot.

Elle permet notamment de gérer l'ancienneté du backlog.

### delaiBackoffActuel

Délai de base courant du backoff.

Le jitter n'est pas inclus dans cette valeur.

### prochainEnvoi

Date à partir de laquelle le lot peut être envoyé ou retransmis.

### contenuMetriques

Objet structuré contenant les métriques du lot.

L'agent ne fait pas d'analyse métier sur ce contenu. Il le conserve et le transmet.

### Informations non répétées

`machineId` n'est pas stocké dans chaque lot.

Il est conservé une seule fois dans la configuration de l'agent puis ajouté au message réseau lors de l'envoi.

Aucun champ `etat = transmis / non transmis` n'est nécessaire :

- lot présent localement -> pas encore acquitté ;
- ACK reçu -> lot supprimé.

## 14. Persistance avant création du lot

Chaque mesure est persistée dès sa collecte.

```text
collecte
-> timestamp
-> insertion dans MetriqueTemporaire
```

Les métriques ne restent donc pas uniquement en mémoire pendant la constitution d'un lot.

En cas de crash, les mesures déjà persistées sont récupérées au redémarrage.

## 15. Constructeur de lots

Un seul constructeur de lots est prévu par agent.

Les collecteurs et le constructeur ont des rôles séparés :

```text
Collecteur CPU ------|
Collecteur RAM ------|-> MetriqueTemporaire
Collecteur Service --|          |
                                v
                       Constructeur de lots
                                |
                                v
                            LotLocal
```

Les collecteurs continuent à fonctionner pendant la création des lots.

### Sélection des métriques

Le constructeur prend en priorité les métriques les plus anciennes selon leur timestamp.

En cas de timestamp identique, l'ordre entre les métriques n'a pas d'importance fonctionnelle. L'identifiant local peut seulement servir à obtenir un ordre déterministe.

### Lots complets

Tant que :

```text
nombreMetriques >= N
```

le constructeur crée un lot de `N` métriques.

Exemple avec `N = 100` et 250 métriques :

```text
Lot 1 = 100
Lot 2 = 100
Reste = 50
```

### Reste inférieur à N

Si le reste est inférieur à `N` :

- si la métrique la plus ancienne attend depuis au moins `X` -> création d'un lot avec le reste ;
- sinon -> les métriques restent temporaires jusqu'à atteindre `N` ou `X`.

## 16. Atomicité de la construction d'un lot

Pour chaque lot, les opérations suivantes forment une seule transaction :

```text
sélectionner les métriques
-> créer LotLocal
-> supprimer les MetriqueTemporaire correspondantes
-> COMMIT
```

Soit toutes les opérations réussissent, soit aucune n'est validée.

Chaque lot utilise sa propre transaction.

Ainsi, si plusieurs lots doivent être construits et qu'un crash survient pendant le deuxième, le premier reste validé.

## 17. Réveil du constructeur

Le constructeur n'est pas réveillé après chaque nouvelle métrique.

Deux mécanismes sont utilisés.

### Condition de quantité

Un compteur en mémoire représente le nombre de métriques temporaires.

Après persistance d'une nouvelle métrique :

```text
compteur++
```

Le constructeur est réveillé uniquement lorsque le compteur franchit le seuil `N`.

Exemple :

```text
99 -> 100 : réveil
100 -> 101 : pas de nouveau réveil
```

Le compteur doit être modifié de manière sûre vis-à-vis de la concurrence.

Une opération atomique ou un verrouillage très court pourra être utilisé selon la technologie choisie.

### Condition temporelle

Un timer est basé sur :

```text
timestamp de la métrique la plus ancienne + X
```

Lorsque l'échéance est atteinte, le constructeur est réveillé.

Le timer n'est pas la source de vérité.

La source de vérité reste le timestamp persistant.

## 18. Gestion du compteur

Le compteur est uniquement une optimisation en mémoire.

Le stockage `MetriqueTemporaire` reste la source de vérité.

Après redémarrage :

```text
compteur = nombre réel de MetriqueTemporaire persistées
```

Lorsqu'un lot est créé avec succès :

```text
compteur =
compteur - nombreMetriquesRetirees
```

Le compteur n'est jamais remis arbitrairement à zéro.

Il est décrémenté uniquement après le `COMMIT` de la transaction.

Ainsi, les nouvelles métriques ajoutées pendant la construction d'un lot ne sont pas perdues dans le comptage.

## 19. Gestion du timer

Si un lot est créé avant l'échéance temporelle, le timer devenu inutile est annulé.

S'il reste des métriques temporaires :

```text
nouvelle échéance =
timestamp de la plus ancienne restante + X
```

Un nouveau timer est alors programmé.

Après un redémarrage :

- si l'échéance est déjà dépassée -> construction immédiate du lot ;
- sinon -> reprogrammation du timer pour le temps restant.

## 20. Initialisation et envoi d'un lot

Un nouveau lot est initialisé avec :

```text
batchId = nouvel UUID
dateCreation = maintenant
delaiBackoffActuel = 0
prochainEnvoi = dateCreation
contenuMetriques = métriques du lot
```

Il est donc immédiatement éligible à une première tentative d'envoi.

## 21. ACK et suppression locale

Un lot reste dans le stockage local tant qu'aucun ACK positif n'a été reçu.

```text
Agent
-> envoie Lot X

Central
-> traite X

si succès
-> ACK(X)

Agent
-> supprime X
```

Si le traitement échoue :

- aucun ACK positif n'est envoyé ;
- le lot reste stocké ;
- il sera retransmis plus tard.

Règle :

```text
Pas d'ACK positif = pas de suppression locale
```

## 22. Idempotence et doublons

Un même lot peut être reçu plusieurs fois.

Exemple :

```text
Agent envoie X
Central traite X
Central envoie ACK
ACK perdu
Agent renvoie X
```

Le central doit reconnaître `batchId = X`.

Il ne doit pas :

- stocker les métriques une deuxième fois ;
- reproduire les effets du traitement.

Il doit cependant renvoyer l'ACK.

Le central conserve donc de manière persistante les `batchId` déjà traités.

## 23. Atomicité côté central

Lors de la réception d'un lot, les opérations suivantes doivent être atomiques :

```text
enregistrer les métriques
+
enregistrer batchId comme traité
```

Une transaction garantit :

```text
soit les deux réussissent
soit aucune n'est validée
```

Cela évite qu'un lot soit partiellement enregistré puis retraité après un crash.

## 24. Backoff et jitter

Après un échec, l'agent utilise un backoff exponentiel.

Exemple conceptuel :

```text
2 s
4 s
8 s
16 s
...
```

Un délai maximal est défini.

Une fois le plafond atteint :

```text
2 -> 4 -> 8 -> 16 -> 30 -> 30 -> 30 ...
```

Le délai ne revient pas à zéro tant que le même lot continue à échouer.

Le jitter ajoute une petite variation aléatoire afin d'éviter que tous les agents réessaient exactement au même moment.

Conceptuellement :

```text
nouveauBackoff =
min(backoffActuel * 2, backoffMax)

prochainEnvoi =
maintenant + nouveauBackoff + jitter
```

Le jitter ne modifie pas `delaiBackoffActuel`.

## 25. Redémarrage et retransmission

Le backoff survit au redémarrage grâce à `prochainEnvoi`.

Après redémarrage :

```text
si prochainEnvoi > maintenant
-> attendre

si prochainEnvoi <= maintenant
-> lot éligible
```

Un lot éligible n'est pas forcément envoyé immédiatement si beaucoup d'autres lots attendent.

Les mécanismes de limitation du backlog restent appliqués.

## 26. Gestion du backlog

Le backlog correspond aux lots non acquittés accumulés localement.

Après une longue coupure, l'agent ne doit pas envoyer tout le backlog d'un seul coup.

Une fenêtre limitée de retransmission sera utilisée.

La taille exacte reste à définir.

### Anciennes et nouvelles données

L'agent doit à la fois :

- vider progressivement le backlog ;
- continuer à transmettre les données récentes.

Principe initial :

```text
1 ancien lot
2 nouveaux lots
1 ancien lot
2 nouveaux lots
...
```

Les valeurs exactes pourront être ajustées.

### Mode adaptatif

Si le backlog devient très important, une plus grande part de la capacité de transmission peut être réservée aux anciens lots.

Deux seuils seront utilisés pour éviter les oscillations :

```text
backlog > seuil haut
-> mode rattrapage fort

backlog < seuil bas
-> mode normal

entre les deux
-> conserver le mode précédent
```

Il s'agit d'une hystérésis.

## 27. Limites de la V1

SentinelOps V1 est un système de supervision et de suivi d'incidents.

Il ne réalise pas encore :

- le redémarrage distant d'un service ;
- la modification distante de la configuration d'une machine ;
- l'isolation automatique d'une machine ;
- l'application de configurations à plusieurs machines ;
- les rapports avancés hebdomadaires ou mensuels ;
- la journalisation complète de toutes les actions critiques.

Ces fonctionnalités pourront être étudiées dans de futures versions.

## 28. Points encore ouverts

Les décisions suivantes ne sont pas encore figées :

- valeur de `N` ;
- valeur de `X` ;
- hystérésis éventuelle pour la RAM ;
- articulation entre disponibilité active d'une machine et perte de communication avec l'agent ;
- délai déclenchant une alerte de perte de communication ;
- délai maximal du backoff ;
- amplitude exacte du jitter ;
- durée de conservation des lots locaux ;
- durée de conservation des `batchId` traités côté central ;
- taille de la fenêtre de retransmission ;
- seuils du mode de rattrapage ;
- proportions définitives entre lots anciens et nouveaux ;
- technologie du stockage local ;
- technologie de stockage central ;
- protocole de communication ;
- technologies d'implémentation des agents et du central.

Ces valeurs et technologies seront choisies plus tard à partir des besoins, des tests et des contraintes réelles.
