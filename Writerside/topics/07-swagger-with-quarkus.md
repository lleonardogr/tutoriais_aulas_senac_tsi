# 5. Documentação de API com Swagger e OpenAPI

A documentação é uma parte essencial de qualquer API. Uma boa documentação torna sua API mais acessível, facilitando o entendimento e uso por outros desenvolvedores. Neste tópico, vamos explorar como implementar e configurar documentação de API usando OpenAPI (anteriormente conhecido como Swagger) com Quarkus.

## O que é OpenAPI e Swagger?

**OpenAPI** é uma especificação para arquivos de definição de interface para APIs RESTful. Ela define um formato padrão para descrever, produzir, consumir e visualizar APIs. A especificação OpenAPI é agnóstica de linguagem de programação e permite que humanos e computadores descubram e compreendam os recursos de um serviço sem necessidade de acesso ao código-fonte.

**Swagger** é um conjunto de ferramentas que implementam a especificação OpenAPI. Originalmente, Swagger era o nome da especificação, mas após a versão 2.0, ela foi doada para a Open API Initiative e renomeada para OpenAPI.

**Swagger UI** é uma ferramenta que gera uma interface interativa a partir da especificação OpenAPI, permitindo que usuários visualizem e interajam com os endpoints da API diretamente no navegador.

## Implementando OpenAPI e Swagger UI no Quarkus

### Adicionando as Extensões

O primeiro passo é adicionar as extensões necessárias ao seu projeto:

```bash
# Maven
./mvnw quarkus:add-extension -Dextensions="smallrye-openapi"

# Gradle
./gradlew addExtension --extensions="smallrye-openapi"
```

Ou adicione manualmente ao `pom.xml`:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-openapi</artifactId>
</dependency>
```

### Configurações Básicas

Por padrão, o Quarkus gera automaticamente documentação OpenAPI baseada em seus endpoints REST. A documentação é disponibilizada nos seguintes endpoints:

- `/q/openapi`: Retorna a especificação OpenAPI no formato YAML
- `/q/openapi?format=json`: Retorna a especificação OpenAPI no formato JSON
- `/q/swagger-ui`: Interface de usuário do Swagger para interagir com sua API

### Personalizando Informações Gerais da API

Você pode personalizar as informações gerais da documentação de sua API através do arquivo `application.properties`:

```text
# Título da API
quarkus.smallrye-openapi.info-title=API de Produtos
# Versão da API
quarkus.smallrye-openapi.info-version=1.0.0
# Descrição da API
quarkus.smallrye-openapi.info-description=API para gerenciamento de produtos e estoque
# Termos de serviço
quarkus.smallrye-openapi.info-terms-of-service=https://exemplo.com/termos
# Informações de contato
quarkus.smallrye-openapi.info-contact-email=api@exemplo.com
quarkus.smallrye-openapi.info-contact-name=API Support
quarkus.smallrye-openapi.info-contact-url=https://exemplo.com/suporte
# Informações de licença
quarkus.smallrye-openapi.info-license-name=Apache 2.0
quarkus.smallrye-openapi.info-license-url=https://www.apache.org/licenses/LICENSE-2.0.html
# Tags globais
quarkus.smallrye-openapi.tags=Produtos,Estoque,Administração

# Customizando a URL para o servidor da API
quarkus.smallrye-openapi.servers=https://api.prod.exemplo.com,https://api.stage.exemplo.com
```

Além de configurar via `application.properties`, você também pode definir essas informações programaticamente usando a classe `OpenAPIDefinition`:

```java
package org.acme.config;

import javax.ws.rs.core.Application;
import org.eclipse.microprofile.openapi.annotations.OpenAPIDefinition;
import org.eclipse.microprofile.openapi.annotations.info.Contact;
import org.eclipse.microprofile.openapi.annotations.info.Info;
import org.eclipse.microprofile.openapi.annotations.info.License;
import org.eclipse.microprofile.openapi.annotations.tags.Tag;

@OpenAPIDefinition(
    info = @Info(
        title = "API de Produtos",
        version = "1.0.0",
        description = "API para gerenciamento de produtos e estoque",
        termsOfService = "https://exemplo.com/termos",
        contact = @Contact(
            name = "API Support",
            url = "https://exemplo.com/suporte",
            email = "api@exemplo.com"
        ),
        license = @License(
            name = "Apache 2.0",
            url = "https://www.apache.org/licenses/LICENSE-2.0.html"
        )
    ),
    tags = {
        @Tag(name = "Produtos", description = "Operações relacionadas a produtos"),
        @Tag(name = "Estoque", description = "Operações de gerenciamento de estoque"),
        @Tag(name = "Administração", description = "Operações administrativas")
    }
)
public class OpenAPIConfig extends Application {
    // Vazio, apenas para configuração
}
```

### Customizando o Swagger UI

Você pode personalizar a aparência e comportamento do Swagger UI através de configurações no `application.properties`:

```text
# Sempre incluir o Swagger UI, mesmo em produção
quarkus.swagger-ui.always-include=true

