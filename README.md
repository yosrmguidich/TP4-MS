# TP4 Spring Cloud : Microservices « Calculatrice » avec Eureka, Gateway et OpenFeign

Matière : Architecture MicroServices, filière MPSRCC1.
Projet : `calculatrice-spring-cloud` (pom parent + 4 modules Maven).

## 0. Présentation du projet

| Programme | Port | Rôle |
|---|---|---|
| `eureka-service` | 8761 | Annuaire des services (Eureka Server) |
| `calcul-service` (instance 1) | 8081 | Effectue les quatre calculs |
| `calcul-service` (instance 2) | 8083 | Second exemplaire du même programme (`--server.port=8083`) |
| `historique-service` | 8082 | Enregistre les opérations, appelle `CALCUL-SERVICE` par OpenFeign |
| `gateway-service` | 8888 | Passerelle : point d'entrée unique des clients |

Ordre de lancement : `eureka-service` → `calcul-service` (×2) → `historique-service` → `gateway-service`.

```
calculatrice-spring-cloud/
├── pom.xml                (pom parent, packaging pom, 4 modules)
├── eureka-service/
├── calcul-service/
├── historique-service/
├── gateway-service/
├── captures/              (captures d'écran du compte rendu)
└── README.md
```

**Versions utilisées.** Spring Boot **3.3.1**, Spring Cloud **2023.0.2**, JDK 21, springdoc-openapi **2.6.0**. Le sujet est écrit pour Spring Boot 4.0.9 / Spring Cloud 2025.1.x. La version 4.0.9 n'étant pas utilisable sur mon poste, j'ai gardé la ligne 3.x (comme au TP3). Le code Java est identique. Seuls changent quelques noms :

| Sujet (Boot 4 / Cloud 2025.x) | Mon projet (Boot 3.3.1 / Cloud 2023.0.2) |
|---|---|
| `spring-cloud-starter-gateway-server-webflux` | `spring-cloud-starter-gateway` |
| `spring.cloud.gateway.server.webflux.routes` | `spring.cloud.gateway.routes` |
| `spring.cloud.gateway.server.webflux.discovery.locator` | `spring.cloud.gateway.discovery.locator` |
| `spring-boot-starter-webmvc` | `spring-boot-starter-web` |
| springdoc 3.0.3 | springdoc 2.6.0 |

---

## 1. Réponses aux questions

### Partie A : conception

**A.4.1 Quel composant fait la correspondance entre le nom et les adresses, et quand est-elle mise à jour ?**
C'est l'**annuaire Eureka** (`eureka-service`). Chaque microservice (client Eureka) s'y inscrit au démarrage sous son `spring.application.name`, envoie ensuite un battement de cœur régulier (toutes les 5 s dans ce TP) et se désinscrit à l'arrêt. L'annuaire retire aussi une instance qui ne donne plus de nouvelles (éviction). La liste est mise à jour à chaque inscription, désinscription ou éviction. Côté client, la copie locale de l'annuaire est rafraîchie périodiquement (toutes les 5 s ici).

**A.4.2 Un avantage et un inconvénient du point d'entrée unique.**
Avantage : le client ne connaît qu'une adresse (le port 8888), quel que soit le nombre de microservices et leurs adresses. La passerelle est aussi le lieu naturel pour centraliser la sécurité, les journaux ou les filtres. Inconvénient : la passerelle devient un **point de défaillance unique** (si elle tombe, plus rien n'est accessible) et un goulot d'étranglement possible. Elle ajoute aussi un saut réseau à chaque requête.

**A.4.3 Six requêtes `GET :8888/calcul/somme?a=1&b=1` sur deux instances.**
Chaque instance reçoit **3 requêtes** (répartition à tour de rôle : 8081, 8083, 8081, 8083, 8081, 8083). Le client ne s'en aperçoit pas : il reçoit toujours la même réponse (`2`), quelle que soit l'instance.

### Partie B : l'annuaire

