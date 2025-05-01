# 8-Idempotencia-no-quarkus

Este guia prático mostra como implementar idempotência em suas APIs Quarkus. A idempotência é um conceito fundamental para criar APIs robustas e confiáveis, especialmente quando lidamos com operações que não devem ser executadas mais de uma vez, como transações financeiras, processamento de pedidos ou atualizações críticas de dados.

> **A idempotência é crucial para APIs resilientes**
>
> Implementar idempotência nas suas APIs garante que, mesmo quando clientes reenviam a mesma requisição devido a falhas de rede ou timeouts, suas operações não sejam duplicadas acidentalmente, evitando cobranças duplicadas, processamentos redundantes e outros problemas críticos.
>
{style="note"}

## Antes de começar
Certifique-se de que você tem:
- Um projeto Quarkus existente
- JDK 11+ instalado
- Maven ou Gradle configurado
- Conhecimento básico de RESTful APIs
- Familiaridade com conceitos de idempotência


## O que é idempotência?
Idempotência é a propriedade de certas operações matemáticas e de computação que podem ser aplicadas várias vezes sem alterar o resultado além da aplicação inicial. No contexto de APIs REST:
- Uma requisição é idempotente quando produz o mesmo estado no servidor e retorna o mesmo resultado, independentemente de quantas vezes for executada.
- Métodos HTTP GET, HEAD, PUT e DELETE são naturalmente idempotentes pela definição do protocolo HTTP.
- Métodos HTTP POST normalmente não são idempotentes por padrão e requerem implementação adicional para garantir essa propriedade.

## Como implementar idempotência básica no Quarkus
Vamos implementar um mecanismo de idempotência usando chaves de idempotência (idempotency keys) e um cache para rastrear requisições processadas.

1. Adicione as dependências necessárias ao seu arquivo `pom.xml` (Maven) ou `build.gradle` (Gradle)


   ```xml
    <!-- Para Maven -->
   <dependency>
     <groupId>io.quarkus</groupId>
     <artifactId>quarkus-cache</artifactId>
   </dependency>
   <dependency>
     <groupId>io.quarkus</groupId>
     <artifactId>quarkus-resteasy-jackson</artifactId>
   </dependency>
   ```

2. Crie uma classe de anotação para marcar os endpoints que necessitam de idempotência

```Java
package org.acme.idempotency;

   import java.lang.annotation.ElementType;
   import java.lang.annotation.Retention;
   import java.lang.annotation.RetentionPolicy;
   import java.lang.annotation.Target;

   @Target({ElementType.METHOD, ElementType.TYPE})
   @Retention(RetentionPolicy.RUNTIME)
   public @interface Idempotent {
       /**
        * Tempo de expiração da chave de idempotência em segundos.
        * Depois desse período, a mesma requisição será tratada como nova.
        */
       int expireAfter() default 3600; // 1 hora por padrão
   }
```

3. Implemente um filtro para processar requisições idempotentes

