# Criando um Modelo de Recomendação de Filmes com Google Cloud

Este relatório descreve o processo completo de criação de um modelo de recomendação de filmes utilizando a plataforma Google Cloud. A tarefa foi realizada como parte das atividades da disciplina de Sistemas Distribuídos 

## Passo 1: Importação do Conjunto de Dados

1. Na sessão do Cloud Shell, executei os seguintes comandos para baixar e descompactar o conjunto de dados de origem:

```shell
wget https://files.grouplens.org/datasets/movielens/ml-latest.zip
unzip ml-latest.zip
```

2. Criei um bucket de armazenamento no Cloud Storage e fiz o upload dos dados para ele:

```shell
gsutil mb gs://meu-projeto-movielens-data
gsutil cp ml-latest/movies.csv ml-latest/ratings.csv \
  gs://meu-projeto-movielens-data
```

3. Criei um conjunto de dados no BigQuery:

```shell
bq mk movielens
```

4. Carreguei o arquivo `movies.csv` em uma nova tabela chamada `movies` no BigQuery:

```shell
bq load --skip_leading_rows=1 movielens.movies \
  gs://meu-projeto-movielens-data/movies.csv \
  movieId:integer,title,genres
```

5. Carreguei o arquivo `ratings.csv` em uma nova tabela chamada `ratings` no BigQuery:

```shell
bq load --skip_leading_rows=1 movielens.ratings \
  gs://meu-projeto-movielens-data/ratings.csv \
  userId:integer,movieId:integer,rating:float,time:timestamp
```

## Passo 2: Criação de Visualizações no BigQuery

1. Criei uma view que converteu a tabela `movies` em um retail product catalog schema:

```shell
bq mk --project_id=meu-projeto \
 --use_legacy_sql=false \
 --view '
 SELECT
   CAST(movieId AS string) AS id,
   SUBSTR(title, 0, 128) AS title,
   SPLIT(genres, "|") AS categories
 FROM `meu-projeto.movielens.movies`' \
movielens.products
```

2. Acessei o BigQuery na barra lateral esquerda, e abri a página de consulta da view. através de meu projeto -> movielens -> products


## Passo 3: Convertendo classificações de filmes em eventos de usuário

1. Para converter as classificações de filmes em eventos de página de detalhes do produto para os usuários, segui os seguintes passos:

   - Ignorei classificações de filmes negativas (<4).
   - Tratei cada classificação positiva como um evento de visualização de página de produto (detail-page-view).
   - Redimensionei a linha do tempo do Movielens para os últimos 90 dias.

2. Criei uma visualização no BigQuery usando o seguinte comando SQL que atende aos requisitos de conversão para o Retail API:

```shell
bq mk --project_id=meu-projeto \
 --use_legacy_sql=false \
 --view '
 WITH t AS (
   SELECT
     MIN(UNIX_SECONDS(time)) AS old_start,
     MAX(UNIX_SECONDS(time)) AS old_end,
     UNIX_SECONDS(TIMESTAMP_SUB(
       CURRENT_TIMESTAMP(), INTERVAL 90 DAY)) AS new_start,
     UNIX_SECONDS(CURRENT_TIMESTAMP()) AS new_end
   FROM `meu-projeto.movielens.ratings`)
 SELECT
   CAST(userId AS STRING) AS visitorId,
   "detail-page-view" AS eventType,
   FORMAT_TIMESTAMP(
     "%Y-%m-%dT%X%Ez",
     TIMESTAMP_SECONDS(CAST(
       (t.new_start + (UNIX_SECONDS(time) - t.old_start) *
         (t.new_end - t.new_start) / (t.old_end - t.old_start))
     AS int64))) AS eventTime,
   [STRUCT(STRUCT(movieId AS id) AS product)] AS productDetails,
 FROM `meu-projeto.movielens.ratings`, t
 WHERE rating >= 4' \
movielens.user_events
```

## Passo 4: Importando catálogo de produtos e eventos de usuário no Retail API

1. Agora que as visualizações estão prontas, foi o momento de importar o catálogo de produtos e os dados de eventos de usuário no Retail API. Segui estas etapas:

   - Ativei o Retail API.

   - Acessei a página "Retail Data" no console do Google Cloud.

   - Cliquei em "Importar".

### Importando catálogo de produtos

2. Preenchi o formulário para importar produtos da visualização BigQuery que criei anteriormente:

   - Selecionei o tipo de importação: Product Catalog.
   - Escolhi a branch padrão.
   - Selecionei a fonte dos dados: BigQuery.
   - Escolhi o esquema dos dados: Retail Product Schema.
   - Inseri o nome da view de BigQuery de produtos que criei (PROJECT_ID.movielens.products).

   Cliquei em "Importar".

3.  Esperei até que todos os produtos fossem importados, o que levou de 5 a 10 minutos.

### Importando eventos de usuário

4. Importei a visualização BigQuery de eventos de usuário:

   - Selecionei o tipo de importação: User Events.
   - Escolhi a fonte dos dados: BigQuery.
   - Selecionei o esquema dos dados: Retail User Events Schema.
   - Inseri o nome da visualização BigQuery de eventos de usuário que criei anteriormente.

   Cliquei em "Importar".

5. Esperei até que pelo menos um milhão de eventos fossem importados antes de prosseguir para a próxima etapa, a fim de atender aos requisitos de dados para treinar um novo modelo.

## Passo 5: Treinando e avaliando modelos de recomendação

6. Para criar um modelo de recomendação, fiz o seguinte:

   - Acessei a página "Retail Models" no console do Google Cloud.
   - Cliquei em "Criar modelo".
   - Dei um nome ao modelo.
   - Selecionei "Others you may like" como o tipo de modelo.
   - Escolhi "Click-through rate (CTR)" como objetivo de negócios.
   - Cliquei em "Criar".

7.  O treinamento leva cerca de dois a cinco dias para ser concluído


## Passo 6: Visualizando recomendações e resultados

1. Depois que o modelo estava pronto para consulta:

   - Acessei a página "Retail Serving Configs" no console do Google Cloud.
   - Cliquei na aba "Evaluate".
   - Inseri o ID de um filme. No vídeo, escolhi 4993, que é "O Senhor dos Anéis: A Sociedade do Anel".
   - Cliquei em "Prediction Preview" para ver a lista de itens recomendados com base no ID que enviei como input.

