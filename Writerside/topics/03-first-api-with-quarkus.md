# 3. Criando sua Primeira API com Quarkus

Neste tópico, vamos criar nossa primeira API RESTful utilizando o framework Quarkus. Aprenderemos como configurar um projeto Quarkus do zero com suporte a banco de dados H2, criar endpoints REST básicos e executar a aplicação.

## Pré-requisitos

Antes de começarmos, certifique-se de ter instalado:

- **JDK 11+** (recomendado JDK 17)
- **Maven 3.8.1+** ou **Gradle 7.4.1+**
- **GraalVM** (opcional, para compilação nativa)
- **Docker** (opcional, para containers)
- **Editor de código** (como VS Code, IntelliJ IDEA, Eclipse)

## Métodos para Criar um Projeto Quarkus

Existem duas formas principais de criar um projeto Quarkus:

1. Utilizando o **Quarkus Code Generator** (interface web)
2. Utilizando a **CLI do Quarkus** (linha de comando)

Vamos explorar ambas as abordagens.

## Método 1: Utilizando o Quarkus Code Generator (code.quarkus.io)

O Quarkus Code Generator é uma interface web que permite configurar e baixar um projeto Quarkus com as extensões desejadas.

### Passos:

1. **Acesse o site**: [code.quarkus.io](https://code.quarkus.io/)

2. **Configure seu projeto**:
   - **Group**: `org.acme` (substitua pelo seu pacote)
   - **Artifact**: `quarkus-api-demo`
   - **Build Tool**: Maven (ou Gradle, conforme sua preferência)
   - **Version**: deixe a versão padrão ou escolha uma específica
   - **Java Version**: 17 (ou a versão que você estiver utilizando)

3. **Adicione as extensões necessárias**:
   - **RESTEasy Classic**: para criar endpoints REST
   - **RESTEasy Classic Jackson**: para serialização/deserialização JSON
   - **H2 Database**: banco de dados em memória
   - **Hibernate ORM with Panache**: para persistência de dados
   - **JDBC Driver - H2**: driver JDBC para o H2

   Use a barra de pesquisa para encontrar e selecionar estas extensões.

4. **Gere o projeto**:
   - Clique no botão "Generate your application"
   - O download do arquivo ZIP começará automaticamente

5. **Extraia e abra o projeto**:
   - Extraia o arquivo ZIP para o diretório de sua escolha
   - Abra o projeto em seu IDE favorito

## Método 2: Utilizando a CLI do Quarkus

A CLI do Quarkus oferece uma maneira rápida de criar projetos diretamente do terminal.

### Instalação da CLI:

#### No Linux/macOS:

```bash
curl -Ls https://sh.jbang.dev | bash -s - trust add https://repo1.maven.org/maven2/io/quarkus/quarkus-cli/
curl -Ls https://sh.jbang.dev | bash -s - app install --fresh --force quarkus@quarkusio
```

#### No Windows (PowerShell):

```shell
iex "& { $(iwr https://ps.jbang.dev) } trust add https://repo1.maven.org/maven2/io/quarkus/quarkus-cli/"
iex "& { $(iwr https://ps.jbang.dev) } app install --fresh --force quarkus@quarkusio"
```

### Criando o projeto com CLI:

```bash
quarkus create app org.acme:quarkus-api-demo \
    --extension="resteasy-classic,resteasy-jackson,hibernate-orm-panache,jdbc-h2"
```

## Estrutura do Projeto

Independentemente do método escolhido, a estrutura do projeto Quarkus será semelhante:

```
quarkus-api-demo/
├── mvnw
├── mvnw.cmd
├── pom.xml
└── src/
    ├── main/
    │   ├── docker/
    │   ├── java/
    │   │   └── org/
    │   │       └── acme/
    │   ├── resources/
    │   │   ├── META-INF/
    │   │   │   └── resources/
    │   │   └── application.properties
    │   └── webapp/
    └── test/
```

## Configurando o Banco de Dados H2

Vamos configurar nosso banco de dados H2. Edite o arquivo `application.properties` em `src/main/resources`:

```text
# Configurações da base de dados
quarkus.datasource.db-kind=h2
quarkus.datasource.username=username
quarkus.datasource.password=password
quarkus.datasource.jdbc.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1

# Em modo de desenvolvimento, crie e atualize o banco automaticamente
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.hibernate-orm.log.sql=true
quarkus.hibernate-orm.sql-load-script=import.sql

# Configuração para o console H2 (opcional)
quarkus.h2.console.enabled=true
quarkus.h2.console.path=/h2-console
```

## Criando uma Entidade

Vamos criar uma entidade simples para nosso exemplo. Crie uma classe chamada `Produto.java` no pacote `org.acme.entity`:

```java
package org.acme.entity;

import io.quarkus.hibernate.orm.panache.PanacheEntity;
import javax.persistence.Entity;

@Entity
public class Produto extends PanacheEntity {
    public String nome;
    public String descricao;
    public Double preco;
    
    // Utilizando Panache, não precisamos criar getters/setters
    // nem implementar operações CRUD básicas
}
```

## Criando Dados de Teste

Crie um arquivo `import.sql` em `src/main/resources` para inserir alguns dados de teste:

```sql
-- Inserindo produtos de teste
INSERT INTO Produto(id, nome, descricao, preco) VALUES (nextval('hibernate_sequence'), 'Notebook', 'Notebook Dell 16GB RAM', 4500.0);
INSERT INTO Produto(id, nome, descricao, preco) VALUES (nextval('hibernate_sequence'), 'Smartphone', 'iPhone 13 Pro 256GB', 6700.0);
INSERT INTO Produto(id, nome, descricao, preco) VALUES (nextval('hibernate_sequence'), 'Tablet', 'iPad Air 4 64GB', 4200.0);
```

## Criando um Resource (Endpoint REST)

Agora vamos criar um endpoint REST para expor nossa entidade. Crie uma classe `ProdutoResource.java` no pacote `org.acme.resource`:

```java
package org.acme.resource;

import io.quarkus.hibernate.orm.rest.data.panache.PanacheEntityResource;
import org.acme.entity.Produto;

import javax.enterprise.context.ApplicationScoped;
import javax.ws.rs.*;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;
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
    public Produto buscarPorId(@PathParam("id") Long id) {
        return Produto.findById(id);
    }
    
    @POST
    public Response criar(Produto produto) {
        produto.persist();
        return Response.status(Response.Status.CREATED).entity(produto).build();
    }
    
    @PUT
    @Path("/{id}")
    public Produto atualizar(@PathParam("id") Long id, Produto produto) {
        Produto entidade = Produto.findById(id);
        if (entidade == null) {
            throw new NotFoundException("Produto não encontrado com o ID: " + id);
        }
        
        entidade.nome = produto.nome;
        entidade.descricao = produto.descricao;
        entidade.preco = produto.preco;
        
        return entidade;
    }
    
    @DELETE
    @Path("/{id}")
    public Response excluir(@PathParam("id") Long id) {
        boolean excluido = Produto.deleteById(id);
        if (excluido) {
            return Response.noContent().build();
        }
        throw new NotFoundException("Produto não encontrado com o ID: " + id);
    }
}
```

## Executando a Aplicação

Para executar a aplicação Quarkus em modo de desenvolvimento:

```bash
# Se estiver usando Maven
./mvnw quarkus:dev

# Se estiver usando Gradle
./gradlew quarkusDev
```

O modo dev do Quarkus oferece:
- Hot reload automático quando você altera o código
- Console de desenvolvimento em http://localhost:8080/q/dev/
- Acesso ao console do H2 em http://localhost:8080/h2-console/

## Testando a API

Agora você pode testar sua API usando curl, Postman ou qualquer cliente HTTP:

### Listar todos os produtos:
```bash
curl http://localhost:8080/produtos
```

### Buscar um produto específico:
```bash
curl http://localhost:8080/produtos/1
```

### Criar um novo produto:
```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"nome":"Monitor", "descricao":"Monitor 27 polegadas 4K", "preco":2100.0}' \
  http://localhost:8080/produtos
```

### Atualizar um produto:
```bash
curl -X PUT -H "Content-Type: application/json" \
  -d '{"nome":"Monitor UHD", "descricao":"Monitor 27 polegadas 4K UHD", "preco":2300.0}' \
  http://localhost:8080/produtos/4
```

### Excluir um produto:
```bash
curl -X DELETE http://localhost:8080/produtos/4
```

## Verificando o Console H2

Para acessar o console H2 e verificar os dados diretamente:

1. Acesse http://localhost:8080/h2-console/
2. Configure a conexão:
   - JDBC URL: `jdbc:h2:mem:testdb`
   - User Name: `username`
   - Password: `password`
3. Clique em "Connect" para acessar o banco e executar consultas SQL

## Empacotando a Aplicação

Para criar um jar executável:

```bash
# Maven
./mvnw package

# Gradle
./gradlew build
```

O arquivo JAR será gerado no diretório `target/` (Maven) ou `build/libs/` (Gradle).

## Criando uma Imagem Docker

Para criar uma imagem Docker da sua aplicação:

```bash
# Maven
./mvnw package -Dquarkus.container-image.build=true

# Gradle
./gradlew build -Dquarkus.container-image.build=true
```

## Compilação Nativa (Opcional)

Para criar uma versão nativa da sua aplicação (requer GraalVM):

```bash
# Maven
./mvnw package -Pnative

# Gradle
./gradlew build -Pnative
```

## Benefícios da Abordagem Quarkus

- **Startup ultra-rápido**: Inicia em milissegundos
- **Baixo consumo de memória**: Ideal para ambientes cloud
- **Live coding**: Desenvolvimento mais produtivo com hot reload
- **Implementação simplificada**: Frameworks como Panache reduzem o código boilerplate
- **Extensibilidade**: Grande ecossistema de extensões

## Próximos Passos

Agora que você tem sua primeira API Quarkus funcionando com banco de dados H2, pode:

1. **Adicionar validações** com Hibernate Validator
2. **Implementar autenticação** com JWT ou OAuth2
3. **Adicionar documentação** usando Swagger/OpenAPI

[//]: # (4. **Explorar testes** com JUnit e RestAssured)
4. **Implantar em produção** em ambientes cloud

No próximo tópico, expandiremos este conceito para criar uma API CRUD mais completa com validações e documentação Swagger.

## Solução de Problemas (Troubleshooting)

Ao trabalhar com projetos Quarkus no IntelliJ Community Edition, alguns ajustes são necessários para garantir uma experiência de desenvolvimento fluida.

### Utilizando IntelliJ Community Edition

O IntelliJ Community Edition não possui suporte nativo ao Quarkus, mas é possível configurar o ambiente para trabalhar com o framework:

- Baixe e instale o plugin <b>Quarkus Tools</b> disponível no marketplace do IntelliJ.
- Crie uma nova configuração de execução (<b>Run Configuration</b>) para rodar sua aplicação Quarkus.
- Certifique-se de que o projeto está indexado corretamente na versão do Java utilizada (JDK 11+ ou JDK 17 recomendado).

<procedure title="Criando uma configuração de execução no IntelliJ" id="run_config_intellij" collapsible="true">
    <step>Vá em <b>Run &gt; Edit Configurations</b>.</step>
    <step>Clique em <b>+</b> e selecione <b>Application</b>.</step>
    <step>Defina o nome, classe principal (<code>io.quarkus.runner.GeneratedMain</code> para projetos padrão) e o módulo do projeto.</step>
    <step>Configure as variáveis de ambiente e argumentos conforme necessário.</step>
</procedure>

<warning>
Se o projeto não estiver indexado na versão correta do Java, você pode enfrentar problemas de depuração e execução. Verifique em <b>File &gt; Project Structure &gt; Project</b> se o SDK está configurado para a versão recomendada.
</warning>

<tip>
Após instalar o plugin Quarkus Tools, você terá acesso a assistentes de criação de projetos, gerenciamento de extensões e integração facilitada com o Quarkus Dev Mode.
</tip>

<warning>
O IntelliJ Community Edition não possui suporte completo para depuração de aplicações Quarkus nativas. Para recursos avançados, considere utilizar o IntelliJ Ultimate ou outras IDEs como VS Code.
</warning>
