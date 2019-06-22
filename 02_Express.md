# Express

No último tópico foi apresentada uma revisão de aplicações web, depois uma análise do modelo de execução do JavaScript, e também uma introdução ao Node.js. No final foi abordado o desenvolvimento de uma API HTTP usando nada além do que já está disponível nativamente na plataforma.

Durante o desenvolvimento da API HTTP, algumas coisas claramente se mostraram difíceis de resolver, produzindo código frágil e mal organizado, como por exemplo:

- *Processamento do corpo da requisição:* foi necessário tratar o recebimento dos dados em "chunks". A implementação que foi feita (concatenação de strings) só funciona com dados textuais, não funcionaria para upload de arquivos, por exemplo. Se a entrada for JSON, a conversão para um objeto JavaScript e validação dos dados de entrada também foi feita manualmente, o que não é o ideal.

- *Roteamento da requisição:* foi usada uma grande carga de manipulação de strings para rotear as chamadas. Isso rapidamente deixa o código poluído e difícil de entender.

- *Tratamento de casos de erro:* como o módulo HTTP não oferece nada com relação a tratamento de erros, foi necessário codificar com cuidado para garantir que toda requisição tivesse uma resposta, mesmo em casos de exceção. Isso fica ainda pior se considerar o fato de que o processamento costuma ser assíncrono (baseado em callbacks/Promises).

Além disso, algumas coisas nem chegaram a ser abordadas, como:

- *Disponibilização de arquivos estáticos (HTML/CSS/JavaScript de front-end):* não é a única maneira, mas uma forma bem simples de disponibilizar sua aplicação para os usuários é através do próprio servidor de back-end.

- *Upload/download de arquivos*

- *Segurança:* como trabalhar a autenticação dos usuários da aplicação? Quais os mecanismos disponíveis e quando usar cada um deles?

Nesta seção será abordado como usar o framework Express para permitir o desenvolvimento de uma API com todas essas características.

## Node Package Manager (NPM)

Antes de prosseguir, vale discutir sobre o gestor de pacotes do Node, o NPM. Essa ferramenta controla as dependências dos seus projetos, além de permitir a instalação de ferramentas disponíveis globalmente na instalação do Node. Para exemplificar, considere a sequência de comandos para iniciar um projeto usando Angular:

```
npm install -g @angular/cli
```

Isso instala o pacote `@angular/cli` de modo global. A partir desse comando, a ferramenta `ng` ficará disponível em qualquer terminal e em qualquer diretório. Use o site `https://www.npmjs.com` para encontrar detalhes deste e qualquer outro pacote. Antes de adicionar qualquer dependência no seu projeto, é interessante analisar a "saúde" do mesmo, que é uma análise subjetiva feita sobre a quantidade de downloads, issues abertas, data do último commit no GitHub, etc.

```
ng new
```

Esse comando inicia o projeto Angular.

```
npm install @angular/material
```

Esse comando instala a dependência `@angular/material` no projeto. Repare que o comando reclama de um `peer dependency` que você precisa instalar manualmente, o `@angular/cdk`. Uma ótima leitura sobre isso pode ser vista aqui: https://stackoverflow.com/questions/26737819/why-use-peer-dependencies-in-npm-for-plugins, e aqui: https://nodejs.org/en/blog/npm/peer-dependencies/. Também discutiremos sobre isso mais abaixo.

```
npm install @angular/cdk
```

Veja como fica o `package.json`:

