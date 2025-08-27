# 4. Implementando CRUD Completo com Validações

Neste tópico, vamos expandir nossa API Quarkus para implementar um CRUD completo com validações robustas utilizando Hibernate Validator. As validações são essenciais para garantir a integridade dos dados e melhorar a experiência do usuário ao fornecer feedback sobre problemas de dados.

## Por que Validar Dados?

A validação de dados é crucial em APIs por várias razões:

- **Segurança**: Previne ataques de injeção e outras vulnerabilidades
- **Integridade de dados**: Garante que apenas dados válidos sejam persistidos
- **Experiência do usuário**: Fornece feedback imediato sobre erros de dados
- **Simplificação de código**: Reduz verificações condicionais no código
- **Documentação implícita**: Define claramente as regras de negócio

## Adicionando Dependências de Validação

Primeiro, precisamos adicionar a extensão de validação ao nosso projeto. Para isso, você pode executar:

```bash
# Maven
./mvnw quarkus:add-extension -Dextensions="hibernate-validator"

# Gradle
./gradlew addExtension --extensions="hibernate-validator"
```

Ou adicionar manualmente ao arquivo `pom.xml`:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-hibernate-validator</artifactId>
</dependency>
```

## Atualizando a Entidade com Validações

Vamos adicionar validações à nossa entidade `Produto`. Aqui está a versão atualizada com anotações de validação:

```java
package org.acme.entity;

import io.quarkus.hibernate.orm.panache.PanacheEntity;
import javax.persistence.Entity;
import javax.persistence.Column;
import javax.validation.constraints.*;

@Entity
public class Produto extends PanacheEntity {
    
    @NotBlank(message = "O nome do produto é obrigatório")
    @Size(min = 3, max = 100, message = "O nome deve ter entre 3 e 100 caracteres")
    @Column(nullable = false)
    public String nome;
    
    @Size(max = 500, message = "A descrição não pode exceder 500 caracteres")
    public String descricao;
    
    @NotNull(message = "O preço do produto é obrigatório")
    @DecimalMin(value = "0.01", message = "O preço deve ser maior que zero")
    @Column(nullable = false)
    public Double preco;
    
    @Min(value = 0, message = "A quantidade em estoque não pode ser negativa")
    public Integer quantidadeEstoque = 0;
    
    @Pattern(regexp = "^[A-Z0-9]{5,20}$", message = "O código deve conter entre 5 e 20 caracteres alfanuméricos maiúsculos")
    @Column(unique = true)
    public String codigo;
    
    // Data de criação do produto
    public java.time.LocalDateTime dataCriacao = java.time.LocalDateTime.now();
}
```

## Principais Anotações de Validação

Aqui estão algumas das anotações de validação mais comuns:

| Anotação | Descrição |
|----------|-----------|
| `@NotNull` | O campo não pode ser nulo |
| `@NotEmpty` | O campo não pode ser nulo nem vazio (para coleções, arrays, strings) |
| `@NotBlank` | O campo de texto não pode ser nulo, vazio ou apenas espaços em branco |
| `@Size` | Define tamanho mínimo e máximo (para strings, coleções, arrays) |
| `@Min` / `@Max` | Define valores mínimos e máximos para números |
| `@DecimalMin` / `@DecimalMax` | Define valores mínimos e máximos para decimais (com inclusão opcional) |
| `@Pattern` | Valida o campo contra uma expressão regular |
| `@Email` | Valida se o campo é um e-mail válido |
| `@Past` / `@Future` | Valida se a data está no passado/futuro |
| `@Positive` / `@Negative` | Valida se o número é positivo/negativo |

## Atualizando o Resource para Usar Validações

Agora, vamos atualizar o `ProdutoResource` para usar as validações:

```java
package org.acme.resource;

import io.quarkus.hibernate.orm.rest.data.panache.PanacheEntityResource;
import org.acme.entity.Produto;

import javax.enterprise.context.ApplicationScoped;
import javax.validation.Valid;
import javax.validation.constraints.NotNull;
import javax.ws.rs.*;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;
import java.net.URI;
import java.util.List;