# Customizando o caminho do Swagger UI
quarkus.swagger-ui.path=/documentacao

# Tema do Swagger UI
quarkus.swagger-ui.theme=material

# Define se a UI deve iniciar expandida
quarkus.swagger-ui.doc-expansion=none

# Título da página HTML do Swagger UI
quarkus.swagger-ui.title=Documentação da API de Produtos

# URL para o favicon da página
quarkus.swagger-ui.favicon-href=https://quarkus.io/favicon.ico
```

## Documentando Endpoints e Operações

O Quarkus gera automaticamente documentação para os endpoints REST, mas você pode enriquecer essa documentação usando anotações do OpenAPI.

### Documentando Classes de Recursos (Controllers)

Use a anotação `@Tag` para organizar seus endpoints em grupos:

```java
package org.acme.resource;

import org.eclipse.microprofile.openapi.annotations.tags.Tag;
// imports...

@Path("/produtos")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@Tag(name = "Produtos", description = "Operações relacionadas a produtos")
public class ProdutoResource {
    // métodos...
}
```

### Documentando Operações (Métodos)

Use a anotação `@Operation` para fornecer informações detalhadas sobre cada operação:

```java
@GET
@Path("/{id}")
@Operation(
    summary = "Buscar produto por ID",
    description = "Recupera um produto específico pelo seu identificador único",
    operationId = "getProdutoById"
)
@APIResponses(value = {
    @APIResponse(
        responseCode = "200",
        description = "Produto encontrado",
        content = @Content(
            mediaType = "application/json",
            schema = @Schema(implementation = Produto.class)
        )
    ),
    @APIResponse(
        responseCode = "404",
        description = "Produto não encontrado"
    )
})
public Response buscarPorId(
    @Parameter(description = "ID do produto", required = true)
    @PathParam("id") Long id
) {
    // implementação...
}
```

### Documentando Parâmetros

Use a anotação `@Parameter` para descrever parâmetros:

```java
@GET
@Path("/busca")
@Operation(summary = "Buscar produtos por critérios")
public List<Produto> buscarProdutos(
    @Parameter(description = "Nome do produto (busca parcial)")
    @QueryParam("nome") String nome,
    
    @Parameter(description = "Preço mínimo")
    @QueryParam("precoMin") Double precoMin,
    
    @Parameter(description = "Preço máximo")
    @QueryParam("precoMax") Double precoMax,
    
    @Parameter(description = "Página da paginação (começa em 0)", schema = @Schema(defaultValue = "0"))
    @QueryParam("pagina") @DefaultValue("0") Integer pagina,
    
    @Parameter(description = "Tamanho da página", schema = @Schema(defaultValue = "10"))
    @QueryParam("tamanho") @DefaultValue("10") Integer tamanho
) {
    // implementação...
}
```

### Documentando Entidades e Esquemas

Use anotações do OpenAPI para documentar as entidades:

```java
package org.acme.entity;

import org.eclipse.microprofile.openapi.annotations.media.Schema;
// imports...

@Entity
@Schema(description = "Representa um produto no catálogo")
public class Produto extends PanacheEntity {
    
    @Schema(description = "Nome do produto", example = "Notebook Dell XPS 13", required = true)
    @NotBlank(message = "O nome do produto é obrigatório")
    public String nome;
    
    @Schema(description = "Descrição detalhada do produto", example = "Notebook Dell XPS 13 com 16GB RAM, 512GB SSD")
    public String descricao;
    
    @Schema(description = "Preço em reais", example = "5499.99", required = true, minimum = "0.01")
    @NotNull(message = "O preço do produto é obrigatório")
    public Double preco;
    
    @Schema(description = "Quantidade disponível em estoque", example = "15", defaultValue = "0", minimum = "0")
    public Integer quantidadeEstoque = 0;
    
    @Schema(description = "Código SKU do produto", example = "NBDELL-XPS13-16-512", pattern = "^[A-Z0-9]{5,20}$")
    public String codigo;
    
    @Schema(description = "Data de cadastro do produto", example = "2023-05-28T14:30:00", readOnly = true)
    public java.time.LocalDateTime dataCriacao = java.time.LocalDateTime.now();
}
```

## Documentando Códigos de Status e Respostas

Você pode documentar os códigos de resposta HTTP e exemplos:

```java
@POST
@Operation(summary = "Criar novo produto")
@APIResponses(value = {
    @APIResponse(
        responseCode = "201",
        description = "Produto criado com sucesso",
        content = @Content(
            mediaType = "application/json",
            schema = @Schema(implementation = Produto.class),
            examples = {
                @ExampleObject(
                    name = "notebook",
                    summary = "Exemplo de notebook",
                    value = "{\n" +
                            "  \"id\": 1,\n" +
                            "  \"nome\": \"Notebook Dell XPS\",\n" +
                            "  \"preco\": 5499.99,\n" +
                            "  \"quantidadeEstoque\": 10\n" +
                            "}"
                )
            }
        )
    ),
    @APIResponse(
        responseCode = "400",
        description = "Dados de entrada inválidos",
        content = @Content(
            mediaType = "application/json",
            schema = @Schema(implementation = ErroValidacao.class)
        )
    ),
    @APIResponse(
        responseCode = "500",
        description = "Erro interno do servidor"
    )
})
public Response criar(
    @RequestBody(
        description = "Produto a ser criado", 
        required = true,
        content = @Content(
            schema = @Schema(implementation = Produto.class)
        )
    )
    Produto produto
) {
    // implementação...
}
```

## Agrupando Endpoints em Tags

Você pode agrupar endpoints relacionados usando tags:

```java
@Tag(name = "Produtos", description = "Gerenciamento de produtos")
@Path("/produtos")
public class ProdutoResource {
    // endpoints de produtos
}

