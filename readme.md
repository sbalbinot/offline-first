# PouchDB / CouchDB

Foram projetados com foco em sincronização utilizando uma arquitetura **multi-master**, ao invés da tradicional master/slave. Ou seja, todas as instâncias aceitam gravações que serão replicadas entre todas elas (não existe apenas uma "fonte da verdade").

- Ambos são bancos de dados **Open Source**,  **NoSQL** e utilizam **JSON** para a formatação e armazenamento de dados. O **CouchDB** roda no servidor e o **PouchDB** roda no **navegador** ou ambiente **Node.js**.
- Utilizam uma API baseada em HTTP ao invés de drivers, podendo ser utilizados em qualquer linguagem (existem bibliotecas disponíveis para fornecer uma interface mais conveniente).
- Linguagem de consulta similar ao MongoDB chamada **Mango Query**

> [!WARNING]
> Por ter uma API baseada em HTTP, não é possível utilizar transações da maneira convencional. Há suporte para transação de documento único, porém se as operações envolvem múltiplos documentos será necessário implementar uma lógica de transação customizada. Como a criação de um documento que representa a transação e contém os IDs de todos os documentos envolvidos, por exemplo.

## Sincronização

Durante a sincronização, todas as alterações feitas localmente são enviadas para o **CouchDB**, e todas as alterações feitas no **CouchDB** que ainda não foram replicadas localmente são puxadas para o **PouchDB**. A sincronização pode ser **bidirecional** (troca de informação entre cliente e servidor) ou **unidirecional** (apenas um lado gera dados para serem consumidos pelo outro).

```javascript
var localDB = new PouchDB('mylocaldb')
var remoteDB = new PouchDB('http://localhost:5984/myremotedb')

// Exemplos de sincronização unidirecional

// replicar do banco remoto para o local
localDB.replicate.from(remoteDB);

// replicar do banco local para o remoto
localDB.replicate.to(remoteDB)

// Exemplo de sincronização bidirecional
localDB.sync(remoteDB)
```

A sincronização pode ser feita de forma automática pela biblioteca do **PouchDB**, que fica tentando reestabelecer a conexão a cada intervalo de tempo (esse intervalo entre as tentativas não é especificado na documentação).

```javascript
localDB.sync(remoteDB, {
    live: true,
    retry: true
}).on('change', function (change) {
    // yo, something changed!
}).on('paused', function (info) {
    // replication was paused, usually because of a lost connection
}).on('active', function (info) {
    // replication was resumed
}).on('error', function (err) {
    // totally unhandled error (shouldn't happen)
});
```

Também é possível cancelar a sincronização, caso o usuário faça logout por exemplo.

```javascript
var syncHandler = localDB.sync(remoteDB, {
    live: true,
    retry: true
});

syncHandler.on('complete', function (info) {
    // replication was canceled!
});

syncHandler.cancel(); // <-- this cancels it
```

É possível também acompanhar as alterações feitas no banco através **changes feed**, que é uma API que fornece um fluxo contínuo de atualizações que representam todas as alterações. Pode ser utilizado para sincronizar dados do CouchDB para outro banco ou serviço, por exemplo.

## Resolução de conflitos

Por padrão a resolução de conflitos é feita de forma automática através de um algoritmo determinístico (sempre escolherá a mesma revisão vencedora, não importa quantas vezes seja executado ou em que ordem as revisões foram recebidas). O algoritmo escolhe a "revisão vencedora" com base nos seguintes critérios, **em ordem**:
1. **Número da revisão:** A revisão de maior número vence. Por exemplo, entre `2-def` e `3-ghi`, a `3-ghi` venceria porque 3 é maior que 2.

2. **Hash do conteúdo:** Se o número da revisão for o mesmo, a revisão com o hash de conteúdo lexicograficamente maior vence. Por exemplo, entre `2-abc` e `2-def`, a `2-def` venceria porque "def" é lexicograficamente maior que "abc".

3. **Origem da revisão:** Se o número da revisão e o hash do conteúdo forem os mesmos, a revisão que foi recebida primeiro pelo banco de dados vence.

Há a possiblidade de resolver conflitos manualmente aplicando lógicas customizadas, como por exemplo buscar todas as revisões conflitantes e perguntar ao usuário qual é a "vencedora", ou em um aplicativo como o Notion poderia ser decidido como revisão vencedora o documento que tem mais palavras, ou um aplicativo de entrega de EPIs caso os mesmos sejam entregues ao mesmo tempo em múltiplos lugares, somar a quantidade de entregas registradas.

