# 🧭 Mapowanie obiektowo‑relacyjne w Spring (JPA/Hibernate)

---

## 📚 Spis treści

1. [Podstawy JPA w Spring](#-podstawy-jpa-w-spring)
2. [Mapowanie klasy na tabelę](#-mapowanie-klasy-na-tabelę)
3. [Klucz główny i generowanie ID](#-klucz-główny-i-generowanie-id)
4. [Mapowanie pól na kolumny](#-mapowanie-pól-na-kolumny)
5. [Relacje między encjami](#-relacje-między-encjami)

   * [@OneToOne](#-relacja-onetoone)
   * [@ManyToOne / @OneToMany](#-relacja-manytoone--onetomany)
   * [@ManyToMany](#-relacja-manytomany)
6. [Typy ładowania i kaskady](#-typy-ładowania-i-kaskady)
7. [Wartości osadzone (@Embeddable)](#-wartości-osadzone-embeddable)

---

## 🧱 Podstawy JPA w Spring

W Springu do mapowania obiektowo‑relacyjnego najczęściej używamy:

* **JPA** (Java Persistence API) – standard
* **Hibernate** – najpopularniejsza implementacja JPA

W projekcie Spring Boot najczęściej dodajemy starter:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

Dodatkowo zależność do wybranej bazy (np. PostgreSQL, MySQL, H2 itd.).

---

## 🏗 Mapowanie klasy na tabelę

Podstawowe adnotacje:

* `@Entity` – oznacza klasę jako encję JPA (odpowiada tabeli w bazie)
* `@Table(name = "nazwa_tabeli")` – pozwala wskazać konkretną nazwę tabeli

```java
import jakarta.persistence.*;

@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String username;

    // gettery/settery/konstruktory
}
```

Jeśli nie podamy `@Table`, nazwa tabeli zostanie wywnioskowana z nazwy klasy (zwykle z uwzględnieniem strategii namingu ustawionej w Hibernate).

---

## 🔑 Klucz główny i generowanie ID

Każda encja **musi mieć klucz główny** oznaczony `@Id`.

Popularne strategie generowania klucza:

* `GenerationType.IDENTITY` – auto‑increment po stronie bazy (np. MySQL, PostgreSQL)
* `GenerationType.SEQUENCE` – korzysta z sekwencji (często w PostgreSQL)
* `GenerationType.AUTO` – JPA wybierze strategię na podstawie bazy

### Przykład – IDENTITY

```java
@Entity
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
}
```

### Przykład – SEQUENCE z własną sekwencją

```java
@Entity
@SequenceGenerator(
        name = "order_seq",
        sequenceName = "order_seq",
        allocationSize = 1
)
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
    private Long id;

    private String number;
}
```

---

## 🧩 Mapowanie pól na kolumny

Podstawowe adnotacje dla pól:

* `@Column(name = "kolumna", nullable = false, unique = true, length = 100, ...)`
* `@Temporal` (dla typów `Date` – rzadziej używane, lepiej `LocalDate`, `LocalDateTime`)
* `@Enumerated(EnumType.STRING)` – dla enumów

### Przykład z @Column

```java
@Entity
@Table(name = "customers")
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "first_name", nullable = false, length = 50)
    private String firstName;

    @Column(name = "last_name", nullable = false, length = 50)
    private String lastName;

    @Column(name = "email", unique = true)
    private String email;
}
```

### Enum w kolumnie

```java
public enum OrderStatus {
    NEW, PAID, SHIPPED, CANCELLED
}

@Entity
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private OrderStatus status;
}
```

> 🔎 **Ważne:** `EnumType.STRING` jest bezpieczniejsze niż `ORDINAL`, bo nie psuje mapowania przy zmianie kolejności wartości w enumie.

---

## 🤝 Relacje między encjami

Relacje w JPA odwzorowują powiązania między tabelami (klucze obce). Główne typy relacji:

* `@OneToOne`
* `@ManyToOne`
* `@OneToMany`
* `@ManyToMany`

Każda relacja może mieć dodatkowe parametry:

* `fetch = FetchType.LAZY/EAGER`
* `cascade = {CascadeType.PERSIST, ...}`
* `mappedBy = "pole"` – definiuje stronę odwrotną relacji

---

### 🔗 Relacja @OneToOne

Przykład: `User` ↔ `UserProfile` (jeden do jednego).

```java
@Entity
@Table(name = "user_profiles")
public class UserProfile {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String firstName;
    private String lastName;

    @OneToOne
    @JoinColumn(name = "user_id", nullable = false, unique = true)
    private User user;
}

@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String username;

    @OneToOne(mappedBy = "user", fetch = FetchType.LAZY)
    private UserProfile profile;
}
```

* `@JoinColumn` wskazuje kolumnę z kluczem obcym.
* `mappedBy` po stronie `User` oznacza, że kolumna z kluczem obcym jest w tabeli `user_profiles`.

---

### 🌿 Relacja @ManyToOne / @OneToMany

Najczęstszy przypadek: wiele encji „dzieci” należy do jednego „rodzica”. Np. `Order` → `Customer`.

```java
@Entity
@Table(name = "customers")
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Order> orders = new ArrayList<>();
}

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    private BigDecimal totalAmount;
}
```

**Uwagi:**

* Strona `@ManyToOne` jest **właścicielem relacji** (trzyma klucz obcy w tabeli `orders`).
* Strona `@OneToMany(mappedBy = "customer")` jest stroną odwrotną.
* `orphanRemoval = true` usuwa „osierocone” encje `Order` po usunięciu z kolekcji.

---

### 🔁 Relacja @ManyToMany

Przykład: `Student` ↔ `Course` (wielu studentów na wielu kursach).

```java
@Entity
@Table(name = "students")
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToMany
    @JoinTable(
            name = "students_courses",
            joinColumns = @JoinColumn(name = "student_id"),
            inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
}

@Entity
@Table(name = "courses")
public class Course {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

> ⚠️ **Uwaga:** `@ManyToMany` często warto zastąpić dwiema relacjami `@OneToMany` + encją pośrednią (np. `Enrollment`) – daje to większą kontrolę i możliwość dodawania dodatkowych pól.

---

## ⚙️ Typy ładowania i kaskady

### FetchType

* `EAGER` – natychmiastowe ładowanie powiązanej encji (może powodować „lawinowe” zapytania)
* `LAZY` – ładowanie odroczone, gdy naprawdę potrzebujemy danych

**Domyślne wartości:**

* `@ManyToOne`, `@OneToOne` → domyślnie **EAGER**
* `@OneToMany`, `@ManyToMany` → domyślnie **LAZY**

Najczęściej zaleca się **LAZY wszędzie**, jeśli to możliwe.

```java
@ManyToOne(fetch = FetchType.LAZY)
private Customer customer;
```

### CascadeType

* `PERSIST` – zapisuje encję zależną razem z właścicielem
* `MERGE` – scala zmiany
* `REMOVE` – usuwa encję zależną przy usunięciu właściciela
* `ALL` – wszystkie powyższe

```java
@OneToMany(
        mappedBy = "customer",
        cascade = {CascadeType.PERSIST, CascadeType.MERGE},
        orphanRemoval = true
)
private List<Order> orders;
```

> 💡 **Praktyka:** ostrożnie z `CascadeType.REMOVE` – szczególnie w relacjach do encji współdzielonych.

---

## 🧱 Wartości osadzone (@Embeddable)

Czasem chcemy grupę pól trzymać razem, ale **bez osobnej tabeli**. Używamy wtedy `@Embeddable` i `@Embedded`.

### Przykład – adres jako obiekt osadzony

```java
@Embeddable
public class Address {

    @Column(name = "street")
    private String street;

    @Column(name = "city")
    private String city;

    @Column(name = "zip_code")
    private String zipCode;
}

@Entity
@Table(name = "companies")
public class Company {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Embedded
    private Address address;
}
```

W tabeli `companies` pojawią się kolumny `street`, `city`, `zip_code`.

---
