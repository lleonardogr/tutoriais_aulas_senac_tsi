# 0. Solução de Problemas - Projetos Java e Quarkus

Este guia de troubleshooting ajudará você a resolver os problemas mais comuns encontrados durante o desenvolvimento de projetos Java com Quarkus, especialmente quando usando o IntelliJ IDEA Community Edition.

## Problemas Comuns e Soluções

### 1. Problemas de Configuração do IntelliJ IDEA Community Edition

#### 1.1 Plugins Essenciais para Quarkus

Se você está usando uma instalação limpa do IntelliJ Community Edition, instale os seguintes plugins:

**Plugins Obrigatórios:**
- **Quarkus Tools** - Suporte oficial para desenvolvimento Quarkus
- **Quarkus Config** - Autocomplete e validação para arquivos de configuração

**Como instalar:**
1. Abra o IntelliJ IDEA
2. Vá em `File > Settings` (Windows/Linux) ou `IntelliJ IDEA > Preferences` (macOS)
3. Navegue para `Plugins`
4. Clique em `Marketplace`
5. Pesquise por "Quarkus Tools" e instale
6. Pesquise por "Quarkus Config" e instale
7. Reinicie o IntelliJ

**Plugins Recomendados Adicionais:**
- **Maven Helper** - Facilita o gerenciamento de dependências Maven
- **String Manipulation** - Útil para formatação de código
- **Rainbow Brackets** - Melhora a visualização de código

#### 1.2 Configuração do SDK Java

**Problema:** IntelliJ não reconhece o JDK ou usa versão incorreta.

**Solução:**
1. Vá em `File > Project Structure`
2. Em `Project Settings > Project`
3. Configure:
   - **Project SDK**: Selecione JDK 11+ (recomendado JDK 17)
   - **Project language level**: Correspondente ao SDK escolhido
4. Em `Platform Settings > SDKs`
   - Verifique se o caminho do JDK está correto
   - Se não aparecer, clique no `+` e adicione o JDK

**Verificação via terminal:**
```bash
java -version
javac -version
echo $JAVA_HOME
```

### 2. Problemas com Maven

#### 2.1 Redownload de Dependências Maven

**Problema:** Dependências corrompidas ou não baixadas corretamente.

**Solução 1 - Via IntelliJ:**
1. Abra a aba `Maven` (lateral direita)
2. Clique no ícone de refresh (🔄)
3. Ou clique com botão direito no projeto e selecione `Maven > Reload project`

**Solução 2 - Via Terminal:**
```bash
# Limpar cache local e redownload
./mvnw clean
./mvnw dependency:purge-local-repository
./mvnw compile

# Ou se usando Maven global
mvn clean
mvn dependency:purge-local-repository
mvn compile
```

**Solução 3 - Limpeza Completa:**
```bash
# Deletar pasta .m2 (cuidado: remove TODAS as dependências)
rm -rf ~/.m2/repository
./mvnw clean compile
```

#### 2.2 Problemas com Maven Wrapper

**Problema:** `./mvnw` não funciona ou permissão negada.

**Solução no Linux/macOS:**
```bash
# Dar permissão de execução
chmod +x mvnw

# Executar
./mvnw clean compile
```

**Solução no Windows:**
```bash
# Usar o arquivo .cmd
mvnw.cmd clean compile
```

#### 2.3 Configuração do Maven no IntelliJ

**Problema:** IntelliJ não usa o Maven wrapper ou configuração incorreta.

**Solução:**
1. `File > Settings > Build, Execution, Deployment > Build Tools > Maven`
2. Configure:
   - **Maven home path**: Use bundled ou aponte para instalação local
   - **User settings file**: Deixe em branco para usar padrão
   - **Local repository**: Geralmente `~/.m2/repository`
3. Em `Maven > Importing`:
   - ✅ **Import Maven projects automatically**
   - ✅ **Download sources**
   - ✅ **Download documentation**

### 3. Problemas com Quarkus

#### 3.1 Modo Dev Não Inicia

**Problema:** `quarkus:dev` falha ao iniciar.

