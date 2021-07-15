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
3. Criar as 3 vizualizações pelo Spark com os dados enviados para o HDFS:

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
tabela.printSchema
```

![foto09_spark](https://user-images.githubusercontent.com/62483710/125826115-0e551ebe-ed62-456f-9ea9-17bda2a5f99c.PNG)

3.4 - Utilizando *spark.sql* criar variáveis que salvam os indicadores:

```pyspark
total_casos_acumulados = spark.sql("select max (casos_acumulados) as casos_acumulados from dados_covid group by cod_mun order by casos_acumulados desc limit 1").show()
total_casos_recuperados = spark.sql("select max (recuperados_novos) as novos_recuperados from dados_covid group by cod_mun order by novos_recuperados desc limit 1").show()
total_obitos = spark.sql("select max(obitos_acumulado) as obitos_acumulados from dados_covid group by cod_mun order by obitos_acumulados desc limit 1").show()
casos_novos = spark.sql("select max(casos_novos) as novos_casos from dados_covid where data = ('2021-07-06') group by cod_mun order by novos_casos desc limit 1").show()
obitos_novos = spark.sql("select max(obitos_novos) as novos_obitos from dados_covid where data = ('2021-07-06') group by cod_mun order by novos_obitos desc limit 1").show()

```
![FOTO11_SPARK](https://user-images.githubusercontent.com/62483710/125826145-20c38ebf-6fb4-4c27-bc95-2f5f4f6c2ed7.PNG)
![FOTO10_SPARK](https://user-images.githubusercontent.com/62483710/125826154-29f995a4-7c35-4a94-b4d0-e8f07e313dac.PNG)


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
