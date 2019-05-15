# Architecture logicielle

## Préambule
Ce dossier technique décrit les éléments logiciels mis en jeu dans le projet signaux-faibles. Cet ensemble est en cours de construction et est susceptible d'évoluer sensiblement. 

Il est à noter que la brique keycloak est en cours d'intégration suite aux discussions qui ont eu lieu avec les experts des différents partenaires et devrait à terme remplacer le middleware gin-jwt interfacé avec gin-gonic.
Il se peut que des modifications soient opérées pour tirer le meilleur parti de cette brique logicielle, ces modifications seront répercutées au fil de l'eau dans cette documentation.

## Objectifs du dispositif
- récolter les données brutes en provenance des partenaires (goup + stockage POSIX) 
- traiter/importer les données brutes (opensignauxfaibles + mongodb)
- raffiner les données pour obtenir les variables nécessaires à la bonne marche de l'algorithme (opensignauxfaibles + mongodb)
- exécuter la détection algorithmique (opensignauxfaibles + mongodb)
- publier les données à destination des agents (datapi + postgresql + signauxfaibles-web)

## Schéma fonctionnel
![schéma](architecture-logicielle/archi.png)

## goup
Plus de détails sont disponibles [ici](https://github.com/signaux-faibles/goup)

### Objectif
goup permet aux partenaires de déposer les fichiers de données brutes sur la plateforme.  
### Composition
goup est basé sur [tusd](https://github.com/tus/tusd) et lui ajoute des fonctionnalités:
- authentification par JWT
- espaces privatifs par utilisateurs (gestion des droits et des chemins)
- supprime la possibilité de télécharger les fichiers
### Authentification JWT
En l'absence d'un JWT valide, le service refuse les sollicitations.  
Ce token devra en plus fournir dans son chargement une propriété `goup-path` correspondant au nom d'utilisateur posix ciblé par le versement.  


### Traitement des fichiers uploadés
- Lors de l'envoi, le fichier n'est accessible qu'au serveur
- La métadonnée `private` associée à l'upload permettra au serveur de décider quel traitement effectuer sur le fichier pour sa mise à disposition.
- Une fois l'envoi complété, le fichier reste disponible dans l'espace du serveur de sorte qu'il pourra naturellement détecté que le fichier est déjà complet et éviter un deuxième envoi.
- Les droits sont fixés sur le fichier pour limiter l'accès aux utilisateurs souhaités (grace à la propriété `goup-path`)
- Un lien est créé dans l'espace de stockage ad-hoc.


### Structure du stockage
Le fichiers uploadés sont matérialisés sur le disque dur par deux fichiers nommés d'après le hash de l'upload généré par TUS:
- un fichier .bin qui contient les données du fichier
- un fichier .info qui contient les métadonnées

Le stockage est organisé dans 2 répertoires permanents:
- tusd: espace de traitement des upload par le serveur et accessible uniquement par ce dernier
- public: espace commun à tous les utilisateurs

1 répertoire privé pour chaque utilisateur (voir exemple ci-dessous)

Voici l'état du stockage après l'upload de 4 fichiers par deux utilisateurs:
- file1: user1, envoi public
- file2: user1, envoi privé
- file3: user2, envoi public
- file4: user2, envoie privé

```
chemin                           user:group            mode 

.
+-- tusd                         goup:goup             770
|   +-- file1.bin                user1:users           660
|   +-- file1.info               user1:users           660
|   +-- file2.bin                user1:goup            660
|   +-- file2.info               user1:goup            660
|   +-- file3.bin                user2:users           660
|   +-- file3.info               user2:users           660
|   +-- file4.bin                user2:goup            660
|   +-- file4.info               user2:goup            660
+-- public                       user1:users           770
|   +-- file1.bin                user1:users           660
|   +-- file1.info               user1:users           660
|   +-- file3.bin                user2:users           660
|   +-- file3.info               user2:users           660
+-- user1                        user1:goup            770
|   +-- file2.bin                user1:goup            660
|   +-- file2.info               user1:goup            660
+-- user2                        user2:goup            770
|   +-- file4.bin                user2:goup            660
|   +-- file4.info               user2:goup            660
```

### dépendances logicielles
goup est développé en go (v1.10) et exploite les packages suivants:
- https://github.com/tus/tusd
- https://github.com/tus/tusd/filestore
- https://github.com/appleboy/gin-jwt
- https://github.com/gin-contrib/cors
- https://github.com/gin-gonic/gin
- https://github.com/spf13/viper

## keycloak
Il s'agit du produit officiel développé par Red-Hat.
KeyCloak fournit les services d'authentification pour `goup` et `datapi` en forgeant les JWT des utilisateurs.

### Structure du chargement JWT
Deux métadonnées sont utilisées:
- goup-path: ouvre la possibilité de téléverger des fichiers sur goup et désigne l'utilisateur. Il s'agit d'une chaine de caractère
- scope: fixe les attribution de l'utilisateurs au sein de datapi. Il s'agit d'une liste de chaines de caractères.

## opensignauxfaibles
### Objectif
opensignauxfaibles se charge du traitement des données:
- import des fichiers bruts
- calcul des données d'entrée de l'algorithme
- stockage des résultats

Cette brique intègre le stockage de l'historique des données brutes, mais aussi calculées, ainsi que des résultats de façon à permettre l'audit des calculs a posteriori.

### Modules
#### dbmongo
Ce module est écrit en go (1.10) et centralise les fonctions de traitement des données suivantes:
- analyse des fichiers bruts
- conversion/insert dans mongodb
- ordonnancement des traitements mapreduce/aggregations mongodb
- publication vers datapi

##### Dépendances logicielles
- https://github.com/appleboy/gin-jwt
- https://github.com/gin-contrib/cors
- https://github.com/gin-contrib/static
- https://github.com/gin-gonic/gin
- https://github.com/gorilla/websocket
- https://github.com/spf13/viper
- https://github.com/swaggo/gin-swagger
- https://github.com/swaggo/gin-swagger/swaggerFiles
- https://github.com/globalsign/mgo
- https://github.com/globalsign/mgo/bson
- https://github.com/spf13/viper
- https://github.com/tealeg/xlsx
- https://github.com/chrnin/gournal
- mongodb 3.6

##### utilisation
dbmongo propose une api pour ordonner les traitements. Une documentation openapi est disponible [ici](https://raw.githubusercontent.com/signaux-faibles/opensignauxfaibles/master/dbmongo/docs/swagger/swagger.yaml)

#### module R/H2O
Ce module permet le traitement algorithmique. (à écrire)

### Flux de traitement
![schéma](architecture-logicielle/workflow-osf.png)
1. Lecture des fichiers bruts ([dbmongo](#dbmongo))
1. Les données brutes sont converties et insérées dans mongodb ([dbmongo](#dbmongo))
1. Les données sont compactées dans mongodb par un traitement map-reduce ([dbmongo](#dbmongo))
1. Les variables sont calculées dans mongodb par un traitement map-reduce ([dbmongo](#dbmongo))
1. Le traitement algorithmique est effectué par le module [R/H2O](#module-rh2o)
1. Les résultats sont injectés dans mongodb par le module [R/H2O](#module-rh2o)
1. Les données nécessaires au frontend sont exportées dans datapi par le module ([dbmongo](#dbmongo))

Pour plus de détail sur le traitement des données et les transformations qui leur sont appliquées, voir [ici](processus-traitement-donnees.md).

### Publication vers datapi
opensignauxfaibles dispose d'un identifiant lui ouvrant la possibilité de publier des données sur datapi.
Il lui incombe de fournir à datapi:
- des données détaillées sur les établissements et les entreprises (niveau A)
- des données synthétiques sur les établissements et les entreprises (niveau B)
- les listes de détection (niveau A)
- les badges de sécurité pour tous les objets exportés afin d'appliquer les niveaux de sécurité.

Ce traitement est écrit en dur dans le code de dbmongo [ici](https://github.com/signaux-faibles/opensignauxfaibles/blob/master/dbmongo/lib/engine/datapi.go)

## datapi
Le détail sur le fonctionnement de Datapi est disponible [ici](https://github.com/signaux-faibles/datapi)

### Objectif
Dans cette infrastructure, datapi est la brique permettant la diffusion contrôlée des données produites dans le projet et sert notamment de back-end pour le signauxfaibles-web.  
Parmi les fonctionnalités notables:
- stockage arbitraire d'objets JSON
- gestion dynamique des permissions en lecture/écriture
- journalisation des requêtes
- programmation de la péremption des données

### Principe d'alimentation
![workflow](architecture-logicielle/workflow-datapi.png)
Il est à noter que:
- les insertions et lectures de données sont effectuées au sein de transactions postgres.
- la prise en compte de l'ajout de données et de politiques de sécurité est intégrée dans le système transactionnel de façon à présenter un comportement synchrone
- les permissions accordées aux utilisateurs sont véhiculées dans le token JWT et reposent sur la sécurité de keycloak.

### Dépendances logicielles
- https://github.com/gin-contrib/cors
- https://github.com/gin-gonic/gin
- https://github.com/appleboy/gin-jwt
- https://github.com/swaggo/gin-swagger
- https://github.com/swaggo/gin-swagger/swaggerFiles
- https://github.com/gin-gonic/gin
- https://github.com/lib/pq
- https://github.com/lib/pq/hstore
- https://golang.org/x/crypto/bcrypt

## signauxfaibles-web

## Produits extérieurs
### mongodb
### keycloak
### postgresql
### client TUS 
Un exemple de client tus est fourni [ici](https://github.com/signaux-faibles/goup/tree/master/goup-client) et permet de voir une implémentation javascript basée sur le client officiel.  
On trouve toutefois des clients dans de nombreux langages qui permettront aux utilisateurs d'intégrer l'upload de fichier dans leurs plateformes. 