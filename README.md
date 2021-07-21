# projeto_semantix
## Campanha Nacional de vacinação contra COVID

O projeto foi dividido em dois níveis, básico e avançado. Recomendo fortemente fazer primeiro o básico e se sobrar tempo, pode aventurar no avançado.

Os exercícios podem ser feitos em qualquer linguagem e todas as questões são bem abertas, tendo várias formas de serem realizadas e interpretadas, pois a idéia é não termos projetos iguais.

O projeto deve estar no *github.com*, a forma de organizar o conteúdo é por sua conta, caso nunca tenha usado, este já é seu primeiro desafio.

Ao final do projeto você precisa preencher o formulário com o seu nome completo, e-mail utilizado no treinamento e o link do github do seu projeto.

-Link do formulário para envio: https://forms.office.com/Pages/ResponsePage.aspx?id=2H_OZbilA0GZftoGjNhf1Y4a9bNsmMNEil2MBcFKJolUMFlTQVBNUVhRTVlSNVJUUDBWM0ZIRDVKQS4u

O formulário também estará na página do treinamento.

OBS: Todas as imagens de exemplo (Visualizações) são apenas para referencias, o projeto irá ter valores diferentes e as formas de se criar tabelas com dataframe/dataset das visualizações, pode ser realizado da maneira que preferir.

---
## Nível Básico

Dados:  https://mobileapps.saude.gov.br/esus-vepi/files/unAFkcaNDeXajurGB7LChj8SgQYS2ptm/04bd3419b22b9cc5c6efac2c6528100d_HIST_PAINEL_COVIDBR_06jul2021.rar

Referência das visualizações:
  - Site: https://covid.saude.gov.br
  - Guia do site: Painel Geral

---

*O projeto foi realizado utilizando Docker e Docker-Compose através do WSL2*

Primeiramente, entrar na pasta onde estão instalados os arquivos docker-compose e inicializar os serviços através do ```docker-compose up -d```

Os arquivos .yml estão disponíveis em : XXXXX

### 1. Enviar os dados para o hdfs

#### 1.1 - Primeiramente é feito download dos dados e posterior extração dos arquivos em pasta local.
   
#### 1.2 - Utilizar ```docker cp``` para fazer a cópia dos arquivos para o namenome, através do segunite script:
   
   ```
   docker cp HIST_PAINEL_COVIDBR_2020_Parte1_06jul2021.csv namenode:/
   ```
   ```
   docker cp HIST_PAINEL_COVIDBR_2020_Parte2_06jul2021.csv namenode:/
   ```
   ```
   docker cp HIST_PAINEL_COVIDBR_2021_Parte1_06jul2021.csv namenode:/
   ```
   ```
   docker cp HIST_PAINEL_COVIDBR_2021_Parte2_06jul2021.csv namenode:/
   ```
   
#### 1.3 - Entrar no container namenode ```docker exec -it namenode bash``` e listar os arquivos através do ```ls```