**Diagnóstico:**
```bash
# Verificar se o projeto compila
./mvnw clean compile

# Verificar dependências
./mvnw dependency:tree

# Tentar modo dev com log detalhado
./mvnw quarkus:dev -X
```

**Soluções Comuns:**
1. **Limpar target e recompilar:**
   ```bash
   ./mvnw clean compile quarkus:dev
   ```

2. **Verificar porta em uso:**
   ```bash
   # Linux/macOS
   lsof -i :8080
   
   # Windows
   netstat -ano | findstr :8080
   ```

3. **Mudar porta temporariamente:**
   ```bash
   ./mvnw quarkus:dev -Dquarkus.http.port=8081
   ```

#### 3.2 Hot Reload Não Funciona

**Problema:** Mudanças no código não são detectadas automaticamente.

**Solução:**
1. Verificar se está em modo dev: `./mvnw quarkus:dev`
2. No IntelliJ, habilitar build automático:
   - `File > Settings > Build, Execution, Deployment > Compiler`
   - ✅ **Build project automatically**
3. Verificar se o arquivo foi salvo (Ctrl+S)

#### 3.3 Problemas com Extensões

**Problema:** Extensão não é reconhecida após adicionar ao `pom.xml`.

**Solução:**
```bash
# Listar extensões disponíveis
./mvnw quarkus:list-extensions

# Adicionar extensão via CLI
./mvnw quarkus:add-extension -Dextensions="hibernate-orm-panache"

# Verificar se foi adicionada corretamente
./mvnw quarkus:info
```

### 4. Problemas de Compilação

#### 4.1 Erro "Package does not exist"

**Problema:** Imports não são reconhecidos.

**Solução:**
1. **Reimport do projeto:**
   ```bash
   ./mvnw clean compile
   ```

2. **No IntelliJ:**
   - `File > Invalidate Caches and Restart`
   - Escolha `Invalidate and Restart`

3. **Verificar estrutura de pastas:**
   ```
   src/
   ├── main/
   │   ├── java/
   │   │   └── org/acme/
   │   └── resources/
   └── test/
       └── java/
           └── org/acme/
   ```

#### 4.2 Problemas com annotation processing

**Problema:** Anotações do Panache ou outras não funcionam.

**Solução:**
1. **Habilitar annotation processing no IntelliJ:**
   - `File > Settings > Build, Execution, Deployment > Compiler > Annotation Processors`
   - ✅ **Enable annotation processing**

2. **Verificar dependências no pom.xml:**
   ```xml
   <dependency>
       <groupId>io.quarkus</groupId>
       <artifactId>quarkus-hibernate-orm-panache</artifactId>
   </dependency>
   ```

### 5. Problemas com Banco de Dados

#### 5.1 H2 Database não inicializa

**Problema:** Erro ao conectar com H2 em memória.

**Solução:**
1. **Verificar application.properties:**
   ```text
   # Configuração H2 em memória
   quarkus.datasource.db-kind=h2
   quarkus.datasource.jdbc.url=jdbc:h2:mem:testdb
   
   # Para modo dev, criar tabelas automaticamente
   quarkus.hibernate-orm.database.generation=drop-and-create
   
   # Habilitar console H2 (apenas dev)
   quarkus.datasource.jdbc.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
   ```

2. **Acessar console H2:**
   - URL: `http://localhost:8080/q/h2-console`
   - JDBC URL: `jdbc:h2:mem:testdb`
   - User: `sa`
   - Password: (deixar em branco)

#### 5.2 Problemas de Schema

**Problema:** Tabelas não são criadas ou dados não persistem.

**Solução:**
1. **Verificar configurações de schema:**
   ```properties
   # Para desenvolvimento - recria schema a cada inicialização
   quarkus.hibernate-orm.database.generation=drop-and-create
   
   # Para produção - apenas valida
   quarkus.hibernate-orm.database.generation=validate
   ```

2. **Importar dados iniciais:**
   ```text
   # Arquivo import.sql na pasta resources
   quarkus.hibernate-orm.sql-load-script=import.sql
   ```

### 6. Problemas de Rede e Proxy

