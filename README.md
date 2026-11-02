# TP3 REST : Architecture Microservices avec Spring Boot

**Faculté des Sciences de Sfax — Département d'informatique et des Communications**  
*Matière :* Architecture MicroServices | *Filière :* MPSRCC1  
*Enseignantes :* Corinne Amel ZAYANI, Amal ABID  

---

## 🏗️ Structure des Projets et Démarrage

Le projet est composé de deux microservices Spring Boot autonomes :
- **`calcul-service`** (Port `8081`) : Effectue les opérations arithmétiques pures (`/calcul`).
- **`historique-service`** (Port `8082`) : Stocke l'historique en mémoire vive et délègue les calculs à `calcul-service` via `RestClient`.

### Démarrage des microservices :
```bash
# Terminal 1 : calcul-service (Port 8081)
cd calcul-service && mvn spring-boot:run

# Terminal 2 : historique-service (Port 8082)
cd historique-service && mvn spring-boot:run
```
- Console 1 : `Tomcat started on port 8081 (http) with context path '/'`
- Console 2 : `Tomcat started on port 8082 (http) with context path '/'`

---

## 📝 Réponses aux Questions du TP

### Partie A : Questions de Conception (A.4)

#### Question A.4.1
**Dans le TP2, le résultat d'une opération était calculé par un simple appel de méthode. Dans ce TP, il est obtenu par une requête HTTP vers un autre microservice. Citez un avantage et un inconvénient de ce choix.**
- **Avantage :** Découplage fort et passage à l'échelle (*scalabilité*) indépendant : chaque service peut évoluer, être déployé ou répliqué séparément selon sa charge.
- **Inconvénient :** Surcharge et latence réseau (appels HTTP, sérialisation JSON), et obligation de gérer les pannes partielles ou l'indisponibilité du service distant.

#### Question A.4.2
**Si calcul-service est arrêté, quelles requêtes du tableau A.2 continuent de fonctionner ? Lesquelles échouent ?**
- **Continuent de fonctionner :** Les lectures et suppressions locales dans l'historique : `GET /operations`, `GET /operations/{id}` et `DELETE /operations/{id}`.
- **Échouent :** Toutes les requêtes nécessitant un calcul : `GET /calcul/...`, ainsi que `POST /operations` et `PUT /operations/{id}` (qui doivent interroger `calcul-service`).

#### Question A.4.3
**Pourquoi les deux microservices ne peuvent-ils pas écouter sur le même port ?**
Au niveau de la pile réseau TCP/IP de l'OS, un port d'écoute ne peut être réservé que par un unique processus à la fois. Tenter d'ouvrir le même port déclenche une `java.net.BindException: Address already in use`.

#### Question A.4.4
**Quel code de statut HTTP historique-service doit-il renvoyer quand il ne parvient pas à joindre calcul-service ?**
Le code **`503 Service Unavailable`** (ou `502 Bad Gateway`), indiquant au client que le service distant requis pour compléter la demande est temporairement indisponible.

---

### Partie C : Premier Microservice : calcul-service

#### Question C.1
**Comparez cette URL (`http://localhost:8081/calcul/somme?a=2&b=3`) avec celle du TP2 (`http://localhost:8080/RestWebApp/api/calcul/somme?a=2&b=3`). Quelles parties ont disparu, et pourquoi ?**
Le nom du WAR (`/RestWebApp`) et le préfixe JAX-RS (`/api`) ont disparu. Avec Spring Boot, chaque microservice intègre son propre Tomcat et expose son API directement à la racine (`context path = '/'`) sur son propre port dédié (`8081`).

#### Question C.2
**Remplacez GET par POST dans Postman et renvoyez la requête. Quel code de statut obtenez-vous ? Est-ce le même qu'au TP2 ?**
On obtient le code **`405 Method Not Allowed`**. Oui, c'est exactement le même code qu'au TP2, car la route est configurée exclusivement pour la méthode HTTP `GET`.

---

### Partie D : Compléter la Calculatrice

