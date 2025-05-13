# Implementando Autenticação JWT com Quarkus

A segurança é um aspecto crucial no desenvolvimento de APIs. Uma API sem proteção adequada pode expor dados sensíveis ou permitir operações não autorizadas. Neste tópico, vamos implementar autenticação e autorização usando JSON Web Tokens (JWT) com Quarkus.

## O que é JWT?

JWT (JSON Web Token) é um padrão aberto (RFC 7519) que define uma maneira compacta e autocontida para transmitir informações com segurança entre partes como um objeto JSON. Esses tokens podem ser verificados e confiáveis porque são assinados digitalmente.

Um JWT consiste em três partes separadas por pontos (`.`):
- **Header**: Contém o tipo do token e o algoritmo de assinatura
- **Payload**: Contém as reivindicações (claims) - dados sobre uma entidade (geralmente o usuário) e metadados
- **Signature**: Usada para verificar que o remetente do JWT é quem diz ser e garantir que a mensagem não foi alterada

Exemplo de um JWT:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

## Vantagens do JWT em APIs REST

- **Stateless**: O servidor não precisa manter um estado de sessão
- **Portabilidade**: Pode ser usado em diferentes domínios e plataformas
- **Segurança**: Assinado digitalmente usando um segredo ou chave pública/privada
- **Autocontido**: Contém todas as informações necessárias sobre o usuário
- **Extensível**: Pode armazenar informações adicionais nas claims

## Configurando JWT em um Projeto Quarkus

### 1. Adicionando as Extensões Necessárias

Primeiro, vamos adicionar as extensões de segurança JWT ao nosso projeto:

```bash
# Maven
./mvnw quarkus:add-extension -Dextensions="smallrye-jwt,jwt"

# Gradle
./gradlew addExtension --extensions="smallrye-jwt,jwt"
```

Ou adicionar manualmente ao `pom.xml`:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-jwt</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-jwt</artifactId>
</dependency>
```

### 2. Gerando Chaves Públicas e Privadas

Para assinar e verificar tokens JWT, precisamos de um par de chaves. Vamos gerar um par de chaves RSA:

```bash
# Gerar chave privada
openssl genrsa -out src/main/resources/privateKey.pem 2048

# Gerar chave pública
openssl rsa -in src/main/resources/privateKey.pem -pubout -out src/main/resources/publicKey.pem
```

### 3. Configurando o Quarkus para JWT

Adicione as seguintes configurações ao arquivo `application.properties`:

```properties
# Caminho para a chave pública usada para verificar JWTs
mp.jwt.verify.publickey.location=publicKey.pem
# Emissor dos tokens
mp.jwt.verify.issuer=https://api.meuservico.com
# Ative o suporte a JWT
quarkus.smallrye-jwt.enabled=true
```

## Implementando a Geração de Tokens

Vamos criar um serviço para gerar tokens JWT:

```java
package org.acme.security;

import io.smallrye.jwt.build.Jwt;
import java.io.InputStream;
import java.security.KeyFactory;
import java.security.PrivateKey;
import java.security.spec.PKCS8EncodedKeySpec;
import java.util.Base64;
import java.util.Set;
import javax.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class TokenService {

    private PrivateKey loadPrivateKey() {
        try {
            // Carrega a chave privada do arquivo
            InputStream is = getClass().getResourceAsStream("/privateKey.pem");
            byte[] bytes = new byte[is.available()];
            is.read(bytes);
            String key = new String(bytes);
            
            // Remove cabeçalho, rodapé e linhas em branco
            key = key.replaceAll("-----BEGIN PRIVATE KEY-----", "")
                     .replaceAll("-----END PRIVATE KEY-----", "")
                     .replaceAll("\\s+", "");
            
            // Decodifica a chave e a carrega
            byte[] keyBytes = Base64.getDecoder().decode(key);
            PKCS8EncodedKeySpec spec = new PKCS8EncodedKeySpec(keyBytes);
            KeyFactory kf = KeyFactory.getInstance("RSA");
            return kf.generatePrivate(spec);
        } catch (Exception e) {
            throw new RuntimeException("Não foi possível carregar a chave privada", e);
        }
    }
    
    public String generateToken(String username, Set<String> roles, Long durationInSeconds) {
        long currentTimeInSeconds = System.currentTimeMillis() / 1000;
        
        return Jwt.issuer("https://api.meuservico.com")
                .subject(username)
                .groups(roles)
                .issuedAt(currentTimeInSeconds)
                .expiresAt(currentTimeInSeconds + durationInSeconds)
                .sign(loadPrivateKey());
    }
}
```

## Implementando Autenticação de Usuário

### 1. Criando uma Entidade de Usuário

```java
package org.acme.entity;