**Question B.1. Pourquoi l'annuaire peut-il tourner sur le même poste que `calcul-service` et `historique-service` ?**
Chaque programme est un processus Java indépendant, avec son propre Tomcat embarqué, et chacun écoute sur un **port différent** (8761, 8081, 8082). Un port ne peut être ouvert que par un seul processus à la fois : il n'y a donc pas de conflit tant que les ports sont distincts.

### Partie C : enregistrement des microservices

**Question C.1. À quoi servent le nom commun et l'identifiant différent des deux instances ?**
Le **nom** (`CALCUL-SERVICE`) désigne le service logique : c'est lui qu'un microservice appelant utilise pour demander à l'annuaire « donne-moi les instances de ce service ». L'**identifiant** (`calcul-service:8081`, `calcul-service:8083`) distingue chaque instance précise : l'annuaire s'en sert pour suivre le battement de cœur de chaque exemplaire, le retirer individuellement et l'afficher dans le tableau de bord. L'appelant choisit parmi les instances grâce au nom, et le répartiteur de charge sélectionne ensuite une instance précise.

**Question C.2. Que se passe-t-il si on lance la seconde instance sans `--server.port=8083` ?**
La seconde instance essaie d'utiliser le port 8081 de `application.properties`, déjà occupé par la première. Le démarrage échoue avec `Port 8081 was already in use` : deux processus ne peuvent pas écouter sur le même port, donc il n'y a pas de deuxième instance.

### Partie D : OpenFeign

**Question D.1. Six fois la requête 1 : quelle est la règle de choix, et comment s'appelle cette stratégie ?**
Les traces alternent entre les consoles [calcul-service:8081] et [calcul-service:8083] (relevé détaillé dans le tableau de la section 2.1). La règle est le **tour de rôle** : à chaque appel, LoadBalancer choisit l'instance suivante de la liste. Cette stratégie s'appelle **Round Robin** (tourniquet).

**Question D.2. Faute volontaire `name = "CALCUL-SERVIC"` : statut obtenu et bloc `catch` ?**
Le statut obtenu est **503 Service Unavailable**, avec le corps `{"erreur":"calcul-service injoignable"}`. L'annuaire ne connaît aucune instance de `CALCUL-SERVIC` : LoadBalancer répond lui-même 503 (aucune instance disponible), et Feign lève une `FeignException`. La réponse est produite par le **second bloc** `catch (FeignException e)` (la classe mère). Le premier bloc, `FeignException.BadRequest`, ne correspond qu'à une réponse 400 d'un service qui répond. Le bon nom a ensuite été remis.

### Partie E : la passerelle

**Question E.1. Pourquoi l'en-tête `Location` indique-t-il le port 8888 et non 8082 ?**
La passerelle ajoute à la requête qu'elle transmet les en-têtes `X-Forwarded-Host`, `X-Forwarded-Port` et `X-Forwarded-Proto` (et `X-Forwarded-Prefix` en configuration dynamique), qui contiennent l'adresse **vue par le client** (`localhost`, `8888`, `http`). Avec `server.forward-headers-strategy=framework`, `historique-service` lit ces en-têtes. Ainsi `ServletUriComponentsBuilder.fromCurrentRequest()` reconstruit l'adresse du client et non celle du microservice. Sans cette propriété, l'en-tête indiquerait `localhost:8082`.

**Question E.2. Comparaison des URI des étapes E.3 et E.5, avantages de chaque forme.**
Pour la même requête de somme : `GET :8888/calcul/somme?a=2&b=3` (E.3) et `GET :8888/calcul-service/calcul/somme?a=2&b=3` (E.5).

