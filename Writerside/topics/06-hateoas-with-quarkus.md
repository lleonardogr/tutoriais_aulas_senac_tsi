# 6. HATEOAS com Quarkus

HATEOAS (Hypermedia as the Engine of Application State) é um dos princípios fundamentais da arquitetura REST que permite que APIs sejam verdadeiramente auto-descritivas. Neste tutorial, aprenderemos como implementar HATEOAS em APIs Quarkus, criando respostas que incluem links de navegação e ações disponíveis.

## O que é HATEOAS?

HATEOAS é um princípio da arquitetura REST que determina que:
- As respostas da API devem incluir links para ações relacionadas
- O cliente não precisa conhecer previamente todas as URLs da API
- A API guia o cliente através dos estados possíveis
- Melhora a evolução e manutenibilidade da API

### Exemplo Conceitual

Sem HATEOAS:
```json
{
  "id": 1,
  "nome": "João Silva",
  "email": "joao@email.com"
}
```

Com HATEOAS:
```json
{
  "id": 1,
  "nome": "João Silva", 
  "email": "joao@email.com",
  "_links": {
    "self": {
      "href": "/usuarios/1"
    },
    "editar": {
      "href": "/usuarios/1",
      "method": "PUT"
    },
    "deletar": {
      "href": "/usuarios/1",
      "method": "DELETE"
    },
    "pedidos": {
      "href": "/usuarios/1/pedidos"
    }
  }
}
```

## Configurando o Projeto

### Dependências Necessárias

Adicione ao seu `pom.xml`:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-resteasy-reactive-jackson</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-hibernate-orm-panache</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-jdbc-h2</artifactId>
</dependency>
```

## Implementação Básica de HATEOAS

### 1. Criando a Classe de Links

Primeiro, vamos criar uma estrutura para representar links HATEOAS:

```java
package org.acme.model;

import com.fasterxml.jackson.annotation.JsonProperty;
import java.util.HashMap;
import java.util.Map;

public class HateoasLink {
    private String href;
    private String method;
    private String rel;

    public HateoasLink() {}

    public HateoasLink(String href, String method, String rel) {
        this.href = href;
        this.method = method;
        this.rel = rel;
    }

    // Getters e Setters
    public String getHref() { return href; }
    public void setHref(String href) { this.href = href; }
    
    public String getMethod() { return method; }
    public void setMethod(String method) { this.method = method; }
    
    public String getRel() { return rel; }
    public void setRel(String rel) { this.rel = rel; }
}
```

### 2. Criando uma Classe Base para Recursos HATEOAS

```java
package org.acme.model;

import com.fasterxml.jackson.annotation.JsonProperty;
import java.util.HashMap;
import java.util.Map;

public abstract class HateoasResource {
    
    @JsonProperty("_links")
    private Map<String, HateoasLink> links = new HashMap<>();

    public void addLink(String rel, String href, String method) {
        links.put(rel, new HateoasLink(href, method, rel));
    }

    public void addLink(String rel, String href) {
        addLink(rel, href, "GET");
    }

    public Map<String, HateoasLink> getLinks() {
        return links;
    }

    public void setLinks(Map<String, HateoasLink> links) {
        this.links = links;
    }
}
```

### 3. Atualizando a Entidade Usuario

```java
package org.acme.model;

import io.quarkus.hibernate.orm.panache.PanacheEntity;
import javax.persistence.Entity;
import javax.persistence.Table;
import javax.validation.constraints.Email;
import javax.validation.constraints.NotBlank;

@Entity
@Table(name = "usuarios")
public class Usuario extends PanacheEntity {
    
    @NotBlank(message = "Nome é obrigatório")
    public String nome;
    
    @Email(message = "Email deve ter formato válido")
    @NotBlank(message = "Email é obrigatório")
    public String email;

    public String telefone;
    
    // Construtor padrão
    public Usuario() {}
    
    public Usuario(String nome, String email, String telefone) {
        this.nome = nome;
        this.email = email;
        this.telefone = telefone;
    }
}
```

### 4. Criando o DTO com HATEOAS

```java
package org.acme.dto;