import io.quarkus.hibernate.orm.panache.PanacheEntity;
import javax.persistence.Entity;
import javax.persistence.Table;
import javax.persistence.Column;
import java.util.Set;
import java.util.HashSet;
import javax.persistence.ElementCollection;
import javax.persistence.FetchType;
import javax.persistence.CollectionTable;
import javax.persistence.JoinColumn;

@Entity
@Table(name = "usuarios")
public class Usuario extends PanacheEntity {
    
    @Column(unique = true, nullable = false)
    public String username;
    
    @Column(nullable = false)
    public String password;
    
    @Column(nullable = false)
    public String nome;
    
    @Column(nullable = false)
    public String email;
    
    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(name = "usuario_roles", joinColumns = @JoinColumn(name = "usuario_id"))
    @Column(name = "role")
    public Set<String> roles = new HashSet<>();
    
    // Método para verificar senha (exemplo simples, em produção use BCrypt ou similar)
    public boolean verificarSenha(String senhaDigitada) {
        // Em uma aplicação real, use um algoritmo de hash seguro como BCrypt
        return this.password.equals(senhaDigitada);
    }
    
    // Método para encontrar usuário por nome de usuário
    public static Usuario findByUsername(String username) {
        return find("username", username).firstResult();
    }
}
```

### 2. Criando um Endpoint de Autenticação

```java
package org.acme.resource;

import org.acme.entity.Usuario;
import org.acme.security.TokenService;
import javax.inject.Inject;
import javax.ws.rs.*;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;
import java.util.Collections;
import javax.validation.constraints.NotBlank;

@Path("/auth")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class AuthResource {

    @Inject
    TokenService tokenService;
    
    @POST
    @Path("/login")
    public Response login(@FormParam("username") @NotBlank String username, 
                          @FormParam("password") @NotBlank String password) {
        // Busca o usuário pelo nome de usuário
        Usuario usuario = Usuario.findByUsername(username);
        
        // Se o usuário não existe ou a senha está incorreta
        if (usuario == null || !usuario.verificarSenha(password)) {
            return Response.status(Response.Status.UNAUTHORIZED)
                    .entity(Collections.singletonMap("message", "Credenciais inválidas"))
                    .build();
        }
        
        // Gera o token com validade de 24 horas (86400 segundos)
        String token = tokenService.generateToken(username, usuario.roles, 86400L);
        
        // Retorna o token
        return Response.ok()
                .entity(Collections.singletonMap("token", token))
                .build();
    }
}
```

## Protegendo Endpoints com JWT

### 1. Endpoint Protegido por Role

```java
package org.acme.resource;

import javax.annotation.security.RolesAllowed;
import javax.ws.rs.*;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Context;
import javax.ws.rs.core.SecurityContext;
import java.util.HashMap;
import java.util.Map;
import org.eclipse.microprofile.jwt.JsonWebToken;
import javax.inject.Inject;

@Path("/api/protegido")
@Produces(MediaType.APPLICATION_JSON)
public class RecursoProtegido {

    @Inject
    JsonWebToken jwt;

    @GET
    @Path("/publico")
    public Map<String, String> rotaPublica() {
        Map<String, String> result = new HashMap<>();
        result.put("message", "Esta é uma rota pública, acessível sem autenticação");
        return result;
    }

    @GET
    @Path("/user")
    @RolesAllowed({"user", "admin"})
    public Map<String, String> rotaUsuario(@Context SecurityContext securityContext) {
        Map<String, String> result = new HashMap<>();
        result.put("message", "Olá " + jwt.getName() + ", você está autenticado como usuário");
        result.put("username", jwt.getName());
        return result;
    }

