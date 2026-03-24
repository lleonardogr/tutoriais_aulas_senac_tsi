# Projeto Final: Desenvolvimento de API com Spring Boot

Este documento apresenta as diretrizes para o projeto final de avaliação do curso de Desenvolvimento de APIs com Spring Boot. O projeto consiste em desenvolver uma API RESTful completa utilizando os conceitos e técnicas aprendidos ao longo do curso.

[Entregas](https://docs.google.com/spreadsheets/d/1TgiupynOKxU5VC3NO4aw3jPTwYkGGXrWc_H8Pxj4vHc/edit?usp=sharing)

## Objetivo

Desenvolver uma API RESTful usando Spring Boot que represente um domínio de negócio à sua escolha, aplicando os conceitos de WebServices, REST, validação de dados e documentação de API com Swagger.

## Requisitos Técnicos - Parte 1

### 1. Estrutura do Projeto

- Utilizar o framework Spring Boot em sua versão mais recente
- Implementar o projeto com Java 17 ou superior
- Utilizar o Maven ou Gradle como gerenciador de dependências
- Configurar o banco de dados H2 para persistência de dados
- Utilizar Spring Data JPA para ORM

### 2. Entidades e Relacionamentos

- **Mínimo de 5 entidades** com relacionamentos entre si
- Pelo menos um relacionamento de cada tipo: One-to-One, One-to-Many e Many-to-Many
- Utilizar validações adequadas (Bean Validation) em cada entidade
- Implementar pelo menos um enum em uma das entidades

### 3. Endpoints e Operações

- **Mínimo de 5 endpoints REST para cada entidade**
- Implementar operações CRUD completas para cada entidade
- **Todas as rotas de listagem devem ser paginadas** (utilizando `Pageable`)
- Incluir pelo menos 1 endpoint com consultas personalizadas por entidade
- Utilizar códigos de status HTTP apropriados para cada operação

### 4. Documentação
- Documentar todos os endpoints usando Springdoc OpenAPI (Swagger)
- Incluir descrições detalhadas, exemplos e possíveis códigos de resposta
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

#### 4.4. CORS (Cross-Origin Resource Sharing)

- Configurar adequadamente a política de CORS para sua API
- Permitir acesso de origens específicas
- Configurar os métodos HTTP permitidos em requisições cross-origin

#### 4.5. Versionamento da API

- Implementar versionamento da API usando um dos seguintes métodos:
  - Versionamento via URL (ex: `/api/v1/recurso`)
  - Versionamento via cabeçalho personalizado
- Manter pelo menos duas versões de um endpoint para demonstrar a funcionalidade

### 5. Validações e Tratamento de Erros

- Validar todos os dados de entrada usando Bean Validation
- Implementar tratamento global de exceções com `@ControllerAdvice`
- Retornar códigos de status HTTP apropriados para cada situação
- Fornecer mensagens de erro claras e úteis

### 6. Documentação com Swagger/OpenAPI

- Documentar todos os endpoints usando Springdoc OpenAPI
- Garantir que a documentação esteja completa e acessível via Swagger UI

## Temas Sugeridos

Você tem liberdade para escolher qualquer domínio de negócio para sua API. Alguns exemplos:

1. **Sistema de Biblioteca**
2. **Gerenciamento de Eventos**
3. **E-commerce**
4. **Rede Social**
5. **Sistema Hospitalar**

## Critérios de Avaliação

| Critério | Peso | Descrição |
|----------|------|-----------|
| Arquitetura e Design | 20% | Organização do código e boas práticas |
| Implementação de Entidades | 15% | Modelagem correta e relacionamentos |
| Endpoints REST | 20% | Uso correto de HTTP e paginação |
| Validações e Erros | 15% | Robustez e tratamento adequado |
| Documentação | 15% | Qualidade da documentação Swagger |
| Funcionalidade Geral | 15% | Funcionamento completo da API |

## Entrega e Apresentação

### Material a Ser Entregue

1. Código-fonte em repositório Git
2. README.md com instruções e descrição
3. Coleção do Postman

### Apresentação

- Introdução (2 min)
- Demonstração (8 min)
- Arquitetura (3 min)
- Perguntas (2 min)

## Exemplo de Endpoint com Paginação (Spring Boot)

```java
@RestController
@RequestMapping("/livros")
public class LivroController {

    @Autowired
    private LivroRepository repository;

    @GetMapping
    public Page<Livro> listar(Pageable pageable) {
        return repository.findAll(pageable);
    }
}
```

## Dicas e Boas Práticas
Comece pelo modelo de dados
Use paginação em todas as listagens
Separe camadas (Controller, Service, Repository)
Teste constantemente
Documente desde o início
Perguntas Frequentes

P: Posso usar outro banco além do H2?
R: Sim, desde que facilite a execução.

P: Posso usar autenticação?
R: Sim, é um diferencial.

P: Posso trabalhar em grupo?
R: Não, o projeto é individual.

## Prazo de Entrega

A entrega da primeira parte deverá ser realizada até 07 de abril às 23:59h.

A entrega da segunda parte será definida posteriormente (a definir).

As apresentações ocorrerão em data a definir.

Boa sorte no desenvolvimento do seu projeto!
