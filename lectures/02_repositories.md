# 📦 Dokumentacja Spring Data – Repozytoria JPA

---

## 📘 1. Czym są repozytoria Spring Data JPA?

Repozytoria to warstwa pośrednia między encjami JPA a bazą danych. Pozwalają one wykonywać operacje CRUD i zapytania bez konieczności pisania SQL/JPQL.

### 🧰 Najpopularniejsze interfejsy repozytoriów

| Interfejs                           | Przeznaczenie                                                      |
| ----------------------------------- | ------------------------------------------------------------------ |
| `CrudRepository<T, ID>`             | podstawowe operacje CRUD                                           |
| `JpaRepository<T, ID>`              | rozszerza `CrudRepository`, dodaje paginację, sortowanie, batching |
| `PagingAndSortingRepository<T, ID>` | dodaje paginację i sortowanie                                      |

Najczęściej używa się **`JpaRepository`**.

### Przykład podstawowego repozytorium

```java
public interface UserRepository extends JpaRepository<User, Long> {
}
```

Spring automatycznie tworzy implementację interfejsu podczas uruchamiania aplikacji.

---

## 🔍 2. Query derivation — konwencje nazewnicze metod

Spring potrafi generować zapytania na podstawie **nazwy metody**. Dzięki temu wiele operacji można wykonać bez @Query.

### Schemat:

```
findBy | readBy | getBy + Kryteria + Operatory
```

### Przykłady

```java
Optional<User> findByEmail(String email);
List<User> findByActive(boolean active);
List<User> findByUsernameOrEmail(String username, String email);
List<User> findByCreatedAtAfter(LocalDateTime time);
List<User> findByAgeBetween(int min, int max);
List<User> findTop10ByActiveOrderByCreatedAtDesc(boolean active);
```

### Przydatne operatory w nazwach

* `And`, `Or`
* `Between`, `LessThan`, `GreaterThan`
* `In`, `NotIn`
* `Containing`, `StartsWith`, `EndsWith`

### Metody pomocnicze

```java
long countByActive(boolean active);
boolean existsByEmail(String email);
void deleteByActive(boolean active);
```

---

## 📦 3. Optional jako typ zwracany

`Optional<T>` jest używany, gdy wynik może nie istnieć.

### Standardowe przykłady

```java
Optional<User> findById(Long id);
Optional<User> findByEmail(String email);
```

### Obsługa Optional w serwisie

```java
public User getUserByEmail(String email) {
    return userRepository.findByEmail(email)
            .orElseThrow(() -> new EntityNotFoundException("User not found"));
}

public User getOrAnonymous(Long id) {
    return userRepository.findById(id)
            .orElseGet(() -> anonymousUser());
}

public void activate(Long id) {
    userRepository.findById(id)
            .ifPresent(user -> {
                user.setActive(true);
                userRepository.save(user);
            });
}
```

`Optional` wymusza jawne obsłużenie braku wyniku – dzięki temu unikamy `NullPointerException`.

---

## 🧾 4. Adnotacja @Query — własne zapytania

Gdy nazwa metody nie wystarcza lub zapytanie jest złożone, używamy `@Query`.

### ⭐ JPQL (domyślny tryb)

```java
@Query("SELECT u FROM User u WHERE u.active = true AND u.email LIKE %:domain")
List<User> findActiveUsersFromDomain(@Param("domain") String domain);
```

* operujemy na **nazwach encji i pól**, nie tabel;
* wspiera nawigację po relacjach, np. `u.profile.address.city`.

### ⭐ Zapytania natywne (SQL)

```java
@Query(value = "SELECT * FROM users WHERE email LIKE %:domain", nativeQuery = true)
List<User> findByEmailDomain(@Param("domain") String domain);
```

Uwaga: natywne zapytania wymagają używania **nazw tabel i kolumn** z bazy.

### ⭐ Zapytania modyfikujące (UPDATE/DELETE)

```java
@Modifying
@Transactional
@Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :threshold")
int deactivateInactive(@Param("threshold") LocalDateTime threshold);
```

* `@Modifying` — informuje Spring, że zapytanie modyfikuje dane;
* `@Transactional` — wymaga transakcji;
* metoda może zwracać liczbę zmodyfikowanych rekordów.

---

## 🧵 5. Przykład serwisu korzystającego z repozytorium

```java
@Service
public class UserService {

    private final UserRepository repo;

    public UserService(UserRepository repo) {
        this.repo = repo;
    }

    public User register(String email, String username) {
        if (repo.existsByEmail(email)) {
            throw new IllegalArgumentException("Email already in use");
        }

        User user = new User(username, email);
        return repo.save(user);
    }

    public List<User> getActiveUsers() {
        return repo.findByActive(true);
    }
}
```

