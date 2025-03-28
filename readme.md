# Spring Boot Project Overview

This repository demonstrates several core concepts and techniques in Spring Boot development. The project includes:

- **JPA & Hibernate:** Creating models/entities and repositories to interact with a database.
- **Flyway Migrations:** Managing and versioning database schema changes.
- **Java Faker:** Seeding the database with dummy data.
- **Cart System Example:** A simple shopping cart application that uses session data.
- **Dependency Inversion & Dependency Injection (DIP):** An image storage example comparing DI versus manual instantiation.

---

## Table of Contents

- [Project Setup](#project-setup)
- [JPA & Hibernate Models and Repositories](#jpa--hibernate-models-and-repositories)
- [Flyway Migrations](#flyway-migrations)
- [Generating Dummy Data with Java Faker](#generating-dummy-data-with-java-faker)
- [Cart System Example](#cart-system-example)
- [Dependency Inversion & Image Storage Example](#dependency-inversion--image-storage-example)
- [Running the Application](#running-the-application)
- [Common Issues & Troubleshooting](#common-issues--troubleshooting)

---

## Project Setup

This Spring Boot project is built using Maven and includes the following dependencies:
- **Spring Web:** To create REST controllers and serve web pages.
- **Spring Data JPA:** For ORM using Hibernate (JPA implementation).
- **Database Connector:** For MySQL/MariaDB/PostgreSQL/SQLite/SQL Server, etc.
- **Flyway:** For database migrations.
- **Java Faker:** For generating dummy data during development.

Ensure your `pom.xml` includes the relevant dependencies, for example:

```xml
<!-- Spring Boot Starter Web and Data JPA -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- Example for MySQL -->
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <version>8.0.33</version>
</dependency>

<!-- Flyway -->
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-core</artifactId>
  <version>10.20.1</version>
</dependency>

<!-- Java Faker -->
<dependency>
  <groupId>com.github.javafaker</groupId>
  <artifactId>javafaker</artifactId>
  <version>1.0.2</version>
</dependency>
```

---

## JPA & Hibernate Models and Repositories

### Models

Entities are mapped to database tables using standard JPA annotations. For example, the `Product` and `Comment` entities:

```java
// Product.java
package com.docencia.clase10.models;

import jakarta.persistence.*;
import java.util.ArrayList;
import java.util.List;
import org.hibernate.annotations.Fetch;
import org.hibernate.annotations.FetchMode;

@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private Integer price;

    @OneToMany(mappedBy = "product", fetch = FetchType.EAGER, cascade = CascadeType.ALL)
    @Fetch(FetchMode.SUBSELECT)
    private List<Comment> comments = new ArrayList<>();

    // Constructors, getters and setters...
}
```

```java
// Comment.java
package com.docencia.clase10.models;

import jakarta.persistence.*;

@Entity
@Table(name = "comments")
public class Comment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String description;

    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name="product_id")
    private Product product;

    // Constructors, getters and setters...
}
```

### Repositories

Repositories are defined by extending Spring Data JPA’s `JpaRepository`:

```java
// ProductRepository.java
package com.docencia.clase10.repositories;

import com.docencia.clase10.models.Product;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
}
```

```java
// CommentRepository.java
package com.docencia.clase10.repositories;

import com.docencia.clase10.models.Comment;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface CommentRepository extends JpaRepository<Comment, Long> {
}
```

---

## Flyway Migrations

Flyway is used for managing database schema changes via versioned SQL scripts.

1. **Migration Scripts Location:**  
   Place your migration files in `src/main/resources/db/migration`.

2. **Example Migration Script:**

```sql
-- V1__create_person_table.sql
CREATE TABLE PERSON (
    ID INT NOT NULL PRIMARY KEY AUTO_INCREMENT,
    NAME VARCHAR(100) NOT NULL
);
```

3. **Configuration in application.properties:**

```properties
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
spring.flyway.baseline-on-migrate=true
```

When the application starts, Flyway will automatically run pending migrations.

---

## Generating Dummy Data with Java Faker

A class implementing `CommandLineRunner` is used to seed the database at startup. For example, a `DataLoader` that creates products and comments:

```java
package com.docencia.clase10.bootstrap;

import com.docencia.clase10.models.Product;
import com.docencia.clase10.models.Comment;
import com.docencia.clase10.repositories.ProductRepository;
import com.docencia.clase10.repositories.CommentRepository;
import com.github.javafaker.Faker;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;
import java.util.Locale;
import java.util.Random;

@Component
public class DataLoader implements CommandLineRunner {

    private final ProductRepository productRepository;
    private final CommentRepository commentRepository;

    public DataLoader(ProductRepository productRepository, CommentRepository commentRepository) {
        this.productRepository = productRepository;
        this.commentRepository = commentRepository;
    }

    @Override
    public void run(String... args) throws Exception {
        Faker faker = new Faker(new Locale("es"));
        Random random = new Random();

        // Insert 5 products with 1-3 comments each
        for (int i = 0; i < 5; i++) {
            String productName = faker.commerce().productName();
            int price = random.nextInt(500) + 50;
            Product product = new Product(productName, price);

            int numComments = random.nextInt(3) + 1;
            for (int j = 0; j < numComments; j++) {
                String description = faker.lorem().sentence();
                Comment comment = new Comment(description, product);
                product.getComments().add(comment);
            }
            productRepository.save(product);
        }

        System.out.println("Products and comments generated.");
    }
}
```

---

## Cart System Example

### Overview

A simple shopping cart application uses session attributes to track products added to the cart. The product “database” is simulated with a hard-coded map.

### Controller

```java
package com.docencia.clase10.controllers;

import com.docencia.clase10.models.Product;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import java.util.HashMap;
import java.util.Map;

@Controller
@RequestMapping("/cart")
public class CartController {

    // Simulated database of products
    private final Map<Integer, Product> products = new HashMap<>();

    public CartController() {
        products.put(121, new Product(121, "TV Samsung", 1000));
        products.put(11, new Product(11, "iPhone", 2000));
    }

    @GetMapping
    public String index(HttpSession session, Model model) {
        Map<Integer, Integer> cartProductData = (Map<Integer, Integer>) session.getAttribute("cart_product_data");
        Map<Integer, Product> cartProducts = new HashMap<>();

        if (cartProductData != null) {
            for (Integer id : cartProductData.keySet()) {
                if (products.containsKey(id)) {
                    cartProducts.put(id, products.get(id));
                }
            }
        }

        model.addAttribute("title", "Cart - Online Store");
        model.addAttribute("subtitle", "Shopping Cart");
        model.addAttribute("products", products);
        model.addAttribute("cartProducts", cartProducts);
        return "cart/index";
    }

    @GetMapping("/add/{id}")
    public String add(@PathVariable Integer id, HttpSession session) {
        Map<Integer, Integer> cartProductData = (Map<Integer, Integer>) session.getAttribute("cart_product_data");
        if (cartProductData == null) {
            cartProductData = new HashMap<>();
        }
        cartProductData.put(id, id);
        session.setAttribute("cart_product_data", cartProductData);
        return "redirect:/cart";
    }

    @GetMapping("/removeAll")
    public String removeAll(HttpSession session) {
        session.removeAttribute("cart_product_data");
        return "redirect:/cart";
    }
}
```

### Thymeleaf View

Create `src/main/resources/templates/cart/index.html`:

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
  <meta charset="UTF-8">
  <title th:text="${title}">Cart - Online Store</title>
</head>
<body>
  <h1 th:text="${subtitle}">Shopping Cart</h1>

  <h2>Available Products</h2>
  <ul>
    <li th:each="entry : ${products}">
      <span>Id: <span th:text="${entry.key}"></span> - </span>
      <span>Name: <span th:text="${entry.value.name}"></span> - </span>
      <span>Price: <span th:text="${entry.value.price}"></span> - </span>
      <a th:href="@{'/cart/add/' + ${entry.key}}">Add to cart</a>
    </li>
  </ul>

  <h2>Products in Cart</h2>
  <ul>
    <li th:each="entry : ${cartProducts}">
      <span>Id: <span th:text="${entry.key}"></span> - </span>
      <span>Name: <span th:text="${entry.value.name}"></span> - </span>
      <span>Price: <span th:text="${entry.value.price}"></span></span>
    </li>
  </ul>
  <a th:href="@{/cart/removeAll}">Remove all products from cart</a>
</body>
</html>
```

> **Note:** If you encounter issues iterating over a map, iterate directly (as shown above) instead of using `#maps.entries(...)`.

---

## Dependency Inversion & Image Storage Example

### Overview

This module demonstrates the Dependency Inversion Principle (DIP) using an interface for image storage and two implementations:
- **With DI:** The implementation is injected via Spring's container.
- **Without DI:** The implementation is directly instantiated in the controller.

### Components

#### 1. ImageStorage Interface

```java
package com.docencia.clase10.interfaces;

import org.springframework.web.multipart.MultipartFile;

public interface ImageStorage {
    void store(MultipartFile file);
}
```

#### 2. ImageLocalStorage Implementation

```java
package com.docencia.clase10.util;

import com.docencia.clase10.interfaces.ImageStorage;
import org.springframework.stereotype.Component;
import org.springframework.web.multipart.MultipartFile;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;

@Component
public class ImageLocalStorage implements ImageStorage {
    private static final String STORAGE_DIR = "uploads/";

    @Override
    public void store(MultipartFile file) {
        if (file != null && !file.isEmpty()) {
            try {
                Path storageDir = Paths.get(STORAGE_DIR);
                if (!Files.exists(storageDir)) {
                    Files.createDirectories(storageDir);
                }
                Path destinationFile = storageDir.resolve("test.png").normalize().toAbsolutePath();
                Files.copy(file.getInputStream(), destinationFile);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

#### 3. Service Provider (Configuration)

```java
package com.docencia.clase10.config;

import com.docencia.clase10.interfaces.ImageStorage;
import com.docencia.clase10.util.ImageLocalStorage;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ImageServiceProvider {
    @Bean
    public ImageStorage imageStorage() {
        return new ImageLocalStorage();
    }
}
```

#### 4. Controllers

- **With Dependency Injection:**

```java
package com.docencia.clase10.controllers;

import com.docencia.clase10.interfaces.ImageStorage;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Controller
@RequestMapping("/image")
public class ImageController {

    private final ImageStorage imageStorage;

    @Autowired
    public ImageController(ImageStorage imageStorage) {
        this.imageStorage = imageStorage;
    }

    @GetMapping
    public String index(Model model) {
        return "image/index";
    }

    @PostMapping("/save")
    public String save(@RequestParam("profile_image") MultipartFile profileImage,
                       RedirectAttributes redirectAttributes) {
        imageStorage.store(profileImage);
        redirectAttributes.addFlashAttribute("message", "Image uploaded successfully!");
        return "redirect:/image";
    }
}
```

- **Without Dependency Injection:**

```java
package com.docencia.clase10.controllers;

import com.docencia.clase10.util.ImageLocalStorage;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Controller
@RequestMapping("/image-not-di")
public class ImageNotDIController {

    @GetMapping
    public String index(Model model) {
        return "imagenotdi/index";
    }

    @PostMapping("/save")
    public String save(@RequestParam("profile_image") MultipartFile profileImage,
                       RedirectAttributes redirectAttributes) {
        ImageLocalStorage storage = new ImageLocalStorage();
        storage.store(profileImage);
        redirectAttributes.addFlashAttribute("message", "Image uploaded successfully (not DI)!");
        return "redirect:/image-not-di";
    }
}
```

#### 5. Thymeleaf Views

- **For DI (in `src/main/resources/templates/image/index.html`):**

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Image Storage - DI</title>
</head>
<body>
<div class="container">
    <h1>Upload Image</h1>
    <form th:action="@{/image/save}" method="post" enctype="multipart/form-data">
        <input type="file" name="profile_image"/>
        <button type="submit">Submit</button>
    </form>
    <div th:if="${message}">
        <p th:text="${message}"></p>
    </div>
    <div>
        <img th:src="@{'/uploads/test.png'}" alt="Uploaded Image"/>
    </div>
</div>
</body>
</html>
```

- **For Non-DI (in `src/main/resources/templates/imagenotdi/index.html`):**

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Image Storage - Without DI</title>
</head>
<body>
<div class="container">
    <h1>Upload Image (Without Dependency Injection)</h1>
    <form th:action="@{/image-not-di/save}" method="post" enctype="multipart/form-data">
        <input type="file" name="profile_image"/>
        <button type="submit">Submit</button>
    </form>
    <div th:if="${message}">
        <p th:text="${message}"></p>
    </div>
    <div>
        <img th:src="@{'/uploads/test.png'}" alt="Uploaded Image"/>
    </div>
</div>
</body>
</html>
```

#### 6. Static Resource Configuration

To serve the uploaded images, add a resource handler:

```java
package com.docencia.clase10.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class StaticResourceConfiguration implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/uploads/**")
                .addResourceLocations("file:uploads/");
    }
}
```

---

## Running the Application

1. **Build and Run:**

   Use Maven:
   ```bash
   mvn spring-boot:run
   ```
   Or build an executable JAR:
   ```bash
   mvn clean package
   java -jar target/your-application-name.jar
   ```

2. **Test the Endpoints:**
   - **Cart System:** Visit [http://localhost:8080/cart](http://localhost:8080/cart).
   - **Image Storage with DI:** Visit [http://localhost:8080/image](http://localhost:8080/image).
   - **Image Storage without DI:** Visit [http://localhost:8080/image-not-di](http://localhost:8080/image-not-di).

---

## Common Issues & Troubleshooting

- **HTTP 413 – Maximum Upload Size Exceeded:**  
  Adjust file upload limits in `application.properties`:
  ```properties
  spring.servlet.multipart.max-file-size=10MB
  spring.servlet.multipart.max-request-size=10MB
  ```
- **Image Not Displaying:**  
  - Confirm the image is saved in the `uploads` folder.
  - Verify the resource handler mapping and file permissions.
  - Directly access the image via `http://localhost:8080/uploads/test.png`.
- **Thymeleaf Map Iteration Errors:**  
  Iterate directly over the map rather than using `#maps.entries(...)`.

---

## Additional Notes

- **Hibernate & JPA:**  
  Hibernate automatically maps your Java entities to database tables based on your model definitions.
- **Flyway Migrations:**  
  Use Flyway to version and manage database schema changes with SQL migration files.
- **Java Faker:**  
  Seed your database with dummy data using a `CommandLineRunner`.
- **Dependency Injection:**  
  Utilize Spring's DI to decouple your code and improve testability and flexibility.
- **Cart System:**  
  A simple example using session management to simulate a shopping cart.

---