```Java
package org.acme.idempotency;

   import io.quarkus.cache.Cache;
   import io.quarkus.cache.CacheName;

   import javax.annotation.Priority;
   import javax.enterprise.context.ApplicationScoped;
   import javax.inject.Inject;
   import javax.ws.rs.Priorities;
   import javax.ws.rs.container.ContainerRequestContext;
   import javax.ws.rs.container.ContainerRequestFilter;
   import javax.ws.rs.container.ContainerResponseContext;
   import javax.ws.rs.container.ContainerResponseFilter;
   import javax.ws.rs.container.ResourceInfo;
   import javax.ws.rs.core.Context;
   import javax.ws.rs.core.Response;
   import javax.ws.rs.ext.Provider;
   import java.io.ByteArrayInputStream;
   import java.io.ByteArrayOutputStream;
   import java.io.IOException;
   import java.lang.reflect.Method;
   import java.time.Instant;
   import java.util.concurrent.CompletableFuture;
   import java.util.concurrent.CompletionStage;
   import java.util.concurrent.ExecutionException;

   @Provider
   @ApplicationScoped
   @Priority(Priorities.HEADER_DECORATOR)
   public class IdempotencyFilter implements ContainerRequestFilter, ContainerResponseFilter {

       private static final String IDEMPOTENCY_KEY_HEADER = "X-Idempotency-Key";
       private static final String IDEMPOTENT_CONTEXT_PROPERTY = "idempotent-context";

       @Inject
       @CacheName("idempotency-cache")
       Cache cache;

       @Context
       ResourceInfo resourceInfo;

       @Override
       public void filter(ContainerRequestContext requestContext) throws IOException {
           // Verifica se o método/classe tem a anotação @Idempotent
           Method method = resourceInfo.getResourceMethod();
           Class<?> clazz = resourceInfo.getResourceClass();

           Idempotent methodAnnotation = method.getAnnotation(Idempotent.class);
           Idempotent classAnnotation = clazz.getAnnotation(Idempotent.class);

           if (methodAnnotation == null && classAnnotation == null) {
               // Endpoint não requer idempotência
               return;
           }

           // Usa a anotação do método, se existir, senão usa a da classe
           Idempotent idempotentConfig = methodAnnotation != null ? methodAnnotation : classAnnotation;

           // Obtém a chave de idempotência do cabeçalho
           String idempotencyKey = requestContext.getHeaderString(IDEMPOTENCY_KEY_HEADER);

           if (idempotencyKey == null || idempotencyKey.isBlank()) {
               // Chave de idempotência não fornecida
               requestContext.abortWith(Response
                   .status(Response.Status.BAD_REQUEST)
                   .entity("O cabeçalho X-Idempotency-Key é obrigatório para esta operação")
                   .build());
               return;
           }

           // Cria o identificador único para esta requisição
           String cacheKey = createCacheKey(requestContext, idempotencyKey);

           try {
               // Verifica se a requisição já foi processada
               CompletionStage<IdempotencyRecord> recordStage = cache.get(cacheKey, k -> CompletableFuture.completedFuture(null));
               IdempotencyRecord record = recordStage.toCompletableFuture().get();

               if (record != null) {
                   // A requisição já foi processada, retorna o resultado cacheado
                   requestContext.abortWith(Response
                       .status(record.getStatus())
                       .entity(record.getBody())
                       .build());
                   return;
               }

               // Armazena o contexto para uso no filtro de resposta
               requestContext.setProperty(IDEMPOTENT_CONTEXT_PROPERTY, 
                   new IdempotentContext(cacheKey, idempotentConfig.expireAfter()));

           } catch (InterruptedException | ExecutionException e) {
               Thread.currentThread().interrupt();
               // Em caso de erro, permite a requisição continuar (fail-open)
               // Pode ser alterado para fail-closed se necessário
           }
       }

       @Override
       public void filter(ContainerRequestContext requestContext, ContainerResponseContext responseContext) {
           IdempotentContext context = (IdempotentContext) requestContext.getProperty(IDEMPOTENT_CONTEXT_PROPERTY);
           if (context == null) {
               // Não é uma requisição idempotente
               return;
           }

           // Cria um registro para armazenar no cache
           IdempotencyRecord record = new IdempotencyRecord(
               responseContext.getStatus(),
               responseContext.getEntity(),
               Instant.now().plusSeconds(context.getExpireAfter())
           );

           // Armazena o resultado no cache
           cache.put(context.getCacheKey(), record);
       }

       private String createCacheKey(ContainerRequestContext requestContext, String idempotencyKey) {
           // Combina método, caminho e chave de idempotência para criar uma chave única
           return requestContext.getMethod() + ":" + 
                  requestContext.getUriInfo().getPath() + ":" + 
                  idempotencyKey;
       }

       // Classe para armazenar o contexto entre os filtros de requisição e resposta
       private static class IdempotentContext {
           private final String cacheKey;
           private final int expireAfter;

           public IdempotentContext(String cacheKey, int expireAfter) {
               this.cacheKey = cacheKey;
               this.expireAfter = expireAfter;
           }

           public String getCacheKey() {
               return cacheKey;
           }

           public int getExpireAfter() {
               return expireAfter;
           }
       }

       // Classe para armazenar o resultado de uma requisição idempotente
       private static class IdempotencyRecord {
           private final int status;
           private final Object body;
           private final Instant expiry;

           public IdempotencyRecord(int status, Object body, Instant expiry) {
               this.status = status;
               this.body = body;
               this.expiry = expiry;
           }

           public int getStatus() {
               return status;
           }

           public Object getBody() {
               return body;
           }

           public Instant getExpiry() {
               return expiry;
           }
       }
   }
```