| | Routes statiques (E.3, `lb://`) | Routes dynamiques (E.5, annuaire) |
|---|---|---|
| Avantage | URI propres et stables pour le client (le nom interne du service n'apparaît pas) ; on garde la main sur chaque route (prédicats, filtres). | Aucune route à écrire : un nouveau microservice enregistré dans l'annuaire est accessible tout de suite. Peu d'erreurs de configuration. |
| Limite | Chaque route est écrite à la main. | Le nom du service apparaît dans l'URI ; toutes les routes sont créées, y compris celles qu'on ne voudrait pas exposer. |

La forme **statique** oblige à modifier la passerelle (et la relancer) quand on ajoute un microservice. La forme dynamique, non.

**Question E.3. Pourquoi la passerelle a-t-elle une route vers elle-même ? Est-ce gênant ?**
La passerelle est elle-même enregistrée dans l'annuaire (`GATEWAY-SERVICE`). Le localisateur dynamique crée une route par service de l'annuaire, donc aussi pour elle (`/gateway-service/**`). Ici ce n'est pas gênant : cela permet même de consulter `/actuator/gateway/routes` par ce chemin. En production, ce serait à éviter : cette route expose des endpoints internes (actuator) par la passerelle, il faudrait donc la filtrer ou la sécuriser.

---

## 2. Tests

> Chaque ligne « statut / corps obtenus » est à remplir avec ce que montre **la zone Responses de Swagger UI** (Request URL, Code, Response body, et pour les POST Response headers). Les colonnes « attendu » viennent de l'énoncé. Les captures sont à placer dans `captures/` avec les noms indiqués.

### 2.1 D.4 : `historique-service` seul (`http://localhost:8082/swagger-ui.html`)

| N° | Requête | Corps envoyé | Statut attendu | Réponse attendue | Statut obtenu | Corps obtenu | Instance qui a affiché la trace | Capture |
|---|---|---|---|---|---|---|---|---|
| 1 | POST /operations | `{"operateur":"somme","a":8,"b":2}` | 201 | `{"id":1,"operateur":"somme","a":8,"b":2,"resultat":10.0}` + `Location: http://localhost:8082/operations/1` | | | 8081 / 8083 | `captures/d4-1.png` |
| 2 | POST /operations | `{"operateur":"produit","a":6,"b":7}` | 201 | `{"id":2,...,"resultat":42.0}` | | | | `captures/d4-2.png` |
| 3 | POST /operations | `{"operateur":"division","a":7,"b":2}` | 201 | `{"id":3,...,"resultat":3.5}` | | | | `captures/d4-3.png` |
| 4 | POST /operations | `{"operateur":"division","a":7,"b":0}` | 400 | `{"erreur":"Erreur : division par 0 impossible"}` | | | 8081 ou 8083 | `captures/d4-4.png` |
| 5 | POST /operations | `{"operateur":"racine","a":9,"b":0}` | 400 | `{"erreur":"Operateur inconnu : racine"}` | | | aucune (pas d'appel) | `captures/d4-5.png` |
| 6 | GET /operations | aucun | 200 | tableau JSON des opérations 1, 2 et 3 | | | | `captures/d4-6.png` |
| 7 | GET /operations/2 | aucun | 200 | `{"id":2,"operateur":"produit","a":6,"b":7,"resultat":42.0}` | | | | `captures/d4-7.png` |

**Relevé pour la question D.1 (six fois la requête 1).**

| Envoi | 1 | 2 | 3 | 4 | 5 | 6 |
|---|---|---|---|---|---|---|
| Instance (trace) | | | | | | |

Les requêtes 1, 2 et 3 doivent se répartir en alternance entre `[calcul-service:8081]` et `[calcul-service:8083]`. Les erreurs 400 et 5 ne consomment pas d'identifiant.

### 2.2 E.2 : passerelle, routes statiques YAML (`http://localhost:8888/swagger-ui.html`)

| N° | Requête (préfixe `http://localhost:8888`) | Statut attendu | Réponse attendue | Statut obtenu | Corps obtenu | Qui a répondu | Capture |
|---|---|---|---|---|---|---|---|
| 1 | GET /calcul/somme?a=2&b=3 | 200 | `5` | | | calcul-service 8081 (trace dans sa console) | `captures/e2-1.png` |
| 2 | GET /calcul/division?a=7&b=0 | 400 | `Erreur : division par 0 impossible` | | | calcul-service 8081 | `captures/e2-2.png` |
| 3 | GET /operations | 200 | tableau JSON des opérations créées en D.4 | | | historique-service | `captures/e2-3.png` |
| 4 | POST /operations `{"operateur":"soustraction","a":10,"b":4}` | 201 | `{"id":4,...,"resultat":6.0}` + `Location: http://localhost:8888/operations/4` | | | historique-service, puis une instance de calcul-service | `captures/e2-4.png` |
| 5 | GET /calcul/somme/detail?a=2&b=3 | 200 | `{"operation":"somme","a":2,"b":3,"resultat":5.0}` | | | calcul-service 8081 | `captures/e2-5.png` |
| 6 | GET /bonjour (barre d'adresse) | 404 | corps JSON d'erreur de la passerelle | | | la passerelle elle-même : aucune route ne correspond | `captures/e2-6.png` |

Observation avec les adresses fixes : la trace de la requête 1 apparaît toujours dans la console de 8081, l'instance 8083 est ignorée.

**E.3 (`lb://`)** : six clics sur la requête 1 : les traces alternent entre 8081 et 8083 (relevé : _à compléter_). **E.4 (routes Java)** : mêmes résultats que E.2 (version Java conservée, routes YAML en commentaire).

### 2.3 E.5 : routes dynamiques (créées à partir de l'annuaire)

| N° | Requête (préfixe `http://localhost:8888`) | Requête transmise au microservice | Statut attendu | Réponse attendue | Statut obtenu | Corps obtenu | Capture |
|---|---|---|---|---|---|---|---|
| 1 | GET /calcul-service/calcul/somme?a=2&b=3 | GET /calcul/somme?a=2&b=3 vers une instance de CALCUL-SERVICE | 200 | `5` | | | `captures/e5-1.png` |
| 2 | GET /calcul-service/calcul/produit/6/7 | GET /calcul/produit/6/7 | 200 | `42` | | | `captures/e5-2.png` |
| 3 | GET /historique-service/operations | GET /operations vers HISTORIQUE-SERVICE | 200 | tableau JSON des opérations | | | `captures/e5-3.png` |
| 4 | POST /historique-service/operations `{"operateur":"somme","a":1,"b":1}` | POST /operations | 201 | `{"id":5,...,"resultat":2.0}` | | | `captures/e5-4.png` |
| 5 | GET /calcul/somme?a=2&b=3 (navigateur) | aucune : plus de route `/calcul/**` | 404 | corps JSON d'erreur de la passerelle | | | `captures/e5-5.png` |
| 6 | GET /gateway-service/actuator/gateway/routes (navigateur) | GET /actuator/gateway/routes vers la passerelle elle-même | 200 | liste JSON des routes | | | `captures/e5-6.png` |

L'identifiant attendu en 4 suppose les identifiants 1 à 4 déjà créés (D.4 et E.2) et aucun autre POST réussi entre-temps. Les routes `ReactiveCompositeDiscoveryClient_CALCUL-SERVICE`, `..._HISTORIQUE-SERVICE` et `..._GATEWAY-SERVICE` sont listées ; après l'arrêt de `historique-service` (attente d'une dizaine de secondes) sa route disparaît, puis elle réapparaît au redémarrage.

---

## 3. Captures d'écran

- [ ] **Tableau de bord Eureka** (`http://localhost:8761`) : `CALCUL-SERVICE` (2 instances : 8081 et 8083), `HISTORIQUE-SERVICE`, `GATEWAY-SERVICE` → `captures/eureka.png`
- [ ] **`http://localhost:8888/actuator/gateway/routes`** en configuration dynamique → `captures/routes-dynamiques.png`
- [ ] **Swagger UI de la passerelle, configuration statique (E.2)** : liste *Select a definition* et zone *Servers* (`http://localhost:8888`) → `captures/swagger-statique.png`
- [ ] **Swagger UI de la passerelle, configuration dynamique (E.5)** : liste *Select a definition* et zone *Servers* (`http://localhost:8888/calcul-service`) → `captures/swagger-dynamique.png`
- [ ] **Cinq consoles** (ou fenêtre *Services* d'IntelliJ) avec les lignes `started on port 8761`, `8081`, `8083`, `8082` et `8888` → `captures/consoles.png`
- [ ] Captures de la zone *Responses* des tableaux 2.1, 2.2 et 2.3 (noms `d4-*.png`, `e2-*.png`, `e5-*.png`)

---

## 4. Questions de synthèse

### 1. Tableau A.3 : étape du TP où chaque différence est observée

| Aspect | TP3 → TP4 | Étape |
|---|---|---|
| Nombre de programmes | 2 microservices → 2 + annuaire + passerelle + seconde instance | B.5, C.5, E.2 (cinq consoles) |
| Dépendances Maven | `webmvc` → + Eureka client, OpenFeign, Gateway, Eureka server, springdoc, Lombok | B.2, C.1, C.7, E.1 |
| Adresse d'un microservice | adresse écrite en dur → nom obtenu de l'annuaire | C.2, C.7 (disparition de `calcul.service.url`), D.2 |
| Appel entre microservices | classe `RestClient` → interface `@FeignClient` | D.2 |
| Erreurs d'appel | `HttpClientErrorException`, `ResourceAccessException` → `FeignException.BadRequest`, `FeignException` | D.3, question D.2 |
| Point d'entrée du client | un port par microservice → un seul port (8888) | E.2 |
| Plusieurs instances d'un service | possibles mais inutilisées → tour de rôle | C.5, D.4 (question D.1), E.3 |
| Serveur embarqué | Tomcat partout → Tomcat + Netty pour la passerelle | B.5, E.1/E.2 (`Netty started on port 8888`) |
| Arrêt d'un microservice | le client échoue (503) → les autres instances prennent le relais, 503 seulement si aucune | C.6, question D.2 |
| Organisation du code | deux projets séparés → un projet, un module par microservice | B.0, C.0 |
| Outil de test | curl, Postman → Swagger UI | C.4, D.4, E.2 |
| Classes de données | écrites à la main → générées par Lombok | C.1 bis, C.7 |

### 2. Trajet complet de `POST :8888/historique-service/operations` (corps `{"operateur":"somme","a":1,"b":2}`)

1. **Client → passerelle.** Swagger UI envoie `POST http://localhost:8888/historique-service/operations` avec le corps JSON (1ʳᵉ requête HTTP).
2. **Passerelle (`gateway-service`).** La route dynamique `Path=/historique-service/**` correspond. La passerelle consulte sa **copie locale de l'annuaire** (rafraîchie toutes les 5 s) : LoadBalancer choisit une instance de `HISTORIQUE-SERVICE` (8082). Elle retire le préfixe `/historique-service` (`RewritePath`), ajoute les en-têtes `X-Forwarded-*` (dont `X-Forwarded-Prefix: /historique-service`) et envoie `POST http://<ip>:8082/operations` (2ᵉ requête HTTP).
3. **`historique-service`.** `OperationController.creer` appelle `calculer`, qui appelle `calculClient.calculer("somme", 1, 2)`. OpenFeign demande à LoadBalancer une instance de `CALCUL-SERVICE`. LoadBalancer consulte la **copie locale de l'annuaire** (8081 et 8083) et en choisit une, à tour de rôle. Feign envoie `GET http://<ip>:808x/calcul/somme?a=1&b=2` (3ᵉ requête HTTP).
4. **`calcul-service`.** `CalculController` affiche la trace `[calcul-service:808x] somme 1+2` et répond **200**, corps texte `3`.
5. **Retour dans `historique-service`.** Feign rend `"3"`, converti en `3.0` ; l'opération reçoit son identifiant et est enregistrée. L'en-tête `Location` est construit à partir des `X-Forwarded-*` (`http://localhost:8888/historique-service/operations/<id>`). La réponse **201** avec le JSON `{"id":…,"operateur":"somme","a":1,"b":2,"resultat":3.0}` part vers la passerelle.
6. **Passerelle → client.** La passerelle renvoie la réponse (201, corps JSON, en-tête `Location`) à Swagger UI.

Il y a donc **trois requêtes HTTP** et **deux consultations de la copie locale** de l'annuaire (passerelle, puis `historique-service`). Le serveur Eureka lui-même n'est pas interrogé à chaque requête : seuls les rafraîchissements périodiques le sollicitent.

### 3. Les trois programmes qui contiennent Spring Cloud LoadBalancer

LoadBalancer est inclus avec le client Eureka (A.1) : les trois programmes qui ont cette dépendance en contiennent un.

| Programme | Nom de service résolu |
|---|---|
| `historique-service` | `CALCUL-SERVICE` (via `@FeignClient(name = "CALCUL-SERVICE")`) |
| `gateway-service` | `CALCUL-SERVICE` et `HISTORIQUE-SERVICE` (routes `lb://` en E.3/E.4) ; en E.5, un nom par service de l'annuaire, y compris `GATEWAY-SERVICE` |
| `calcul-service` | aucun : il contient le répartiteur (avec le client Eureka) mais n'appelle aucun autre service, il ne résout donc aucun nom |

### 4. Client Eureka, OpenFeign, Gateway : où et pourquoi ?

| Dépendance | `calcul-service` | `historique-service` | `gateway-service` | Pourquoi |
|---|---|---|---|---|
| **Client Eureka** | oui | oui | oui | `calcul-service` doit s'inscrire pour être trouvé ; `historique-service` s'inscrit et doit obtenir les instances de `CALCUL-SERVICE` ; la passerelle consulte l'annuaire pour `lb://` et les routes dynamiques. |
| **OpenFeign** | non | **oui** | non | Seul `historique-service` appelle un autre microservice par son nom, en Java. `calcul-service` n'appelle personne ; la passerelle transmet les requêtes par ses routes, sans interface Java. |
| **Gateway** | non | non | **oui** | Seule la passerelle est le point d'entrée qui route les requêtes. |

### 5. Problème de l'adresse écrite en dur et composants qui le résolvent

Avec `calcul.service.url=http://localhost:8081`, `historique-service` ne connaît **qu'une seule instance** : si on lance trois exemplaires de `calcul-service`, un seul reçoit toutes les requêtes et les autres ne servent à rien, donc aucune répartition de charge. Si cette instance change d'adresse ou tombe, il faut modifier la configuration et relancer `historique-service` : il ne sait pas qu'il existe d'autres instances. Le client doit aussi connaître l'adresse de chaque microservice.

Ce TP résout ces trois problèmes avec :
- **Eureka Server** : l'annuaire de toutes les instances, mis à jour automatiquement (inscription, battement de cœur, éviction).
- **Client Eureka** : chaque service s'inscrit et obtient la liste des instances des autres.
- **Spring Cloud LoadBalancer** : choisit une instance à chaque appel, à tour de rôle.
- **OpenFeign** : le nom `CALCUL-SERVICE` remplace l'adresse dans `historique-service`, sans écrire d'implémentation.
- **Spring Cloud Gateway** : le client n'utilise plus qu'une adresse (8888), et les routes `lb://` ou dynamiques répartissent la charge.

### 6. Deux instances d'`historique-service` (8082 et 8084) : que renvoie `GET /historique-service/operations` après trois `POST` ?

La passerelle répartit les trois `POST` à tour de rôle : deux sur une instance, un sur l'autre (par exemple 2 sur 8082 et 1 sur 8084). Chaque instance garde sa liste **en mémoire**. Le `GET` ne montre alors que la liste de **l'instance choisie**, et le résultat **change d'un appel à l'autre** (liste de 2 opérations, puis de 1, etc.). Les identifiants sont aussi dupliqués (chaque instance recommence à 1).

Le principe mis en défaut est celui de **service sans état** (*stateless*) : une instance ne doit pas conserver de données qui lui sont propres, sinon les instances ne sont plus interchangeables. On le corrige en **externalisant l'état** dans un stockage partagé par toutes les instances (base de données, cache distribué), `historique-service` ne gardant plus rien en mémoire.

### 7. Construire et lancer les cinq programmes sans IntelliJ

À la racine de `calculatrice-spring-cloud`, une seule commande construit les quatre modules :

```
mvn clean package -DskipTests
```

Puis, dans **cinq terminaux** (chaque programme dans le sien), dans cet ordre, en attendant à chaque fois la ligne `Started ...` :

```
java -jar eureka-service/target/eureka-service-0.0.1-SNAPSHOT.jar
java -jar calcul-service/target/calcul-service-0.0.1-SNAPSHOT.jar
java -jar calcul-service/target/calcul-service-0.0.1-SNAPSHOT.jar --server.port=8083
java -jar historique-service/target/historique-service-0.0.1-SNAPSHOT.jar
java -jar gateway-service/target/gateway-service-0.0.1-SNAPSHOT.jar
```

### 8. Rôle de `@GetMapping("/calcul/{operateur}")` dans `CalculController` et dans `CalculClient`

Dans `CalculController`, l'annotation décrit la requête **reçue** : quand un `GET /calcul/...` arrive, Spring exécute cette méthode (côté serveur). Dans l'interface `CalculClient`, la même annotation décrit la requête **envoyée** : OpenFeign construit un `GET /calcul/{operateur}?a=..&b=..` et l'envoie à une instance de `CALCUL-SERVICE` (côté client). Dans le premier cas la méthode a un corps qui calcule ; dans le second, elle n'en a pas, Feign génère l'implémentation.

### 9. Le regroupement en modules remet-il en cause l'indépendance ?

Non : le regroupement ne concerne que **l'organisation du code et du build** (un seul projet IntelliJ, un pom parent). À l'exécution, chaque microservice reste indépendant :

| Élément | Chaque module |
|---|---|
| JVM | son propre processus Java |
| Port | le sien (8761, 8081, 8083, 8082, 8888) |
| Fichier `.jar` | son propre `.jar` exécutable, construit à partir de son `pom.xml` |
| Cycle de vie | lancement, arrêt et redéploiement indépendants (en C.6, on arrête une instance sans toucher aux autres) |

### 10. D'où vient l'adresse de la zone *Servers*, et que se passe-t-il sans `server.forward-headers-strategy=framework` ?

L'adresse (`http://localhost:8888` en E.2, `http://localhost:8888/calcul-service` en E.5) vient des **en-têtes `X-Forwarded-Host`, `X-Forwarded-Port` et `X-Forwarded-Prefix`** que la passerelle ajoute en transmettant la requête de description OpenAPI. Avec la propriété, le microservice les lit et springdoc indique l'adresse vue par le client dans la description.

Sans la propriété, le microservice ignore ces en-têtes et utilise l'adresse de la requête qu'il a reçue : la zone *Servers* indiquerait l'adresse du microservice (`http://localhost:8081`, `http://localhost:8082` ou son adresse IP). Chaque *Execute* enverrait alors la requête **directement au microservice**, en contournant la passerelle (le navigateur connaîtrait les ports), et pourrait même échouer (blocage CORS entre deux origines). De même, l'en-tête `Location` des POST indiquerait le port 8082 (question E.1).

### 11. Où se trouvent les méthodes générées par Lombok, et pourquoi exclure Lombok du `.jar` ?

Les constructeurs, getters et setters ne figurent **pas dans le code source** `Operation.java`, mais dans le fichier compilé **`Operation.class`** (dossier `target/classes/com/exemple/historique/model/`) : en ouvrant ce fichier avec IntelliJ, on voit `getId()`, `setOperateur(...)`, etc. Lombok n'est qu'un **processeur d'annotations utilisé à la compilation** : une fois les `.class` produits, ils contiennent déjà tout le code nécessaire, et la bibliothèque n'est plus utile à l'exécution. On peut donc l'exclure du `.jar` exécutable (et la déclarer `optional`), ce qui allège le `.jar`.
