# Setmana 12 — Dimarts: Spring Security — Protegir el Backend Java

## Objectiu del Dia

Integrar Spring Security al backend Java d'EsportsPulse. Al final del dia has de tenir un endpoint de registre (que guarda contrasenyes hashejades amb BCrypt) i un endpoint de login (que retorna un JWT valid). Tot protegit per defecte — cap endpoint accessible sense autenticacio excepte `/auth/**`.

---

## Teoria

### Spring Security: Tot Tancat per Defecte

Quan afegeixes Spring Security al projecte, passa una cosa radical: **tots els endpoints queden bloquejats immediatament**. Aixo es un patro de seguretat anomenat **deny by default** — es molt mes segur oblidar-se d'obrir un endpoint (detectes l'error rapidament) que oblidar-se de tancar-lo (pot passar desapercebut mesos).

```xml
<!-- Afegir al pom.xml — nomes amb aixo, TOT queda bloquejat -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

Prova-ho: afegeix la dependencia, reinicia el servidor, i veuras que `curl http://localhost:8080/api/champions` retorna `401 Unauthorized`. Fins i tot el navegador et mostrara un formulari de login generic.

### SecurityFilterChain: Definir Qui Pot Accedir a Que

La configuracio de seguretat es fa amb un `SecurityFilterChain` — una cadena de filtres que processa cada peticio HTTP:

```java
// Cada peticio HTTP passa per aquesta cadena de filtres:
// 1. CorsFilter          → Comprova si l'origen esta permès
// 2. CsrfFilter          → Proteccio contra CSRF (la desactivem per APIs REST)
// 3. AuthenticationFilter → Extreu i valida el token JWT
// 4. AuthorizationFilter  → Comprova si l'usuari te el rol necessari
// 5. El teu Controller    → Finalment executa la logica de negoci
```

### Password Hashing: Mai Guardar Contrasenyes en Text Pla

```
MAL:  password = "s3cr3t"          → Si la BD es compromesa, totes les contrasenyes exposades
BE:   password = "$2a$10$xK9..."   → Hash irreversible, inutil sense la contrasenya original
```

**BCrypt** es l'algorisme recomanat per OWASP:
- Es lent per disseny (preveu atacs de forca bruta)
- Inclou un "salt" aleatori (dues contrasenyes iguals generen hashos diferents)
- Te un factor de cost configurable (pots fer-lo mes lent a mesura que el hardware millora)