4. Configure o cache no arquivo `application.properties`

```Plain Text
# Configurações do cache para idempotência
quarkus.cache.caffeine."idempotency-cache".initial-capacity=100
quarkus.cache.caffeine."idempotency-cache".maximum-size=1000
quarkus.cache.caffeine."idempotency-cache".expire-after-write=PT1H
```

5. Aplique a anotação de idempotência aos seus recursos (endpoints)

```Java
package org.acme.resource;

   import javax.ws.rs.*;
   import javax.ws.rs.core.MediaType;
   import javax.ws.rs.core.Response;
   import java.net.URI;
   import java.util.UUID;
   import javax.validation.Valid;

   import org.acme.idempotency.Idempotent;
   import org.acme.model.Pedido;
   import org.acme.service.PedidoService;

   @Path("/pedidos")
   @Produces(MediaType.APPLICATION_JSON)
   @Consumes(MediaType.APPLICATION_JSON)
   public class PedidoResource {

       private final PedidoService pedidoService;

       public PedidoResource(PedidoService pedidoService) {
           this.pedidoService = pedidoService;
       }

       @GET
       public Response listarPedidos() {
           return Response.ok(pedidoService.listarTodos()).build();
       }

       @POST
       @Idempotent(expireAfter = 7200) // 2 horas
       public Response criarPedido(@Valid Pedido pedido) {
           Pedido novoPedido = pedidoService.criar(pedido);
           return Response
               .created(URI.create("/pedidos/" + novoPedido.getId()))
               .entity(novoPedido)
               .build();
       }

       @PUT
       @Path("/{id}")
       @Idempotent
       public Response atualizarPedido(@PathParam("id") Long id, @Valid Pedido pedido) {
           Pedido pedidoAtualizado = pedidoService.atualizar(id, pedido);
           return Response.ok(pedidoAtualizado).build();
       }

       @DELETE
       @Path("/{id}")
       @Idempotent
       public Response excluirPedido(@PathParam("id") Long id) {
           pedidoService.excluir(id);
           return Response.noContent().build();
       }

       @POST
       @Path("/{id}/pagamento")
       @Idempotent(expireAfter = 86400) // 24 horas
       public Response processarPagamento(@PathParam("id") Long pedidoId) {
           // Esta operação delicada de pagamento será executada apenas uma vez,
           // mesmo se o cliente reenviar a mesma requisição com a mesma chave de idempotência
           return Response.ok(pedidoService.processarPagamento(pedidoId)).build();
       }
   }
```

## Implementando idempotência com Redis (para aplicações em cluster)
Para aplicações que executam em múltiplas instâncias, é recomendado usar um armazenamento distribuído como o Redis para manter as chaves de idempotência.
1. Adicione a extensão Redis ao seu projeto Quarkus

```Shell
./mvnw quarkus:add-extension -Dextensions="quarkus-redis-client"
```

2. Configure o Redis no arquivo `application.properties`

3. Substitua o `IdempotencyFilter` por uma versão que usa Redis