@Path("/produtos")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class ProdutoResource {
    
    @GET
    public List<Produto> listarTodos() {
        return Produto.listAll();
    }
    
    @GET
    @Path("/{id}")
    public Response buscarPorId(@PathParam("id") Long id) {
        Produto produto = Produto.findById(id);
        if (produto == null) {
            return Response.status(Response.Status.NOT_FOUND).build();
        }
        return Response.ok(produto).build();
    }
    
    @POST
    public Response criar(@NotNull @Valid Produto produto) {
        produto.persist();
        return Response.created(URI.create("/produtos/" + produto.id)).entity(produto).build();
    }
    
    @PUT
    @Path("/{id}")
    public Response atualizar(@PathParam("id") Long id, @NotNull @Valid Produto produto) {
        Produto entidade = Produto.findById(id);
        if (entidade == null) {
            return Response.status(Response.Status.NOT_FOUND).build();
        }
        
        entidade.nome = produto.nome;
        entidade.descricao = produto.descricao;
        entidade.preco = produto.preco;
        entidade.quantidadeEstoque = produto.quantidadeEstoque;
        entidade.codigo = produto.codigo;
        
        return Response.ok(entidade).build();
    }
    
    @DELETE
    @Path("/{id}")
    public Response excluir(@PathParam("id") Long id) {
        boolean excluido = Produto.deleteById(id);
        if (excluido) {
            return Response.noContent().build();
        }
        return Response.status(Response.Status.NOT_FOUND).build();
    }
    
    @GET
    @Path("/busca/nome/{nome}")
    public List<Produto> buscarPorNome(@PathParam("nome") String nome) {
        return Produto.list("nome LIKE ?1", "%" + nome + "%");
    }
    
    @GET
    @Path("/busca/preco")
    public List<Produto> buscarPorFaixaDePreco(@QueryParam("min") Double precoMinimo, 
                                               @QueryParam("max") Double precoMaximo) {
        if (precoMinimo == null) {
            precoMinimo = 0.0;
        }
        
        if (precoMaximo == null) {
            return Produto.list("preco >= ?1", precoMinimo);
        } else {
            return Produto.list("preco >= ?1 AND preco <= ?2", precoMinimo, precoMaximo);
        }
    }
}
```

## Tratamento Personalizado de Erros de Validação

Por padrão, quando uma validação falha, o Quarkus retorna um status HTTP 400 (Bad Request) com uma mensagem de erro genérica. Podemos melhorar isso criando um handler personalizado para erros de validação:

```java
package org.acme.exception;

import javax.validation.ConstraintViolation;
import javax.validation.ConstraintViolationException;
import javax.ws.rs.core.Response;
import javax.ws.rs.ext.ExceptionMapper;
import javax.ws.rs.ext.Provider;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Provider
public class ValidationExceptionMapper implements ExceptionMapper<ConstraintViolationException> {

    @Override
    public Response toResponse(ConstraintViolationException exception) {
        Map<String, List<String>> erros = new HashMap<>();
        
        // Agrupa erros por propriedade
        Map<String, List<String>> violationsByPropertyPath = exception.getConstraintViolations()
                .stream()
                .collect(Collectors.groupingBy(
                        violation -> violation.getPropertyPath().toString(),
                        Collectors.mapping(ConstraintViolation::getMessage, Collectors.toList())
                ));
        
        erros.put("validacao", violationsByPropertyPath.entrySet().stream()
                .map(entry -> entry.getKey() + ": " + String.join(", ", entry.getValue()))
                .collect(Collectors.toList()));
        
        return Response.status(Response.Status.BAD_REQUEST)
                .entity(erros)
                .build();
    }
}
```

## Criando Validações Customizadas

Às vezes, as validações padrão não são suficientes para regras de negócio específicas. Vamos criar uma anotação de validação personalizada para verificar se o código do produto é válido:

1. Primeiro, crie a anotação:

```java
package org.acme.validation;

