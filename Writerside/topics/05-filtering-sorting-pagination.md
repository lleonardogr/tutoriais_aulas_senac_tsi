# 5. Filtragem, Ordenação e Paginação com Query Parameters

Neste tópico, exploraremos como implementar recursos avançados de consulta em APIs RESTful utilizando query parameters para filtragem, ordenação e paginação de dados. Esses recursos são essenciais para criar APIs eficientes e escaláveis.

## Introdução aos Query Parameters

Query parameters são parâmetros opcionais passados na URL após o símbolo `?`, separados por `&`. Eles são ideais para funcionalidades como:

- **Filtragem**: Filtrar resultados com base em critérios específicos
- **Ordenação**: Definir a ordem dos resultados retornados
- **Paginação**: Dividir grandes conjuntos de dados em páginas menores
- **Busca**: Implementar funcionalidades de pesquisa textual

### Exemplo de URL com Query Parameters

```http
GET /api/produtos?categoria=eletronicos&preco_min=100&preco_max=500&ordenar=preco&ordem=asc&pagina=1&tamanho=10
```

## Implementando Filtragem

A filtragem permite que os clientes solicitem apenas os dados que atendem a critérios específicos.

### Exemplo Prático: Sistema de Produtos

Vamos criar um endpoint para listar produtos com diferentes opções de filtragem:

```java
@Path("/produtos")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class ProdutoResource {

    @GET
    @Operation(summary = "Listar produtos com filtros", 
               description = "Recupera produtos aplicando filtros opcionais")
    public Response listarProdutos(
            @QueryParam("categoria") String categoria,
            @QueryParam("nome") String nome,
            @QueryParam("preco_min") BigDecimal precoMinimo,
            @QueryParam("preco_max") BigDecimal precoMaximo,
            @QueryParam("ativo") Boolean ativo,
            @QueryParam("ordenar") @DefaultValue("id") String ordenarPor,
            @QueryParam("ordem") @DefaultValue("asc") String ordem,
            @QueryParam("pagina") @DefaultValue("1") @Min(1) int pagina,
            @QueryParam("tamanho") @DefaultValue("10") @Min(1) @Max(100) int tamanho) {

        // Construir query dinâmica
        StringBuilder query = new StringBuilder("SELECT p FROM Produto p WHERE 1=1");
        Map<String, Object> parametros = new HashMap<>();

        // Aplicar filtros condicionalmente
        if (categoria != null && !categoria.trim().isEmpty()) {
            query.append(" AND LOWER(p.categoria) = LOWER(:categoria)");
            parametros.put("categoria", categoria);
        }

        if (nome != null && !nome.trim().isEmpty()) {
            query.append(" AND LOWER(p.nome) LIKE LOWER(:nome)");
            parametros.put("nome", "%" + nome + "%");
        }

        if (precoMinimo != null) {
            query.append(" AND p.preco >= :precoMinimo");
            parametros.put("precoMinimo", precoMinimo);
        }

        if (precoMaximo != null) {
            query.append(" AND p.preco <= :precoMaximo");
            parametros.put("precoMaximo", precoMaximo);
        }

        if (ativo != null) {
            query.append(" AND p.ativo = :ativo");
            parametros.put("ativo", ativo);
        }

        // Aplicar ordenação
        query.append(" ORDER BY p.").append(sanitizarCampoOrdenacao(ordenarPor))
             .append(" ").append(sanitizarOrdem(ordem));

        // Executar query com paginação
        PanacheQuery<Produto> panacheQuery = Produto.find(query.toString(), parametros);
        
        // Aplicar paginação
        List<Produto> produtos = panacheQuery.page(pagina - 1, tamanho).list();
        long totalItens = panacheQuery.count();
        int totalPaginas = (int) Math.ceil((double) totalItens / tamanho);

        // Criar resposta com metadados de paginação
        Map<String, Object> resposta = new HashMap<>();
        resposta.put("produtos", produtos);
        resposta.put("pagina_atual", pagina);
        resposta.put("tamanho_pagina", tamanho);
        resposta.put("total_itens", totalItens);
        resposta.put("total_paginas", totalPaginas);
        resposta.put("tem_proxima", pagina < totalPaginas);
        resposta.put("tem_anterior", pagina > 1);

        return Response.ok(resposta).build();
    }

    private String sanitizarCampoOrdenacao(String campo) {
        // Lista de campos permitidos para ordenação
        Set<String> camposPermitidos = Set.of("id", "nome", "preco", "categoria", "criadoEm");
        return camposPermitidos.contains(campo) ? campo : "id";
    }

    private String sanitizarOrdem(String ordem) {
        return "desc".equalsIgnoreCase(ordem) ? "DESC" : "ASC";
    }
}
```

## Implementando com Panache Active Record

Uma abordagem mais elegante usando o padrão Active Record do Panache:

```java
@Entity
public class Produto extends PanacheEntity {
    
    public String nome;
    public String categoria;
    public BigDecimal preco;
    public Boolean ativo;
    
    @CreationTimestamp
    public LocalDateTime criadoEm;

    // Método personalizado para busca com filtros
    public static PanacheQuery<Produto> buscarComFiltros(String categoria, 
                                                         String nome, 
                                                         BigDecimal precoMin, 
                                                         BigDecimal precoMax, 
                                                         Boolean ativo) {
        StringBuilder query = new StringBuilder();
        Map<String, Object> parametros = new HashMap<>();
        List<String> condicoes = new ArrayList<>();

        if (categoria != null && !categoria.trim().isEmpty()) {
            condicoes.add("LOWER(categoria) = LOWER(:categoria)");
            parametros.put("categoria", categoria);
        }

        if (nome != null && !nome.trim().isEmpty()) {
            condicoes.add("LOWER(nome) LIKE LOWER(:nome)");
            parametros.put("nome", "%" + nome + "%");
        }

        if (precoMin != null) {
            condicoes.add("preco >= :precoMin");
            parametros.put("precoMin", precoMin);
        }

        if (precoMax != null) {
            condicoes.add("preco <= :precoMax");
            parametros.put("precoMax", precoMax);
        }

        if (ativo != null) {
            condicoes.add("ativo = :ativo");
            parametros.put("ativo", ativo);
        }

        if (condicoes.isEmpty()) {
            return findAll();
        }

        String queryString = String.join(" AND ", condicoes);
        return find(queryString, parametros);
    }

    // Método para busca por texto (full-text search simulado)
    public static PanacheQuery<Produto> buscarPorTexto(String texto) {
        return find("LOWER(nome) LIKE LOWER(?1) OR LOWER(categoria) LIKE LOWER(?1)", 
                   "%" + texto + "%");
    }
}
```

## Resource Simplificado

```java
@Path("/produtos")
@Produces(MediaType.APPLICATION_JSON)
public class ProdutoResource {

    @GET
    public Response listarProdutos(
            @QueryParam("categoria") String categoria,
            @QueryParam("nome") String nome,
            @QueryParam("preco_min") BigDecimal precoMin,
            @QueryParam("preco_max") BigDecimal precoMax,
            @QueryParam("ativo") Boolean ativo,
            @QueryParam("busca") String textoBusca,
            @QueryParam("ordenar") @DefaultValue("id") String ordenarPor,
            @QueryParam("ordem") @DefaultValue("asc") String ordem,
            @QueryParam("pagina") @DefaultValue("1") @Min(1) int pagina,
            @QueryParam("tamanho") @DefaultValue("10") @Min(1) @Max(100) int tamanho) {

        PanacheQuery<Produto> query;

        // Priorizar busca por texto se fornecida
        if (textoBusca != null && !textoBusca.trim().isEmpty()) {
            query = Produto.buscarPorTexto(textoBusca);
        } else {
            query = Produto.buscarComFiltros(categoria, nome, precoMin, precoMax, ativo);
        }

        // Aplicar ordenação
        String ordenacao = sanitizarCampoOrdenacao(ordenarPor) + " " + sanitizarOrdem(ordem);
        query = query.sort(ordenacao);

        // Aplicar paginação
        List<Produto> produtos = query.page(pagina - 1, tamanho).list();
        long totalItens = query.count();

        // Criar resposta paginada
        RespostaPaginada<Produto> resposta = new RespostaPaginada<>(
            produtos, pagina, tamanho, totalItens
        );

        return Response.ok(resposta).build();
    }

    // Métodos de sanitização...
}
```

## Classe de Resposta Paginada

Criar uma classe utilitária para padronizar respostas paginadas:

```java
public class RespostaPaginada<T> {
    public List<T> dados;
    public int paginaAtual;
    public int tamanhoPagina;
    public long totalItens;
    public int totalPaginas;
    public boolean temProxima;
    public boolean temAnterior;
    public String proximaUrl;
    public String anteriorUrl;

    public RespostaPaginada(List<T> dados, int pagina, int tamanho, long total) {
        this.dados = dados;
        this.paginaAtual = pagina;
        this.tamanhoPagina = tamanho;
        this.totalItens = total;
        this.totalPaginas = (int) Math.ceil((double) total / tamanho);
        this.temProxima = pagina < totalPaginas;
        this.temAnterior = pagina > 1;
    }

    // Método para adicionar URLs de navegação
    public RespostaPaginada<T> comUrls(String baseUrl, Map<String, String> parametros) {
        if (temProxima) {
            proximaUrl = construirUrl(baseUrl, parametros, paginaAtual + 1);
        }
        if (temAnterior) {
            anteriorUrl = construirUrl(baseUrl, parametros, paginaAtual - 1);
        }
        return this;
    }

    private String construirUrl(String baseUrl, Map<String, String> params, int pagina) {
        StringBuilder url = new StringBuilder(baseUrl + "?pagina=" + pagina);
        params.forEach((key, value) -> {
            if (value != null && !value.trim().isEmpty()) {
                url.append("&").append(key).append("=").append(value);
            }
        });
        return url.toString();
    }
}
```

## Implementando Busca Avançada

Para implementar funcionalidades de busca mais sofisticadas:

```java
@GET
@Path("/busca")
@Operation(summary = "Busca avançada de produtos")
public Response buscaAvancada(
        @QueryParam("q") String query,
        @QueryParam("campos") @DefaultValue("nome,categoria") String campos,
        @QueryParam("operador") @DefaultValue("AND") String operador,
        @QueryParam("pagina") @DefaultValue("1") int pagina,
        @QueryParam("tamanho") @DefaultValue("10") int tamanho) {

    if (query == null || query.trim().isEmpty()) {
        return Response.status(Response.Status.BAD_REQUEST)
                      .entity("Parâmetro 'q' é obrigatório para busca")
                      .build();
    }

    List<String> camposBusca = Arrays.asList(campos.split(","));
    List<String> condicoes = new ArrayList<>();
    
    String conectivo = "OR".equalsIgnoreCase(operador) ? "OR" : "AND";

    for (String campo : camposBusca) {
        if (Set.of("nome", "categoria", "descricao").contains(campo.trim())) {
            condicoes.add("LOWER(" + campo.trim() + ") LIKE LOWER(:query)");
        }
    }

    String queryString = String.join(" " + conectivo + " ", condicoes);
    Map<String, Object> parametros = Map.of("query", "%" + query + "%");

    PanacheQuery<Produto> panacheQuery = Produto.find(queryString, parametros);
    
    List<Produto> produtos = panacheQuery.page(pagina - 1, tamanho).list();
    long total = panacheQuery.count();

    RespostaPaginada<Produto> resposta = new RespostaPaginada<>(produtos, pagina, tamanho, total);

    return Response.ok(resposta).build();
}
```

## Boas Práticas e Considerações

### 1. Validação de Parâmetros

```java
// Usar Bean Validation nos query parameters
public Response listar(
    @QueryParam("pagina") @Min(1) @DefaultValue("1") int pagina,
    @QueryParam("tamanho") @Min(1) @Max(100) @DefaultValue("10") int tamanho) {
    // ...
}
```

### 2. Limitações de Performance

```java
// Configurar limites máximos
private static final int MAX_PAGE_SIZE = 100;
private static final int DEFAULT_PAGE_SIZE = 10;

if (tamanho > MAX_PAGE_SIZE) {
    tamanho = MAX_PAGE_SIZE;
}
```

### 3. Cache de Consultas Frequentes

```java
@GET
@Path("/populares")
@CacheResult(cacheName = "produtos-populares")
public List<Produto> produtosPopulares(@QueryParam("limite") @DefaultValue("10") int limite) {
    return Produto.find("ORDER BY vendas DESC").page(0, limite).list();
}
```

### 4. Documentação OpenAPI

```java
@GET
@Operation(
    summary = "Listar produtos com filtros e paginação",
    description = "Endpoint para listar produtos com múltiplas opções de filtragem, ordenação e paginação"
)
@APIResponses({
    @APIResponse(
        responseCode = "200",
        description = "Lista de produtos retornada com sucesso",
        content = @Content(
            mediaType = MediaType.APPLICATION_JSON,
            schema = @Schema(implementation = RespostaPaginada.class)
        )
    )
})
public Response listarProdutos(
    @Parameter(description = "Filtrar por categoria", example = "eletronicos")
    @QueryParam("categoria") String categoria,
    
    @Parameter(description = "Buscar por nome (busca parcial)", example = "smartphone")
    @QueryParam("nome") String nome,
    
    @Parameter(description = "Preço mínimo", example = "100.00")
    @QueryParam("preco_min") BigDecimal precoMin,
    
    @Parameter(description = "Campo para ordenação", example = "preco")
    @QueryParam("ordenar") @DefaultValue("id") String ordenarPor,
    
    @Parameter(description = "Direção da ordenação: asc ou desc", example = "desc")
    @QueryParam("ordem") @DefaultValue("asc") String ordem,
    
    @Parameter(description = "Número da página (inicia em 1)", example = "1")
    @QueryParam("pagina") @DefaultValue("1") int pagina) {
    // ...
}
```

## Exemplos de Uso

### Filtrar produtos por categoria

```bash
curl "http://localhost:8080/produtos?categoria=eletronicos"
```

### Buscar produtos com preço entre R$ 100 e R$ 500

```bash
curl "http://localhost:8080/produtos?preco_min=100&preco_max=500"
```

### Ordenar por preço decrescente

```bash
curl "http://localhost:8080/produtos?ordenar=preco&ordem=desc"
```

### Paginação (página 2, 5 itens por página)

```bash
curl "http://localhost:8080/produtos?pagina=2&tamanho=5"
```

### Combinando múltiplos filtros

```bash
curl "http://localhost:8080/produtos?categoria=eletronicos&preco_max=1000&ordenar=nome&ordem=asc&pagina=1&tamanho=20"
```

## Conclusão

A implementação de filtragem, ordenação e paginação através de query parameters é fundamental para criar APIs RESTful eficientes e escaláveis. Essas funcionalidades permitem que os clientes obtenham apenas os dados necessários, melhoram a performance da aplicação e proporcionam uma melhor experiência do usuário.

No próximo tópico, exploraremos como documentar essas funcionalidades avançadas utilizando Swagger/OpenAPI, garantindo que nossa API seja não apenas funcional, mas também bem documentada para outros desenvolvedores.
