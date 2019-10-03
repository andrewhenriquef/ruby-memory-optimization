- [Índice](https://github.com/andrewhenriquef/ruby-memory-optimization)
- [1ª parte - Introdução](https://github.com/andrewhenriquef/ruby-memory-optimization/blob/master/introduction.md)
- [2ª parte - Análise](https://github.com/andrewhenriquef/ruby-memory-optimization/blob/master/optimization.md)

# Otimizando o uso de memória dos workers de nossa aplicação rails, parte 3

Nessa terceira parte falo sobre as otimizações que realizamos nos relatórios. Mudanças essas que trouxeram para baixo o nosso uso de memória. Baixamos o custo de memória de 3.518mb para cerca de 480mb.

## Desacoplando a escrita dos relatórios

A primeira melhoria que aplicamos ao sistema foi o desacoplamento do código que faz a escrita dos relatórios. Criamos um micro serviço que recebe os dados em formato json, escreve os dados e gospe de volta um relatório nos formatos xlsx ou csv.

Fazendo isso economizamos aproximadamento `1.248mb` da nossa memória. Estavamos gastando o total de `3.518mb` para gerar o relatório de vendas. Ao remover a parte de escrita dos relatórios do sistemas passamos a gastar `2.270mb` do nosso sistema principal.

O custo dessa alteração é que teriamos de refatorar a parte do código que faz a extração de dados. Essa parte ela usava uma interface que sair criando objetos a rodo, um objetos por celula contento o dado extraído + o formato da célula(data, moeda, porcentagem, etc) e a saída dessa interface não era em um formato json.


## Reescrita da extração dos dados

 Por conta de removermos a escrita dos dados no arquivo xlsx/csv do nosso sistema, tivemos de reescrever o como é feito a extração de dados no sistema, para que pudessemos enviar um json para o novo micro-serviço:
 No modelo antigo de extração dos dados utilizavamos uma interface que funcionava como uma fabrica de objetos. Para cada célula extraíamos, formatavamos os dados e criavamos um novo objeto. Cada novo objeto continha o valor extraído e o formato dessa célula(data, moeda, etc). Reescrevemos essa interface para que ela gere um hash com os dados formatados e com os metadados necessários para gerar o relatório corretamente. No final, pegamos o resultado dessa interface, convertemos o hash para json e enviamos para o micro-serviço gerar o relatório.

Com as alterações realizadas conseguimos reduzir o custo de memória que tinhamos em nossa aplicação. Medindo o uso de memória obtivemos os seguintes resultados:

### Especificações da máquina

- Memória disponível: 15,5gb
- Intel® Core™ i7-7500U CPU @ 2.70GHz × 4
- Intel® HD Graphics 620 (Kaby Lake GT2)

### Modelo velho de extração dos dados

- Pico de uso de memória: 29.6% equivalente a 4,5gb(4588mb)
- Ao termino da extração dos dados manteve alocado 4.2gb(4223.96mb)

### Modelo novo de extração dos dados

- Pico de uso de memória: 14.2% equivalente a 3.6gb(3621mb)
- Ao termino da extração dos dados manteve alocado 2.2gb(2244.59mb)

Com a retirada da parte de escrita e com a reescrita da camada de extração dos dados dos relatórios conseguimos evitar o uso de aproximadamente 2gb de nossa memória.

Conseguimos uma redução de praticamente 50% do uso da memória, contudo ainda estavamos longe do que alvejavamos. Então utilizamos de algumas coisas que o Ruby On Rails nos oferece como alternativas para controlar e reduzir o uso de memória.


### Método find_each({})

Esse método tem a intenção de ser usado quando uma grande quantidade de dados não cabem na memoria de uma só vez. Ele divide a query em lotes de 1000 por padrao,

This method is only intended to use for batch processing of large amounts of records that wouldn’t fit in memory all at once. If you just need to loop over less than 1000 records, it’s probably better just to use the regular find methods.
 falar sobre o find_each e explicar que não estava desalocando toda a memoria necessaria, que ai vieram as mudanças para lazy loading

### lazy loading e Enumerator#to_json

  Evita o uso de memoria mas desconta no processamento
  falar sobre a implementação que faz com que o to_json se torne lazy, evitando assim o uso de memória excessivo, olhar no artigo que faz isso para explicar melhor, pois é meio confuso
  [link para o enumerator_to_json](https://gist.github.com/brianhempel/56823fb777fb567b676a)

  datas:

  mb 2062.59 using find_each(batch_size: 1000)

  all with lazy here:
  mb 1834.16 using to_json
  mb 1522.98 using JSON.generate

### uncached

  Força com que o rails não cacheie os resultados das queries

### JSON.generate vs .to_json

 Explicar aqui o porque o JSON.generate usa menos memória que o to_json

### solução ou workaround em cima disso

### outras ferramentas não exploradas

- ferramentas
- o que olhar e como


# report datas

# OLD (29.6 * 15500) / 100 = pico de 4588 mb/ manteve alocado = 4223.96
# NEW (14.2 * 15500) / 100 = pico de 3621mb / 2244.59mb manteve alocado

# old way results

file size 32.8 mb (csv)

22.7 % de 15.5 gb -> 3.518 mb usados para gerar um arquivo de 32.8 mb

9 min para gerar

# new way resultd