```json
{
  "name": "?????????????",
  "version": "0.0.0",
  "scripts": {
    "ng": "ng",
    "start": "ng serve",
    "build": "ng build",
    "test": "ng test",
    "lint": "ng lint",
    "e2e": "ng e2e"
  },
  "private": true,
  "dependencies": {
    "@angular/animations": "~8.0.0",
    "@angular/cdk": "^8.0.1",
    "@angular/common": "~8.0.0",
    "@angular/compiler": "~8.0.0",
    "@angular/core": "~8.0.0",
    "@angular/forms": "~8.0.0",
    "@angular/material": "^8.0.1",
    "@angular/platform-browser": "~8.0.0",
    "@angular/platform-browser-dynamic": "~8.0.0",
    "@angular/router": "~8.0.0",
    "rxjs": "~6.4.0",
    "tslib": "^1.9.0",
    "zone.js": "~0.9.1"
  },
  "devDependencies": {
    "@angular-devkit/build-angular": "~0.800.0",
    "@angular/cli": "~8.0.2",
    "@angular/compiler-cli": "~8.0.0",
    "@angular/language-service": "~8.0.0",
    "@types/node": "~8.9.4",
    "@types/jasmine": "~3.3.8",
    "@types/jasminewd2": "~2.0.3",
    "codelyzer": "^5.0.0",
    "jasmine-core": "~3.4.0",
    "jasmine-spec-reporter": "~4.2.1",
    "karma": "~4.1.0",
    "karma-chrome-launcher": "~2.2.0",
    "karma-coverage-istanbul-reporter": "~2.0.1",
    "karma-jasmine": "~2.0.1",
    "karma-jasmine-html-reporter": "^1.4.0",
    "protractor": "~5.4.0",
    "ts-node": "~7.0.0",
    "tslint": "~5.15.0",
    "typescript": "~3.4.3"
  }
}
```

O que são os `scripts`? São utilitários que você deseja deixar documentado no próprio projeto, acessíveis através do comando `npm run <script>`, ex: `npm run start`. Alguns desses scripts podem ser chamados sem o uso do comando `run`, como `start`: `npm start`.

As `dependencies` são todas as bibliotecas usadas diretamente pelo seu projeto. Elas são instaladas no diretório `node_modules` (esse diretório *não* deve ser versionado no repositório do projeto). Sempre que uma cópia limpa do projeto for criada, basta executar o comando `npm install` para baixar novamente todas as dependências.

As `devDependencies` são normalmente ferramentas usadas no ambiente de desenvolvimento, mas que não são necessárias para usar uma biblioteca ou rodar o projeto em um ambiente.

Execute o comando abaixo para analisar o gráfico de dependências do projeto:

```
npm ls
```

Sim, praticamente a Internet toda. Para ter uma saída um pouco mais legível, limite o nível de folhas:

```
npm ls --depth=0
```

## Semantic Versioning (SemVer)

Dê uma olhada nas dependências do projeto usado para testes na seção anterior. O que significam, exatamente, as definições de versão nas dependências? Por exemplo:

```
"@angular/cli": "~8.0.2"
```

Todo pacote Node que é publicado em um repositório (como o npmjs.com, por exemplo) precisa ter um nome e uma versão. Essa versão precisa ser compatível com o conceito de versão semântica (SemVer [1][2]). De forma resumida, versionamento semântico é composto por três números:

```
1.2.3
```

O primeiro é a versão `MAJOR`. O segundo, a `MINOR`, e o terceiro, `PATCH`. A versão `MAJOR` só deve mudar quando a biblioteca alterar sua *interface externa* de modo *incompatível* com as versões anteriores, por exemplo: alteração de assinatura de método de modo que chamadas existentes deixem de funcionar, remoção completa de um método. A `MINOR` é incrementada quando houver *adição de funcionalidade*, sem quebrar código existente. Por fim, a versão `PATCH` só é incrementada quando existirem apenas correções de bugs, melhorias de performance e coisas do tipo em funcionalidades existentes.

Com isso, é possível dizer para o NPM quais versões de determinada biblioteca você entende que seu projeto suporta. A maneira mais engessada seria dizer *exatamente* qual versão usar, por exemplo:

```
"@angular/cli": "8.0.2"
```

Neste caso, não importa quantas novas versões da biblioteca já foram publicadas, esse projeto sempre vai usar a versão `8.0.2`. Uma outra forma é permitir que o npm baixe versões `PATCH` mais novas automaticamente. Isso pode ser atingido de diversas formas:

```
"@angular/cli": "8.0"
"@angular/cli": "8.0.x"
"@angular/cli": "8.0.X"
"@angular/cli": "8.0.*"
"@angular/cli": "~8.0.2"
"@angular/cli": ">=8.0.2 <8.1"
"@angular/cli": "8.0.2 - 8.0"
```

Note que as 4 primeiras opções vão aceitar a versão `8.0.1` se esta estiver disponível, enquanto as 3 últimas são um pouco mais restritivas.

De modo similar, você pode ser ainda mais flexível, e aceitar incrementos `MINOR` de forma automática:

```
"@angular/cli": "~8"
"@angular/cli": "8.x"
"@angular/cli": "8"
"@angular/cli": ">=8.0.2 <9"
"@angular/cli": ">=8.0.2 - 8"
"@angular/cli": "^8.0.2"
```

E se estiver se sentindo aventureiro, pode até mesmo aceitar qualquer versão da biblioteca:

```
"@angular/cli": "*"
```

[1] https://semver.org/

[2] https://docs.npmjs.com/misc/semver

## Instalação do Express e primeiro projeto

Agora que ficou claro o uso do `npm`, é possível iniciar um novo projeto e adicionar o framework `express` como dependência. Vá até um diretório vazio na sua máquina e digite o seguinte comando:

```
npm init
```

Se o seu projeto não vai ser consumido por outros como uma biblioteca, é uma boa ideia remover o atributo `main` e adicionar `private: true` no arquivo `package.json` que foi gerado.

Adicione agora a dependência para a versão mais recente do Express:

```
npm install express
```

O que é esse arquivo `package-lock.json` que foi criado? Está fora do escopo desta disciplina, mas duas ótimas discussões sobre este arquivo podem ser encontradas aqui: https://renovatebot.com/docs/dependency-pinning/ e aqui: https://github.com/commitizen/cz-conventional-changelog-default-export/pull/4#issuecomment-358038966.

Agora crie um arquivo chamado `index.js`, colocando o conteúdo abaixo:

```js
const express = require('express');
const app = express()


app.get('/ola', (req, res) => {
    const nome = req.query.nome;
    res.send(`Olá, ${nome}!`);
});

app.listen(3000, () => {
    console.log('Primeira API com Express.');
});
```

Note que já é possível observar algumas diferenças drásticas com relação ao tópico anterior. Primeiro, não foi necessário fechar a resposta, o próprio método `send` já faz isso. Segundo, o objeto `req` vem enriquecido com dados da requisição, não há a necessidade de processar esses dados na mão. Terceiro, o roteamento fica muito mais simples, por padrão a resposta sempre será 404, a não ser que o caminho e método se mostre compatível a alguma das chamadas no estilo `app.get`. Veja esse outro exemplo, envolvendo parâmetros de caminho e corpo:

```js
app.use(express.json());

app.post('/pessoas/:id/telefones', (req, res) => {
    const idPessoa = req.params.id;
    const telefone = req.body;
    console.log(idPessoa);
    console.log(telefone);
    res.send();
});
```

Note que o atributo `body` só ficou disponível depois de instalarmos um `middleware` de processamento de corpo de requisição.

TODO:

- Instalação e primeiro projeto
- Express generator
- Middlewares
- CRUD de tarefas refeito
    - Arquivos estáticos (front html, JS e CSS simples)
    - Roteamento (mountable routers)
    - Validação de dados de entrada e saída
    - Tratamentos de casos excepcionais
- Trabalhando com anexos (upload)
    - Tratar imagens de forma especial (crop/resize?)
        https://stackoverflow.com/questions/13938686/can-i-load-a-local-file-into-an-html-canvas-element
        https://github.com/fengyuanchen/cropperjs/blob/master/README.md#features
- Adicionando segurança (login)
    basic
    https://www.npmjs.com/package/express-session
    guid localstorage/sessionstorage
    jwt no localstorage/sessionstorage