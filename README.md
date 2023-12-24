# Integração Contínua (CI) Para Laravel Usando Ações Do Github

PHP LARAVEL GITHUB

O Github fornece pipelines de construção de integração contínua (CI) usando seu serviço chamado [github actions](https://github.com/features/actions).

Ações do Github As compilações de CI são chamadas de fluxos de trabalho, e um fluxo de trabalho pode ser acionado quando um determinado evento acontece em seu repositório do github.

Eventos como um commit é feito, uma solicitação pull é aberta, etc. irão acionar execuções de fluxo de trabalho em ações do GitHub.

Se você estiver trabalhando com uma equipe de desenvolvedores, as ações do Github podem ajudá-lo a validar as solicitações pull executando casos de teste necessários em uma solicitação pull, para que possamos mesclar com segurança uma solicitação pull quando ela for aberta e todos os casos de teste forem aprovados.

> As ações do Github oferecem minutos de construção ilimitados para repositórios públicos. Para repositórios privados, a conta gratuita nos dá 2.000 minutos de construção/mês. Para um desenvolvedor solo, isso é mais que suficiente.

O Laravel CI geralmente consiste em várias etapas para garantir que o aplicativo funcione sem problemas quando for para produção. Criaremos um fluxo de trabalho de ações no github para executar as seguintes tarefas quando uma solicitação pull for aberta.

- Verifique se podemos instalar dependências do compositor
- Verifique se as dependências do npm podem ser instaladas com sucesso
- Execute o comando frontend assets build para verificar se podemos reduzir com sucesso nossos arquivos css e js
- Verifique se podemos executar migrações em um banco de dados sem problemas (usaremos um banco de dados sqlite temporário para isso)
- Execute nossos testes de unidade e certifique-se de que as alterações de código na solicitação pull não quebraram a funcionalidade existente.
- Depois que as tarefas acima forem executadas com êxito e sem problemas, poderemos mesclar com segurança a solicitação pull se as alterações no código parecerem boas.

## Criar arquivo de fluxo de trabalho para Laravel

Todos os arquivos de fluxo de trabalho devem residir no .github/workflowsdiretório raiz do projeto.

Então, primeiro, crie esses dois diretórios.

```
mkdir .github && mkdir .github/workflows
```

Arquivos de fluxo de trabalho usados yaml syntax​​para definir as tarefas. Um único arquivo de fluxo de trabalho pode conter diversas tarefas.

Vamos criar nosso arquivo de fluxo de trabalho dentro .github/workflowsdo diretório. Vou chamar esse arquivo de laravel-ci.yml. Você pode usar qualquer nome de arquivo que desejar. Apenas certifique-se de que a extensão do arquivo seja .ymlou .yaml.

```
touch .github/workflows/laravel-ci.yml
```

Abra o arquivo em seu editor de código favorito e adicione o seguinte código nele.

```
name: Laravel CI
on:
  pull_request:
    branches: [ master, staging ]
jobs:
  laravel-tests:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
```

O arquivo acima fornece um esqueleto básico para nossas ações do github laravel ci build. Ele é configurado para executar o fluxo de trabalho na versão mais recente do sistema operacional Ubuntu quando uma solicitação pull é aberta em ramificações master ou de teste.

Nosso fluxo de trabalho não está executando nenhuma tarefa no momento. Então, vamos começar adicionando algumas tarefas a este fluxo de trabalho.

### 1. Configure o PHP (opcional)

A ubuntu-latestimagem já vem com a versão mais recente do php instalada. Se o seu aplicativo requer a versão php 7.3 e inferior, adicione o código abaixo à stepsseção do seu arquivo de fluxo de trabalho para mudar para a versão php que seu aplicativo está usando.

```
- name: Setup PHP
  uses: shivammathur/setup-php@v2
  with:
    php-version: 7.2 # Change to the php version your application is using
    extensions: mbstring, bcmath # Install any required php extensions
```

### 2. Crie o arquivo .env

Todo aplicativo laravel deve ter um arquivo .env para gerenciar suas variáveis ​​de ambiente. A primeira tarefa que executamos em nosso fluxo de trabalho é copiar .env.exampleo arquivo para .env.

Atualize a stepsseção no arquivo de fluxo de trabalho com abaixo

```
- name: Copy .env.example to .env
  run: php -r "file_exists('.env') || copy('.env.example', '.env');"
3. Instale dependências do compositor
A próxima etapa é instalar as dependências do compositor. Ao testar a instalação das dependências do compositor, garantimos que não haja pacotes quebrados na solicitação pull.

- name: Install composer dependencies
  run: composer install
```

### 4. Defina nossas permissões de diretório corretamente

O Laravel exige que certos diretórios possam ser gravados pelo servidor web. Podemos definir essas permissões de diretório, 777pois este é apenas um servidor de CI.

Você nunca deve definir permissões de diretório para 777 em seu servidor real. Definir isso pode resultar em acesso não autorizado a arquivos restritos.

```
- name: Set required directory permissions
  run: chmod -R 777 storage bootstrap/cache
```

### 5. Gere a chave de criptografia do aplicativo

Como estamos criando nosso .envarquivo recentemente, ele não terá uma chave de criptografia. Então, vamos criar e definir uma chave de criptografia.

```
- name: Generate encryption key
  run: php artisan key:generate
```

### 6. Crie banco de dados temporário

Nas próximas etapas, executaremos migrações de banco de dados e testes unitários. Precisamos de um banco de dados temporário para executar essas tarefas. Podemos criar um banco de dados sqlite temporário adicionando a seguinte tarefa.

```
- name: Create temporary sqlite database
  run: |
    mkdir -p database
    touch database/database.sqlite
```

### 7. Execute migrações de banco de dados

Podemos executar migrações de banco de dados no banco de dados temporário que criamos. Fazendo isso, podemos garantir que os arquivos de migração novos e existentes ainda funcionem e faça as alterações de esquema necessárias.

Antes de executarmos as migrações, precisamos definir nossas variáveis ​​de ambiente DB_CONNECTIONe DB_DATABASEpara o banco de dados sqlite temporário recém-criado. Podemos definir essas variáveis ​​de ambiente usando o envcomando mostrado abaixo.

```
- name: Run laravel database migrations
  env:
    DB_CONNECTION: sqlite
    DB_DATABASE: database/database.sqlite
  run: php artisan migrate --force
```

### 8. Instale dependências NPM

A próxima etapa é instalar as dependências do nó. Ao testar a instalação de dependências npm, garantimos que não haja pacotes npm quebrados na solicitação pull.

```
- name: Install NPM dependencies
  run: npm install
```

### 9. Minimize arquivos CSS e JS

Podemos executar npm run prodo comando para testar e garantir que nosso comando de construção de frontend funcione conforme o esperado e reduza nossos arquivos css e js.

```
- name: Minify CSS and JS files
  run: npm run prod
```

### 10. Execute testes de unidade

Finalmente, nosso aplicativo está completamente pronto com todos os pacotes necessários do compositor e do nó instalados. Possui um banco de dados temporário, arquivo env e os ativos de frontend são compilados e reduzidos.

Agora podemos mergulhar na execução de nossos testes de unidade para garantir que as novas alterações no código não prejudiquem a funcionalidade existente.

```
- name: Run unit tests via PHPUnit
  env:
    DB_CONNECTION: sqlite
    DB_DATABASE: database/database.sqlite
  run: ./vendor/bin/phpunit
```

## Arquivo de fluxo de trabalho concluído

Se você seguir todas as etapas corretamente, seu arquivo de fluxo de trabalho concluído deverá ser semelhante ao mostrado abaixo.

```
name: Laravel CI
on:
  pull_request:
    branches: [ master, staging ]
jobs:
  laravel-tests:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Copy .env.example to .env
      run: php -r "file_exists('.env') || copy('.env.example', '.env');"
    - name: Install composer dependencies
      run: composer install
    - name: Set required directory permissions
      run: chmod -R 777 storage bootstrap/cache
    - name: Generate encryption key
      run: php artisan key:generate
    - name: Create temporary sqlite database
      run: |
        mkdir -p database
        touch database/database.sqlite
    - name: Run laravel database migrations
      env:
        DB_CONNECTION: sqlite
        DB_DATABASE: database/database.sqlite
      run: php artisan migrate --force
    - name: Install NPM dependencies
      run: npm install
    - name: Minify CSS and JS files
      run: npm run prod
    - name: Run unit tests via PHPUnit
      env:
        DB_CONNECTION: sqlite
        DB_DATABASE: database/database.sqlite
      run: ./vendor/bin/phpunit
```

Depois de enviar o arquivo acima para seu repositório github, o Github executará as referidas tarefas sempre que uma nova solicitação pull for aberta mastere stagingramificada.

O status de falha/sucesso da execução do fluxo de trabalho será exibido na solicitação pull em que foi executado.

Você pode visualizar todas as execuções de fluxo de trabalho acionadas em um repositório específico clicando na actionsguia no menu de navegação de uma página do repositório.