import org.acme.model.HateoasResource;
import org.acme.model.Usuario;

public class UsuarioDTO extends HateoasResource {
    
    public Long id;
    public String nome;
    public String email;
    public String telefone;

    public UsuarioDTO() {}

    public UsuarioDTO(Usuario usuario) {
        this.id = usuario.id;
        this.nome = usuario.nome;
        this.email = usuario.email;
        this.telefone = usuario.telefone;
    }

    public static UsuarioDTO fromEntity(Usuario usuario) {
        return new UsuarioDTO(usuario);
    }
}
```

### 5. Criando Utility para Construção de Links

```java
package org.acme.util;

import org.acme.dto.UsuarioDTO;
import javax.ws.rs.core.UriInfo;

public class HateoasBuilder {

    public static void addUsuarioLinks(UsuarioDTO dto, UriInfo uriInfo) {
        String baseUri = uriInfo.getBaseUriBuilder()
                .path("usuarios")
                .toString();

        // Link para si mesmo
        dto.addLink("self", baseUri + "/" + dto.id);
        
        // Link para editar
        dto.addLink("editar", baseUri + "/" + dto.id, "PUT");
        
        // Link para deletar
        dto.addLink("deletar", baseUri + "/" + dto.id, "DELETE");
        
        // Link para listar todos
        dto.addLink("todos", baseUri);
        
        // Links condicionais baseados no estado
        if (dto.telefone != null && !dto.telefone.isEmpty()) {
            dto.addLink("contato", baseUri + "/" + dto.id + "/contato");
        }
    }

    public static void addCollectionLinks(UriInfo uriInfo, Object response) {
        // Implementar links para coleções (próximo tutorial)
    }
}
```

## Implementando o Resource com HATEOAS

### UsuarioResource Atualizado

```java
package org.acme.resource;

import org.acme.dto.UsuarioDTO;
import org.acme.model.Usuario;
import org.acme.util.HateoasBuilder;

import javax.transaction.Transactional;
import javax.validation.Valid;
import javax.ws.rs.*;
import javax.ws.rs.core.*;
import java.net.URI;
import java.util.List;
import java.util.stream.Collectors;

