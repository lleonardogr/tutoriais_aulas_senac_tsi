# 7. Como Implementar Rate Limit no Quarkus

Este guia prático mostra como implementar limites de taxa (rate limiting) em suas APIs Quarkus. O rate limiting é uma técnica essencial para proteger suas APIs contra uso excessivo, ataques de negação de serviço (DoS) e para garantir um uso justo dos recursos entre todos os usuários da sua aplicação.

> **O rate limiting é crucial para APIs em produção**
>
> Implementar limites de taxa não apenas protege sua infraestrutura, mas também ajuda a manter a qualidade de serviço para todos os usuários e pode reduzir custos operacionais.
>
{style="note"}

## Antes de começar

Certifique-se de que você tem:

- Um projeto Quarkus existente
- JDK 11+ instalado
- Maven ou Gradle configurado
- Conhecimento básico de RESTful APIs
- Familiaridade com filtros JAX-RS

## Como implementar rate limiting com SmallRye Fault Tolerance
O SmallRye Fault Tolerance é uma implementação da especificação MicroProfile Fault Tolerance que oferece recursos como circuit breaker, timeout, retry, fallback e rate limiting nativamente integrados ao Quarkus.
1. Adicione a extensão SmallRye Fault Tolerance ao seu projeto

   ```xml
   <!-- Para Maven -->
   <dependency>
     <groupId>io.quarkus</groupId>
     <artifactId>quarkus-smallrye-fault-tolerance</artifactId>
   </dependency>

   ```

2. Use a anotação `@RateLimit` nos métodos que deseja limita

   ```java
   package org.acme.resource;
   
   import javax.enterprise.context.ApplicationScoped;
   import javax.ws.rs.GET;
   import javax.ws.rs.Path;
   import javax.ws.rs.Produces;
   import javax.ws.rs.core.MediaType;
   
   import org.eclipse.microprofile.faulttolerance.Fallback;
   import org.eclipse.microprofile.faulttolerance.RateLimit;
   import org.eclipse.microprofile.faulttolerance.exceptions.RateLimitException;
   
   import java.time.temporal.ChronoUnit;
   
   @Path("/api")
   @ApplicationScoped
   public class ExemploResource {
   
       @GET
       @Path("/publico")
       @Produces(MediaType.TEXT_PLAIN)
       public String endpointPublico() {
           return "Este endpoint não tem limite de taxa";
       }
       
       @GET
       @Path("/limitado")
       @Produces(MediaType.TEXT_PLAIN)
       @RateLimit(value = 5, windowDuration = 1, windowDurationUnit = ChronoUnit.MINUTES)
       @Fallback(fallbackMethod = "fallbackParaRateLimit")
       public String endpointLimitado() {
           return "Este endpoint permite apenas 5 requisições por minuto";
       }
       
       @GET
       @Path("/muito-limitado")
       @Produces(MediaType.TEXT_PLAIN)
       @RateLimit(value = 2, windowDuration = 1, windowDurationUnit = ChronoUnit.MINUTES)
       @Fallback(fallbackMethod = "fallbackParaRateLimit")
       public String endpointMuitoLimitado() {
           return "Este endpoint permite apenas 2 requisições por minuto";
       }
       
       /**
        * Método de fallback para quando o rate limit é excedido
        */
       public String fallbackParaRateLimit() {
           return "Taxa de requisições excedida. Por favor, tente novamente mais tarde.";
       }
   }
   ```

3. Configurando o fallback para exceções específicas

```Java
@GET
   @Path("/limitado-com-mensagem")
   @Produces(MediaType.TEXT_PLAIN)
   @RateLimit(value = 3, windowDuration = 1, windowDurationUnit = ChronoUnit.MINUTES)
   @Fallback(fallbackMethod = "fallbackComMensagem")
   public String endpointLimitadoComMensagem() {
       return "Este endpoint permite apenas 3 requisições por minuto";
   }
   
   /**
    * Método de fallback que recebe a exceção
    */
   public String fallbackComMensagem(RateLimitException e) {
       return "Taxa de requisições excedida. O limite é de 3 requisições por minuto. " +
              "Por favor, tente novamente mais tarde.";
   }
```

4. Aplicando rate limit a uma classe inteira

```Java
package org.acme.resource;
   
   import javax.enterprise.context.ApplicationScoped;
   import javax.ws.rs.GET;
   import javax.ws.rs.Path;
   import javax.ws.rs.Produces;
   import javax.ws.rs.core.MediaType;
   
   import org.eclipse.microprofile.faulttolerance.Fallback;
   import org.eclipse.microprofile.faulttolerance.RateLimit;
   
   import java.time.temporal.ChronoUnit;
   
   @Path("/produtos")
   @ApplicationScoped
   @RateLimit(value = 20, windowDuration = 1, windowDurationUnit = ChronoUnit.MINUTES)
   @Fallback(fallbackMethod = "fallbackGlobal")
   public class ProdutoResource {
   
       @GET
       @Produces(MediaType.APPLICATION_JSON)
       public String listarProdutos() {
           // Este método herda o rate limit da classe (20 requisições/minuto)
           return "[{\"nome\":\"Produto 1\"}, {\"nome\":\"Produto 2\"}]";
       }
       
       @GET
       @Path("/{id}")
       @Produces(MediaType.APPLICATION_JSON)
       public String buscarProduto(Long id) {
           // Este método também herda o rate limit da classe
           return "{\"id\":" + id + ", \"nome\":\"Produto " + id + "\"}";
       }
       
       @GET
       @Path("/premium")
       @Produces(MediaType.APPLICATION_JSON)
       @RateLimit(value = 5, windowDuration = 1, windowDurationUnit = ChronoUnit.MINUTES)
       public String listarProdutosPremium() {
           // Este método sobrescreve o rate limit da classe
           return "[{\"nome\":\"Produto Premium 1\"}, {\"nome\":\"Produto Premium 2\"}]";
       }
       
       public String fallbackGlobal() {
           return "{\"erro\":\"Taxa de requisições excedida para a API de produtos.\"}";
       }
   }
```