> **Lectura recomanada (no bloquejant):**
> - [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
> - [Spring Security Reference](https://docs.spring.io/spring-security/reference/) — Seccions "Architecture" i "Authentication"

---

## Activitat

### 1. Afegir Dependencies (10 min)

Afegeix al `pom.xml` les dependencies de seguretat i JWT:

```xml
<!-- Spring Security: framework d'autenticacio i autoritzacio -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- JJWT: llibreria per crear i validar JSON Web Tokens
     Necessitem 3 artefactes: API, implementacio i serialitzacio Jackson -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

Executa `mvn compile` per verificar que les dependencies es descarreguen correctament.

### 2. Crear l'Entitat User (15 min)

```java
// fitxer: src/main/java/com/esportspulse/engine/model/User.java
// Entitat JPA que representa un usuari del sistema.
// Connecta amb la feina de JPA de la Setmana 6 — mateixa estructura d'entitat.

package com.esportspulse.engine.model;

import jakarta.persistence.*;

@Entity
@Table(name = "app_users")  // "users" es paraula reservada a PostgreSQL, usem "app_users"
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String username;

    @Column(nullable = false)
    private String password;  // Mai en text pla — sempre hash BCrypt

    @Column(nullable = false)
    @Enumerated(EnumType.STRING)  // Guarda el nom del enum ("USER", "ADMIN"), no l'ordinal
    private Role role;

    // Enum amb els rols possibles — per ara nomes dos
    public enum Role {
        USER,   // Pot llegir dades
        ADMIN   // Pot llegir, crear, modificar i esborrar
    }

    // Constructor buit obligatori per JPA
    public User() {}

    // Constructor per crear usuaris nous
    public User(String username, String password, Role role) {
        this.username = username;
        this.password = password;
        this.role = role;
    }

    // Getters i setters
    public Long getId() { return id; }
    public String getUsername() { return username; }
    public String getPassword() { return password; }
    public Role getRole() { return role; }

    public void setId(Long id) { this.id = id; }
    public void setUsername(String username) { this.username = username; }
    public void setPassword(String password) { this.password = password; }
    public void setRole(Role role) { this.role = role; }
}
```

### 3. Crear el UserRepository (5 min)

```java
// fitxer: src/main/java/com/esportspulse/engine/repository/UserRepository.java
// Repository JPA per accedir a la taula d'usuaris.
// Spring Data genera la implementacio automaticament (igual que a S6).

package com.esportspulse.engine.repository;

import com.esportspulse.engine.model.User;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {

    // Spring Data genera el SQL: SELECT * FROM app_users WHERE username = ?
    Optional<User> findByUsername(String username);

    // Necessari per validar que no es registrin usernames duplicats
    boolean existsByUsername(String username);
}
```

### 4. Servei JWT (20 min)

```java
// fitxer: src/main/java/com/esportspulse/engine/security/JwtService.java
// Encapsula tota la logica de creacio i validacio de tokens JWT.
// Usa la llibreria JJWT (io.jsonwebtoken) — molt mes robusta que fer-ho manualment.

package com.esportspulse.engine.security;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;

@Service
public class JwtService {

    private final SecretKey signingKey;
    private final long expirationMs;

    // Injecta valors des de application.properties (bona practica: no hardcodejar secrets)
    public JwtService(
            @Value("${jwt.secret}") String secret,
            @Value("${jwt.expiration-ms:3600000}") long expirationMs) {  // Default: 1 hora
        // Crea una clau criptografica a partir del secret string
        // La clau ha de tenir minim 256 bits (32 bytes) per HS256
        this.signingKey = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.expirationMs = expirationMs;
    }

    /**
     * Genera un JWT amb el username i rol de l'usuari.
     * El token inclou: subject (username), role (claim personalitzat),
     * data de creacio i data d'expiracio.
     */
    public String generateToken(String username, String role) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + expirationMs);

        return Jwts.builder()
                .subject(username)                    // Claim estàndard: qui es l'usuari
                .claim("role", role)                  // Claim personalitzat: quin rol te
                .issuedAt(now)                        // Quan s'ha creat
                .expiration(expiry)                   // Quan caduca
                .signWith(signingKey)                 // Signa amb la clau secreta
                .compact();                           // Serialitza a String
    }

    /**
     * Extreu el username (subject) d'un token.
     * Llanca excepcio si el token es invalid o ha caducat.
     */
    public String extractUsername(String token) {
        return extractClaims(token).getSubject();
    }

    /** Extreu el rol de l'usuari del token. */
    public String extractRole(String token) {
        return extractClaims(token).get("role", String.class);
    }

    /** Valida que el token no hagi caducat i la signatura sigui correcta. */
    public boolean isTokenValid(String token) {
        try {
            extractClaims(token);  // Si no llanca excepcio, el token es valid
            return true;
        } catch (Exception e) {
            return false;  // Token invalid, caducat o manipulat
        }
    }

    /**
     * Parseja i valida el token, retornant els claims.
     * JJWT verifica automaticament: signatura, expiracio, format.
     */
    private Claims extractClaims(String token) {
        return Jwts.parser()
                .verifyWith(signingKey)       // Usa la mateixa clau per verificar
                .build()
                .parseSignedClaims(token)     // Parseja i valida
                .getPayload();                // Retorna els claims
    }
}
```

### 5. Configurar application.properties (5 min)

```properties
# fitxer: src/main/resources/application.properties
# Afegir aquestes propietats (NO commitejis la clau secreta real)

