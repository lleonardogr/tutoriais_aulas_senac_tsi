# Projeto Final: Desenvolvimento de API com Quarkus

Este documento apresenta as diretrizes para o projeto final de avaliação do curso de Desenvolvimento de APIs com Quarkus. O projeto consiste em desenvolver uma API RESTful completa utilizando os conceitos e técnicas aprendidos ao longo do curso.

[Entregas](https://docs.google.com/spreadsheets/d/1TgiupynOKxU5VC3NO4aw3jPTwYkGGXrWc_H8Pxj4vHc/edit?usp=sharing)

## Objetivo

Desenvolver uma API RESTful usando Quarkus que represente um domínio de negócio à sua escolha, aplicando os conceitos de WebServices, REST, validação de dados e documentação de API com Swagger.

## Requisitos Técnicos - Parte 1

### 1. Estrutura do Projeto

- Utilizar o framework Quarkus em sua versão mais recente
- Implementar o projeto com Java 11 ou superior
- Utilizar o Maven ou Gradle como gerenciador de dependências
- Configurar o banco de dados H2 para persistência de dados
- Utilizar Hibernate+Panache para ORM

### 2. Entidades e Relacionamentos

- **Mínimo de 3 entidades** com relacionamentos entre si
- Pelo menos um relacionamento de cada tipo: One-to-One, One-to-Many e Many-to-Many
- Utilizar validações adequadas (Bean Validation) em cada entidade
- Implementar pelo menos um enum em uma das entidades

### 3. Endpoints e Operações

- **Mínimo de 5 endpoints REST para cada entidade** (total de pelo menos 15 endpoints)
- Implementar operações CRUD completas para cada entidade
- Incluir pelo menos 1 endpoints com consultas personalizadas por entidade
- Incluir HATEOAS em todos os endpoints necessários
- Utilizar códigos de status HTTP apropriados para cada operação

### 4. Documentação
- Documentar todos os endpoints usando anotações OpenAPI
- Incluir descrições detalhadas, exemplos e possíveis códigos de resposta
- Personalizar a interface do Swagger UI com informações do projeto
- Garantir que a documentação esteja completa e precisa

## Requisitos Técnicos - Parte 2

### 4. Recursos Avançados de API

Para esta fase do projeto, você deve implementar os seguintes recursos avançados na sua API:

#### 4.1. Idempotência

- Implementar idempotência em operações POST utilizando chaves idempotentes
- Garantir que a mesma operação não seja processada duas vezes acidentalmente
- Exemplo: implementar mecanismo para armazenar e verificar cabeçalhos de idempotência (`Idempotency-Key`)

#### 4.2. Autenticação com Chave de API

- Implementar um sistema de autenticação baseado em chaves de API
- Criar endpoint para geração e gerenciamento de chaves de API para usuários
- Proteger endpoints sensíveis para exigir uma chave de API válida
- Implementar diferentes níveis de acesso (opcional)

#### 4.3. Rate Limiting

- Configurar limitação de taxa de requisições por cliente/IP
- Implementar cabeçalhos de resposta informando limites e uso (`X-RateLimit-Limit`, `X-RateLimit-Remaining`)
- Retornar status HTTP 429 (Too Many Requests) quando o limite for excedido
- Configurar diferentes limites para diferentes endpoints ou tipo de usuário (opcional)

#### 4.4. CORS (Cross-Origin Resource Sharing)

- Configurar adequadamente a política de CORS para sua API
- Permitir acesso de origens específicas
- Configurar os métodos HTTP permitidos em requisições cross-origin
- Configurar cabeçalhos personalizados permitidos
- Implementar suporte para cookies em requisições cross-origin (opcional)

#### 4.5. Versionamento da API

- Implementar versionamento da API usando um dos seguintes métodos:
  - Versionamento via URL (ex: `/api/v1/recurso`)
  - Versionamento via cabeçalho personalizado (ex: `X-API-Version: 1`)
  - Versionamento via parâmetro de consulta (ex: `?version=1`)
  - Versionamento via content negotiation (ex: `Accept: application/vnd.empresa.api.v1+json`)
- Manter pelo menos duas versões de um endpoint para demonstrar a funcionalidade

### 5. Validações e Tratamento de Erros

- Validar todos os dados de entrada usando Bean Validation
- Implementar tratamento global de exceções
- Retornar códigos de status HTTP apropriados para cada situação
- Fornecer mensagens de erro claras e úteis

### 6. Documentação com Swagger/OpenAPI

- Documentar todos os endpoints usando anotações OpenAPI
- Incluir descrições detalhadas, exemplos e possíveis códigos de resposta
- Personalizar a interface do Swagger UI com informações do projeto
- Garantir que a documentação esteja completa e precisa

## Temas Sugeridos

Você tem liberdade para escolher qualquer domínio de negócio para sua API. Alguns exemplos:

1. **Sistema de Biblioteca**
   - Entidades: Livro, Autor, Empréstimo, Usuário
   - Funcionalidades: cadastro de livros, empréstimos, devoluções, reservas

2. **Gerenciamento de Eventos**
   - Entidades: Evento, Participante, Local, Palestrante
   - Funcionalidades: inscrição, check-in, programação, avaliações

3. **E-commerce**
   - Entidades: Produto, Categoria, Cliente, Pedido, ItemPedido
   - Funcionalidades: catálogo, carrinho, checkout, histórico de pedidos

4. **Rede Social**
   - Entidades: Usuário, Post, Comentário, Grupo
   - Funcionalidades: publicações, amizades, curtidas, compartilhamentos

5. **Sistema Hospitalar**
   - Entidades: Médico, Paciente, Consulta, Especialidade
   - Funcionalidades: agendamento, prontuário, receitas, exames

## Critérios de Avaliação

| Critério | Peso | Descrição |
|----------|------|-----------|
| **Arquitetura e Design** | 20% | Organização do código, separação de responsabilidades, uso de padrões de projeto |
| **Implementação de Entidades** | 15% | Modelagem correta, relacionamentos, validações adequadas |
| **Endpoints REST** | 20% | Implementação correta dos verbos HTTP, rotas bem definidas, parâmetros adequados |
| **Validações e Tratamento de Erros** | 15% | Robustez da API, validações adequadas, mensagens claras de erro |
| **Documentação Swagger** | 15% | Qualidade e completude da documentação OpenAPI |
| **Funcionalidade Geral** | 15% | Funcionamento correto da API como um todo |

## Entrega e Apresentação

### Material a Ser Entregue

1. **Código-fonte completo** em um repositório Git (GitHub, GitLab, etc.)
2. **Arquivo README.md** com:
   - Descrição do projeto
   - Instruções detalhadas para execução
   - Explicação das entidades e endpoints
   - Exemplos de uso (curl ou código)

3. **Coleção do Postman** com exemplos de todas as chamadas da API

### Apresentação

Cada estudante terá **15 minutos** para apresentar seu projeto, divididos da seguinte forma:

1. **Introdução** (2 min): Apresentação do domínio escolhido e justificativa
2. **Demonstração da API** (8 min): Mostrar o funcionamento dos principais endpoints usando Swagger UI e/ou Postman
3. **Código e Arquitetura** (3 min): Explicar as decisões de design mais importantes
4. **Perguntas e Respostas** (2 min): Esclarecer dúvidas dos avaliadores

## Exemplo de Implementação Parcial

Para ajudar a visualizar o que é esperado, segue um exemplo parcial de implementação para um Sistema de Biblioteca:

### Entidade Livro

```java
@Entity
@Schema(description = "Representa um livro no acervo da biblioteca")
public class Livro extends PanacheEntity {
    
    @NotBlank(message = "O título do livro é obrigatório")
    @Size(min = 2, max = 100, message = "O título deve ter entre 2 e 100 caracteres")
    @Schema(description = "Título do livro", example = "O Senhor dos Anéis")
    public String titulo;
    
    @NotBlank(message = "O ISBN é obrigatório")
    @Pattern(regexp = "^\\d{13}$", message = "O ISBN deve conter 13 dígitos numéricos")
    @Schema(description = "ISBN do livro (13 dígitos)", example = "9788533613379")
    public String isbn;
    
    @NotNull(message = "O ano de publicação é obrigatório")
    @Min(value = 1000, message = "O ano de publicação deve ser maior que 1000")
    @Max(value = 2100, message = "O ano de publicação não pode ser maior que 2100")
    @Schema(description = "Ano de publicação", example = "1954")
    public Integer anoPublicacao;
    
    @ManyToMany
    public List<Autor> autores;
    
    @Enumerated(EnumType.STRING)
    public StatusLivro status = StatusLivro.DISPONIVEL;
    
    public enum StatusLivro {
        DISPONIVEL,
        EMPRESTADO,
        EM_MANUTENCAO,
        EXTRAVIADO
    }
}
```

### Endpoints para Livro

```java
@Path("/livros")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@Tag(name = "Livros", description = "Operações relacionadas a livros")
public class LivroResource {

    @GET
    @Operation(summary = "Listar todos os livros", description = "Retorna a lista de todos os livros cadastrados")
    public List<Livro> listarTodos() {
        return Livro.listAll();
    }
    
    @GET
    @Path("/{id}")
    @Operation(summary = "Buscar livro por ID", description = "Recupera um livro específico pelo ID")
    public Response buscarPorId(@PathParam("id") Long id) {
        return Livro.findByIdOptional(id)
                .map(livro -> Response.ok(livro).build())
                .orElse(Response.status(Response.Status.NOT_FOUND).build());
    }
    
    @POST
    @Operation(summary = "Cadastrar novo livro", description = "Adiciona um novo livro ao acervo")
    public Response criar(@Valid Livro livro) {
        livro.persist();
        return Response.created(URI.create("/livros/" + livro.id)).entity(livro).build();
    }
    
    @PUT
    @Path("/{id}")
    @Operation(summary = "Atualizar livro", description = "Atualiza os dados de um livro existente")
    public Response atualizar(@PathParam("id") Long id, @Valid Livro livroAtualizado) {
        return Livro.findByIdOptional(id)
                .map(livro -> {
                    livro.titulo = livroAtualizado.titulo;
                    livro.isbn = livroAtualizado.isbn;
                    livro.anoPublicacao = livroAtualizado.anoPublicacao;
                    livro.autores = livroAtualizado.autores;
                    livro.status = livroAtualizado.status;
                    return Response.ok(livro).build();
                })
                .orElse(Response.status(Response.Status.NOT_FOUND).build());
    }
    
    @DELETE
    @Path("/{id}")
    @Operation(summary = "Excluir livro", description = "Remove um livro do acervo")
    public Response excluir(@PathParam("id") Long id) {
        return Livro.findByIdOptional(id)
                .map(livro -> {
                    livro.delete();
                    return Response.noContent().build();
                })
                .orElse(Response.status(Response.Status.NOT_FOUND).build());
    }
    
    @GET
    @Path("/busca/titulo/{titulo}")
    @Operation(summary = "Buscar livros por título", description = "Pesquisa livros contendo o texto especificado no título")
    public List<Livro> buscarPorTitulo(@PathParam("titulo") String titulo) {
        return Livro.list("titulo LIKE ?1", "%" + titulo + "%");
    }
    
    @GET
    @Path("/status/{status}")
    @Operation(summary = "Filtrar livros por status", description = "Lista todos os livros com o status especificado")
    public List<Livro> filtrarPorStatus(@PathParam("status") StatusLivro status) {
        return Livro.list("status", status);
    }
    
    @PUT
    @Path("/{id}/status")
    @Operation(summary = "Atualizar status do livro", description = "Altera o status de um livro (disponível, emprestado, etc)")
    public Response atualizarStatus(@PathParam("id") Long id, @QueryParam("status") StatusLivro novoStatus) {
        return Livro.findByIdOptional(id)
                .map(livro -> {
                    livro.status = novoStatus;
                    return Response.ok(livro).build();
                })
                .orElse(Response.status(Response.Status.NOT_FOUND).build());
    }
}
```

## Dicas e Boas Práticas

1. **Comece pelo modelo de dados**: Defina bem suas entidades antes de começar a implementar os endpoints.

2. **Use branches no Git**: Desenvolva cada funcionalidade em uma branch separada e faça merge quando estiver funcionando.

3. **Teste constantemente**: Use curl, Postman ou o próprio Swagger UI para testar cada endpoint à medida que desenvolve.

4. **Documentação em primeiro lugar**: Pense na documentação desde o início, não deixe para documentar apenas no final.

5. **Modularize seu código**: Separe responsabilidades usando serviços quando necessário, não coloque toda a lógica nos recursos REST.

6. **Respeite os princípios REST**: Use os verbos HTTP corretamente, defina URIs significativas e utilize os códigos de status apropriados.

7. **Validação é crucial**: Valide todos os dados de entrada rigorosamente para evitar corrupção de dados ou comportamentos inesperados.

## Perguntas Frequentes

**P: Posso usar outras extensões do Quarkus além das mencionadas?**  
R: Sim, você pode usar quaisquer extensões que achar úteis para seu projeto.

**P: Preciso implementar autenticação e autorização?**  
R: Não é obrigatório, mas será considerado um diferencial positivo.

**P: Posso usar outro banco de dados que não seja o H2?**  
R: Sim, mas sua API deve ser executável sem configurações externas complexas. Se usar outro banco, forneça um `docker-compose.yml`.

**P: Como devo entregar o projeto?**  
R: O código deve estar em um repositório Git público, e o link deve ser enviado por email até a data limite.

**P: Posso trabalhar em dupla ou grupo?**  
R: Não, este é um projeto individual para avaliação das competências de cada aluno.

## Prazo de Entrega

A entrega final do primeiro projeto deverá ser realizada até **[DATA DE ENTREGA - 23/09]** às 23:59h.

A entrega final do segundo projeto deverá ser realizada até **[DATA DE ENTREGA - A definir]** às 23:59h.

As apresentações ocorrerão no dia **[DATA DA APRESENTAÇÃO - A definir]** em sala de aula.

---

Boa sorte no desenvolvimento do seu projeto! Este trabalho é uma oportunidade de demonstrar suas habilidades e consolidar os conhecimentos adquiridos durante o curso. Aproveite para criar algo que possa ser incluído no seu portfólio profissional.