import javax.validation.Constraint;
import javax.validation.Payload;
import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = CodigoProdutoValidator.class)
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface CodigoProdutoValido {
    String message() default "Código de produto inválido";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

2. Em seguida, crie o validador:

```java
package org.acme.validation;

import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;

public class CodigoProdutoValidator implements ConstraintValidator<CodigoProdutoValido, String> {
    
    @Override
    public void initialize(CodigoProdutoValido constraintAnnotation) {
        // Inicialização, se necessário
    }
    
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) {
            return true; // Use @NotNull se quiser que o campo seja obrigatório
        }
        
        // Regra: códigos de produto precisam começar com 2 letras seguidas de 3-18 números
        return value.matches("^[A-Z]{2}[0-9]{3,18}$");
    }
}
```

3. Agora, aplique a validação personalizada à entidade:

```java
@CodigoProdutoValido(message = "O código do produto deve começar com 2 letras maiúsculas seguidas de 3 a 18 números")
public String codigo;
```

## Validação em Grupo e Validação em Cascata

### Grupos de Validação

Grupos de validação permitem aplicar diferentes validações em contextos diferentes:

```java
// Definindo interfaces para grupos
public interface OnCreate {}
public interface OnUpdate {}

// Na entidade
@NotBlank(groups = {OnCreate.class, OnUpdate.class})
public String nome;

@NotNull(groups = OnCreate.class)
public String codigo;
```

### Validação em Cascata

Para validar objetos aninhados, use `@Valid` em campos de objeto:

```java
public class Pedido extends PanacheEntity {
    @NotNull
    @Valid // Valida o cliente também
    public Cliente cliente;
    
    @NotEmpty
    @Valid // Valida cada item do pedido
    public List<ItemPedido> itens;
}
```

## Usando Bean Validation em Métodos

Você também pode aplicar validações diretamente aos parâmetros de método:

```java
@GET
@Path("/busca/preco")
public List<Produto> buscarPorFaixaDePreco(
        @QueryParam("min") @DecimalMin("0.0") Double precoMinimo, 
        @QueryParam("max") @DecimalMin("0.0") Double precoMaximo) {
    // implementação
}
```

## Testando Validações

É importante testar se suas validações estão funcionando. Crie testes específicos:

```java
package org.acme.tests;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;
import org.acme.entity.Produto;

import static io.restassured.RestAssured.given;
import static org.hamcrest.CoreMatchers.*;

@QuarkusTest
public class ProdutoValidationTest {
    
    @Test
    public void testCriarProdutoValido() {
        Produto produto = new Produto();
        produto.nome = "Notebook Dell";
        produto.descricao = "Notebook Dell XPS 13";
        produto.preco = 5000.0;
        produto.quantidadeEstoque = 10;
        produto.codigo = "AB12345";
        
        given()
            .contentType(ContentType.JSON)
            .body(produto)
        .when()
            .post("/produtos")
        .then()
            .statusCode(201);
    }
    
    @Test
    public void testCriarProdutoInvalido() {
        Produto produto = new Produto();
        produto.nome = ""; // Nome vazio (inválido)
        produto.preco = -100.0; // Preço negativo (inválido)
        
        given()
            .contentType(ContentType.JSON)
            .body(produto)
        .when()
            .post("/produtos")
        .then()
            .statusCode(400)
            .body("validacao", hasItems(
                containsString("nome"),
                containsString("preco")
            ));
    }
}
```

## Considerações Importantes

1. **Validação Vs. Regras de Negócio**: Use validações de bean para regras simples. Para regras de negócio complexas, considere usar um serviço separado.

2. **Desempenho**: Validações são rápidas, mas considere o impacto de validações personalizadas complexas.

3. **Internacionalização**: Para aplicações multi-idioma, externalize as mensagens de erro para arquivos de propriedades.

4. **Segurança**: Nunca confie apenas em validações do lado do cliente. Sempre valide os dados no servidor.

5. **Documentação**: As validações ajudam a documentar sua API. Use OpenAPI para expor suas regras de validação.

## Integração com OpenAPI/Swagger

As anotações de validação são automaticamente incorporadas à documentação OpenAPI. Para habilitar:

```text
# application.properties
quarkus.swagger-ui.always-include=true
```

## Conclusão

Com as validações implementadas, nossa API agora é mais robusta e confiável. Os clientes recebem feedback claro sobre erros de dados, e a integridade dos dados é garantida.

No próximo tópico, exploraremos como documentar nossa API usando a especificação OpenAPI/Swagger e como implementá-la facilmente com Quarkus.
