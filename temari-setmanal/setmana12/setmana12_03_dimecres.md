# Setmana 12 — Dimecres: JWT Filter, Roles i @PreAuthorize

## Objectiu del Dia

Implementar el filtre JWT que intercepta cada peticio, valida el token i estableix el context de seguretat. Afegir control d'acces basat en rols (USER vs ADMIN) als endpoints de l'API. Configurar CORS per preparar la integracio amb el frontend. Al final del dia, els endpoints de lectura estaran oberts a qualsevol usuari autenticat, pero nomes els ADMIN podran crear o esborrar recursos.

---

## Teoria

### El JWT Authentication Filter

Ahir vam configurar Spring Security perque bloqueges tot excepte `/auth/**`. Pero ara necessitem un mecanisme per **desbloquejar** peticions que portin un JWT valid. Aixo es fa amb un filtre que s'executa ABANS de cada peticio:

```
Peticio HTTP amb "Authorization: Bearer eyJhb..."
    ↓
JwtAuthenticationFilter
    ↓ extreu el token de la capcalera
    ↓ valida la signatura i l'expiracio
    ↓ crea un objecte Authentication amb username + rol
    ↓ el posa al SecurityContext (perque Spring Security el trobi)
    ↓
SecurityFilterChain
    ↓ comprova si l'usuari te el rol necessari per l'endpoint
    ↓
Controller (finalment executa la logica)
```

**Per que un filtre i no un interceptor?**
Els filtres de Servlet s'executen abans que Spring processi la peticio. Aixo vol dir que si el token es invalid, la peticio ni tan sols arriba al controller — es rebutjada al nivell mes baix possible.

### Rols i @PreAuthorize

Spring Security permet controlar l'acces a nivell de metode amb anotacions:

```java
// Qualsevol usuari autenticat pot llegir
@GetMapping("/champions")
public List<Champion> getAll() { ... }

// Nomes ADMIN pot crear
@PreAuthorize("hasRole('ADMIN')")
@PostMapping("/champions")
public Champion create(@RequestBody Champion champion) { ... }

// Nomes ADMIN pot esborrar
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/champions/{id}")
public void delete(@PathVariable Long id) { ... }
```

**Convencio de rols a Spring Security:**
Quan uses `hasRole('ADMIN')`, Spring Security busca l'authority `ROLE_ADMIN` (afegeix el prefix `ROLE_` automaticament). Ho has de tenir en compte quan crees els `GrantedAuthority`.

### CORS (Cross-Origin Resource Sharing)

Quan el frontend (Streamlit a S13, o qualsevol app web) esta en un domini/port diferent del backend, el navegador bloqueja les peticions per defecte. Aixo es una mesura de seguretat:

```
Frontend (http://localhost:8501)  →  Backend (http://localhost:8080)
                                     ❌ Bloquejat pel navegador!
```

**Per que?** Sense CORS, una pagina maliciosa podria fer peticions al teu backend usant les cookies de l'usuari (atac CSRF). El navegador demana al backend: "Acceptes peticions des de localhost:8501?" Si el backend no respon afirmativament, el navegador bloqueja la resposta.

```
1. Frontend fa peticio → Navegador envia primer una peticio OPTIONS (preflight)
2. Backend respon amb headers CORS: "Accepto peticions de localhost:8501"
3. Navegador permet la peticio real (GET, POST, etc.)
```

> **Lectura recomanada (no bloquejant):**
> - [Spring Security Method Security](https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html)
> - [MDN: Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)

---

## Activitat

### 1. Implementar el JWT Authentication Filter (30 min)

```java
// fitxer: src/main/java/com/esportspulse/engine/security/JwtAuthenticationFilter.java
// Filtre que s'executa a CADA peticio HTTP.
// Extreu el JWT de la capcalera Authorization, el valida,
// i estableix l'autenticacio al SecurityContext de Spring.

package com.esportspulse.engine.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.lang.NonNull;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;

    public JwtAuthenticationFilter(JwtService jwtService) {
        this.jwtService = jwtService;
    }

    @Override
    protected void doFilterInternal(
            @NonNull HttpServletRequest request,
            @NonNull HttpServletResponse response,
            @NonNull FilterChain filterChain
    ) throws ServletException, IOException {

        // 1. Extreu la capcalera Authorization
        String authHeader = request.getHeader("Authorization");

        // Si no hi ha capcalera o no comença per "Bearer ", passa al seguent filtre
        // (sera rebutjat mes endavant per Spring Security si l'endpoint requereix auth)
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        // 2. Extreu el token (tot despres de "Bearer ")
        String token = authHeader.substring(7);

        // 3. Valida el token i extreu les dades
        if (jwtService.isTokenValid(token)) {
            String username = jwtService.extractUsername(token);
            String role = jwtService.extractRole(token);

            // 4. Crea l'objecte Authentication amb el rol de l'usuari
            // IMPORTANT: el prefix "ROLE_" es necessari perque hasRole() funcioni
            var authorities = List.of(new SimpleGrantedAuthority("ROLE_" + role));

            var authToken = new UsernamePasswordAuthenticationToken(
                    username,       // Principal: qui es l'usuari
                    null,           // Credentials: null perque ja hem validat el JWT
                    authorities     // Authorities: rols de l'usuari
            );

            // Afegeix detalls de la peticio (IP, session ID, etc.)
            authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

            // 5. Estableix l'autenticacio al SecurityContext
            // A partir d'aqui, Spring Security sap qui es l'usuari i quins rols te
            SecurityContextHolder.getContext().setAuthentication(authToken);
        }

        // 6. Continua amb la cadena de filtres
        filterChain.doFilter(request, response);
    }
}
```