#### Question D.1
**Au TP2, JAX-RS donnait la valeur 0 à un paramètre entier absent et la méthode somme s'exécutait. Que fait Spring dans le même cas ? Laquelle des deux attitudes vous paraît la plus conforme à REST ?**
Spring renvoie immédiatement le code **`400 Bad Request`**, car par défaut `@RequestParam` est obligatoire (`required=true`).  
L'attitude de **Spring est plus conforme à REST et plus robuste**, car une requête cliente incomplète ou invalide doit être explicitement rejetée plutôt que de produire silencieusement un calcul faussé par une valeur par défaut arbitraire.

#### Question D.2
**Avec curl ou Postman, Spring renvoie ses messages d'erreur sous forme d'objet JSON. Quels champs contient-il ?**
L'objet JSON d'erreur contient 4 champs :
- `timestamp` : Date et heure de l'erreur.
- `status` : Code numérique HTTP (ex: `400`).
- `error` : Libellé du statut HTTP (ex: `"Bad Request"`).
- `path` : URI de la requête en erreur (ex: `"/calcul/somme"`).

#### Question D.3
**Complétez le tableau de correspondance : à quelles annotations JAX-RS correspondent `@RequestParam` et `@PathVariable` ? Laquelle utilisera-t-on pour lire l'identifiant dans `/operations/{id}` ?**
- `@RequestParam` correspond à **`@QueryParam`** (paramètres de requête après le `?`).
- `@PathVariable` correspond à **`@PathParam`** (variables insérées dans le chemin d'URI).
Pour lire l'identifiant dans `/operations/{id}`, on utilise **`@PathVariable("id") int id`**.

---

### Partie E : Échanger des Données en JSON

#### Question E.1
**Au TP2, JSON-B avait produit `{"a":2,"b":3,"operation":"somme","resultat":5.0}`. Les deux textes JSON sont-ils équivalents pour un client ? Qui a effectué la conversion cette fois-ci ?**
- **Oui, ils sont strictement équivalents** : selon la norme JSON (RFC 8259), un objet est une collection non ordonnée de clés/valeurs.
- C'est la bibliothèque **Jackson** (intégrée par défaut dans Spring Boot) qui a réalisé la sérialisation automatique en JSON.

#### Question E.2
**Pourquoi la seconde requête (`Accept: application/xml`) est-elle refusée, alors que le code de la méthode ne mentionne aucun format ?**
Spring applique la **négociation de contenu** (*Content Negotiation*). Comme le client exige explicitement du XML via l'en-tête `Accept`, et que l'application ne dispose d'aucun convertisseur XML dans son classpath, Spring rejette la demande avec le code **`406 Not Acceptable`**.

---

### Partie F : Second Microservice & Résilience

#### Question F.1
**Pourquoi les lectures continuent-elles de fonctionner alors que calcul-service est arrêté ? Une telle situation était-elle possible au TP2 ?**
- Les lectures (`GET /operations`, `GET /operations/{id}`) consultent uniquement la mémoire locale d'`historique-service` (`OperationService`) sans solliciter `calcul-service`.
- C'était **impossible au TP2**, car toute l'application résidait dans le même WAR sur un serveur unique ; son arrêt coupait toutes les fonctionnalités simultanément.

#### Question F.2
**Pendant l'arrêt de calcul-service, le client a reçu une réponse 503 en moins d'une seconde. Que se serait-il passé sans le bloc catch (`ResourceAccessException e`) ?**
L'exception réseau n'aurait pas été interceptée et Spring aurait retourné une erreur générique **`500 Internal Server Error`** avec une stack trace technique. Le bloc `catch` permet de renvoyer un statut sémantique propre **`503 Service Unavailable`**.

---

### Questions de Synthèse

#### 1. Différences observées par rapport au TP2 (Tableau A.3)
- **Technologie :** Parties B (`pom.xml`) et C.2 (annotations Spring Boot).
- **Découpage :** Parties B (`calcul-service`) et F.1 (`historique-service`).
- **Résultat de la construction :** Parties B.3 et C.5 (JAR autonome exécutable avec Tomcat embarqué).
- **Serveur :** Partie C.5 (`java -jar` au lieu d'un déploiement externe).
- **Port :** Parties C.4 (8081) et F.1 (8082).
- **Adresse :** Parties C.6 et C.7 (disparition de `/RestWebApp/api/`).
- **Activation :** Partie B.4 (`@SpringBootApplication` et méthode `main`).
- **Ressource REST :** Partie C.2 (`@RestController` + `@RequestMapping`).
- **Verbes HTTP :** Parties C.2, D.1, F.5, F.8, F.9 (`@GetMapping`, `@PostMapping`, etc.).
- **Paramètres :** Parties C.2, D.4, F.5 (`@RequestParam`, `@PathVariable`, `@RequestBody`).
- **Réponse HTTP :** Parties C.2, D.2, E.4, F.5 (`ResponseEntity`).
- **Conversion JSON :** Parties E.1 et E.2 (sérialisation Jackson).
- **Création des objets :** Parties C.1, C.2, F.4, F.5 (`@Service` et injection par constructeur).
- **Communication :** Parties F.3 et F.5 (requêtes HTTP distantes via `RestClient`).

#### 2. Tableau de correspondance JAX-RS (TP2) vs Spring Boot (TP3)
| Rôle | JAX-RS (TP2) | Spring Boot (TP3) |
| :--- | :--- | :--- |
| Déclaration de contrôleur REST | *(aucune / implicite)* | `@RestController` |
| Préfixe d'URI commun | `@Path("/calcul")` | `@RequestMapping("/calcul")` |
| Route HTTP GET | `@GET` + `@Path` | `@GetMapping` |
| Route HTTP POST | `@POST` + `@Path` | `@PostMapping` |
| Route HTTP PUT | `@PUT` + `@Path` | `@PutMapping` |
| Route HTTP DELETE | `@DELETE` + `@Path` | `@DeleteMapping` |
| Paramètre query string | `@QueryParam("a")` | `@RequestParam("a")` |
| Paramètre de chemin URI | `@PathParam("id")` | `@PathVariable("id")` |
| Corps JSON de la requête | *(implicite)* + `@Consumes` | `@RequestBody` |
| Format de réponse produit | `@Produces(MediaType.APPLICATION_JSON)` | `produces = MediaType.APPLICATION_JSON_VALUE` |
| Composant injectable | *(aucun / POJO standard)* | `@Service` |
| Injection de dépendance | `@Inject` | Injection par constructeur |

#### 3. Commandes pour construire et lancer sans IDE
```bash
# 1. Construction des JARs exécutables
cd calcul-service && mvn clean package -DskipTests
cd ../historique-service && mvn clean package -DskipTests

# 2. Lancement des microservices (dans deux terminaux)
java -jar calcul-service/target/calcul-service-0.0.1-SNAPSHOT.jar      # Terminal 1
java -jar historique-service/target/historique-service-0.0.1-SNAPSHOT.jar  # Terminal 2
```

#### 4. Un microservice est-il un service web ? Un service web est-il toujours un microservice ?
- **Microservice $\rightarrow$ Service Web ?** Oui, s'il expose des interfaces via les protocoles et formats standards du Web (HTTP, REST, JSON), comme dans ce TP.
- **Service Web $\rightarrow$ Microservice ?** Non. L'application du TP2 était un service web REST complet, mais c'était un **monolithe** regroupant toutes les fonctions dans un seul livrable WAR.

#### 5. Problème de l'URL codée en dur face à la répartition de charge
L'adresse étant fixe (`http://localhost:8081`), `historique-service` enverra toujours ses requêtes à cette seule instance. Il ne peut ni découvrir automatiquement d'autres instances, ni répartir la charge (*load balancing*), ni basculer en cas de panne (*failover*). Cela nécessite un **Service Registry** (ex: Eureka) et un client avec Load Balancer (sujet du TP4).

#### 6. Principe stateless vs conservation de la liste des opérations
Le principe *stateless* interdit de conserver l'**état de session d'un client** spécifique entre deux requêtes. L'historique géré par `historique-service` représente les **données métier de l'application** (comme une table de base de données). Chaque requête reste totalement autonome et indépendante de tout contexte client antérieur.

#### 7. Répétition de POST `/operations` vs répétition de PUT `/operations/1`
- **`POST /operations` n'est pas idempotent :** Deux envois successifs créent deux opérations distinctes avec deux identifiants différents (`id=1` puis `id=2`). L'état change à chaque fois.
- **`PUT /operations/1` est idempotent :** Répéter deux fois la même mise à jour applique les mêmes données sur l'opération n°1. L'état final du système reste rigoureusement inchangé après le second appel.
- Le comportement est **strictement identique à celui du TP2**.

---

## 🧪 Résultats des Tests F.6, F.7, F.8 et F.9

### Tests F.6 : Validation du Contrôleur
| N° | Requête envoyée | Corps envoyé | Statut | Corps de la réponse |
| :---: | :--- | :--- | :---: | :--- |
| **1** | `POST http://localhost:8082/operations` | `{"operateur":"somme","a":8,"b":2}` | **201 Created** | `{"id":1,"operateur":"somme","a":8,"b":2,"resultat":10.0}` |
| **2** | `POST http://localhost:8082/operations` | `{"operateur":"division","a":7,"b":2}` | **201 Created** | `{"id":2,"operateur":"division","a":7,"b":2,"resultat":3.5}` |
| **3** | `GET http://localhost:8082/operations` | *Aucun* | **200 OK** | `[{"id":1,...},{"id":2,...}]` |
| **4** | `GET http://localhost:8082/operations/2` | *Aucun* | **200 OK** | `{"id":2,"operateur":"division","a":7,"b":2,"resultat":3.5}` |
| **5** | `GET http://localhost:8082/operations/99` | *Aucun* | **404 Not Found** | *(Corps vide)* |
| **6** | `POST http://localhost:8082/operations` | `{"operateur":"division","a":7,"b":0}` | **400 Bad Request** | `{"erreur":"Erreur : division par 0 impossible"}` |
| **7** | `POST http://localhost:8082/operations` | `{"operateur":"racine","a":9,"b":0}` | **400 Bad Request** | `{"erreur":"Opérateur inconnu : racine"}` |

### Tests F.7 : Arrêt de `calcul-service` (Résilience)
| N° | Action / Requête | Corps envoyé | Statut | Corps de la réponse |
| :---: | :--- | :--- | :---: | :--- |
| **8** | `GET http://localhost:8082/operations` *(8081 arrêté)* | *Aucun* | **200 OK** | Liste complète lue en mémoire locale |
| **9** | `GET http://localhost:8082/operations/1` *(8081 arrêté)* | *Aucun* | **200 OK** | Opération 1 lue avec succès |
| **10** | `POST http://localhost:8082/operations` *(8081 arrêté)* | `{"operateur":"produit","a":3,"b":4}` | **503 Service Unavailable** | `{"erreur":"Service calcul-service indisponible"}` |
| **10b** | `POST http://localhost:8082/operations` *(8081 relancé)* | `{"operateur":"produit","a":3,"b":4}` | **201 Created** | `{"id":3,"operateur":"produit","a":3,"b":4,"resultat":12.0}` |

### Tests F.8 : Modification (`PUT /operations/{id}`)
| Action / Situation | Requête | Corps envoyé | Statut | Corps de la réponse |
| :--- | :--- | :--- | :---: | :--- |
| ID inexistant | `PUT http://localhost:8082/operations/99` | `{"operateur":"somme","a":1,"b":1}` | **404 Not Found** | *(Corps vide)* |
| Division par 0 | `PUT http://localhost:8082/operations/1` | `{"operateur":"division","a":10,"b":0}` | **400 Bad Request** | `{"erreur":"Erreur : division par 0 impossible"}` |
| Modification réussie | `PUT http://localhost:8082/operations/1` | `{"operateur":"produit","a":8,"b":2}` | **200 OK** | `{"id":1,"operateur":"produit","a":8,"b":2,"resultat":16.0}` |
| Vérification GET | `GET http://localhost:8082/operations/1` | *Aucun* | **200 OK** | `{"id":1,"operateur":"produit","a":8,"b":2,"resultat":16.0}` |

### Tests F.9 : Suppression (`DELETE /operations/{id}`)
| Action / Situation | Requête | Statut | Corps de la réponse |
| :--- | :--- | :---: | :--- |
| Suppression réussie | `DELETE http://localhost:8082/operations/1` | **204 No Content** | *(Corps vide)* |
| Vérification GET | `GET http://localhost:8082/operations/1` | **404 Not Found** | *(Corps vide)* |
| Seconde suppression | `DELETE http://localhost:8082/operations/1` | **404 Not Found** | *(Corps vide)* |