@Tag(name = "Categorias", description = "Gerenciamento de categorias")
@Path("/categorias")
public class CategoriaResource {
    // endpoints de categorias
}

@Tag(name = "Relatórios", description = "Relatórios e estatísticas")
@Path("/relatorios")
public class RelatorioResource {
    // endpoints de relatórios
}
```

## Ocultando Endpoints da Documentação

Há situações em que você pode querer ocultar certos endpoints da documentação:

```java
@GET
@Path("/interno")
@Hidden // Oculta este endpoint da documentação
public String endpointInterno() {
    // implementação...
}
```

Para ocultar classes inteiras:

```java
@Path("/admin")
@Hidden
public class AdminResource {
    // todos os métodos serão ocultados
}
```

## Documentação OpenAPI Personalizada via Arquivo YAML/JSON

Além de usar anotações, você pode fornecer documentação personalizada através de um arquivo estático. Crie um arquivo `src/main/resources/META-INF/openapi.yaml` ou `openapi.json`:

```yaml
openapi: 3.0.3
info:
  title: API de Produtos
  version: 1.0.0
  description: |-
    # API para gerenciamento de produtos
    
    Esta API oferece funcionalidades para:
    * Gerenciamento de produtos
    * Controle de estoque
    * Cadastro de categorias
    
  contact:
    name: API Support Team
    url: https://exemplo.com/support
    email: support@exemplo.com
paths:
  /produtos:
    get:
      summary: Listar produtos
      # ...mais detalhes
```

## Integrando Documentação OpenAPI com Anotações de Validação

As anotações de validação do Bean Validation são automaticamente integradas à documentação OpenAPI no Quarkus. Por exemplo:

```java
@NotBlank(message = "O nome é obrigatório")
@Size(min = 3, max = 100, message = "O nome deve ter entre 3 e 100 caracteres")
public String nome;
```

Esta validação será refletida na documentação OpenAPI como restrições no schema do campo `nome`.

## Personalizando o Modelo de Dados na Documentação

Às vezes, você não quer expor toda a entidade em sua documentação. Use `@Schema` para personalizar:

```java
@Schema(name = "ProdutoDTO", description = "Representação pública de um produto")
public class Produto extends PanacheEntity {
    // campos...
    
    @Schema(hidden = true)
    public String campoInterno;
    
    @Schema(writeOnly = true)
    public String senha;
    
    @Schema(readOnly = true)
    public LocalDateTime dataCriacao;
}
```

## Gerando Documentação em Formato Estático

Para gerar arquivos estáticos de documentação OpenAPI:

```bash
# Maven
./mvnw quarkus:generate-code

# Gradle
./gradlew quarkusGenerateCode
```

Isso gera arquivos no diretório `target/generated-resources/openapi.*`.

## Compatibilidade com Ferramentas de Cliente

A documentação OpenAPI gerada pelo Quarkus pode ser usada com ferramentas como:

- **OpenAPI Generator**: para gerar clientes em várias linguagens
- **Postman**: importando a especificação OpenAPI
- **ReDoc**: como alternativa ao Swagger UI para documentação mais limpa

## Melhores Práticas

1. **Seja detalhista**: Forneça descrições claras, exemplos relevantes e documente todos os possíveis códigos de resposta.

2. **Mantenha consistência**: Use um padrão de documentação consistente em todos os endpoints.

3. **Agrupe com tags**: Organize endpoints relacionados usando tags para facilitar a navegação.

4. **Inclua exemplos**: Forneça exemplos de requisição e resposta para facilitar o entendimento.

5. **Versione sua API**: Inclua informação de versão na documentação e nos endpoints.

6. **Documente erros**: Inclua detalhes sobre possíveis erros e como interpretá-los.

7. **Revise regularmente**: Mantenha sua documentação atualizada conforme sua API evolui.

## Conclusão

O Quarkus, através da integração com MicroProfile OpenAPI e Swagger UI, oferece uma maneira poderosa e flexível de documentar suas APIs RESTful. Uma documentação bem elaborada não só facilita o consumo de sua API por outros desenvolvedores, mas também serve como uma referência valiosa para sua própria equipe durante o desenvolvimento e manutenção.

No próximo tópico, vamos explorar técnicas avançadas de desenvolvimento de APIs com Quarkus, incluindo tópicos como autenticação, autorização e testes.