    @GET
    @Path("/admin")
    @RolesAllowed("admin")
    public Map<String, String> rotaAdmin() {
        Map<String, String> result = new HashMap<>();
        result.put("message", "Acesso administrativo concedido para " + jwt.getName());
        result.put("username", jwt.getName());
        // Você pode recuperar informações adicionais do token
        result.put("issuer", jwt.getIssuer());
        result.put("expiration", String.valueOf(jwt.getExpirationTime()));
        return result;
    }
}
```

### 2. Extraindo Informações do Token

O Quarkus torna fácil injetar e usar o token JWT:

```java
@Inject
JsonWebToken jwt;

// Acessando claims do token
String subject = jwt.getSubject(); // Obtém o subject (normalmente o username)
Set<String> groups = jwt.getGroups(); // Obtém os grupos/roles
String issuer = jwt.getIssuer(); // Obtém o emissor
long expiration = jwt.getExpirationTime(); // Obtém o tempo de expiração

// Acessando claims personalizadas
String email = jwt.getClaim("email");
```

## Implementando Refresh Token

Para uma melhor experiência do usuário, é aconselhável implementar refresh tokens. Vamos adicionar essa funcionalidade:

```java
package org.acme.security;

import io.smallrye.jwt.build.Jwt;
import javax.enterprise.context.ApplicationScoped;
import javax.inject.Inject;
import java.time.LocalDateTime;
import java.time.ZoneOffset;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;
import org.acme.entity.Usuario;

@ApplicationScoped
public class RefreshTokenService {

    @Inject
    TokenService tokenService;
    
    // Armazenamento em memória para fins de demonstração
    // Em produção, use um banco de dados
    private final Map<String, RefreshTokenInfo> refreshTokens = new HashMap<>();
    
    public class RefreshTokenInfo {
        String username;
        LocalDateTime expiry;
        
        public RefreshTokenInfo(String username, LocalDateTime expiry) {
            this.username = username;
            this.expiry = expiry;
        }
    }
    
    public String generateRefreshToken(String username) {
        String refreshToken = UUID.randomUUID().toString();
        // Refresh token válido por 30 dias
        LocalDateTime expiry = LocalDateTime.now().plusDays(30);
        refreshTokens.put(refreshToken, new RefreshTokenInfo(username, expiry));
        return refreshToken;
    }
    
    public String refreshAccessToken(String refreshToken) {
        RefreshTokenInfo info = refreshTokens.get(refreshToken);
        
        if (info == null || info.expiry.isBefore(LocalDateTime.now())) {
            // Token expirado ou inválido
            throw new RuntimeException("Refresh token inválido ou expirado");
        }
        
        Usuario usuario = Usuario.findByUsername(info.username);
        if (usuario == null) {
            throw new RuntimeException("Usuário não encontrado");
        }
        
        // Gera um novo access token com validade de 24 horas
        return tokenService.generateToken(info.username, usuario.roles, 86400L);
    }
    
    public void invalidateRefreshToken(String refreshToken) {
        refreshTokens.remove(refreshToken);
    }
}
```

Atualizando o endpoint de autenticação:

```java
@Path("/auth")
public class AuthResource {

    @Inject
    TokenService tokenService;
    
    @Inject
    RefreshTokenService refreshTokenService;
    
    @POST
    @Path("/login")
    public Response login(@FormParam("username") @NotBlank String username, 
                          @FormParam("password") @NotBlank String password) {
        // ... lógica de verificação do usuário
        
        String token = tokenService.generateToken(username, usuario.roles, 86400L);
        String refreshToken = refreshTokenService.generateRefreshToken(username);
        
        Map<String, String> result = new HashMap<>();
        result.put("token", token);
        result.put("refreshToken", refreshToken);
        
        return Response.ok(result).build();
    }
    
    @POST
    @Path("/refresh")
    public Response refresh(@FormParam("refreshToken") @NotBlank String refreshToken) {
        try {
            String newToken = refreshTokenService.refreshAccessToken(refreshToken);
            return Response.ok(Collections.singletonMap("token", newToken)).build();
        } catch (Exception e) {
            return Response.status(Response.Status.UNAUTHORIZED)
                    .entity(Collections.singletonMap("message", e.getMessage()))
                    .build();
        }
    }
    
