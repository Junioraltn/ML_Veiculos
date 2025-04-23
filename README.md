# 🚗 Clusterização de carros com PySpark: Uma viagem com Aprendizados Reais

Durante a Pós-Graduação um dos meus professores fez uma apresentação incrivel mostrando como era feito a clusterização de dados utilizando Apache Spark, eu fiquei maravilhado, mas não tive a oportunidade de desenvolver a ideia aprensentada na prática. Depois que tive algumas orientações e concluir alguns cursos pelo programa ONE, eu me senti capaz e fiquei convicto que poderia tentar replicar o que o professor fez. No inicio do ano eu tinha conseguido tal feito, mas não estava satisfeito, muitas questões me vieram a mente: "E se?", "Por que isso?", "Tem outra forma?", "essa é a melhor forma?", "Como isso é feito?", enfim, hoje eu conclui e apresento a vocês o que eu fiz além do que me foi apresentado.

Em resumo o Apache Spark é uma poderosa ferramenta para processamento distribuido. E esse projeto tem como objetivo explorar dados de automóveis e agrupalos em clusters com base em caracteristicas como conumo, potência e tipo de carro.

Abaixo está a estrutura do DataFrame:

![Schema do Dataframe](https://github.com/user-attachments/assets/c67800f9-11fd-4be7-9cc1-8cb3504509b4)

E antes de começar é bom visualizar as 5 primeiras linhas do Dataframe:
![Top 5 das linhas do DF](https://github.com/user-attachments/assets/6c3ba80d-035e-41f4-bc19-a0d168a6407f)


## ⚙️ Versão Inicial — Simplicidade e Controle Manual
Essa é a versão que me foi apresentado em aula, onde o foco estava em mostrar cada etapa do processo. Foi ótimo, pois nos dá controle do processo controle, todavia tornou o código mais extenso e difícil de escalar.

Nessa versão, todo o pré-processamento foi feito "a mão" como:
> 🛠️ Converção de variáveis categóricas:
<p>De forma simplificada, para os casos em que os itens no dataframe correspondia aos que estavam em List (Lista de pontas ou de tipo de carro), era atribuido ao valor da categoria; sua posição na "List" mais 1. Quando não correspondia era atribuido o valor "2.0". E isso era feito para cada coluna em "Colunas".</p>

~~~python
def transform (coluna,Df,List):
    for item in List:
        i = List.index(item) + 1
        Df = Df.withColumn(coluna, when(Df[coluna] == item, float(i) ).otherwise(2.0))
        
    return Df

# Percorrer entre colunas e listas de valores unicos
Colunas = ["portas","tipo"]
Listas = [portas,tipos]
for i in range(0,len(Colunas)):
    CarrosDF2 = transform(Colunas[i],CarrosDF2,Listas[i])
~~~
Um dos motivos de eu desenvolver mudanças no código é devido ao caso anterior. A função percorre a coluna do Dataframe inteira para todo o "Item" em "List", além de ser ineficiente sobrescreve os valores da coluna a cada nova interação. A solução que pensei inicialmente era retirar as linhas que correspondiam aos itens da lista e depois adicioná-los ao dataframe principal. A ideia era só realizar a transformação nos casos em que havia correspondência e depois de percorrido a "List" realizar a transformação nos casos que não correspondia. Pensei também na possibilidade apenas listar a posição de cada linha que não correspondia e ir percorrendo do Dataframe através dessas posições em uma lista por exemplo, e ir retirando a cada nova correspondência, no fim, teria uma lista com a posição de cada linha que não houve correspondência com "List" e assim seria feito a transformação nesses casos. Esta ultima alternativa seria preferível ao meu ver.
> 🎛️ Normalização estatística manual usando médias e desvios:
<p>Aproveitando o ".describe()" do pandas e convertendo-o para dataframe, foi possivel gerar as médias e desvio padrão para as colunas analisadas. A normalização foi feita para cada "i" em um intervalo de 0 ao tamanho da lista de médias calculadas.</p>

~~~python
#💉Extraindo a média e o desvio padrão
medias = descricaoNova.iloc[1,1:6].values.tolist()
desvios = descricaoNova.iloc[2,1:6].values.tolist()
~~~

~~~python
# Função para Normalizar e criar os vetores densos
def centerAndScale(inRow):
  #Variáveis Globais
  global bc_media
  global bc_desvio
  #Array de médias e desvios
  meanArray = bc_media.value
  stdArray = bc_desvio.value
  #Array para o resultados
  retArray= []
for i in range(len(meanArray)):
    retArray.append((float(inRow[i])-float(meanArray[i]))/float(stdArray[i])) # 🎛️ Normalização
  return Row(features = Vectors.dense(retArray)) # 🪄 Criação de vetores densos com Vectors.dense:
~~~
## 🔁 Mudança de Rota — Explorando o Pipeline do Spark MLlib

Percebi que poderia melhorar a robustez e modularidade do código utilizando o Pipeline do Spark MLlib.

~~~python
pipeline = Pipeline(stages=[indexer_portas, indexer_tipo, encoder, assembler, scaler])
~~~

Então Implementei:

* StringIndexer + OneHotEncoder para tratar variáveis categóricas:
~~~python
indexer_portas = StringIndexer(inputCol="portas", outputCol="portas_indexadas")
indexer_tipo = StringIndexer(inputCol="tipo", outputCol="tipo_indexado")
~~~
A classe StringIndexer converte colunas categóricas em índices numéricos, atribuindo um número inteiro a cada categoria.
O índice atribuído é baseado na frequência da categoria, onde a categoria mais frequente recebe o índice 0, a segunda mais frequente recebe o índice 1 e assim por diante.
* VectorAssembler para consolidar as features:

~~~python
assembler = VectorAssembler(inputCols=["portas_encoded", "tipo_encoded", "horsepowerf", "rpmf", "consumo_cidadef"], outputCol="features_nao_escaladas")
~~~

O VectorAssembler combina várias colunas em uma única coluna de vetor.
O parâmetro inputCols especifica as colunas de entrada que serão combinadas, e o parâmetro outputCol especifica o nome da coluna de saída que conterá o vetor resultante.
O parâmetro handleInvalid="skip" é usado para lidar com valores inválidos durante a transformação. Nesse caso, os valores inválidos serão ignorados e não incluídos no vetor resultante.

* StandardScaler para normalização:
  
~~~python
scaler = StandardScaler(inputCol="features_nao_escaladas", outputCol="features", withStd=True, withMean=True)
~~~

O StandardScaler é usado para padronizar as features, ou seja, ajustar a média e o desvio padrão de cada feature para que tenham média 0 e desvio padrão 1.
O parâmetro withStd=True indica que o escalonamento deve ser feito com base no desvio padrão, enquanto withMean=True indica que a média deve ser subtraída.
Isso é útil para garantir que todas as features tenham a mesma escala e contribuam igualmente para o modelo de aprendizado de máquina.

* KMeans para clusterização:
~~~python
# Criando modelo
kmeans = KMeans(k=4, seed=1)
# Treinando o modelo
modelo = kmeans.fit(carrosDfPreparado.select("features"))
~~~

O modelo é treinado usando o método fit, que ajusta o modelo aos dados de entrada. O resultado é um modelo KMeans treinado que pode ser usado para prever os clusters dos dados.
O modelo KMeans é um algoritmo de aprendizado não supervisionado que agrupa os dados em k clusters com base nas características fornecidas.
O parâmetro seed é usado para garantir a reprodutibilidade dos resultados.

> *Essa mudança exigiu aprender algumas novas práticas, mas deixou o projeto preparado para datasets maiores.*

## Resultado:
O gráfico abaixo mostra a distribuição dos veiculos em função do "Housepower" e "Concumo_cidade". O modelo identificou 5 grupos distintos de carros que possuem perfis semelhante, e são representado por cores diferentes no gráfico. Esse agrupamento (segmentação) considerou variáveis como potência, consumo, tipo do carro e número de portas.


![Gráfico de Dispersão](https://github.com/user-attachments/assets/0a93ef07-3909-41c4-91ea-7c9a74755fee)


Relação de Vaiculos por Clusters:
![Gráfico de Barras](https://github.com/user-attachments/assets/66ecc819-c0f4-4f15-b2fd-82b638529d63)


## 📊 Próximos passos
Agora que a estrutura está bem definida, pretendo:

* Tentar caracterizar os clusters realizando "groupBy" dos dados pela predição ou por outra forma;
* Aplicar o modelo em datasets maiores;
* Explorar outros métodos de avaliação além da silhueta;
* Testar pipelines com outras técnicas de normalização e redução de dimensionalidade (como Análise de Componentes Principais - PCA);

## ✍️ Publicado por:
Ailton Júnior – Estudante em [Gestão de Big Data e Anlise de negocios]
#Spark #BigData #MachineLearning #DataScience #Python #Clusterização #Storytelling