@Path("/usuarios")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class UsuarioResource {

    @Context
    UriInfo uriInfo;

    @GET
    public Response listarTodos() {
        List<UsuarioDTO> usuarios = Usuario.<Usuario>listAll()
                .stream()
                .map(usuario -> {
                    UsuarioDTO dto = UsuarioDTO.fromEntity(usuario);
                    HateoasBuilder.addUsuarioLinks(dto, uriInfo);
                    return dto;
                })
                .collect(Collectors.toList());

        return Response.ok(usuarios).build();
    }

    @GET
    @Path("/{id}")
    public Response buscarPorId(@PathParam("id") Long id) {
        Usuario usuario = Usuario.findById(id);
        
        if (usuario == null) {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity("Usuário não encontrado")
                    .build();
        }

        UsuarioDTO dto = UsuarioDTO.fromEntity(usuario);
        HateoasBuilder.addUsuarioLinks(dto, uriInfo);

        return Response.ok(dto).build();
    }

    @POST
    @Transactional
    public Response criar(@Valid Usuario usuario) {
        usuario.persist();

        UsuarioDTO dto = UsuarioDTO.fromEntity(usuario);
        HateoasBuilder.addUsuarioLinks(dto, uriInfo);

        URI location = uriInfo.getAbsolutePathBuilder()
                .path(String.valueOf(usuario.id))
                .build();

        return Response.created(location).entity(dto).build();
    }

    @PUT
    @Path("/{id}")
    @Transactional
    public Response atualizar(@PathParam("id") Long id, @Valid Usuario usuarioAtualizado) {
        Usuario usuario = Usuario.findById(id);
        
        if (usuario == null) {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity("Usuário não encontrado")
                    .build();
        }

        usuario.nome = usuarioAtualizado.nome;
        usuario.email = usuarioAtualizado.email;
        usuario.telefone = usuarioAtualizado.telefone;

        UsuarioDTO dto = UsuarioDTO.fromEntity(usuario);
        HateoasBuilder.addUsuarioLinks(dto, uriInfo);

        return Response.ok(dto).build();
    }

    @DELETE
    @Path("/{id}")
    @Transactional
    public Response deletar(@PathParam("id") Long id) {
        Usuario usuario = Usuario.findById(id);
        
        if (usuario == null) {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity("Usuário não encontrado")
                    .build();
        }

        usuario.delete();
        
        // Retorna links úteis após a deleção
        return Response.noContent()
                .header("Link", "</usuarios>; rel=\"collection\"")
                .build();
    }

    @GET
    @Path("/{id}/contato")
    public Response obterContato(@PathParam("id") Long id) {
        Usuario usuario = Usuario.findById(id);
        
        if (usuario == null) {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity("Usuário não encontrado")
                    .build();
        }

        // Criar resposta específica para contato
        var contato = Map.of(
            "nome", usuario.nome,
            "email", usuario.email,
            "telefone", usuario.telefone != null ? usuario.telefone : "Não informado",
            "_links", Map.of(
                "usuario", Map.of("href", uriInfo.getBaseUriBuilder()
                    .path("usuarios").path(id.toString()).build().toString()),
                "editar", Map.of("href", uriInfo.getBaseUriBuilder()
                    .path("usuarios").path(id.toString()).build().toString(), "method", "PUT")
            )
        );

        return Response.ok(contato).build();
    }
}
```

## Testando a API com HATEOAS

### 1. Iniciando a Aplicação

```bash
./mvnw quarkus:dev
```

### 2. Testando com curl

**Criar um usuário:**
```bash
curl -X POST http://localhost:8080/usuarios \
  -H "Content-Type: application/json" \
  -d '{
    "nome": "Maria Silva",
    "email": "maria@email.com",
    "telefone": "(11) 99999-9999"
  }'
```

**Resposta esperada:**
```json
{
  "id": 1,
  "nome": "Maria Silva",
  "email": "maria@email.com",
  "telefone": "(11) 99999-9999",
  "_links": {
    "self": {
      "href": "http://localhost:8080/usuarios/1",
      "method": "GET",
      "rel": "self"
    },
    "editar": {
      "href": "http://localhost:8080/usuarios/1",
      "method": "PUT",
      "rel": "editar"
    },
    "deletar": {
      "href": "http://localhost:8080/usuarios/1",
      "method": "DELETE",
      "rel": "deletar"
    },
    "todos": {
      "href": "http://localhost:8080/usuarios",
      "method": "GET",
      "rel": "todos"
    },
    "contato": {
      "href": "http://localhost:8080/usuarios/1/contato",
      "method": "GET",
      "rel": "contato"
    }
  }
}
```

**Buscar usuário por ID:**
```bash
curl -X GET http://localhost:8080/usuarios/1
```

**Acessar informações de contato:**
```bash
curl -X GET http://localhost:8080/usuarios/1/contato
```

## HATEOAS Avançado

### 1. Links Condicionais e Estados

```java
public static void addUsuarioLinksAvancado(UsuarioDTO dto, UriInfo uriInfo, String userRole) {
    String baseUri = uriInfo.getBaseUriBuilder()
            .path("usuarios")
            .toString();

    // Links básicos sempre presentes
    dto.addLink("self", baseUri + "/" + dto.id);
    
    // Links condicionais baseados em permissões
    if ("ADMIN".equals(userRole)) {
        dto.addLink("editar", baseUri + "/" + dto.id, "PUT");
        dto.addLink("deletar", baseUri + "/" + dto.id, "DELETE");
    }
    
    // Links baseados no estado do recurso
    if (dto.telefone != null && !dto.telefone.isEmpty()) {
        dto.addLink("ligar", baseUri + "/" + dto.id + "/ligar", "POST");
    }
    
    if (dto.email != null && !dto.email.isEmpty()) {
        dto.addLink("enviar-email", baseUri + "/" + dto.id + "/email", "POST");
    }
}
```

### 2. Paginação com HATEOAS

```java
public class PaginatedResponse<T> extends HateoasResource {
    public List<T> content;
    public int page;
    public int size;
    public long totalElements;
    public int totalPages;
    