5. Configurando o rate limit via arquivo `application.properties` (opcional)

``` Plain Text
# Configurar rate limit para classes específicas
# Formato: quarkus.fault-tolerance.[identificador].rate-limit.[propriedade]=[valor]

# Habilitar métricas para monitoramento
quarkus.smallrye-metrics.jaxrs.enabled=true

# Taxa de limite global para o serviço de produtos (sobrescreve a anotação)
org.acme.resource.ProdutoResource/listarProdutos.RateLimit/value=30
org.acme.resource.ProdutoResource/listarProdutos.RateLimit/windowDuration=1

# Configurações diferentes por ambiente (desenvolvimento vs produção)
%dev.org.acme.resource.ProdutoResource/listarProdutos.RateLimit/value=50
%prod.org.acme.resource.ProdutoResource/listarProdutos.RateLimit/value=20
```

## Tipo de janelas de tempo para Rate Limit
O SmallRye Fault Tolerance oferece três tipos de janelas de tempo para rate limiting:
1. **Janela fixa (padrão)** - Divide o tempo em intervalos fixos. Por exemplo, com limite de 10 requisições por minuto, o contador é zerado no início de cada minuto.
2. **Janela deslizante (rolling)** - Considera o número de requisições feitas no passado recente. Por exemplo, com limite de 10 requisições por minuto, verifica quantas requisições foram feitas nos últimos 60 segundos.
3. **Janela suave (smooth)** - Similar à janela deslizante, mas com uma transição mais gradual.

```Plain Text
# Configurar janela deslizante para rate limiting
org.acme.resource.ExemploResource/limitado.RateLimit/windowKind=ROLLING
```
## Combinando rate limit com outras funcionalidades de resiliência
SmallRye Fault Tolerance permite combinar diferentes estratégias de resiliência:

```Java
@GET
@Path("/completo")
@Produces(MediaType.TEXT_PLAIN)
@RateLimit(value = 10, windowDuration = 1, windowDurationUnit = ChronoUnit.MINUTES)
@Timeout(value = 2, unit = ChronoUnit.SECONDS)
@Retry(maxRetries = 3, delay = 200, delayUnit = ChronoUnit.MILLIS)
@Fallback(fallbackMethod = "fallbackCompleto")
public String endpointCompleto() {
    // Lógica de negócio
    return "Este endpoint tem rate limit, timeout e retry configurados";
}

public String fallbackCompleto(Exception e) {
    if (e instanceof RateLimitException) {
        return "Taxa de requisições excedida. Tente novamente em breve.";
    } else if (e instanceof TimeoutException) {
        return "A operação atingiu o tempo limite. Tente novamente.";
    } else {
        return "Ocorreu um erro: " + e.getMessage();
    }
}
```

## Monitoramento e métricas
O SmallRye Fault Tolerance se integra naturalmente com as métricas do MicroProfile, permitindo monitorar a eficácia das suas estratégias de resiliência:
1. Adicione a extensão de métricas ao projeto:

```Shell
./mvnw quarkus:add-extension -Dextensions="quarkus-smallrye-metrics"
```

2. As métricas estão disponíveis em `/q/metrics` e incluem:
   - Quantas vezes o rate limit foi excedido
   - Número de requisições aceitas
   - Tempo médio de resposta


## Testando seu rate limiting

Para testar sua implementação, você pode usar ferramentas como `curl` em um script bash:

```bash
#!/bin/bash
# Script para testar rate limiting

URL="http://localhost:8080/api/limitado"

for i in {1..10}; do
  echo "Requisição $i:"
  curl -v "$URL" 2>&1 | grep -E 'HTTP/|X-Rate-Limit|Taxa de requisições excedida'
  echo "---------------------------------------"
  sleep 1
done
```

## Considerações para produção
1. **Escolha limites adequados**: Configure os limites com base na capacidade dos seus servidores e no padrão de uso esperado.
2. **Feedback claro para usuários**: Retorne mensagens claras quando os limites forem excedidos, incluindo o tempo de espera recomendado.
3. **Métricas e alertas**: Monitore a quantidade de requisições rejeitadas e configure alertas para quando os limites são frequentemente atingidos.
4. **Documentação**: Documente seus limites de taxa na documentação da API para que os desenvolvedores possam projetar seus clientes adequadamente.
5. **Configuração por ambiente**: Use perfis do Quarkus para definir limites diferentes em ambientes de desenvolvimento, teste e produção.
