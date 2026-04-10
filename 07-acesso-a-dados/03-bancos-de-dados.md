# Bancos de Dados

## Geral

Numa aplicação com grande pressão de escrita, digamos um e-commerce em dia de Black Friday, fechando várias compras por segundo, você deve programar sua aplicação para usar **filas**. Toda nova compra entra na fila e vai sendo escrito no banco à medida que der. Você nunca pode só aumentar a quantidade de servidores de aplicação web e deixar conectar todo mundo no mesmo banco ao mesmo tempo, isso só vai aumentar a contenção e no final ninguém consegue escrever.

É melhor todo mundo entrar numa fila e só a fila gerenciar o banco. As leituras usam caches de Redis e isso tira a carga de pesquisa no banco.

O grande gargalo numa aplicação web é o banco de dados, porque mesmo com replicação e sharding, você sempre vai ter contenção por causa das garantias ACID.

## Bancos NoSQL

Diferente de um banco relacional, a maioria não é nem um banco transacional e nem um banco analítico, ou seja, que você pode sair fazendo queries como em SQL com qualquer critério e a qualquer momento.

Pesquisas em bancos NoSQL precisam ser **planejadas, indexadas com antecedência**. NoSQLs tendem a ser bancos que funcionem em cluster, com shards, ou seja, nenhum servidor tem todos os dados que você precisa ao mesmo tempo, e se sua pesquisa for mal planejada, ela vai ter que ir vasculhando em todos os nós da rede, e isso custa processamento e custa tempo, muito tempo.

## Bancos Relacionais

Num banco relacional, o que mais custa performance é escrever, mais do que ler. A escrita precisa ter o tal ACID garantido: **atomicidade, consistência, isolamento e durabilidade**. Dados escritos precisam garantidamente estar no disco de tal forma que uma pane inesperada não faça você perder dados.

Múltiplas conexões escrevendo ao mesmo tempo não podem um pisar no pé do outro, você precisa de um esquema de concorrência como **multiversion concurrency control ou MVCC** que a maioria dos bancos usa hoje em dia. E você precisa de **transaction logs** pra conseguir dar rollback ou voltar ao estado original dos dados se uma transação de múltiplas operações dá pau na metade.

## 1. MongoDB

É um banco de dados NoSQL. Todas as operações são feitas em RAM, é o que garante sua performance. Mas isso significa que se a máquina der pane antes da RAM ter chance de dar flush pro disco, **você pode perder dados**.

Você pode configurar pra que ele garanta mais durabilidade. Por exemplo, hoje quando você manda gravar alguma coisa no Mongo ele devolve ok antes de ter realmente gravado. O ok do Mongo é mais tipo "belê, recebi a ordem pra gravar, quando der eu gravo". Diferente de um banco relacional que só dá ok se garantidamente o dado foi gravado.

Você pode configurar o Mongo pra devolver ok só se garantidamente uma outra instância de replica master recebeu o dado. Mas aí você perde a performance que era justamente o que te fez escolher Mongo em primeiro lugar.

## 2. Cassandra

Um NoSQL wide column store. Apesar de ter similaridades com bancos relacionais só na superfície — especialmente nos schemas e no CQL que é similar a SQL — não tem absolutamente nada a ver. Eles são como **key-value stores multi dimensionais com column families** e foram feitos pra serem bancos altamente distribuídos, com múltiplos nós num cluster, de preferência em múltiplas regiões diferentes.

Cassandra **não foi feito pra usar numa única instância**, como banco de um blog. O sweet spot dele é funcionar em cluster, com múltiplos nós, e com muita escrita.

## 3. Redis

Na prática a maioria de nós vai sempre acabar usando uma **combinação de um banco relacional com um cache**, usando algo como um Redis. Nessa configuração o Redis pode até ser um pouco mais relaxado porque o certo é fazer com que os dados sejam consultados no Redis e o que não estiver lá você carrega do banco relacional e grava no Redis à medida que for precisando.

Portanto o Redis pode ser reconstruído do zero mesmo se der pane e precisar derrubar tudo. E você usa o Redis pra manter coisas pesadas de calcular e gravar num banco relacional como **agregações**, coisas com contadores, médias e outras métricas que agregam múltiplas linhas do banco.

## Dicas de Performance

1. **Desnormalização controlada** — Quanto mais normalizada a tabela, mais lento vai ficar pois vai exigir mais do banco. Ex: ao invés de criar uma tabela ESTADOS (São Paulo, Minas Gerais), a sigla e nome do estado podem ser salvos diretamente na tabela de usuário. Isso irá deixar o banco com menos joins e aumentar a performance.

2. **Cache** — Colocar cache para evitar ao máximo ir até o banco.

3. **Filas para escrita** — Programar a aplicação para deixar o pedido de escrita do banco em uma fila (como RabbitMQ), para evitar gargalos e dar timeout para o usuário.

4. **Full-text search** — Quando precisar fazer pesquisa textual e mostrar resultados por relevância (como busca em e-commerces), o mais correto e performático é usar algo como **ElasticSearch**.

5. **Índices** — Criar os índices necessários, mas não criar demais pois pode prejudicar a performance (cada índice precisa ser atualizado a cada escrita).

---

[← Anterior: Entity Framework](02-entity-framework.md) | [Voltar ao índice](README.md)