# JWT Configuration
# IMPORTANT: en produccio, usa una variable d'entorn: ${JWT_SECRET}
# La clau ha de tenir minim 32 caracters per HS256
jwt.secret=${JWT_SECRET:clau-secreta-de-desenvolupament-que-te-32-chars-minim}
jwt.expiration-ms=3600000
```

Afegeix al `.env` (que ja esta al `.gitignore` des de S1):
```bash
JWT_SECRET=la-teva-clau-secreta-de-produccio-molt-llarga-i-aleatoria
```

### 6. DTOs de Registre i Login (10 min)

```java
// fitxer: src/main/java/com/esportspulse/engine/dto/RegisterRequest.java
// DTO (Data Transfer Object) per les peticions de registre.
// Usa records de Java (S2) — immutables i compactes.

package com.esportspulse.engine.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record RegisterRequest(
    @NotBlank(message = "El username es obligatori")
    @Size(min = 3, max = 50, message = "El username ha de tenir entre 3 i 50 caracters")
    String username,

    @NotBlank(message = "La contrasenya es obligatoria")
    @Size(min = 8, message = "La contrasenya ha de tenir minim 8 caracters")
    String password
) {}
```

```java
// fitxer: src/main/java/com/esportspulse/engine/dto/LoginRequest.java
package com.esportspulse.engine.dto;

import jakarta.validation.constraints.NotBlank;

public record LoginRequest(
    @NotBlank String username,
    @NotBlank String password
) {}
```

```java
// fitxer: src/main/java/com/esportspulse/engine/dto/AuthResponse.java
// Resposta que retorna el token JWT al client despres del login/registre.

package com.esportspulse.engine.dto;

public record AuthResponse(
    String token,
    String username,
    String role
) {}
```

### 7. AuthController: Registre i Login (25 min)

```java
// fitxer: src/main/java/com/esportspulse/engine/controller/AuthController.java
// Controller amb els endpoints publics d'autenticacio.
// Endpoints: POST /auth/register i POST /auth/login

package com.esportspulse.engine.controller;

import com.esportspulse.engine.dto.*;
import com.esportspulse.engine.model.User;
import com.esportspulse.engine.model.User.Role;
import com.esportspulse.engine.repository.UserRepository;
import com.esportspulse.engine.security.JwtService;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/auth")  // Tots els endpoints d'auth comencen per /auth
public class AuthController {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtService jwtService;

    // Injeccio per constructor (bona practica de S5 Clean Code)
    public AuthController(UserRepository userRepository,
                          PasswordEncoder passwordEncoder,
                          JwtService jwtService) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
        this.jwtService = jwtService;
    }

    /**
     * Registra un nou usuari.
     * 1. Verifica que el username no existeixi
     * 2. Hasheja la contrasenya amb BCrypt (MAI guardar en text pla)
     * 3. Guarda l'usuari a la BD amb rol USER per defecte
     * 4. Retorna un JWT perque l'usuari pugui fer login immediatament
     */
    @PostMapping("/register")
    public ResponseEntity<AuthResponse> register(@Valid @RequestBody RegisterRequest request) {
        // Comprova duplicats — retorna 409 Conflict si ja existeix
        if (userRepository.existsByUsername(request.username())) {
            return ResponseEntity.status(HttpStatus.CONFLICT).build();
        }

        // Crea l'usuari amb la contrasenya hashejada
        User user = new User(
                request.username(),
                passwordEncoder.encode(request.password()),  // BCrypt hash
                Role.USER  // Rol per defecte — nomes un admin pot canviar-ho
        );
        userRepository.save(user);

        // Genera token perque l'usuari no hagi de fer login just despres de registrar-se
        String token = jwtService.generateToken(user.getUsername(), user.getRole().name());
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(new AuthResponse(token, user.getUsername(), user.getRole().name()));
    }

    /**
     * Login d'un usuari existent.
     * 1. Busca l'usuari per username
     * 2. Verifica la contrasenya contra el hash BCrypt
     * 3. Si tot OK, retorna un JWT
     */
    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@Valid @RequestBody LoginRequest request) {
        // Busca l'usuari — si no existeix, retorna 401
        User user = userRepository.findByUsername(request.username())
                .orElse(null);

        // IMPORTANT: no diguem "username no trobat" vs "contrasenya incorrecta"
        // Aixo donaria informacio a un atacant sobre quins usernames existeixen
        if (user == null || !passwordEncoder.matches(request.password(), user.getPassword())) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }

        String token = jwtService.generateToken(user.getUsername(), user.getRole().name());
        return ResponseEntity.ok(new AuthResponse(token, user.getUsername(), user.getRole().name()));
    }
}
```

### 8. Configuracio de Seguretat (15 min)

```java
// fitxer: src/main/java/com/esportspulse/engine/security/SecurityConfig.java
// Configuracio central de Spring Security.
// Defineix: quins endpoints son publics, com es hashegen contrasenyes,
// i desactiva CSRF per APIs stateless (ho explicarem divendres).