![foto03_projeto_dev](https://user-images.githubusercontent.com/62483710/125337812-ceef0c80-e325-11eb-9aea-ef1feb4877b4.PNG)

#### 1.4 -De dentro do namenode, criar a seguinte estrutura de pasta ** hdfs:/user/projeto/semantix ** através do comando ```hdfs dfs -mkdir user/projeto/semantix ``` e enviar os arquivos para o HDFS através dos comandos:
   
   ```
   hdfs dfs -put HIST_PAINEL_COVIDBR_2020_Parte1_06jul2021.csv /user/projeto/semantix
   ```
   ```
   hdfs dfs -put HIST_PAINEL_COVIDBR_2020_Parte2_06jul2021.csv /user/projeto/semantix
   ```
   ```
   hdfs dfs -put HIST_PAINEL_COVIDBR_2021_Parte1_06jul2021.csv /user/projeto/semantix
   ```
   ```
   hdfs dfs -put HIST_PAINEL_COVIDBR_2021_Parte2_06jul2021.csv /user/projeto/semantix
   ```
   
#### 1.5 - Conferir se arquivos foram devidamente salvos no HDFS através de ``` hdfs dfs -ls /user/projeto/semantix```
   
   ![foto04_projeto_dev](https://user-images.githubusercontent.com/62483710/125338881-1e820800-e327-11eb-9d54-ade20e33fa21.PNG)

---       
### 2. Otimizar todos os dados do hdfs para uma tabela Hive particionada por município.

Ainda no container do namenode, dar um ```hdfs dfs -cat <file> | head ``` para visualizar o formato do arquivo e seu cabeçalho.

```
hdfs dfs -cat /user/projeto/semantix/HIST_PAINEL_COVIDBR_2021_Parte1_06jul2021.csv | head
```

![foto07_projeto](https://user-images.githubusercontent.com/62483710/125356531-b6d6b780-e33c-11eb-8bfd-50af6c0c6954.PNG)


#### 2.1 - Sair do container namenode através do comando *Ctrl+d* e entrar no container Hiver-Server através do comando:

```
docker exec -it hive-server bash
```

#### 2.2 - Conectar-se ao Hive e acessar ao cliente beeline para se conectar ao HiveServer2 através da porta localhost:10000 através do script:

```
beeline -u jdbc:hive2://localhost:10000
```

![foto05_projeto](https://user-images.githubusercontent.com/62483710/125340327-bb917080-e328-11eb-9aa9-26a9b4864d26.PNG)

#### 2.3 - Criar banco de dados *projeto_semantix*

```
create database projeto_semantix;
```

mostrar banco de dados criado: script ```show databases;```

![foto06_projeto_hive](https://user-images.githubusercontent.com/62483710/125344000-204eca00-e32d-11eb-99f4-fd51e799e650.PNG)

#### 2.4 - Criar tabela dados_covid para receber dados do HDFS

```
CREATE TABLE dados_covid(regiao string,
                         estado string,
                         municipio string,
                         cod_uf int,
                         cod_mun int,
                         cod_regiao_saude int,
                         nome_regiao_saude string,
                         data date,
                         semana_epi int,
                         populacaoTCU2019 int,
                         casos_acumulados int,
                         casos_novos int,
                         obitos_acumulado int,
                         obitos_novos int,
                         recuperados_novos int,
                         em_acompanhamento_novos int,
                         interior_metropolitana string)
                         row format delimited
                         fields terminated by ';'
                         lines terminated by '\n'
                         tblproperties("skip.header.line.count"="1");
```

#### 2.5 - Carregar dados do HDFS na tabela dados_covid:

```
load data inpath '/user/projeto/semantix' into table dados_covid;
```
---
### 3. Criar as 3 vizualizações pelo Spark com os dados enviados para o HDFS:

Utilizando o Jupyter Notebook através da porta localhost: 8889.

3.1 - Listar arquivos no hive/warehouse

```
! hdfs dfs -ls /user/hive/warehouse/
```

![foto08_spark](https://user-images.githubusercontent.com/62483710/125826064-3a052972-b69a-490e-8d61-2f3ee61d2d7e.PNG)


3.2 - Ler a tabela Hive 

```pyspark
tabela = spark.read.table("dados_covid")
```

3.3 - Visualizar schema

```pyspark
tabela.printSchema()
```

![SCHEMA](https://user-images.githubusercontent.com/62483710/126519678-cc8c8413-8358-4222-8db6-d82738111139.PNG)


3.4.1 - Utilizando *spark.sql* criar variáveis que salvam os indicadores:

```pyspark
### Criação das querys

total_casos_recuperados = spark.sql("select MAX (recuperados_novos) as Casos_Recuperados from dados_covid").show()
casos_em_acompanhamento = spark.sql("select LAST (em_acompanhamento_novos) as Em_Acompanhamento from dados_covid WHERE em_acompanhamento_novos IS NOT NULL").show()

total_casos_acumulados = spark.sql("select MAX (casos_acumulados) as Acumulado from dados_covid ").show()
casos_novos = spark.sql("select MAX (casos_novos) as Casos_novos from dados_covid where data = ('2021-07-06')").show()
incidencia = spark.sql("SELECT ROUND(((MAX(casos_acumulados) / MAX(populacaotcu2019))*100000),1) as incidencia from dados_covid where data = ('2021-07-06')").show()

###Calculo incidencia (casos confirmados * 1.000.000) / população.
###Calculo letalidade (mortes totais/casos totais)
###Calculo mortalidade (mortes totais/população)

total_obitos_acumulados = spark.sql("select MAX (obitos_acumulado) as Obito_acumulado from dados_covid ").show()
obitos_novos = spark.sql("select MAX (obitos_novos) as Obitos_novos from dados_covid where data = ('2021-07-06')").show()
letalidade = spark.sql("SELECT ROUND(((MAX(obitos_acumulado) / MAX(casos_acumulados))*100),1) as letalidade from dados_covid").show()
mortalidade = spark.sql("SELECT ROUND(((MAX(obitos_acumulado) / MAX(populacaotcu2019))*100000),1) as mortalidade from dados_covid").show()


#populacao = spark.sql("select MAX (populacaotcu2019) as populacao from dados_covid where data = ('2021-07-06')").show()

```
![indic_03](https://user-images.githubusercontent.com/62483710/126370889-bb5e36bf-e27c-4db4-bd89-7642cd12a3d8.PNG)
![indic_01](https://user-images.githubusercontent.com/62483710/126370891-0f6f5b34-caf1-469e-9e91-97c98ab4db7d.PNG)
![indic_02](https://user-images.githubusercontent.com/62483710/126370895-45b5970f-977f-47bf-aa7f-56a54c73f1d2.PNG)

3.4.2 - Criação das views:

Criação das views

```pyspark
spark.sql("use projeto_semantix")

VIEW_01 = spark.sql("ALTER VIEW Casos_Recuperados AS select 1, MAX (recuperados_novos) as total_casos_recuperados from dados_covid UNION select 2, MAX (em_acompanhamento_novos) as Em_Acompanhamento from dados_covid WHERE data = ('2021-07-06')")

VIEW_02 = spark.sql("ALTER VIEW Casos_confirmados AS select 1, (MAX (casos_acumulados)) as casos_confirmados from dados_covid UNION select 2, MAX (casos_novos) as Casos_novos from dados_covid where data = ('2021-07-06') UNION SELECT 3, ROUND(((MAX(casos_acumulados) / MAX(populacaotcu2019))*100000),1) as incidencia from dados_covid where data = ('2021-07-06')")

VIEW_ = spark.sql("ALTER VIEW Obitos_confirmados AS select 1, MAX (obitos_acumulado) as Obitos_confirmados from dados_covid UNION select 2, MAX (obitos_novos) as Obitos_novos from dados_covid where data = ('2021-07-06') UNION SELECT 3, ROUND(((MAX(obitos_acumulado) / MAX(casos_acumulados))*100),1) as letalidade from dados_covid UNION SELECT 4, ROUND(((MAX(obitos_acumulado) / MAX(populacaotcu2019))*100000),1) as mortalidade from dados_covid")

```
3.4.3 Visualização das views:

```pyspark
spark.sql("SELECT * from Casos_Recuperados").show()
spark.sql("SELECT * from Casos_confirmados").show()
spark.sql("SELECT * from Obitos_confirmados").show()
```

![visu_views](https://user-images.githubusercontent.com/62483710/126524088-81f2e96a-6efa-4390-89b7-00d623b9a69c.PNG)


![foto01_projeto](https://user-images.githubusercontent.com/62483710/125175523-287afe00-e1a3-11eb-8aea-4c59b9d79272.PNG)

4. Salvar a primeira visualização como tabela Hive



5. Salvar a segunda visualização com formato parquet e compressão snappy
6. Salvar a terceira visualização em um tópico no Kafka
7. Criar a visualização pelo Spark com os dados enviados para o HDFS:

![foto02_projeto](https://user-images.githubusercontent.com/62483710/125175529-36308380-e1a3-11eb-9306-aafa753b9b54.PNG)

8. Salvar a visualização do exercício 6 em um tópico no Elastic
9. Criar um dashboard no Elastic para visualização dos novos dados enviados

---

## Nível Avançado:

Replicar as visualizações do site “https://covid.saude.gov.br/”, porém acessando diretamente a API de Elastic.

Link oficial para todas as informações: https://opendatasus.saude.gov.br/dataset/covid-19-vacinacao

#### Informações para se conectar ao cluster:

  • URL https://imunizacao-es.saude.gov.br/desc-imunizacao 
  
  • Nome do índice: desc-imunizacao 
  
  • Credenciais de acesso 
  
      o Usuário: imunizacao_public 
      
      o Senha: qlto5t&7r_@+#Tlstigi 

#### Links utéis para a resolução do problema:

##### Consumo do API:
  
https://opendatasus.saude.gov.br/dataset/b772ee55-07cd-44d8-958f-b12edd004e0b/resource/5916b3a4-81e7-4ad5-adb6-b884ff198dc1/download/manual_api_vacina_covid-19.pdf

##### Conexão do Spark com Elastic:
  
https://www.elastic.co/guide/en/elasticsearch/hadoop/current/spark.html

https://docs.databricks.com/data/data-sources/elasticsearch.html#elasticsearch-notebook

https://github.com/elastic/elasticsearch-hadoop

https://www.elastic.co/guide/en/elasticsearch/hadoop/current/configuration.html

##### Instalar Dependências:
  
https://www.elastic.co/guide/en/elasticsearch/hadoop/current/install.html

---
Bom projeto e estudos

Desenvolvido por Rodrigo Rebouças
