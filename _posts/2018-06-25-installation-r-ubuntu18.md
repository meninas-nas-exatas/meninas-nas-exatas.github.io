---
layout: post
draft: false
title: Instalando R e RStudio no Ubuntu 18
subtitle: Bionic Beaver meets R
bigimg: /img/2018-06-26-img1.png
tags: [r, rstudio, ubuntu, 18]
comments: true
---

No fim de abril foi lançado o [ubuntu 18](http://releases.ubuntu.com/18.04/) (bionic beaver). Um mês depois estava na [Campus Party Bahia](http://brasil.campus-party.org/cpbahia/) e precisei formatar meu computador. Após caçar um pendrive bootavel (e achar!), a prioridade foi instalar o R/Rstudio e continuar os trabalhos.

A instalação seguiu com o seguinte fluxo:

```
# Atualização do pacotes e instalação das versões mais novas

sudo apt-get update && sudo apt-get upgrade

# Instalação de pacotes necessários

sudo apt install libxml2-dev gdebi-core libcurl4-openssl-dev libssl-dev

# Instalação do R

sudo apt -y install r-base

# Instalação do R-Studio

cd Downloads/
wget https://download1.rstudio.org/rstudio-xenial-1.1.453-amd64.deb
sudo gdebi rstudio-xenial-1.1.453-amd64.deb

# Instalação de pacotes

sudo su - -c "R -e \"install.packages(c('tidyverse', 'ggplot2', 'shiny'), repos='http://cran.rstudio.com/')\""
```

Nada muito complicado! Na instalação de outros pacotes, houveram algumas bibliotecas do ubuntu que precisaram ser instaladas, mas, felizmente, estes são informados no console do R/Rstudio.

---

Como este é o primeiro post do blog, aproveito para agradecer a @almsou com o design do blog!
