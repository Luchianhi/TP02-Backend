# LAB 2 : Développement d'une API REST CRUD avec Spring Boot

API REST de gestion de produits développée avec **Spring Boot**, organisée en couches (Controller, Service, Repository), utilisant des **DTOs**, la **validation** des données et une **gestion centralisée des erreurs**. Les données sont stockées **en mémoire** (sans base de données).

---

## Table des matières

1. [Introduction et prérequis](#1-introduction-et-prérequis)
2. [Création du projet Spring Boot](#2-création-du-projet-spring-boot)
3. [Architecture et organisation des packages](#3-architecture-et-organisation-des-packages)
4. [Définition de l'entité Produit](#4-définition-de-lentité-produit)
5. [Création des DTOs](#5-création-des-dtos-data-transfer-objects)
6. [Implémentation du Repository en mémoire](#6-implémentation-du-repository-en-mémoire)
7. [Développement de la couche Service](#7-développement-de-la-couche-service)
8. [Gestion des exceptions personnalisées](#8-gestion-des-exceptions-personnalisées)
9. [Gestion centralisée des erreurs](#9-gestion-centralisée-des-erreurs)
10. [Développement du Controller REST](#10-développement-du-controller-rest)
11. [Complétion de la logique Service](#11-complétion-de-la-logique-service)
12. [Configuration et lancement de l'application](#12-configuration-et-lancement-de-lapplication)
13. [Test de l'endpoint de création (POST)](#13-test-de-lendpoint-de-création-post)
14. [Test des endpoints de consultation (GET)](#14-test-des-endpoints-de-consultation-get)
15. [Test des endpoints de modification et suppression](#15-test-des-endpoints-de-modification-et-suppression)
16. [Synthèse, exercices et évaluation](#16-synthèse-exercices-et-évaluation)

---

## 1. Introduction et prérequis

### Objectifs d'apprentissage

Ce labo vise à maîtriser le développement d'APIs RESTful avec Spring Boot. Vous apprendrez à :

- structurer une application en couches (Controller, Service, Repository) ;
- manipuler des données JSON via des DTOs ;
- gérer les erreurs de manière professionnelle.

### Prérequis techniques

- JDK 21 installé et configuré
- Maven ou Gradle pour la gestion des dépendances
- IntelliJ IDEA ou VS Code avec les plugins Java
- Postman ou `curl` pour tester les endpoints

### Ce que vous allez construire

Une API de gestion de produits fonctionnelle, **sans base de données**, en utilisant une collection en mémoire. Cela permet de se concentrer sur la logique REST et l'architecture Spring avant d'aborder la persistance dans le **LAB 3**.

### Architecture

```
Client (Postman / curl)
        │  Requêtes HTTP (REST / JSON)
        ▼
┌─────────────────────────────────────────────┐
│  Application Spring Boot (Product API)      │
│                                             │
│  Controller  →  Service  →  Repository      │
│  (REST)         (Métier)    (en mémoire)    │
│                                             │
│  Modèles & DTOs : Product, ProductRequest,  │
│                   ProductResponse           │
│                                             │
│  Gestion centralisée des exceptions         │
│  (GlobalExceptionHandler)                   │
└─────────────────────────────────────────────┘
```

**Flux de traitement (exemple : création d'un produit)**

1. Le client envoie une requête `POST /products` (JSON).
2. Le contrôleur valide la requête (`@Valid`) et la transmet au service.
3. Le service applique la logique métier et crée le `Product`.
4. Le repository sauvegarde le produit en mémoire et génère l'ID.
5. Le contrôleur retourne un `ProductResponse` (`201 Created`).

**Technologies et concepts clés** : Spring Boot 3.x (API REST), Java 21, Maven, Spring Web, Jakarta Validation (Bean Validation), architecture en couches (Controller / Service / Repository).

---

## 2. Création du projet Spring Boot

### Initialisation avec Spring Initializr

Accédez à [start.spring.io](https://start.spring.io) pour générer le squelette du projet. Sélectionnez :

- **Build tool** : Maven
- **Langage** : Java
- **Java** : 21
- **Spring Boot** : 3.x

### Dépendances à ajouter

- **Spring Web** : pour créer les endpoints REST.
- **Validation** : pour la validation des entrées (Jakarta Bean Validation).

Générez le projet et importez-le dans votre IDE. Vérifiez que la classe principale `ProductApiApplication.java` est correctement générée avec l'annotation `@SpringBootApplication`.

```java
@SpringBootApplication
public class ProductApiApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProductApiApplication.class, args);
    }
}
```

---

## 3. Architecture et organisation des packages

### Structure en couches

Une bonne architecture sépare les responsabilités. Créez les packages suivants sous `com.example.productapi` :

| Package | Rôle |
|---|---|
| `controller` | Gère les requêtes HTTP |
| `service` | Contient la logique métier |
| `repository` | Gère l'accès aux données (en mémoire ici) |
| `model` | Entités métier |
| `dto` | Objets de transfert de données |
| `exception` | Gestion des erreurs |

Structure attendue du projet :

```
com.example.productapi
├── ProductApiApplication.java
├── controller
│   └── ProductController.java
├── service
│   └── ProductService.java
├── repository
│   └── ProductRepository.java
├── model
│   └── Product.java
├── dto
│   ├── ProductRequest.java
│   └── ProductResponse.java
└── exception
    ├── GlobalExceptionHandler.java
    └── ResourceNotFoundException.java
```

> **Conseil :** ne mélangez jamais la logique métier dans les contrôleurs. Le contrôleur doit être mince et déléguer au service.

---

## 4. Définition de l'entité Produit

### Création du modèle de données

Créez la classe `Product` dans le package `model`. Cette classe représente l'entité métier avec ses attributs : `id`, `name`, `description`, `price` et `quantity`.

```java
package com.example.productapi.model;

import java.math.BigDecimal;

public class Product {
    private Long id;
    private String name;
    private String description;
    private BigDecimal price;
    private Integer quantity;

    // Constructeurs, getters et setters
}
```

> Utilisez `BigDecimal` pour les prix afin d'éviter les erreurs d'arrondi des nombres flottants. Ajoutez un **constructeur sans argument** et un **constructeur avec arguments** pour la facilité d'utilisation.

---

## 5. Création des DTOs (Data Transfer Objects)

### Séparation des données d'entrée/sortie

Les DTOs permettent de contrôler exactement ce qui est envoyé et reçu par l'API. Créez `ProductRequest` pour les opérations de création/modification et `ProductResponse` pour les retours.

**`ProductRequest`**

```java
package com.example.productapi.dto;

import jakarta.validation.constraints.*;
import java.math.BigDecimal;

public class ProductRequest {

    @NotBlank(message = "Le nom est obligatoire")
    private String name;

    @NotNull(message = "Le prix est obligatoire")
    @Positive(message = "Le prix doit être positif")
    private BigDecimal price;

    @NotNull(message = "La quantité est obligatoire")
    @Min(value = 0, message = "La quantité ne peut pas être négative")
    private Integer quantity;

    private String description;

    // Getters et Setters
}
```

**`ProductResponse`**

```java
package com.example.productapi.dto;

import java.math.BigDecimal;

public class ProductResponse {
    private Long id;
    private String name;
    private String description;
    private BigDecimal price;
    private Integer quantity;

    public ProductResponse(Long id, String name, String description,
                           BigDecimal price, Integer quantity) {
        this.id = id;
        this.name = name;
        this.description = description;
        this.price = price;
        this.quantity = quantity;
    }

    // Getters
}
```

Les annotations Jakarta Bean Validation sont appliquées directement sur les DTOs pour valider les données **avant** qu'elles n'atteignent la logique métier.

---

## 6. Implémentation du Repository en mémoire

### Stockage des données

Puisque nous n'utilisons pas de base de données, nous créons une classe `ProductRepository` qui gère une `ArrayList` ou une `HashMap` en mémoire.

```java
package com.example.productapi.repository;

import com.example.productapi.model.Product;
import org.springframework.stereotype.Repository;
import java.util.*;
import java.util.concurrent.atomic.AtomicLong;

@Repository
public class ProductRepository {

    private final Map<Long, Product> products = new HashMap<>();
    private final AtomicLong idGenerator = new AtomicLong(1);

    public Product save(Product product) {
        if (product.getId() == null) {
            product.setId(idGenerator.getAndIncrement());
        }
        products.put(product.getId(), product);
        return product;
    }

    public Optional<Product> findById(Long id) {
        return Optional.ofNullable(products.get(id));
    }

    public List<Product> findAll() {
        return new ArrayList<>(products.values());
    }

    public void deleteById(Long id) {
        products.remove(id);
    }
}
```

L'annotation `@Repository` indique à Spring que cette classe est un composant de persistance. L'usage d'`AtomicLong` garantit la génération d'IDs uniques de manière *thread-safe*.

---

## 7. Développement de la couche Service

### Logique métier

La couche service orchestre les opérations. Créez `ProductService` et injectez le repository via le constructeur.

```java
package com.example.productapi.service;

import com.example.productapi.dto.ProductRequest;
import com.example.productapi.dto.ProductResponse;
import com.example.productapi.exception.ResourceNotFoundException;
import com.example.productapi.model.Product;
import com.example.productapi.repository.ProductRepository;
import org.springframework.stereotype.Service;
import java.util.List;
import java.util.stream.Collectors;

@Service
public class ProductService {

    private final ProductRepository repository;

    public ProductService(ProductRepository repository) {
        this.repository = repository;
    }

    public ProductResponse createProduct(ProductRequest request) {
        Product product = new Product();
        product.setName(request.getName());
        product.setDescription(request.getDescription());
        product.setPrice(request.getPrice());
        product.setQuantity(request.getQuantity());
        Product saved = repository.save(product);
        return mapToResponse(saved);
    }

    private ProductResponse mapToResponse(Product product) {
        return new ProductResponse(product.getId(), product.getName(),
                product.getDescription(), product.getPrice(), product.getQuantity());
    }

    // Autres méthodes : getAll, getById, update, delete (voir étape 11)
}
```

> L'injection par constructeur est préférée car elle rend les dépendances explicites et facilite le test unitaire.

---

## 8. Gestion des exceptions personnalisées

### Création des exceptions métier

Créez une exception `ResourceNotFoundException` pour les produits inexistants.

```java
package com.example.productapi.exception;

public class ResourceNotFoundException extends RuntimeException {

    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

Dans le service, lancez cette exception si un produit n'est pas trouvé :

```java
public ProductResponse getProductById(Long id) {
    Product product = repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Produit introuvable avec l'id: " + id));
    return mapToResponse(product);
}
```

Cette approche centralise la logique d'erreur et évite de renvoyer des erreurs génériques du serveur.

---

## 9. Gestion centralisée des erreurs

### Utilisation de `@RestControllerAdvice`

Créez une classe `GlobalExceptionHandler` pour intercepter les exceptions et renvoyer des réponses JSON standardisées.

```java
package com.example.productapi.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<Map<String, String>> handleNotFound(ResourceNotFoundException ex) {
        Map<String, String> errors = new HashMap<>();
        errors.put("error", ex.getMessage());
        return new ResponseEntity<>(errors, HttpStatus.NOT_FOUND);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
                errors.put(error.getField(), error.getDefaultMessage())
        );
        return new ResponseEntity<>(errors, HttpStatus.BAD_REQUEST);
    }
}
```

Cela garantit une cohérence dans les réponses d'erreur de l'API.

---

## 10. Développement du Controller REST

### Exposition des endpoints

Créez `ProductController` dans le package `controller`. Utilisez les annotations Spring pour mapper les requêtes HTTP.

```java
package com.example.productapi.controller;

import com.example.productapi.dto.ProductRequest;
import com.example.productapi.dto.ProductResponse;
import com.example.productapi.service.ProductService;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService service;

    public ProductController(ProductService service) {
        this.service = service;
    }

    @PostMapping
    public ResponseEntity<ProductResponse> createProduct(@Valid @RequestBody ProductRequest request) {
        ProductResponse response = service.createProduct(request);
        return new ResponseEntity<>(response, HttpStatus.CREATED);
    }

    @GetMapping
    public List<ProductResponse> getAllProducts() {
        return service.getAllProducts();
    }

    @GetMapping("/{id}")
    public ProductResponse getProductById(@PathVariable Long id) {
        return service.getProductById(id);
    }

    @PutMapping("/{id}")
    public ProductResponse updateProduct(@PathVariable Long id, @Valid @RequestBody ProductRequest request) {
        return service.updateProduct(id, request);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        service.deleteProduct(id);
        return ResponseEntity.noContent().build();
    }
}
```

L'annotation `@Valid` déclenche la validation des DTOs. `ResponseEntity` permet de contrôler le code de statut HTTP retourné.

---

## 11. Complétion de la logique Service

### Implémentation des méthodes restantes

Complétez le `ProductService` avec les méthodes de récupération, mise à jour et suppression.

```java
public List<ProductResponse> getAllProducts() {
    return repository.findAll().stream()
            .map(this::mapToResponse)
            .collect(Collectors.toList());
}

public ProductResponse updateProduct(Long id, ProductRequest request) {
    Product product = repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Produit introuvable avec l'id: " + id));

    product.setName(request.getName());
    product.setDescription(request.getDescription());
    product.setPrice(request.getPrice());
    product.setQuantity(request.getQuantity());

    Product updated = repository.save(product);
    return mapToResponse(updated);
}

public void deleteProduct(Long id) {
    if (!repository.findById(id).isPresent()) {
        throw new ResourceNotFoundException("Produit introuvable avec l'id: " + id);
    }
    repository.deleteById(id);
}
```

> Assurez-vous que chaque opération vérifie l'existence de la ressource avant de procéder à la modification ou à la suppression.

---

## 12. Configuration et lancement de l'application

### Vérification de la configuration

Assurez-vous que le fichier `application.properties` est vide ou contient uniquement les paramètres de base. Spring Boot configure automatiquement le serveur Tomcat embarqué.

```properties
# application.properties
server.port=8080
spring.application.name=product-api
```

Lancez l'application via la classe principale :

```bash
./mvnw spring-boot:run
```

Vous devriez voir dans la console que le serveur est démarré sur le port **8080**.

> **Note :** si vous avez des erreurs de compilation, vérifiez que tous les imports sont présents et que les packages correspondent à la structure du projet.

---

## 13. Test de l'endpoint de création (POST)

### Test avec Postman

Ouvrez Postman et créez une requête **POST** vers `http://localhost:8080/api/products`.

Dans l'onglet **Body**, sélectionnez **raw** et **JSON**, puis collez :

```json
{
  "name": "Ordinateur Portable",
  "description": "Laptop 15 pouces",
  "price": 1200.50,
  "quantity": 10
}
```

Envoyez la requête. Vous devriez recevoir un statut **201 Created** et le JSON du produit créé avec un `id` généré.

Testez aussi une **requête invalide** (sans nom) pour vérifier que la validation renvoie un **400 Bad Request** avec les messages d'erreur.

---

## 14. Test des endpoints de consultation (GET)

### Récupération des données

- Testez la récupération de **tous** les produits avec une requête **GET** vers `http://localhost:8080/api/products`. Vous devriez voir la liste des produits créés.
- Testez la récupération d'**un produit spécifique** avec **GET** vers `http://localhost:8080/api/products/1` (remplacez `1` par l'id réel).
- Testez la récupération d'un **produit inexistant** (ex : id 999). Vous devriez recevoir un statut **404 Not Found** avec le message d'erreur défini dans le `GlobalExceptionHandler`.

```bash
curl -X GET http://localhost:8080/api/products/1
```

---

## 15. Test des endpoints de modification et suppression

### Mise à jour et suppression

Testez la modification avec une requête **PUT** vers `http://localhost:8080/api/products/1`. Modifiez le prix :

```json
{
  "name": "Ordinateur Portable Pro",
  "description": "Laptop 15 pouces amélioré",
  "price": 1300.00,
  "quantity": 8
}
```

Vous devriez recevoir un **200 OK** avec les nouvelles données.

Testez la suppression avec une requête **DELETE** vers `http://localhost:8080/api/products/1`. Le statut attendu est **204 No Content**. Vérifiez ensuite que le produit n'apparaît plus dans la liste des produits.

```bash
curl -X DELETE http://localhost:8080/api/products/1
```

### Récapitulatif des endpoints

| Méthode | URL | Description | Statut succès |
|---|---|---|---|
| `POST` | `/api/products` | Créer un produit | `201 Created` |
| `GET` | `/api/products` | Lister les produits | `200 OK` |
| `GET` | `/api/products/{id}` | Récupérer un produit | `200 OK` |
| `PUT` | `/api/products/{id}` | Modifier un produit | `200 OK` |
| `DELETE` | `/api/products/{id}` | Supprimer un produit | `204 No Content` |

---

## 16. Synthèse, exercices et évaluation

### Résumé et compétences acquises

Vous avez appris à structurer une API REST avec Spring Boot, à utiliser les DTOs, la validation et la gestion centralisée des exceptions. Vous maîtrisez maintenant le cycle de vie d'une requête REST.

### Exercices d'application

1. Ajoutez un endpoint `GET /api/products/search?name=...` pour filtrer par nom.
2. Créez une exception `InvalidQuantityException` si la quantité dépasse 1000.
3. Refactorisez le code pour utiliser des **records Java** pour les DTOs.

### Livrables attendus

- Code source complet du projet.
- Collection Postman exportée en JSON.
- `README.md` expliquant comment lancer le projet.

### Transition vers le LAB 3

Le prochain labo introduira la persistance avec **Spring Data JPA**, **Hibernate** et **MySQL**, remplaçant la collection en mémoire par une vraie base de données.

---

## Lancer le projet

```bash
# Cloner le dépôt
git clone <url-du-depot>
cd product-api

# Lancer l'application
./mvnw spring-boot:run
```

L'API est ensuite disponible sur `http://localhost:8082/api/products`.