### 2. Registrar el Filtre a SecurityConfig (10 min)

Actualitza la configuracio de seguretat per incloure el filtre JWT:

```java
// fitxer: src/main/java/com/esportspulse/engine/security/SecurityConfig.java
// Afegim el filtre JWT i activem @PreAuthorize

package com.esportspulse.engine.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.List;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // Activa @PreAuthorize, @Secured, etc. als controllers
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;

    public SecurityConfig(JwtAuthenticationFilter jwtAuthFilter) {
        this.jwtAuthFilter = jwtAuthFilter;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            // Registra el filtre JWT ABANS del filtre d'autenticacio per defecte
            // Aixo garanteix que el JWT es processa abans que Spring comprovi l'autenticacio
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    /**
     * Configuracio CORS: quins origens, metodes i capcaleres estan permesos.
     * Preparem per la integracio amb Streamlit (S13) i qualsevol frontend.
     */
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        // Origens permesos (afegir el frontend quan existeixi)
        config.setAllowedOrigins(List.of(
                "http://localhost:8501",   // Streamlit (S13)
                "http://localhost:3000"    // Possible frontend React/Next.js
        ));
        // Metodes HTTP permesos
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        // Capcaleres permeses (Authorization es imprescindible per enviar el JWT)
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        // Permet enviar cookies/credencials (necessari si algun dia uses sessions)
        config.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);  // Aplica a tots els endpoints
        return source;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 3. Protegir Endpoints amb @PreAuthorize (15 min)

Actualitza el `ChampionController` (o el controller principal de l'API) per afegir control de rols:

```java
// fitxer: src/main/java/com/esportspulse/engine/controller/ChampionController.java
// Afegim anotacions de seguretat als metodes CRUD.
// GET: qualsevol usuari autenticat. POST/PUT/DELETE: nomes ADMIN.

package com.esportspulse.engine.controller;

import com.esportspulse.engine.model.Champion;
import com.esportspulse.engine.repository.ChampionRepository;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/champions")
public class ChampionController {

    private final ChampionRepository championRepository;

    public ChampionController(ChampionRepository championRepository) {
        this.championRepository = championRepository;
    }

    // Qualsevol usuari autenticat pot llistar champions
    // No cal @PreAuthorize — el filtre general ja exigeix autenticacio
    @GetMapping
    public List<Champion> getAll() {
        return championRepository.findAll();
    }

    // Qualsevol usuari autenticat pot veure un champion concret
    @GetMapping("/{id}")
    public ResponseEntity<Champion> getById(@PathVariable Long id) {
        return championRepository.findById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    // Nomes ADMIN pot crear champions
    @PreAuthorize("hasRole('ADMIN')")
    @PostMapping
    public ResponseEntity<Champion> create(@RequestBody Champion champion) {
        Champion saved = championRepository.save(champion);
        return ResponseEntity.status(HttpStatus.CREATED).body(saved);
    }

    // Nomes ADMIN pot modificar champions
    @PreAuthorize("hasRole('ADMIN')")
    @PutMapping("/{id}")
    public ResponseEntity<Champion> update(@PathVariable Long id,
                                           @RequestBody Champion champion) {
        return championRepository.findById(id)
                .map(existing -> {
                    champion.setId(id);
                    return ResponseEntity.ok(championRepository.save(champion));
                })
                .orElse(ResponseEntity.notFound().build());
    }

    // Nomes ADMIN pot esborrar champions
    @PreAuthorize("hasRole('ADMIN')")
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        if (!championRepository.existsById(id)) {
            return ResponseEntity.notFound().build();
        }
        championRepository.deleteById(id);
        return ResponseEntity.noContent().build();
    }