#### 6.1 Download de Dependências Falha

**Problema:** Timeout ou erro de conexão ao baixar dependências.

**Solução para ambientes corporativos:**
1. **Configurar proxy no Maven** (`~/.m2/settings.xml`):
   ```xml
   <settings>
     <proxies>
       <proxy>
         <id>proxy</id>
         <active>true</active>
         <protocol>http</protocol>
         <host>proxy.empresa.com</host>
         <port>8080</port>
         <username>usuario</username>
         <password>senha</password>
       </proxy>
     </proxies>
   </settings>
   ```

2. **Configurar proxy no IntelliJ:**
   - `File > Settings > Appearance & Behavior > System Settings > HTTP Proxy`

### 7. Comandos de Diagnóstico

#### 7.1 Informações do Sistema

```bash
# Versão do Java
java -version

# Variáveis de ambiente
echo $JAVA_HOME
echo $MAVEN_HOME
echo $PATH

# Versão do Maven
mvn -version
./mvnw -version

# Informações do projeto Quarkus
./mvnw quarkus:info
```

#### 7.2 Limpeza Completa do Projeto

```bash
# Parar todos os processos
# Ctrl+C no terminal do quarkus:dev

# Limpeza completa
./mvnw clean
rm -rf target/
rm -rf .quarkus/

# Recompilação
./mvnw compile
./mvnw quarkus:dev
```

### 8. IntelliJ IDEA - Configurações Específicas

#### 8.1 Configurações de Performance

Para projetos grandes, otimize o IntelliJ:

1. **Aumentar memória** (`Help > Change Memory Settings`):
   ```
   -Xmx2048m
   -XX:ReservedCodeCacheSize=512m
   ```

2. **Configurações de indexação:**
   - `File > Settings > Build, Execution, Deployment > Compiler`
   - ✅ **Build project automatically**
   - ✅ **Compile independent modules in parallel**

#### 8.2 Configuração de Encoding

**Problema:** Caracteres especiais não aparecem corretamente.

**Solução:**
1. `File > Settings > Editor > File Encodings`
2. Configure:
   - **Global Encoding**: UTF-8
   - **Project Encoding**: UTF-8
   - **Default encoding for properties files**: UTF-8

### 9. Problemas Específicos do Windows

#### 9.1 Caminho muito longo

**Problema:** Erro de caminho muito longo no Windows.

**Solução:**
1. **Usar caminhos mais curtos para projetos**
2. **Habilitar suporte a caminhos longos:**
   ```cmd
   # Como administrador
   git config --system core.longpaths true
   ```

#### 9.2 PowerShell Execution Policy

**Problema:** Scripts não executam no PowerShell.

**Solução:**
```powershell
# Como administrador
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

## Checklist de Verificação Rápida

Quando encontrar problemas, siga esta ordem:

- [ ] ✅ Java JDK 11+ está instalado e configurado
- [ ] ✅ IntelliJ tem plugins Quarkus instalados
- [ ] ✅ Projeto Maven importado corretamente
- [ ] ✅ Dependências baixadas (`./mvnw dependency:resolve`)
- [ ] ✅ Projeto compila (`./mvnw clean compile`)
- [ ] ✅ Annotation processing habilitado no IntelliJ
- [ ] ✅ Encoding configurado para UTF-8
- [ ] ✅ Porta 8080 disponível

## Recursos Adicionais

- **Documentação Oficial Quarkus**: https://quarkus.io/guides/
- **IntelliJ Quarkus Plugin**: https://plugins.jetbrains.com/plugin/13234-quarkus-tools
- **Maven Troubleshooting**: https://maven.apache.org/guides/mini/guide-ide-idea.html

## Contato para Suporte

Se os problemas persistirem:
1. Documente o erro exato (screenshot/log)
2. Informe versões (Java, IntelliJ, Maven, SO)
3. Descreva os passos que levaram ao erro
4. Procure o professor ou monitoria da disciplina

---

**Dica:** Mantenha este guia acessível durante todo o curso. A maioria dos problemas tem soluções simples, mas é importante saber onde procurar!