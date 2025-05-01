# 6. Como Implementar Autenticação com API Key no Quarkus

Este guia explica passo a passo como implementar e usar autenticação por API key em aplicações Quarkus.

## Introdução

A autenticação por API key é uma abordagem simples e eficaz para proteger APIs RESTful. Uma API key é um código único que identifica o chamador da API e autoriza o acesso aos seus recursos. O Quarkus permite implementar essa estratégia de segurança através de filtros personalizados.

## Pré-requisitos

- Quarkus instalado
- Conhecimento básico de desenvolvimento Java
- Um projeto Quarkus existente

## Passo 1: Configurar as Propriedades da Aplicação

Adicione a configuração da API key no arquivo `application.properties`:

```plain text
quarkus.api-key.value=sua-api-key-secreta
quarkus.api-key.header-name=X-API-Key
```

## Passo 2: Criar um Filtro de API Key
Crie uma classe chamada `ApiKeyFilter` para interceptar as requisições e verificar a API key:

```Java
package org.acme.security;

import javax.enterprise.context.ApplicationScoped;
import javax.inject.Inject;
import javax.ws.rs.container.ContainerRequestContext;
import javax.ws.rs.container.ContainerRequestFilter;
import javax.ws.rs.core.Response;
import javax.ws.rs.ext.Provider;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@Provider
@ApplicationScoped
public class ApiKeyFilter implements ContainerRequestFilter {

    @ConfigProperty(name = "quarkus.api-key.value")
    String apiKey;
    
    @ConfigProperty(name = "quarkus.api-key.header-name", defaultValue
            = "X-API-Key")
    String apiKeyHeader;

    @Override
    public void filter(ContainerRequestContext requestContext) {
        // Verificar se é uma rota pública (opcional)
        if (isPublicRoute(requestContext.getUriInfo().getPath())) {
            return;
        }
        
        // Obter o valor do cabeçalho da API key
        String providedKey = 
                requestContext.getHeaderString(apiKeyHeader);
        
        // Verificar se a API key é válida
        if (providedKey == null || !providedKey.equals(apiKey)) {
            requestContext.abortWith(
                Response.status(Response.Status.UNAUTHORIZED)
                    .entity("API key inválida ou ausente")
                    .build());
        }
    }
    
    private boolean isPublicRoute(String path) {
        // Defina aqui suas rotas públicas que não requerem autentica
        // ção
        return path.contains("/public/") || path.startsWith("/health") 
                || path.startsWith("/metrics");
    }
}
```
## Passo 3: Criar um Interceptor para Anotações Personalizadas (Opcional)
Se quiser ter mais controle sobre quais endpoints são protegidos por API key, crie uma anotação personalizada:

```Java
package org.acme.security;

import javax.enterprise.util.Nonbinding;
import javax.interceptor.InterceptorBinding;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@InterceptorBinding
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface RequiresApiKey {
    @Nonbinding String[] roles() default {};
}
```

## Passo 4: Implementar um Serviço de Validação de API Key
Crie um serviço para validar a API key e gerenciar usuários associados a chaves (opcional para casos mais complexos):

```Java
package org.acme.security;

import javax.enterprise.context.ApplicationScoped;
import javax.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
public class ApiKeyService {

    @ConfigProperty(name = "quarkus.api-key.value")
    String configuredApiKey;
    
    public boolean isValid(String apiKey) {
        return apiKey != null && apiKey.equals(configuredApiKey);
    }
    
    // Método opcional para obter informações associadas a uma API key
    public ApiKeyInfo getApiKeyInfo(String apiKey) {
        if (isValid(apiKey)) {
            return new ApiKeyInfo("default-user", new String[]{"user"
            });
        }
        return null;
    }
    
    // Classe interna para representar informações da API key
    public static class ApiKeyInfo {
        private final String username;
        private final String[] roles;
        
        public ApiKeyInfo(String username, String[] roles) {
            this.username = username;
            this.roles = roles;
        }
        
        public String getUsername() {
            return username;
        }
        
        public String[] getRoles() {
            return roles;
        }
    }
}
```

## Passo 5: Integrar com o Sistema de Segurança do Quarkus (Avançado)
Para integração mais profunda com o sistema de segurança do Quarkus, implemente uma classe `SecurityIdentityAugmentor`:

```Java
package org.acme.security;

import io.quarkus.security.identity.AuthenticationRequestContext;
import io.quarkus.security.identity.SecurityIdentity;
import io.quarkus.security.identity.SecurityIdentityAugmentor;
import io.smallrye.mutiny.Uni;

import javax.enterprise.context.ApplicationScoped;
import javax.inject.Inject;
import java.util.function.Supplier;

@ApplicationScoped
public class ApiKeySecurityAugmentor implements SecurityIdentityAugmentor {

    @Inject
    ApiKeyService apiKeyService;

    @Override
    public Uni<SecurityIdentity> augment(SecurityIdentity identity, AuthenticationRequestContext context) {
        return Uni.createFrom().item(() -> identity);
    }
}
```

## Passo 6: Como Usar a API Key no Cliente
Para consumir sua API protegida, os clientes devem incluir a API key no cabeçalho das requisições:

```Shell
# Exemplo usando curl
curl -H "X-API-Key: sua-api-key-secreta" https://seu-servico/api/recurso
```

```Java
HttpClient client = HttpClient.newBuilder().build();
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://seu-servico/api/recurso"))
    .header("X-API-Key", "sua-api-key-secreta")
    .GET()
    .build();
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
```

## Considerações de Segurança
1. **Armazenamento Seguro**: Armazene as API keys de forma segura, usando variáveis de ambiente ou cofres de senhas.
2. **Expiração**: Considere implementar um mecanismo de expiração para suas API keys.
3. **HTTPS**: Sempre use HTTPS para transmitir API keys.
4. **Logs**: Evite registrar API keys completas nos logs.
5. **Rotação**: Implemente um processo para rotação periódica das API keys.

## Conclusão
A implementação de autenticação por API key no Quarkus é uma solução simples e eficaz para proteger seus endpoints REST. Ela oferece um bom equilíbrio entre segurança e facilidade de uso para muitos cenários de API, especialmente em comunicações servidor-para-servidor [[3]](https://ccpatrut.medium.com/quarkus-api-key-implementation-36d9738abda7) [[9]](https://medium.com/arconsis/securing-your-rest-api-with-quarkus-implementing-an-api-key-request-filter-515ead51739f).
Para casos mais complexos de segurança, considere utilizar outras estratégias como OAuth2 ou JWT, que o Quarkus também suporta através de suas extensões de segurança [[1]](https://quarkus.io/guides/security-overview) [[6]](https://quarkus.io/guides/security-authentication-mechanisms).