    // Endpoint que retorna qui esta fent la peticio (util per depurar)
    @GetMapping("/me")
    public ResponseEntity<String> whoAmI(Authentication authentication) {
        // Authentication es injectat automaticament per Spring Security
        String username = authentication.getName();
        String role = authentication.getAuthorities().toString();
        return ResponseEntity.ok("Usuari: " + username + ", Rols: " + role);
    }
}
```

### 4. Crear un Usuari ADMIN per Proves (10 min)

Crea un `DataInitializer` que insereixi un admin al arrencar (nomes per desenvolupament):

```java
// fitxer: src/main/java/com/esportspulse/engine/config/DataInitializer.java
// Crea un usuari ADMIN al arrencar l'aplicacio si no existeix.
// Nomes per a desenvolupament — en produccio usaries una migracio Flyway/Liquibase.

package com.esportspulse.engine.config;

import com.esportspulse.engine.model.User;
import com.esportspulse.engine.model.User.Role;
import com.esportspulse.engine.repository.UserRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class DataInitializer {

    private static final Logger log = LoggerFactory.getLogger(DataInitializer.class);

    @Bean
    @Profile("dev")  // Nomes s'executa amb el perfil "dev" (no en produccio)
    public CommandLineRunner initAdminUser(UserRepository userRepository,
                                           PasswordEncoder passwordEncoder) {
        return args -> {
            if (!userRepository.existsByUsername("admin")) {
                User admin = new User(
                        "admin",
                        passwordEncoder.encode("admin-password-123"),
                        Role.ADMIN
                );
                userRepository.save(admin);
                log.info("Usuari ADMIN creat per desenvolupament");
            }
        };
    }
}
```

A `application.properties`, activa el perfil dev:
```properties
spring.profiles.active=dev
```

### 5. Proves Completes amb curl (20 min)

```bash
# === SETUP: Registrar usuaris ===

# 1. Registrar un USER normal
curl -s -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"viewer","password":"viewer-pass-123"}' | jq .

# Guarda el token USER
USER_TOKEN=$(curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"viewer","password":"viewer-pass-123"}' | jq -r .token)

# 2. Login com ADMIN (creat pel DataInitializer)
ADMIN_TOKEN=$(curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin-password-123"}' | jq -r .token)

echo "USER_TOKEN: $USER_TOKEN"
echo "ADMIN_TOKEN: $ADMIN_TOKEN"

# === PROVES D'AUTORITZACIO ===

# 3. USER pot llegir champions (200 OK)
curl -s -H "Authorization: Bearer $USER_TOKEN" \
  http://localhost:8080/api/champions | jq .

# 4. USER NO pot crear champions (403 Forbidden)
curl -s -w "\nHTTP %{http_code}\n" \
  -X POST http://localhost:8080/api/champions \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Faker","role":"mid"}'
# Esperat: 403 Forbidden

# 5. ADMIN pot crear champions (201 Created)
curl -s -X POST http://localhost:8080/api/champions \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Faker","role":"mid"}' | jq .
# Esperat: 201 Created

# 6. ADMIN pot esborrar champions (204 No Content)
curl -s -w "\nHTTP %{http_code}\n" \
  -X DELETE http://localhost:8080/api/champions/1 \
  -H "Authorization: Bearer $ADMIN_TOKEN"
# Esperat: 204 No Content

# 7. Sense token: 401 Unauthorized
curl -s -w "\nHTTP %{http_code}\n" http://localhost:8080/api/champions

# 8. Endpoint /me per verificar la identitat
curl -s -H "Authorization: Bearer $USER_TOKEN" \
  http://localhost:8080/api/champions/me
# Esperat: "Usuari: viewer, Rols: [ROLE_USER]"

# === PROVES CORS (simula peticio del navegador) ===

# 9. Preflight OPTIONS request (el que fa el navegador automaticament)
curl -s -X OPTIONS http://localhost:8080/api/champions \
  -H "Origin: http://localhost:8501" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: Authorization" \
  -D - -o /dev/null | grep -i "access-control"
# Esperat: Access-Control-Allow-Origin: http://localhost:8501
```

> **Connexio amb S3 (Linux i curl):** Estem usant exactament les mateixes habilitats de curl que vam aprendre a la Setmana 3. La diferencia es que ara afegim capcaleres d'autenticacio.

---

## Checklist de Lliurament

- [ ] El `JwtAuthenticationFilter` intercepta i valida tokens a cada peticio
- [ ] Un USER autenticat pot fer GET pero NO POST/DELETE als endpoints de champions
- [ ] Un ADMIN pot fer GET, POST, PUT i DELETE
- [ ] Peticions sense token retornen `401 Unauthorized`
- [ ] Peticions amb token valid pero rol insuficient retornen `403 Forbidden`
- [ ] L'endpoint `/api/champions/me` retorna el username i rol de l'usuari autenticat
- [ ] CORS esta configurat per acceptar peticions des de `localhost:8501`
- [ ] Commit: `feat(auth): add JWT filter, role-based access control and CORS configuration`
