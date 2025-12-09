# 🌐 Adnotacje dla kontrolerów HTTP w Spring

Krótka dokumentacja najważniejszych adnotacji używanych w kontrolerach HTTP w Spring / Spring Boot.

---

## 📚 Spis treści

1. [Podstawowe typy kontrolerów](#-podstawowe-typy-kontrolerów)

   * `@Controller`
   * `@RestController`
2. [Mapowanie endpointów](#-mapowanie-endpointów)

   * `@RequestMapping`
   * `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`
3. [Parametry żądania](#-parametry-żądania)

   * `@PathVariable`
   * `@RequestParam`
   * `@RequestBody`
   * `@RequestHeader`
4. [Odpowiedzi HTTP](#-odpowiedzi-http)

   * `@ResponseStatus`
   * `ResponseEntity`
5. [Walidacja danych wejściowych](#-walidacja-danych-wejściowych)

   * `@Valid` / `@Validated`
6. [CORS i inne przydatne adnotacje](#-cors-i-inne-przydatne-adnotacje)

   * `@CrossOrigin`
   * `@ExceptionHandler`, `@ControllerAdvice`

---

## 🧭 Podstawowe typy kontrolerów

### `@Controller`

Adnotacja `@Controller` oznacza klasę jako **kontroler webowy** w Spring MVC.

* Zwracane wartości metod są domyślnie traktowane jako **nazwy widoków** (np. szablony Thymeleaf), a nie jako ciało odpowiedzi HTTP.

```java
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class PageController {

    @GetMapping("/hello")
    public String helloPage(Model model) {
        model.addAttribute("message", "Hello world!");
        return "hello"; // nazwa widoku (np. hello.html)
    }
}
```

---

### `@RestController`

`@RestController` = `@Controller` + `@ResponseBody` na każdej metodzie.

* używana do tworzenia **REST API**,
* wartości zwracane z metod są serializowane (np. do JSON) i wysyłane jako **ciało odpowiedzi**.

```java
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.GetMapping;

@RestController
public class UserController {

    @GetMapping("/api/users/me")
    public UserDto currentUser() {
        return new UserDto("john", "john@example.com");
    }
}
```

---

## 🔀 Mapowanie endpointów

### `@RequestMapping`

Służy do mapowania **ścieżki** i **metody HTTP**. Może być stosowana na klasie i metodach.

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping
    public List<UserDto> getAll() {
        // GET /api/users
    }

    @PostMapping
    public UserDto create(@RequestBody CreateUserRequest request) {
        // POST /api/users
    }
}
```

Przykład użycia `@RequestMapping` na metodzie:

```java
@RequestMapping(path = "/api/users", method = RequestMethod.GET)
public List<UserDto> getAll() { ... }
```

---

### Skrócone adnotacje metod

W praktyce używa się częściej skrótów:

* `@GetMapping("/users")`
* `@PostMapping("/users")`
* `@PutMapping("/users/{id}")`
* `@DeleteMapping("/users/{id}")`
* `@PatchMapping("/users/{id}")`

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public UserDto getById(@PathVariable Long id) { ... }

    @PutMapping("/{id}")
    public UserDto update(@PathVariable Long id,
                          @RequestBody UpdateUserRequest request) { ... }

    @DeleteMapping("/{id}")
    public void delete(@PathVariable Long id) { ... }
}
```

---

## 🎯 Parametry żądania

### `@PathVariable`

Do pobierania **parametrów ścieżki**.

```java
@GetMapping("/api/users/{id}")
public UserDto getUser(@PathVariable("id") Long userId) {
    // ścieżka: /api/users/123 -> userId = 123
}
```

Jeżeli nazwa parametru w metodzie jest taka sama jak w ścieżce, można pominąć wartość w adnotacji:

```java
@GetMapping("/api/users/{id}")
public UserDto getUser(@PathVariable Long id) { ... }
```

---

### `@RequestParam`

Do pobierania **parametrów query** (np. `?page=0&size=10`).

```java
@GetMapping("/api/users")
public List<UserDto> searchUsers(@RequestParam(required = false) String email,
                                 @RequestParam(defaultValue = "0") int page,
                                 @RequestParam(defaultValue = "10") int size) {
    // /api/users?email=a@a.pl&page=1&size=20
}
```

Parametry:

* `required = false` – parametr opcjonalny,
* `defaultValue = "..."` – wartość domyślna, gdy parametr nie jest podany.

---

### `@RequestBody`

Do pobierania **ciała żądania** (JSON, XML itd.) i mapowania go na obiekt Java.

```java
@PostMapping("/api/users")
public UserDto createUser(@RequestBody CreateUserRequest request) {
    // JSON z body mapuje się na pola klasy CreateUserRequest
}
```

`@RequestBody` jest typowo używane w kontrolerach REST.

---

### `@RequestHeader`

Do pobierania konkretnych **nagłówków** HTTP.

```java
@GetMapping("/api/data")
public DataDto getData(@RequestHeader("X-Request-Id") String requestId) {
    // odczyt niestandardowego nagłówka
}
```

Można również mapować wiele nagłówków na `Map<String, String>`.

---

## 📤 Odpowiedzi HTTP

### `@ResponseStatus`

Pozwala ustawić **kod statusu HTTP** dla metody lub wyjątku.

```java
@PostMapping("/api/users")
@ResponseStatus(HttpStatus.CREATED)
public UserDto createUser(@RequestBody CreateUserRequest request) {
    // jeśli wszystko OK -> status 201 CREATED
}
```

Można ją też stosować na klasach wyjątków:

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class UserNotFoundException extends RuntimeException {
}
```

---

### `ResponseEntity`

`ResponseEntity<T>` pozwala pełniej kontrolować odpowiedź:

* ciało,
* status,
* nagłówki.

```java
@GetMapping("/api/users/{id}")
public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
    return userService.findById(id)
            .map(user -> ResponseEntity.ok(user))
            .orElseGet(() -> ResponseEntity.notFound().build());
}

@PostMapping("/api/users")
public ResponseEntity<UserDto> createUser(@RequestBody CreateUserRequest request) {
    UserDto created = userService.create(request);
    return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(created);
}
```

---

## ✅ Walidacja danych wejściowych

Spring dobrze współgra z Bean Validation (Jakarta Validation). W kontrolerach najczęściej używamy:

* `@Valid` (z pakietu `jakarta.validation` lub `javax.validation`),
* opcjonalnie `@Validated` na klasie kontrolera.

### Przykład DTO z walidacją

```java
public class CreateUserRequest {

    @NotBlank
    private String username;

    @Email
    @NotBlank
    private String email;

    // gettery/settery
}
```

### Użycie @Valid w kontrolerze

```java
@PostMapping("/api/users")
public UserDto createUser(@Valid @RequestBody CreateUserRequest request) {
    // jeśli walidacja się nie powiedzie -> rzucony zostanie wyjątek MethodArgumentNotValidException
}
```

Można też walidować parametry `@PathVariable` czy `@RequestParam` (przy włączeniu `@Validated`).

---

## 🌍 CORS i inne przydatne adnotacje

### `@CrossOrigin`

Pozwala skonfigurować CORS (Cross-Origin Resource Sharing) na poziomie kontrolera lub metody.

```java
@RestController
@CrossOrigin(origins = "https://example.com")
@RequestMapping("/api/users")
public class UserController { ... }
```

Można też ustawić `@CrossOrigin` tylko na pojedynczej metodzie:

```java
@GetMapping("/public")
@CrossOrigin(origins = "*")
public String publicEndpoint() {
    return "ok";
}
```

---

### `@ExceptionHandler` i `@ControllerAdvice`

Służą do obsługi wyjątków w kontrolerach.

`@ExceptionHandler` — metoda obsługująca konkretny typ wyjątku:

```java
@RestController
public class UserController {

    @GetMapping("/api/users/{id}")
    public UserDto get(@PathVariable Long id) {
        return userService.getById(id); // może rzucić UserNotFoundException
    }

    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Map<String, String> handleNotFound(UserNotFoundException ex) {
        return Map.of("error", ex.getMessage());
    }
}
```

`@ControllerAdvice` — globalny "kontroler" obsługujący błędy dla wielu kontrolerów:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, String> handleValidationErrors(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
                errors.put(error.getField(), error.getDefaultMessage())
        );
        return errors;
    }
}
```

---