package com.esportspulse.engine.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity  // Activa la configuracio de seguretat personalitzada
public class SecurityConfig {

    /**
     * Defineix la cadena de filtres de seguretat.
     * Aqui decidim QUI pot accedir a QUE.
     */
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Desactivem CSRF perque som una API REST stateless (no usem cookies de sessio)
            // Ho explicarem en detall divendres
            .csrf(csrf -> csrf.disable())

            // No volem sessions — cada peticio porta el seu JWT
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            // Regles d'autoritzacio per endpoint
            .authorizeHttpRequests(auth -> auth
                // Endpoints publics: registre i login (tothom hi pot accedir)
                .requestMatchers("/auth/**").permitAll()
                // Endpoint de health check (util per Docker/Kubernetes)
                .requestMatchers("/actuator/health").permitAll()
                // Tot la resta requereix autenticacio
                .anyRequest().authenticated()
            );

        return http.build();
    }

    /**
     * Bean de BCrypt per hasheja contrasenyes.
     * Spring Security el fara servir automaticament quan cridem
     * passwordEncoder.encode() i passwordEncoder.matches()
     */
    @Bean
    public PasswordEncoder passwordEncoder() {
        // BCrypt amb strength 10 (default) — 2^10 = 1024 iteracions
        // Es prou lent per prevenir brute force, prou rapid per no molestar l'usuari
        return new BCryptPasswordEncoder();
    }
}
```

### 9. Provar amb curl (15 min)

```bash
# 1. Registrar un usuari nou
curl -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"pere","password":"contrasenya-segura-123"}' | jq .

# Resposta esperada (201 Created):
# {
#   "token": "eyJhbGciOiJIUzI1NiJ9...",
#   "username": "pere",
#   "role": "USER"
# }

# 2. Intentar registrar el mateix username (ha de fallar)
curl -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"pere","password":"altra-contrasenya"}' -w "\n%{http_code}\n"
# Resposta: 409 Conflict

# 3. Login amb credencials correctes
curl -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"pere","password":"contrasenya-segura-123"}' | jq .
# Resposta: 200 OK amb token

# 4. Login amb credencials incorrectes
curl -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"pere","password":"incorrecta"}' -w "\n%{http_code}\n"
# Resposta: 401 Unauthorized

# 5. Accedir a un endpoint protegit SENSE token
curl http://localhost:8080/api/champions -w "\n%{http_code}\n"
# Resposta: 401 Unauthorized (PERFECTE — abans era 200!)

# 6. Verificar a la BD que la contrasenya esta hashejada
docker exec -it esportspulse-db psql -U postgres -d esportspulse \
  -c "SELECT username, password FROM app_users;"
# La columna password ha de mostrar algo com: $2a$10$xK9m...
# Si veus "contrasenya-segura-123" en text pla, alguna cosa ha anat malament
```

---

## Checklist de Lliurament

- [ ] `mvn compile` passa amb les noves dependencies de seguretat
- [ ] L'entitat `User` existeix amb camps `username`, `password` (hash), `role`
- [ ] `POST /auth/register` crea un usuari i retorna un JWT
- [ ] `POST /auth/login` verifica credencials i retorna un JWT
- [ ] Les contrasenyes a la BD estan hashejades amb BCrypt (verificat amb query SQL)
- [ ] `GET /api/champions` sense token retorna `401 Unauthorized`
- [ ] Registrar un username duplicat retorna `409 Conflict`
- [ ] Login amb contrasenya incorrecta retorna `401` (sense revelar si l'usuari existeix)
- [ ] Commit: `feat(auth): add Spring Security with user registration and JWT login`