    public PaginatedResponse(List<T> content, int page, int size, long totalElements) {
        this.content = content;
        this.page = page;
        this.size = size;
        this.totalElements = totalElements;
        this.totalPages = (int) Math.ceil((double) totalElements / size);
    }
}

public static void addPaginationLinks(PaginatedResponse<?> response, UriInfo uriInfo) {
    String baseUri = uriInfo.getRequestUri().toString().split("\\?")[0];
    
    // Link para a página atual
    response.addLink("self", String.format("%s?page=%d&size=%d", 
        baseUri, response.page, response.size));
    
    // Link para primeira página
    if (response.page > 0) {
        response.addLink("first", String.format("%s?page=0&size=%d", 
            baseUri, response.size));
        response.addLink("prev", String.format("%s?page=%d&size=%d", 
            baseUri, response.page - 1, response.size));
    }
    
    // Link para última página
    if (response.page < response.totalPages - 1) {
        response.addLink("next", String.format("%s?page=%d&size=%d", 
            baseUri, response.page + 1, response.size));
        response.addLink("last", String.format("%s?page=%d&size=%d", 
            baseUri, response.totalPages - 1, response.size));
    }
}
```

## Boas Práticas para HATEOAS

### 1. Padrões de Nomenclatura

```java
// Use nomes descritivos para relações
dto.addLink("editar", href, "PUT");        // ✅ Bom
dto.addLink("edit", href, "PUT");          // ❌ Evitar inglês misturado

// Mantenha consistência
dto.addLink("self", href);                 // ✅ Padrão
dto.addLink("proprio", href);              // ❌ Inconsistente
```

### 2. Métodos HTTP Apropriados

```java
// Especifique o método HTTP para ações não-GET
dto.addLink("criar", href, "POST");
dto.addLink("atualizar", href, "PUT");
dto.addLink("deletar", href, "DELETE");
dto.addLink("patch", href, "PATCH");
```

### 3. Links Contextualmente Relevantes

```java
// Adicione apenas links que fazem sentido no contexto atual
if (usuario.isAtivo()) {
    dto.addLink("desativar", baseUri + "/" + dto.id + "/desativar", "POST");
} else {
    dto.addLink("ativar", baseUri + "/" + dto.id + "/ativar", "POST");
}
```

## Exercícios Práticos

### Exercício 1: Implementar HATEOAS para Produtos
Crie uma entidade `Produto` com campos `nome`, `preco`, `categoria` e implemente HATEOAS incluindo:
- Links básicos (self, editar, deletar)
- Link para produtos da mesma categoria
- Link condicional para desconto (se preço > 100)

### Exercício 2: HATEOAS para Relacionamentos
Expanda o exemplo para incluir uma entidade `Pedido` relacionada a `Usuario` e adicione:
- Links entre usuário e seus pedidos
- Links entre pedido e usuário proprietário
- Estados do pedido (pendente, processando, entregue)

### Exercício 3: API de Descoberta
Crie um endpoint `/` que retorne todos os recursos disponíveis na API com seus links principais.

## Resumo

Neste tutorial, aprendemos:

1. **Conceitos de HATEOAS**: Princípios e benefícios da hipermídia
2. **Implementação básica**: Estruturas para links e recursos
3. **Integração com Quarkus**: Uso de `UriInfo` e DTOs
4. **Links condicionais**: Baseados em estado e permissões
5. **Boas práticas**: Nomenclatura e contextualização

HATEOAS torna suas APIs mais autodescritivas e flexíveis, permitindo que clientes descubram funcionalidades dinamicamente e facilitando a evolução da API sem quebrar compatibilidade.

**Próximo:** No próximo tutorial, aprenderemos como documentar APIs com Swagger, incluindo a documentação dos links HATEOAS.