```Java
package org.acme.idempotency;

import io.quarkus.redis.client.RedisClient;
import io.vertx.redis.client.Response;

import javax.annotation.Priority;
import javax.enterprise.context.ApplicationScoped;
import javax.inject.Inject;
import javax.ws.rs.Priorities;
import javax.ws.rs.container.ContainerRequestContext;
import javax.ws.rs.container.ContainerRequestFilter;
import javax.ws.rs.container.ContainerResponseContext;
import javax.ws.rs.container.ContainerResponseFilter;
import javax.ws.rs.container.ResourceInfo;
import javax.ws.rs.core.Context;
import javax.ws.rs.ext.Provider;
import java.io.IOException;
import java.lang.reflect.Method;
import java.util.Arrays;
import java.util.List;
import javax.ws.rs.core.Response.Status;

import com.fasterxml.jackson.databind.ObjectMapper;

@Provider
@ApplicationScoped
@Priority(Priorities.HEADER_DECORATOR)
public class RedisIdempotencyFilter implements ContainerRequestFilter, ContainerResponseFilter {

    private static final String IDEMPOTENCY_KEY_HEADER = "X-Idempotency-Key";
    private static final String IDEMPOTENT_CONTEXT_PROPERTY = "idempotent-context";

    @Inject
    RedisClient redisClient;

    @Inject
    ObjectMapper objectMapper;

    @Context
    ResourceInfo resourceInfo;

    @Override
    public void filter(ContainerRequestContext requestContext) throws IOException {
        // Verifica se o método/classe tem a anotação @Idempotent
        Method method = resourceInfo.getResourceMethod();
        Class<?> clazz = resourceInfo.getResourceClass();

        Idempotent methodAnnotation = method.getAnnotation(Idempotent.class);
        Idempotent classAnnotation = clazz.getAnnotation(Idempotent.class);

        if (methodAnnotation == null && classAnnotation == null) {
            // Endpoint não requer idempotência
            return;
        }

        // Usa a anotação do método, se existir, senão usa a da classe
        Idempotent idempotentConfig = methodAnnotation != null ? methodAnnotation : classAnnotation;

        // Obtém a chave de idempotência do cabeçalho
        String idempotencyKey = requestContext.getHeaderString(IDEMPOTENCY_KEY_HEADER);

        if (idempotencyKey == null || idempotencyKey.isBlank()) {
            // Chave de idempotência não fornecida
            requestContext.abortWith(javax.ws.rs.core.Response
                    .status(Status.BAD_REQUEST)
                    .entity("O cabeçalho X-Idempotency-Key é obrigatório para esta operação")
                    .build());
            return;
        }

        // Cria a chave para o Redis
        String redisKey = createRedisKey(requestContext, idempotencyKey);

        // Verifica se a requisição já foi processada
        Response redisResponse = redisClient.get(redisKey);

        if (redisResponse != null) {
            // A requisição já foi processada, retorna o resultado armazenado
            IdempotencyRecord record = objectMapper.readValue(
                    redisResponse.toString(),
                    IdempotencyRecord.class
            );

            requestContext.abortWith(javax.ws.rs.core.Response
                    .status(record.getStatus())
                    .entity(record.getBody())
                    .build());
            return;
        }

        // Armazena o contexto para uso no filtro de resposta
        requestContext.setProperty(IDEMPOTENT_CONTEXT_PROPERTY,
                new IdempotentContext(redisKey, idempotentConfig.expireAfter()));
    }

    @Override
    public void filter(ContainerRequestContext requestContext, ContainerResponseContext responseContext) throws IOException {
        IdempotentContext context = (IdempotentContext) requestContext.getProperty(IDEMPOTENT_CONTEXT_PROPERTY);
        if (context == null) {
            // Não é uma requisição idempotente
            return;
        }

        // Cria um registro para armazenar no Redis
        IdempotencyRecord record = new IdempotencyRecord(
                responseContext.getStatus(),
                responseContext.getEntity(),
                null // Não precisamos do expiry aqui pois o Redis gerenciará isso
        );

        // Serializa o registro
        String recordJson = objectMapper.writeValueAsString(record);

        // Armazena o resultado no Redis com expiração
        redisClient.setex(
                context.getRedisKey(),
                String.valueOf(context.getExpireAfter()),
                recordJson
        );
    }

    private String createRedisKey(ContainerRequestContext requestContext, String idempotencyKey) {
        // Combina método, caminho e chave de idempotência para criar uma chave única
        return "idempotency:" +
                requestContext.getMethod() + ":" +
                requestContext.getUriInfo().getPath() + ":" +
                idempotencyKey;
    }

    // Classes auxiliares
    private static class IdempotentContext {
        private final String redisKey;
        private final int expireAfter;

        public IdempotentContext(String redisKey, int expireAfter) {
            this.redisKey = redisKey;
            this.expireAfter = expireAfter;
        }

        public String getRedisKey() {
            return redisKey;
        }

        public int getExpireAfter() {
            return expireAfter;
        }
    }

    // Use o Jackson para serialização/deserialização
    public static class IdempotencyRecord {
        private int status;
        private Object body;
        private String expiry;

        // Construtor padrão para deserialização
        public IdempotencyRecord() {}

        public IdempotencyRecord(int status, Object body, String expiry) {
            this.status = status;
            this.body = body;
            this.expiry = expiry;
        }

        public int getStatus() {
            return status;
        }

        public void setStatus(int status) {
            this.status = status;
        }

        public Object getBody() {
            return body;
        }

        public void setBody(Object body) {
            this.body = body;
        }

        public String getExpiry() {
            return expiry;
        }

        public void setExpiry(String expiry) {
            this.expiry = expiry;
        }
    }
}
```