    @POST
    @Path("/logout")
    public Response logout(@FormParam("refreshToken") @NotBlank String refreshToken) {
        refreshTokenService.invalidateRefreshToken(refreshToken);
        return Response.ok(Collections.singletonMap("message", "Logout realizado com sucesso")).build();
    }
}
```

## Configurando CORS para Compatibilidade com Frontend

Se sua API será consumida por um aplicativo frontend em outro domínio, você precisará configurar CORS:

```properties
# application.properties
quarkus.http.cors=true
quarkus.http.cors.origins=http://localhost:3000,https://meuapp.com
quarkus.http.cors.methods=GET,POST,PUT,DELETE,OPTIONS
quarkus.http.cors.headers=Content-Type,Authorization
quarkus.http.cors.exposed-headers=Content-Disposition
quarkus.http.cors.access-control-max-age=24H
quarkus.http.cors.access-control-allow-credentials=true
```

## Integração com Swagger UI

Você pode documentar suas rotas de autenticação com Swagger UI:

```java
@Path("/auth")
@Tag(name = "Autenticação", description = "Endpoints para autenticação de usuários")
public class AuthResource {
    
    @POST
    @Path("/login")
    @Operation(summary = "Autenticar usuário", description = "Autentica um usuário e retorna um token JWT")
    @APIResponses({
        @APIResponse(responseCode = "200", description = "Autenticação bem-sucedida", 
                     content = @Content(schema = @Schema(implementation = TokenResponse.class))),
        @APIResponse(responseCode = "401", description = "Credenciais inválidas")
    })
    public Response login(...) {
        // ...
    }
    
    // Classe para documentação do Swagger
    public static class TokenResponse {
        @Schema(description = "Token JWT para autenticação", example = "eyJhbGciOiJSUzI1NiJ9...")
        public String token;
        
        @Schema(description = "Token de atualização", example = "a1b2c3d4-e5f6-g7h8-i9j0-k1l2m3n4o5p6")
        public String refreshToken;
    }
}
```

## Testando a Autenticação JWT

Vamos criar alguns testes para verificar se nossa autenticação está funcionando:

```java
package org.acme.test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.CoreMatchers.containsString;
import static org.hamcrest.CoreMatchers.is;

@QuarkusTest
public class AuthTest {

    @Test
    public void testLoginSuccess() {
        String token = given()
            .contentType(ContentType.URLENC)
            .formParam("username", "admin")
            .formParam("password", "senha123")
        .when()
            .post("/auth/login")
        .then()
            .statusCode(200)
            .body("token", containsString("."))
            .body("refreshToken", containsString("-"))
            .extract().path("token");
        
        // Testa o acesso a um recurso protegido com o token obtido
        given()
            .header("Authorization", "Bearer " + token)
        .when()
            .get("/api/protegido/admin")
        .then()
            .statusCode(200)
            .body("message", containsString("Acesso administrativo concedido"));
    }
    
    @Test
    public void testLoginFailed() {
        given()
            .contentType(ContentType.URLENC)
            .formParam("username", "admin")
            .formParam("password", "senhaerrada")
        .when()
            .post("/auth/login")
        .then()
            .statusCode(401)
            .body("message", is("Credenciais inválidas"));
    }
    
    @Test
    public void testUnauthorizedAccess() {
        given()
        .when()
            .get("/api/protegido/admin")
        .then()
            .statusCode(401);
    }
}
```

## Melhores Práticas de Segurança

1. **Use HTTPS**: Sempre use HTTPS em produção para proteger as comunicações.

2. **Defina tempo de expiração curto**: Use tokens de acesso com tempo de vida curto (por exemplo, 15-60 minutos) e implemente refresh tokens para renovação.

3. **Armazene tokens com segurança**: Em clientes web, armazene os tokens no localStorage ou sessionStorage e nunca em cookies sem proteção.

4. **Implemente blacklist de tokens**: Para casos de logout ou comprometimento de token.

5. **Valide todas as entradas**: Implemente validação rigorosa em todos os endpoints.

6. **Use algoritmos seguros**: Prefira algoritmos como RSA ou ECDSA em vez de HMAC para assinaturas de token.

7. **Mantenha suas chaves seguras**: Nunca exponha suas chaves privadas.

8. **Audite seu código**: Realize auditorias de segurança regularmente.

## Conclusão

A implementação de JWT com Quarkus oferece uma maneira robusta e eficiente de proteger suas APIs. Ao seguir as melhores práticas e entender os conceitos fundamentais, você pode garantir que sua aplicação seja segura e escalável.

No próximo tópico, vamos explorar técnicas avançadas de teste para APIs seguras, garantindo que sua implementação de segurança esteja funcionando corretamente em todos os cenários.
