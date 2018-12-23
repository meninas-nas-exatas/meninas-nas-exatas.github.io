---
layout: post
draft: true
title: Crônicas 1
subtitle: R² negativo ?
bigimg: /img/path.jpg
comments: true
---

Com o coeficiente de determinação (R²) medimos a aderência do modelo aos dados experimentais na forma de uma proporção da variabilidade dos dados que é explicada pelo modelo. Já o coeficiente de determinação ajustado (R² adj), penaliza essa aderência do modelo pelo seu número de parâmetros, uma aplicação direta do [principio da parcimônia](https://en.wikipedia.org/wiki/Occam%27s_razor). Nenhuma novidade até aí. 

O problema surgiu quando a criatura me mostrou um output do `summary()` em R com o valor do R² ajustado negativo. A primeira reação foi realmente de dizer "seu modelo é ruim, vamos investigar os dados e propor um modelo melhor". A segunda reação foi entender essa loucura. A equação do R² é apresentada abaixo, onde `n` é o número de observações e `p` o número de parâmetros do modelo.

$$R^2_{adj} = 1 - \frac{n-1}{n - (p+1)}\left ( 1 - R^2 \right )$$

A equação da forma que é escrita deixa claro que é possível obter valores negativos, motivados por um número grande de parâmetros, um pequeno número de observações ou um R² pequeno. Respondida essa pergunta, vem a curiosidade de entender o que isto pode significar em termos práticos. Para isso, modificando a equação anterior para a condição em que o R² ajustado é igual ou menor a zero, encontramos as seguintes expressões.

$$1 - \frac{n-1}{n - (p+1)}\left ( 1 - R^2 \right ) \leqslant 0$$

$$\frac{n-1}{p}  \leqslant  \frac{1}{R^2}$$

O interessante aqui é observar a razão `(n-1)/p`. Seu numerador `n-1` representa o número de graus de liberdade do conjunto de dados que quando dividido pelo número de parâmetros é, na realidade, uma medida do grau de informação adicionado ao modelo.

Era comum nas aulas de estatística e de aprendizado de máquina, a regra prática de que para cada parâmetro adicionado ao modelo deveriamos adicionar 4 a 5 observações ao conjunto de dados. E não é que encontramos um expressão com esta razão definida? 

E se assumirmos que desejamos modelos com um R² de no mínimo 80%? 

Para que o R² ajustado seja numericamente superior a zero, nesse caso, iremos precisar de uma razão `(n-1)/p` superior à ... 5.

<p><img src="https://noliquidificador.files.wordpress.com/2012/07/brain-explode.jpg" alt="É isso mesmo amigos!" align="center"></p>

Pequenos prazeres da estatística.

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>