## Geração automática de chaves de idempotência
Para facilitar o uso pelos clientes, você pode implementar um utilitário para gerar chaves de idempotência automaticamente:

```Java
package org.acme.idempotency;

import java.time.Instant;
import java.util.UUID;

/**
 * Utilitário para trabalhar com chaves de idempotência
 */
public class IdempotencyKeyGenerator {
    
    /**
     * Gera uma chave de idempotência baseada em UUID
     */
    public static String generateKey() {
        return UUID.randomUUID().toString();
    }
    
    /**
     * Gera uma chave de idempotência determinística baseada no conteúdo
     * Útil quando o cliente precisa regenerar a mesma chave para a mesma operação
     */
    public static String generateDeterministicKey(String entityId, String operation, String clientId) {
        return UUID.nameUUIDFromBytes(
            (entityId + "-" + operation + "-" + clientId).getBytes()
        ).toString();
    }
    
    /**
     * Gera uma chave de idempotência que inclui timestamp
     * Útil para depuração e rastreamento
     */
    public static String generateTimeBasedKey(String prefix) {
        String timestamp = String.valueOf(Instant.now().toEpochMilli());
        String random = UUID.randomUUID().toString().substring(0, 8);
        return prefix + "-" + timestamp + "-" + random;
    }
}
```

## Práticas recomendadas para idempotência em APIs
1. **Sempre use HTTPS**: As chaves de idempotência são dados sensíveis que não devem ser interceptadas.
2. **Defina políticas de expiração**: Chaves de idempotência não devem ser armazenadas indefinidamente. Configure períodos de expiração adequados com base nos requisitos do seu negócio.
3. **Documente para os clientes**: Explique claramente como os clientes devem usar as chaves de idempotência e como sua API se comporta com requisições repetidas.
4. **Considere a semântica das operações**: Diferentes operações podem exigir diferentes estratégias de idempotência.
5. **Trate falhas de armazenamento**: Se o cache ou Redis não estiver disponível, decida se sua API deve falhar de forma segura (fail-closed) ou permitir a operação (fail-open).
6. **Use IDs naturais quando possível**: Para operações como criação de pedidos, use identificadores naturais (como número de referência do pedido) como parte da chave de idempotência.
7. **Implemente TTL (Time To Live)**: Defina por quanto tempo uma chave de idempotência deve ser válida. Após esse período, uma requisição com a mesma chave será tratada como nova.
8. **Monitore o uso**: Acompanhe métricas sobre requisições idempotentes repetidas para identificar problemas de clientes ou na rede.

## Conclusão
A implementação de idempotência é um padrão essencial para APIs robustas e confiáveis. No Quarkus, podemos aproveitar a infraestrutura de cache nativa ou integrações com Redis para implementar esse padrão de forma eficiente e escalável.
Ao seguir as práticas recomendadas e implementar corretamente a idempotência, suas APIs poderão lidar de forma segura com falhas de rede, timeouts e retentativas de clientes, garantindo que operações críticas sejam executadas exatamente uma vez, independentemente de quantas vezes a requisição seja enviada.