As revisões "perdedoras" não são imediatamente excluídas, elas ficam flagadas caso seja necessário recuperá-las. Elas não são replicadas entre os dois bancos de forma automática (permanecem apenas no banco local, mas podem ser replicadas manualmente). É necessário realizar limpeza periódica para economizar espaço e melhorar o desempenho, através de um processo chamado "compaction".

```json
// exemplo de um registro com conflito

{
  "_id": "123",
  "_rev": "2-abc",
  "_conflicts": ["2-def"],
  "title": "Capacete",
  "delivered": true
}
```

## Tolerância a falhas

A arquitetura distribuida permite a existência de múltiplas cópias do banco de dados que podem aceitar leituras e gravações, com alterações replicadas de forma assíncrona para alta disponibilidade. O modelo de consistência eventual garante que, após uma gravação, pode haver um atraso até que a gravação seja replicada para todas as cópias, mas eventualmente todos os estados concordarão. E o modelo de gravação **append-only** (novas gravações são sempre anexadas ao final) permite que o sistema se recupere de falhas retornando ao último estado consistente.

# AWS Datastore / AppSync

Ambos da AWS, onde o **AWS Datastore** é uma biblioteca que fornece uma interface de para comunicação com banco de dados locais (IndexedDB e SQLite) e o **AppSync** é um serviço que roda na nuvem fornecendo uma API (GraphQL) para conectar com serviços remotos da AWS (DynamoDB, por exemplo).


## Sincronização

1. Alterações de dados utilizando o **AWS Datastore** são armazenadas localmente
2. **AWS Datastore** monitora o status da conexão de rede para quando estiver disponível sincronizar as alterações com o serviço **AppSync**
3. O **AppSync** recebe as informações e aplica no banco de dados remoto. Se houver conflitos, o própio serviço resolve utilizando a estratégia configurada no painel AWS AppSync.
4. O **AppSync** então envia as alterações confirmadas de volta para o **AWS Datastore**, que atualiza o armazenamento local.
> [!TIP]
> O **AppSync** também pode enviar atualizações em tempo real para o **AWS Datastore** quando outros clientes fazem alterações nos dados, através da função `observe`.
```javascript
import { DataStore } from "@aws-amplify/datastore";
import { Todo } from "./models";

// Inicia a sincronização
async function syncTodos() {
  await DataStore.start();

  console.log("DataStore started and syncing with backend");
}

// Subscrição para atualizações em tempo real (tanto locais quanto remotas)
const subscription = DataStore.observe(Todo).subscribe(msg => {
  console.log(msg.model, msg.opType, msg.element);
});

// Exemplo de criação de um registro
async function createTodo() {
  const newTodo = await DataStore.save(
    new Todo({
      name: "My offline todo",
      description: "This todo was created while offline"
    })
  );

  console.log("New todo created: ", newTodo);
}
```

## Resolução de conflitos
Existem três estratégias de resolução de conflitos disponíveis no **AppSync**, que podem ser configuradas diretamente no painel AWS.

1. **Automerge:** Estratégia padrão. Se as alterações forem feitas em diferentes partes do registro, ambas serão mescladas automaticamente. Se as alterações forem feitas na mesma parte, a alteração mais recente ganhará.
2. **Optimistic concurrency:** Essa estratégia usa um token de versão para detectar conflitos. Se o token de versão no cliente não corresponder ao token de versão no servidor, um erro será retornado.
3. **Lambda:** Permite ser criado uma função lambda personalizada para resolver conflitos.

## Tolerância a falhas
Os serviços têm políticas de retentativa automática para operações de redem, se uma solicitação falhar devido a uma falha temporária, ela será automaticamente retentada. Do lado do servidor, o **AppSync** roda na infrastrutura da AWS podendo escalar automaticamente para lidar com grandes volumes de tráfego e é redudante para proteger contra falhas de ponto único.


# Outras menções

## RxDB

Assim como o **PouchDB**, também suporta o **CouchDB** como banco remoto. Porém, há algumas diferenças:
1. **Reatividade:** RxDB é construido em torno do conceito de programação reativa, o que significa que pode se inscrever para mudanças nos dados e reagir à elas em tempo real.
2. **Validação de Schema**: Suporte a validação de schema usando **JSON Schema**.
3. **Criptografia**: Suporte a criptografia de dados.
> [!WARNING]
> Pode haver uma curva de aprendizado caso não estiver familiarizado com programação reativa.
> [!TIP]
> O PouchDB pode oferecer suporte à validação de schema e criptografia através de plugins (bibliotecas de terceiros).

A maioria dos plugins oficiais são pagos.

Suporta replicação com outras tecnologias também: HTTP, GraphQL, Websocket, Firestore, NATS etc.


## PowerSync + SQLite

Possui SDKs para react native e flutter (para PWAs está em beta).

## Realm + MongoDB Realm