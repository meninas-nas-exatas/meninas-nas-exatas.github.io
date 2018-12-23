---
layout: post
title: Quanto vale um pirula?
bigimg: /img/pirula.gif
tags: [r, rstudio, youtube, statistics, API]
comments: true
draft: true
output:
  html_document:
    keep_md: true
---

Você provavelmente já se bateu com o meme "Hoje vou durmir cedo". 

<center>

<blockquote class="twitter-tweet" data-lang="en"><p lang="pt" dir="ltr">&quot;Hoje eu vou dormir cedo&quot;<br><br>3 da manhã: <a href="https://t.co/Of64NBkTzN">pic.twitter.com/Of64NBkTzN</a></p>&mdash; Carlos ↟ (@_CarlosLz) <a href="https://twitter.com/_CarlosLz/status/805593892150185985?ref_src=twsrc%5Etfw">December 5, 2016</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>


</center>

Bem... foi assim minha madrugada.

**Contextualizando!**

No youtube BR, o [canal do Pirula](https://www.youtube.com/channel/UCdGpd0gNn38UKwoncZd9rmA) se destaca como um dos melhores (na minha opnião!) canais de ciência. Tratando de religião à mudança climática, o rapaz apanha da direita e da esquerda sem querer tretar com nenhum dos dois.

Ele também se destaca pela duração de seus vídeos. Praticamente, um podcast com imagens!

Basta começar um vídeo com "esse vai ser um vídeo curto" que você terá pelo menos 30 minutos de conteúdo.

Bem... eram 11h da noite quando me deparei com o seguinte comentário em um dos vídeos.

<center>

![](wwww/img2.png)
</center>

Então veio a ideia menos razoável do momento.

E você já sabe qual é...

**tl;dr**: Um pirula é igual 24 minutos!

### Extração dos dados

A informação prinicipal a ser extraída dos vídeos é a duração total deles. Com quase 500 vídeos no canal -  sem muita priori sobre a variabilidade da duração dos vídeos e podendo existir alguma característica temporal - a solução foi extrair a informação de todos os vídeos via API do youtube.

O pacote [**tuber**](https://cran.r-project.org/web/packages/tuber/tuber.pdf) realiza função de cliente para o API do youtube, facilitando em muito o nosso trabalho.

Para isso devemos criar uma chave de aplicação na plataforma de [desenvolvedores do google](https://developers.google.com/youtube/v3/getting-started), ativar o [API do youtube](https://developers.google.com/youtube/v3/) e então via tuber::yt_oauth informar a chave gerada.


```r
tuber::yt_oauth(app_id = "youtube", 
         app_secret = "AIzaSyBQyA88Cb1SNRhVfDjOL5rb9cujL9Lh4",
         token = "")
```

Então!

<center>

![](wwww/img3.png)


</center>

<center>
Achou que ia ser fácil? Achou erado otário!
</center>

**1h da manhã**

Após muitas tentativas, a solução foi estudar a documentação do [API do youtube](https://developers.google.com/youtube/v3/getting-started) e utilizar o pacote RJSONIO para converter a resposta das solicitações ao API em um objeto no R.


```r
# Id do canal do Pirula
# https://www.youtube.com/channel/UCdGpd0gNn38UKwoncZd9rmA
channel <- "UCdGpd0gNn38UKwoncZd9rmA"

# Chave do API
# https://console.developers.google.com/apis/credentials?project=a2tw-1527955801428
api_key <- "<Chave do API>"

# Requisição das informações do canal
url <- paste0("https://www.googleapis.com/youtube/v3/search?",
              "channelId=", channel,
              "&key=",api_key,
              "&part=snippet",
              "&maxResults=1",
              "&type=video")

channel_info <- fromJSON(url, simplify = FALSE)
```

A url é formada pela id do canal, a chave da API e os argumentos: part (informações a serem extraídas), maxResults, e type (filtro entre videos ou playlists).

O objeto `channel_info` acumula informações gerais tanto do canal, quanto de um único vídeo do canal (maxResults=1).


```r
# Número de vídeos do canal
videos_n <- channel_info$pageInfo$totalResults

# Id, nome, data de publicação e título de 1 vídeo do canal
video_id    <- channel_info$items[[1]]$id$videoId
video_time  <- substr(channel_info$items[[1]]$snippet$publishedAt, 1, 10)
video_title <- channel_info$items[[1]]$snippet$title
```

Para extrair informações detalhados do vídeo, podemos realizar a seguinte requisição ao API do youtube.


```r
# Requisição das informações do vídeo
url_video <- paste0("https://www.googleapis.com/youtube/v3/videos?",
                    "id=", video_id,
                    "&key=",key,
                    "&part=snippet,contentDetails,statistics,status")


video_info <- fromJSON(url_video, simplify = FALSE)
```

Com a seleção dos argumentos `contentDetails`, `statistics`,`status` extraímos dados de duração do vídeo, tags, comentários, likes e dislikes.


```r
# Extração da duração do vídeo e variáveis de contagem
video_duration_raw <- video_info$items[[1]]$contentDetails$duration

viewCount <-  isNull(video_info$items[[1]]$statistics$viewCount)

likeCount <- isNull(video_info$items[[1]]$statistics$likeCount)

dislikeCount <- isNull(video_info$items[[1]]$statistics$dislikeCount)

favoriteCount <- isNull(video_info$items[[1]]$statistics$favoriteCount)

commentCount <- isNull(video_info$items[[1]]$statistics$commentCount)
```

Para o caso de alguma informação retorne `NULL` e não comprometa o armazenamento dos dados, escrevi uma função isNull.


```r
isNull <- function(x){
  
  if (is.null(x)) { y <- NA } 
  else { y <- x }
  
  return(y)
}
```

Após a extração dos dados do primeiro vídeo, podemos navegar para o próximo utilizando a saída da requisição de nome `nextPageToken`.


```r
# Requisição das informações do canal (próxima página)
url <- paste0("https://www.googleapis.com/youtube/v3/search?",
                "channelId=", channel,
                "&key=",api_key,
                "&part=snippet",
                "&maxResults=1",
                "&pageToken=",channel_info$nextPageToken,
                "&type=video")
  
channel_info <- fromJSON(url, simplify = FALSE)
```

A partir deste ponto, pode-se escrever um loop (limitado pelo número de vídeos do canal) que extrai as informações do vídeo e acessa a próxima página pelo `nextPageToken` para continuar extraindo informações dos vídeos.

O Código como um todo está disponível no seguinte [Gist](https://gist.github.com/adelmofilho/cb028fc5d6dbc8e08a1695de5bfc2a54).

### Data Prep

**3h da manhã**

<center>
![](https://pbs.twimg.com/media/DqCX1voXQAEG875.jpg)
</center>

<br><br>



Após a extração os dados tinham a seguinte cara.


```r
dados %>% 
  dplyr::as_tibble() %>% 
  purrr::set_names(~sub('Count', '', .x)) %>% 
  DT::datatable(options = list(pageLength = 4, order = list(list(2, 'desc'))))
```

<!--html_preserve--><div id="htmlwidget-2df136372dbbd6da875a" style="width:100%;height:auto;" class="datatables html-widget"></div>
<script type="application/json" data-for="htmlwidget-2df136372dbbd6da875a">{"x":{"filter":"none","data":[["1","2","3","4","5","6","7","8","9","10","11","12","13","14","15","16","17","18","19","20","21","22","23","24","25","26","27","28","29","30","31","32","33","34","35","36","37","38","39","40","41","42","43","44","45","46","47","48","49","50","51","52","53","54","55","56","57","58","59","60","61","62","63","64","65","66","67","68","69","70","71","72","73","74","75","76","77","78","79","80","81","82","83","84","85","86","87","88","89","90","91","92","93","94","95","96","97","98","99","100","101","102","103","104","105","106","107","108","109","110","111","112","113","114","115","116","117","118","119","120","121","122","123","124","125","126","127","128","129","130","131","132","133","134","135","136","137","138","139","140","141","142","143","144","145","146","147","148","149","150","151","152","153","154","155","156","157","158","159","160","161","162","163","164","165","166","167","168","169","170","171","172","173","174","175","176","177","178","179","180","181","182","183","184","185","186","187","188","189","190","191","192","193","194","195","196","197","198","199","200","201","202","203","204","205","206","207","208","209","210","211","212","213","214","215","216","217","218","219","220","221","222","223","224","225","226","227","228","229","230","231","232","233","234","235","236","237","238","239","240","241","242","243","244","245","246","247","248","249","250","251","252","253","254","255","256","257","258","259","260","261","262","263","264","265","266","267","268","269","270","271","272","273","274","275","276","277","278","279","280","281","282","283","284","285","286","287","288","289","290","291","292","293","294","295","296","297","298","299","300","301","302","303","304","305","306","307","308","309","310","311","312","313","314","315","316","317","318","319","320","321","322","323","324","325","326","327","328","329","330","331","332","333","334","335","336","337","338","339","340","341","342","343","344","345","346","347","348","349","350","351","352","353","354","355","356","357","358","359","360","361","362","363","364","365","366","367","368","369","370","371","372","373","374","375","376","377","378","379","380","381","382","383","384","385","386","387","388","389","390","391","392","393","394","395","396","397","398","399","400","401","402","403","404","405","406","407","408","409","410","411","412","413","414","415","416","417","418","419","420"],["Criacionismo [2] - Essência","P.C.R.Evo [3] - Ninguém estava lá pra ver a evolução","Jurassic World - resenha de um paleontólogo (COM SPOILER) (#Pirula 107.1)","Agnóstico ou Ateu? (#Pirula 26)","DUBLANDO OS CANDIDATOS (Pirula em SP e Fortaleza)","A maldição da mente descontínua (#Pirula 20)","O sumiço das abelhas (#Pirula 148)","Evolução e Dispersão dos mamíferos [REPOST] (#Pirula 62)","Cesariana está retardando a evolução? (#Pirula 189)","Fim do Museu Nacional (#Pirula 264)","Algo errado na educação (#Pirula 17)","P.C.R.Evo [7] - Cada novo estudo \"muda tudo\" sobre Evolução","Qual o animal? (#Pirula 136.1)","50 tons de Pirula (#Pirula 97)","O mundo assombrado pelos demônios (#Pirula 99)","Criacionismo [5] - Fim da Transição (criatura e algoz)","O que dizer na hora da perda? | #PergunteAoAteu 2 (#Pirula 234)","O segredo do sucesso dos dinossauros (#Pirula 260)","12 ANOS PARA SALVAR O PLANETA? (#Pirula 32.6C)","P.C.R.Evo [6] - \"Fósseis-vivos\" (#Evolução)","Falseabilidade (ou por que não conto como virei ateu) (#Pirula 42)","Mudanças no youtube &amp; Hotter than Hell (#Pirula 66)","Stephen &amp; Alan (#Pirula 98)","P.C.R.Evo [10] - Existe Micro e Macro Evolução?","Os Alquimistas e o Vigarista (#Pirula 90)","O dilema dos zoológicos (#Pirula 162)","Qual a função do sexo? (Pirula #58)","Erro, arrependimento e perdão (#Pirula 38)","Por que existe gente antivacinas? (OU: o tempo algoz das memórias) (#Pirula 267)","É preciso fé para ser ateu? (agnóstico = ateu em cima do muro?) (#Pirula 2)","PIRULA, O BANDIDO (#Pirula 236)","Floresta pra quê? (#Pirula 12)","Sim, somos todos macacos (#Pirula 79)","Dinossauros: o que mudou em 24 anos? (#Pirula 208)","Difícil aprender inglês? Não mesmo. (#Pirula 89)","As pautas do Cauê Moura (tamanho do membro, interstício, comeu lesma, apiterapia)!! (#Pirula 251)","Cladística - reconstruindo a Evolução (#Pirula 94)","Dinossauros: não foi o meteoro? (#Pirula 145)","Tubby e a Dissonância Cognitiva* (#Pirula 68)","Prove que não existe! (Inversão do Ônus da Prova) (#Pirula 4)","O fantasma dos natais futuros (#Pirula 64)","O homem de 300 mil anos (#Pirula 215)","Um bate-papo com Cortella (#Pirula 263)","\"Escola Livre\" de Evolução? (#Pirula 149.1)","Como eu lido com a morte? | #PergunteAoAteu 1 (#Pirula 234)","Criacionismo [3] - A pergunta certa.","Boa noite, vizinhança (#Pirula 91)","Jurassic World 2 - resenha de um paleontólogo (#Pirula 107.2)","Ateu, neo-ateu ou antiteísta? (ou exemplo de intolerância ao teísmo?) (#Pirula 39)","Vote no Hugo","Recadinhos / Anonymous (#Pirula 31)","P.C.R.Evo [9] - A Evolução justificou o racismo (eugenia)?","In memoriam - Christopher Hitchens (#Pirula 11)","Ética 3 - Morte e dor (#Pirula 18)","Falácia naturalista e Falácia moralista (#Pirula 109)","Ciência de branco? (#Pirula 257.2)","Contar pra família sobre ser ateu. #PergunteAoAteu 6 (#Pirula 234)","De onde vieram os Índios? (#Pirula 172)","A desculpa do entretenimento (#Pirula 100)","Vida eterna (#Pirula 73.1)","Belo Monte é a Gota D'água? (#Pirula 10)","Governo pagando youtubers (#Pirula 199)","Resposta à resposta da \"Inversão do Ônus da Prova\" (#Pirula 4)","Criacionismo [4] - Sujeito a mudanças no tempo","Vida eterna 2 - adendo (#Pirula 73.2)","Os problemas do jornalismo científico (#Pirula 143)","Hipnose | Fui hipnotizado - feat Alberto Dell'isola (#Pirula 197.4)","Fósseis são sempre primitivos? As figuras do livro da Evolução PARTE 3 (#Pirula 210.3)","Criacionismo [1] - Etiologia","E o show do Iron Maiden? (#Pirula 139.1)","Por que aves são dinossauros e pterossauros não são? (#Pirula 261)","In Memoriam - Lynn Margulis e a mitocôndria (#Pirula 9)","PEC 65/2012: o licenciamento ambiental pra escanteio (#Pirula 150)","Romário e a pesquisa no Brasil (#Pirula 47)","A memória sabe mentir (#Pirula 112)","Um sujeito chamado Pirula (#Pirula 168)","O que é Pseudociência? (#Pirula 256)","Efeito Placebo (#Pirula 3)","Detetives Inteligentes","Experimentos em animais e os beagles: FAQs (#Pirula 65.2)","Novos fósseis e a evolução dos primatas (#Pirula 156)","Origem e Evolução das Baleias (#Pirula 266)","Direita, esquerda, direita, esquerda... (#Pirula 204)","Criacionismo [6] - O contra-ataque (Parte 1)","Tempo, Pirula e Medula (#Pirula 61)","Qual a diferença entre Arqueologia e Paleontologia? (#Pirula 44)","Alma, personalidade e astrologia (#Pirula 152)","Sacolas plásticas: precisa proibir? (#Pirula 37)","Tiranossauro tinha penas? (#Pirula 216)","O incrível dino-pato da Mongólia (#Pirula 237)","A \"ditadura da beleza\" (#Pirula 92.1)","Trazendo pó, levando bala (#Pirula 96)","Polvos são aliens??? (#Pirula 252)","A questão Palestina 2: comentários (#Pirula 84.2)","Obesidade infantil (#Pirula 186)","A \"Lava-Jato\" da Biociências da USP (#Pirula 212.1) ENGLISH SUBTITLES","Tenho moral sem Deus? Existe bem e mal? | #PergunteAoAteu 3 (#Pirula 234)","Novo Código Florestal (#Pirula 15)","Argumentos que ateus não deveriam mais usar (#Pirula 52.1)","Recifes amazônicos (#Pirula 157)","A treta dos meus tweets sobre astrologia (#Pirula 262)","\"Peleumonia\" e a arrogância lingüística (#Pirula 171)","Ética 1 - Dilemas Éticos (#Pirula 18)","Corrente do Caraoquê (#Pirula 53)","Respondendo críticas parte 1 - Modo light (#Pirula 185)","Ateísmo - Origem do pensamento religioso (#Pirula 1)","Caiado injuriado (#Pirula 70)","Criacionismo [7] - Mudando de nome","Missão marte? (#Pirula 88)","Dinossauro no âmbar (com penas) (#Pirula 190)","Resposta ao desafio teísta sobre moralidade (#Pirula 7)","Alma pra quê? - debatendo as respostas (#Pirula 135.2)","Cristofobia 2 - adendo (#Pirula 105.2)","Politicamente correto, anti-militância e o efeito mola (#Pirula 104)","P.C.R.Evo [FAQ] - Cadê o ancestral, o elo perdido? (#Evolução)","P.C.R.Evo [8] - Perfeição demais pra ter surgido pela Evolução","O resgate dos beagles (#Pirula 65.1)","Candomblé não é religião (#SQN) (#Pirula 80)","Tíbio e Perônio e Pirula (#Pirula 170)","P.C.R.Evo [2] - Evolução é acaso?","Pílula anticoncepcional masculina X feminina: Parte 1 - mulher (#Pirula 187)","Quem paga a pesquisa no Brasil? (#Pirula 72)","Criacionismo [6] - O contra-ataque (Parte 2)","E a tal da apropriação cultural? (#Pirula 200)","Marco Legal da Ciência (#Pirula 128)","Supremo Tribunal Federal e os anencéfalos (#Pirula 28)","O Romário e o Feliciano (#Pirula 67)","Guzzo, gays e cabras (#Pirula 50)","Nerdologia é educativo? (#Pirula 222)","Tô velho (#Pirula 153.2)","Ghira a zade e shera o delli (#Pirula 69)","Um bate-papo com Drauzio Varella (#Pirula 196)","A treta da homeopatia na USP (#Pirula 213)","Patreon, doações e dízimo (#Pirula 146)","A síndrome de \"Downkins\" (#Pirula 85)","Revoltas na USP (#Pirula 6)","Se Deus não existe, qual o sentido da vida? #PergunteAoAteu 4 (#Pirula 234)","Pirula nas Zoropa 6 - Alemanha","Papo sério (#Pirula 253)","Keep walking, Nicolelis (#Pirula 82)","Sim, é bom vacinar contra H1N1 (#Pirula 78.3)","Fósseis intermediários? As figuras do livro da Evolução PARTE 2 (#Pirula 210.2)","Video Blocker e a bolha social (#Pirula 203)","Ciência em chamas (#Pirula 54)","Estereótipos (#Pirula 21)","Hipnose | Memória e mecanismos gerais - feat Alberto Dell'isola (#Pirula 197.2)","Agrotóxicos: salvação ou tragédia? (#Pirula 242)","Go Chuck, go.....  (#Pirula 205)","Recadinhos: Curitiba, Ensino Médio e Fatos Desconhecidos (#Pirula 168 - 177.3)","2 Centavos de prosa com Tiago Belotti - Intolerância, mimimi e politicamente correto (#Pirula 180)","Inteligência Artificial e Evolução (#Pirula 102)","O britânico negro de 10 mil anos atrás (#Pirula 245)","Por que preservar as espécies? (Resposta a Alex Pyron) (#Pirula 136.4)","Por que não sou cristão? #PergunteAoAteu 5 (#Pirula 234)","Índios: o que fazer com eles? (#Pirula 48.1)","Meritocracia: o que não costumam contar (#Pirula 233)","#Zika, DDT e Narloch (#Pirula 125.5)","Sabotagem à Pesquisa e a \"crueldade com animais\" (#Pirula 257.1)","700.000 e outras coisas... (#Pirula 265)","Ameaça ao estado laico 2 - o retorno (#Pirula 13)","Unidades de medida astronômicas em menos de meio Pirula (feat. Ciência e Astronomia) (#Pirula  234)","Índios: Comentários ao Yuri (EuCiência) (#Pirula 48)","Ateísmo é escolha? (#Pirula 114)","Ozzy Osbourne 2018: EU FUI (#Pirula 139.2)","In memoriam - Cesar Ades (#Pirula 24)","Respondendo críticas parte 2 - Evil Mode (#Pirula 185)","SOPA e PIPA (#Pirula 14)","Corrente literária/cinematográfica (#Pirula 25)","P.C.R.Evo [5] - Evolução é \"só\" uma teoria? Por que não é lei?","ZELÂNDIA: o continente perdido que já era conhecido (#Pirula 201)","O Gafanhoto Fininho","Índios: o que fazer com eles? - ADENDO (#Pirula 48)","Tecnologia e Empreendedorismo (feat. Professor Carlos Julio) (#Pirula 269)","Você tinha que deixar de ser ateu!! (#Pirula 138)","Pesquisas inúteis pagas com seu dinheiro? (#Pirula 247)","Pela união dos seus poderes...      #SVBR  #sciencevlogsbrasil  #Pirula 134","Ciência nas Eleições 2018, viagem pra Irlanda e livros (#Pirula 259)","\"Ditadura da beleza\" - Adendo e FAQs (#Pirula 92.2)","Fantástico e a Arca de Noé 2 - respondendo comentários (#Pirula 193.2)","Zika, FAPESP e o Governador (#Pirula 151)","A questão Palestina (#Pirula 84.1)","Fósseis: as figuras do livro da Evolução PARTE 1 (#Pirula 210.1)","Zika e os boatos zicados (#Pirula 125.1)","Malafaia e os argumentos anti-gay (#Pirula 57)","Criacionismo [8] - Modus operandi","Hipnose é uma farsa? Um resumo (#Pirula 197.5)","Quero ver mexer com Maomé - RESPONDENDO CRÍTICAS PARTE 1 (#Pirula 239.2)","Desenho e reação (feat. Dai Oliveira) (#Pirula  169)","Blablalogia: a ciência contra-ataca (#Pirula 163)","Pirula nas Zoropa 8 - Bonus track (中国)","Whatsapp: a Deep Web de bolso (#Pirula 268)","Tomografia de dinossauro: cérebro e alimentação (#Pirula 228)","Pseudociência no SUS (#Pirula 249)","Tretas, fofocas e a \"TVlização\" do Youtube (#Pirula 129)","Lei Rouanet - respondendo aos comentários (#Pirula 154.2)","Políticos deveriam ganhar menos? (#Pirula 131)","Otário, Gaúcho Ateu, censura e preconceito (#Pirula 45)","Vegans, vegetarianos e o Gary Yourofsky (English subtitles) (#Pirula 16)","Sobre gays e camarão (abominação) (#Pirula 41)","Ética [2.1] - Resposta aos dilemas éticos (#Pirula 18)","Belo Monte - Adendo - Alternativas e respostas (#Pirula 10)","É sem partido? (#Pirula 59.2)","Vacinas valem a pena? (#Pirula 78.1)","Já deu minha cota pras cotas (faz uma cota) (#Pirula 46)","Dicas de leitura 1 (#Pirula 74.1)","Anti-vacinas, terraplanistas e punh....gorilas (#Pirula 214)","História da Irlanda (PARTE 2): Ingleses, Domínio Protestante, Plantations, Grande Fome (#Pirula 255)","#Zika: Água parada pra combater mosquito??? (#Pirula 125.3)","Ética [2.3] - O policial e o traficante (#Pirula 18)","100.000!! (#Pirula 63)","Pirula: doutor em arrogância (#Pirula 142)","Meu encontro com o Beakman na #CPBR8 (#Pirula 115)","Respondendo perguntas sobre AIDS #DesafioUNAIDS (#Pirula 238)","Alma pra quê? (#Pirula 135.1)","Respondendo críticas, politicamente correto, racismo, humor e telhado de vidro (#258.2)","O tarado do busão (#Pirula 226)","Marcelo Crivella - uma ode à ignorância (#Pirula 23)","Forma Vs conteúdo (#Pirula 118)","Código Florestal, Rio + 20 e Belo Monte: ATUALIZAÇÕES (#Pirula 49)","A menina e os 33 (#Pirula 161.1)","Pirula pela América Latina [4] - Argentina (Tucumán)","História da Irlanda (PARTE 1): pré-história, celtas, cristianismo, vikings e normandos (#Pirula 255)","O fóssil de dinossauro mais bem preservado do mundo? (#Pirula 217)","Mudanças no Youtube e os canais pequenos (#Pirula 244)","Quem foi Charles Manson? (#Pirula 235)","Hipnose | Regressão, possessões e exorcismos - feat Alberto Dell'isola (#Pirula 197.3)","PL 122 e a \"mola gay\" (#Pirula 40)","Sci-Hub e a pirataria acadêmica (#Pirula 218)","Há perigo na vacina contra o HPV? (#Pirula 78.2)","A Fosfo não funfa? (#Pirula 117.3/133.4)","Pergunte ao ateu (#Pirula 234.1)","Homossexualidade - ponto final. (#Pirula 29)","Pirula em: a ciência só está certa quando concorda comigo (#Pirula 168)","Limite de dados pra internet fixa? (#Pirula 141)","Quando ser mimado vira crime (#Pirula 220)","Goleiro Bruno e a \"Defesa de bandido\" (#Pirula 206)","Samarco e a lama no mar (#Pirula 121.3)","Fui criticar a Fatos Desconhecidos, e olha no que deu (#Pirula 177.2)","NEVE NO SAARA??? (#Pirula 32.6B)","Dicas de Leitura 2 - O Retorno (#Pirula 74.2)","Incêndio no Museu da Língua Portuguesa (#Pirula 124)","Burkini: opção ou opressão? (#Pirula 174)","Uma RENCA de discórdias entre mineração e meio ambiente (#Pirula 225)","Um bate-papo num sábado qualquer... (feat @sabadoqualquer) (#Pirula 108)","Dr. Pirula, Vicentinho e abertura da Copa (#Pirula 81)","Ética [2.2] - EXTRAS (#Pirula 18)","O fóssil mais antigo do mundo!?! (#Pirula 202)","Entre o bullying e a crítica: qual é o limite? (#Pirula 165)","Apartheid na balada e escravidão \"liberada\" (#Pirula 230)","Violência contra mulher na ciência do Brasil (#Pirula 231)","P.C.R.Evo [1] - Por que ainda existem macacos? (#Evolução)","5 mitos sobre a reciclagem (feat Julia Jolie) (#Pirula 179  #ZooUrbanoSP)","Desarmadendo 2: Chicago e Austrália","\"Justiça\" com as próprias mãos (#Pirula 71)","Somos todos Charlie Hebdo (#Pirula 95.1)","Datafolha e os ateus que rezam por dinheiro (#Pirula 192)","O Algoritmo Maldito do Youtube (#Pirula 188)","Músicas iguais (coincidência musical?) (#Pirula 43)","LIVE DO DIA TRISTE (#Pirula 183)","Testes em animais - um adendo (#Pirula 22)","Hipnose | Separando joio do trigo - feat Alberto Dell'isola (#Pirula 197.1)","R.I.P. Hawking","Não seja BURRO no Youtube: evite o \"hate amigo\" (#Pirula 246)","Eu não acredito na Evolução (#Pirula 119)","Biodiversidade 1 - Quantas são as espécies? (#Pirula 136.2)","Shorty Awards e Mantiqueira Tombada (#Pirula 76)","Mar de lama em Mariana (#Pirula 121.1)","Pirula nas Zoropa 3 - França","\"Ateu\" mata 12 e se mata em Campinas (#Pirula 194)","Existe justificativa boa? (#Pirula 161.2)","Resposta (nervosa) ao meu último vídeo (#Pirula 52.2)","O que o racismo do Cocielo pode nos dizer? (#Pirula 258.1)","Uma mudança no canal (#Pirula 140)","Pirula nas Zoropa 1 - Portugal","Ciência gaúcha à beira do açude (#Pirula 110)","O PT e eu (#Pirula 101)","O brasileiro e a política (#Pirula 159)","Ensino religioso confessional em escolas públicas (#Pirula 229)","Pirula nas Zoropa 7 - Bélgica e Polônia","Pirula pela América Latina [3] - Bolívia","Família - adendo (#Pirula 113.2)","Pesquisa: quem são vocês? (resultados) (#Pirula 106)","Seca no Sudeste, terra sem garoa (#Pirula 87)","Fosfoetanolamina liberada? (#Pirula 133.3)","Pirula nas Zoropa 5 - Inglaterra","Desarmadendo 4 - \"Gun free zones\", drogas e automóveis","O vazamento da barragem em Barcarena (#Pirula 248)","Desarmadendo 1: A \"pesquisa de Harvard\"","Estado Islâmico (الدولة الإسلامية) (#Pirula 86)","Ameaça ao estado laico no Brasil (#Pirula 13)","Pesquisa e a verba de Schrödinger: 90% virá do nada (#Pirula 195.1)","Homofobia e os atentados em Orlando (#Pirula 164)","Um Ponto em Comum com a divulgação científica (#Pirula 166)","Pirula pela América Latina [2] - Chile","Vingadores da Ciência 1 - CAOS NA CAPES (feat Caio Gomes, André Souza e Camila Laranjeira)","Pílula anticoncepcional masculina X feminina: Parte 2 - homem (#Pirula 187)","O ato não compadecido: arreando Suassuna (#Pirula 77)","Detetives Inteligentes/Vote no Hugo - Atrás das Câmeras","Fraudes na Evolução? (resposta ao Rodrigo Silva) (#Pirula 223)","Adblock, fuleco, e outros recados (#Pirula 83)","Todo mundo odeia o Pirula! (#Pirula 185)","Direita Vs Esquerda (feat. Icles - Leitura ObrigaHistória) (#Pirula 250)","Felipe Neto Vs Feliciano - Comentários (PARTE 2) (#Pirula 167.2)","Cura gay, conselhos federais e Marcelo Rezende (#Pirula 227)","Como sabemos a cor dos dinossauros? (#Pirula 243)","EDUCAÇÃO NO BRASIL (feat. Rafael Procópio/MatemáticaRio e Paulo/Inglês Winner) (#Pirula 233)","Explicando a corrente (#Pirula 60)","A Igreja Católica está certa? (#Pirula 8)","É arte ou só mau gosto? (feat Carol Moreira) COM SPOILER #Pirula 191","O porquinho-da-índia no microondas (#Pirula 209)","Vinheta &amp; FAQs (#Pirula 55)","Pirula nas Zoropa 2 - Espanha","Sim, eu continuo sendo Charlie Hebdo (resposta aos críticos) (#Pirula 95.2)","Estado Laico: pode ser ou tá difícil? (#Pirula 207)","Pirula, O Isentão (#Pirula 137.2)","O memorando do ex-engenheiro da Google: ciência ou anti-diversidade? (#Pirula 221)","Mim, mim mesmo e Eu, Ciência (#Pirula 155)","Terra Plana e o Filtro pra Teorias da Conspiração (#Pirula 130)","Estupro (#Pirula 75)","A grana da lei Rouanet (#Pirula 154.1)","Pirula, por favor, fale só de Biologia (#Pirula 175)","Atirador de Campinas ADENDO - ateísmo, loucura e discurso de ódio (#Pirula 194)","Zika: \"Brigada\" Anti-Boato (#Pirula 125.2)","Ajude Oziel - #ForçaOziel (#Pirula 27)","Patrícia Abravanel, ateus e gays (#Pirula 158)","CURITIBA - A ciência só está certa quando concorda comigo (#Pirula 168)","#Zika é propriedade dos Rockefeller? (#Pirula 125.4)","Exemplo de intolerância ao ateísmo (#Pirula 5)","A sexta extinção em massa (#Pirula 136.3/219.2)","Tiffany e transsexuais no esporte (#Pirula 241)","IMPERDÍVEL!! 25 de março com Pirula (#Pirula 168)","Desarmamento - Conclusões","Scientific American, Campinas e Belo Horizonte (#Pirula 168)","Eleições 2016: educação como prioridade (#mapanaseleições #Pirula 178)","Trollando: Só os fortes entenderão (#Pirula 36)","Pirula nas Zoropa 4 - Império Austro-Húngaro","Atentados, refugiados e o Estado Islâmico (#Pirula 120)","Cristofobia?? (#Pirula 105.1)","Mackenzie e o \"Núcleo de Pesquisa\" em Design Inteligente (#Pirula 211)","Fantástico e a Arca de Noé (#Pirula 193.1)","Inglês, civilização e viagens - um bate papo com Rafa Silva (#Pirula 111)","Merchandising e alguns recados (#Pirula 35)","Ensinando a aprender (feat. Emílio Garcia) (#Pirula 184)","Evolution of Mammals and their dispersal","Compreensão científica, robôs e singularidade (feat @milalaranjeira) (#Pirula 132)","13 respostas sobre a FOSFOETANOLAMINA (#Pirula 117.2)","Pesquisa e a verba de Schrödinger: o ministério volta atrás... (#Pirula 195.2)","Aprender inglês na gringa (#Irlanda #ICOT #Pirula 89)","Desarmadendo 3 - ditaduras e os dados no Brasil","Adendo sobre a \"lava-jato\" da Biociências (#Pirula 212.2)","Pirula pela América Latina [1] - Venezuela","LIVE DO PIRULA 2","Quero ver mexer com Maomé - RESPOSTA AO YAGO (#Pirula 239.3)","A água alcalina do Seu Lair (#Pirula 232.1)","O que é uma família? (#Pirula 113.1)","Nova censura do Youtube? (#Pirula 176)","Quero ver mexer com Maomé! (#Pirula 239.1)","Pepsi, fetos e adoçante (#Pirula 33)","Criacionismo [9] - Complexidade Irredutível","LIVE DO PIRULA 3","Nióbio de cool é HOAX (#Pirula 56.1)","Fosfoetanolamina compassiva (#Pirula 133.2)","Vergonha: A escola adventista e os fósseis (#Pirula 30)","Live do Pirula (#Pirula 183)","Jesus era comunista?? (#Pirula 240)","Juntos pela ciência! (#Pirula 168)","LIVE DO MÉRITO ou VALOR (#Pirula 183)","Deputaiada (e a cuspida do Jean) (#Pirula 144.1)","666","ELEIÇÕES 2018: SEGUNDO TURNO (feat. Mas Afinal, Leitura Obrigahistória e Lucio Big)","A USP, o câncer, e a \"cura\" (#Pirula 117.1)","ELEIÇÕES 2018 (feat. Mas Afinal, Leitura Obrigahistória e Lucio Big)","Sim, o estado DEVE financiar pesquisa e tecnologia (RESPOSTA a Ideias Radicais) (#Pirula 247.2)","O céu católico do Porta dos Fundos (#Pirula 181)","300 mil (e sobre crescer com ajuda dos outros) (#Pirula 122)","Meu problema com a Fatos Desconhecidos (#Pirula 177.1)","LIVE DO VELÓRIO DA CIÊNCIA BR (#Pirula 183)","Água alcalina - ADENDO em 12 passos (#Pirula 232.2)","Pirula no Nordeste!! : ENQUETE (#Pirula 168)","Recados - Belo Horizonte, Campinas e TELETON (#Pirula 168)","Vaquejada proibida (#Pirula 182)","Aquecimento Global: TEMPERATURAS (#Pirula 32.8)","10 dicas para não ser um fanático (#Pirula 126)","Desmilitarização da polícia? Um bate-papo com Túlio Vianna (#Pirula 198)","Desabafo e Samarco (#Pirula 121.2)","Tem que experimentar pra saber do que gosta? (#Pirula 149.4)","Fosfoetanolamina, Ratinho e Samarco (#Pirula 133.1)","Domingão do Pirula (#Pirula 168)","Reflexões sobre o desarmamento","Lula, Dilma e o grampo: algumas reflexões (#Pirula 137.1)","Omo e a brincadeira de criança (#Pirula 224.4)","Aquecimento Global: OCEANOS (#Pirula 32.7)","Resposta: O bravo sem trabalho (#Pirula 34)","O nióbio, o Hoax, e as viúvas do Enéas (#Pirula 56.2)","Gato Gold brincando com os gatinhos","Respondendo aos comentários sobre Jean Wyllys e Bolsonaro (#Pirula 144.2)","Aquecimento Global - Último Round (#Pirula 32.3)","Aquecimento Global: Ricardo Felício - Segundo Round (#Pirula 32.2)","A exposição do MAM: nudez, arte e educação (#Pirula 224.3)","Aquecimento global: GELO (#Pirula 32.6)","PIRULA ISENTÃO ALIVIA PRA TODOS NO CASO SANTANDER/QUEERMUSEU?  (#Pirula 224.2)","Meus agradecimentos. (#Pirula 168)","Escola sem partido: resposta aos comentários (#Pirula 149.3)","Vício em redes sociais e Depressão em Youtubers (#Pirula 270)","Recados: nova palestra em São Paulo (#Pirula 168)","Agradecimentos a Curitiba (#Pirula 168)","Pirula \"passando vergonha\" - respondendo Ricardo Felício (de novo) (#Pirula 32.5)","Escola sem partido ou sem juízo? (#Pirula 149.2)","Santander e a exposição fechada (#Pirula 224.1)","Papo reto: Ricardo Felício e o Aquecimento Global (#Pirula 32.4)","LIVE DO AQUECIMENTO (#Pirula 183)","A vez de Campinas (#Pirula 168)","Ajude Oziel: uma justa homenagem e algumas considerações (#Pirula 27)","Balanço dos 30 dias (#Pirula 140)","12 ANOS PARA SALVAR O PLANETA? (#Pirula 32.6C)","Sim, somos todos macacos (#Pirula 79)","Dinossauros: não foi o meteoro? (#Pirula 145)","Prove que não existe! (Inversão do Ônus da Prova) (#Pirula 4)","Jurassic World 2 - resenha de um paleontólogo (#Pirula 107.2)","Contar pra família sobre ser ateu. #PergunteAoAteu 6 (#Pirula 234)"],["2012-03-09","2016-02-01","2015-06-24","2012-04-08","2018-10-05","2012-03-05","2016-04-26","2013-08-12","2016-12-07","2018-09-03","2012-02-19","2016-08-17","2016-03-11","2015-02-06","2015-02-22","2012-08-06","2017-12-14","2018-08-10","2018-10-12","2016-04-17","2012-08-10","2013-11-14","2015-02-17","2018-05-05","2014-11-24","2016-06-03","2013-04-23","2012-07-10","2018-09-28","2009-05-18","2017-11-24","2012-01-04","2014-05-01","2017-04-18","2014-11-03","2018-04-22","2015-01-05","2016-04-22","2013-12-15","2011-06-10","2013-09-04","2017-06-10","2018-08-27","2016-04-28","2017-12-03","2012-04-05","2014-11-29","2018-06-30","2012-07-19","2011-07-30","2012-05-14","2017-06-19","2011-12-26","2012-09-17","2015-07-29","2018-06-23","2018-09-14","2016-08-13","2015-03-03","2014-03-05","2011-12-04","2017-02-18","2011-07-13","2012-06-02","2014-03-15","2016-04-16","2017-02-02","2017-05-04","2012-03-01","2016-04-08","2018-08-12","2011-11-27","2016-04-28","2012-10-16","2015-09-03","2017-04-26","2018-06-08","2011-04-10","2011-07-01","2013-10-22","2016-05-09","2018-09-09","2017-03-16","2013-01-14","2013-07-24","2012-09-20","2016-05-01","2012-06-28","2017-06-16","2017-12-11","2014-12-07","2015-01-20","2018-05-21","2014-08-04","2016-11-03","2017-05-17","2018-02-21","2012-01-27","2012-12-20","2016-05-10","2018-08-20","2016-08-04","2012-02-25","2013-01-17","2016-11-07","2009-05-06","2014-01-04","2014-02-13","2014-10-30","2016-12-11","2011-11-20","2016-04-30","2015-06-16","2015-05-21","2016-01-22","2016-12-01","2013-10-19","2014-05-20","2016-07-30","2016-01-20","2016-11-11","2014-02-21","2013-01-21","2017-02-23","2016-01-29","2012-04-13","2013-11-27","2012-11-20","2017-08-22","2016-05-03","2013-12-30","2017-01-23","2017-05-31","2016-04-23","2014-08-26","2011-11-12","2018-03-11","2013-06-07","2018-05-22","2014-06-24","2016-04-19","2017-05-03","2017-03-12","2013-01-25","2012-03-10","2017-01-31","2018-01-23","2017-03-20","2016-09-24","2016-10-07","2015-04-20","2018-02-10","2017-12-30","2018-04-13","2012-10-25","2017-11-08","2016-02-13","2018-06-16","2018-09-05","2012-09-08","2017-11-19","2012-11-13","2015-09-24","2018-05-16","2012-03-16","2016-11-09","2012-01-23","2012-03-22","2016-03-15","2017-02-26","2006-10-11","2012-10-29","2018-10-16","2016-04-02","2018-03-03","2016-03-03","2018-07-22","2014-12-12","2016-12-29","2016-04-29","2014-07-29","2017-05-02","2015-12-27","2013-02-25","2014-11-12","2017-02-13","2018-01-04","2016-07-22","2016-06-12","2013-07-09","2018-10-09","2017-09-24","2018-03-20","2016-02-09","2016-05-06","2016-02-21","2012-10-02","2012-02-12","2012-07-29","2012-03-26","2011-12-17","2013-06-21","2014-05-01","2012-10-09","2014-03-30","2017-06-04","2018-07-25","2016-02-02","2016-11-25","2013-08-25","2016-04-14","2015-10-04","2017-12-18","2016-03-09","2018-07-10","2017-09-15","2012-03-15","2015-11-05","2012-11-07","2016-05-28","2012-12-09","2018-06-01","2017-06-22","2018-02-05","2017-11-22","2017-02-01","2012-07-24","2017-06-28","2016-04-18","2017-04-03","2017-11-11","2012-04-25","2016-07-15","2016-04-13","2017-08-05","2017-04-07","2016-04-24","2016-09-22","2018-01-18","2015-06-10","2015-12-22","2016-08-29","2017-09-13","2015-07-11","2014-06-06","2012-03-28","2017-03-06","2016-06-25","2017-10-02","2017-10-17","2016-01-14","2016-09-26","2015-10-18","2014-02-08","2015-01-08","2016-12-27","2016-11-18","2012-09-04","2017-06-06","2012-03-11","2017-01-30","2018-03-14","2018-02-12","2015-12-06","2016-03-28","2014-04-20","2015-11-25","2013-04-30","2017-01-03","2016-05-28","2012-12-28","2018-07-04","2016-04-12","2013-04-02","2015-08-09","2015-03-25","2016-05-12","2017-09-26","2013-06-18","2012-12-04","2015-09-18","2015-07-02","2014-10-26","2016-04-15","2013-05-29","2015-10-24","2018-03-05","2015-10-16","2014-10-18","2012-01-12","2017-01-15","2016-06-14","2016-07-08","2012-12-01","2018-08-04","2016-11-14","2014-04-25","2011-09-12","2017-08-31","2014-07-09","2016-10-24","2018-03-26","2016-07-16","2017-09-20","2018-01-30","2017-10-31","2013-07-15","2011-11-24","2016-12-21","2017-04-27","2013-02-09","2013-04-14","2015-01-12","2017-04-12","2016-03-23","2017-08-16","2016-05-08","2016-02-18","2014-04-08","2016-05-05","2016-09-03","2017-01-06","2015-12-27","2012-04-11","2016-05-11","2016-09-07","2016-02-04","2011-09-27","2017-07-18","2018-01-15","2017-03-04","2015-12-09","2017-09-23","2016-09-20","2012-06-28","2013-05-25","2015-11-15","2015-06-13","2017-05-11","2016-12-28","2015-08-17","2012-06-18","2016-10-22","2010-12-07","2016-02-22","2015-10-22","2017-01-18","2018-09-19","2015-10-22","2017-05-30","2012-11-24","2017-11-30","2018-01-11","2017-10-23","2015-09-10","2016-09-06","2017-12-24","2012-06-03","2016-01-07","2018-03-01","2013-02-15","2016-02-29","2012-04-27","2016-10-18","2017-12-27","2017-10-04","2017-11-09","2016-04-20","2018-05-27","2018-10-10","2015-10-19","2018-03-15","2018-03-31","2016-10-10","2015-11-29","2016-09-11","2017-09-06","2017-10-27","2018-01-19","2017-10-25","2016-10-11","2017-08-10","2015-12-30","2017-02-15","2015-11-26","2016-07-26","2016-02-28","2017-09-02","2015-10-11","2016-03-17","2017-10-15","2017-07-30","2012-06-13","2013-03-16","2009-10-04","2016-04-21","2013-10-14","2012-06-09","2017-10-01","2017-07-26","2017-09-13","2016-08-09","2016-07-25","2018-10-21","2016-10-15","2016-09-28","2017-07-24","2016-07-20","2017-09-12","2017-07-05","2017-07-06","2017-06-11","2012-04-22","2016-05-16","2018-10-12","2014-05-01","2016-04-22","2011-06-10","2018-06-30","2018-09-14"],["PT14M20S","PT13M48S","PT24M53S","PT13M16S","PT1M39S","PT10M56S","PT12M12S","PT17M1S","PT20M14S","PT24M11S","PT12M56S","PT15M41S","PT2M17S","PT20M17S","PT17M53S","PT11M42S","PT12M7S","PT25M24S","PT18M32S","PT19M53S","PT13M18S","PT8M21S","PT14M21S","PT21M30S","PT11M54S","PT35M36S","PT11M40S","PT10M50S","PT21M44S","PT7M26S","PT21M7S","PT15M8S","PT21M25S","PT20M45S","PT23M56S","PT21M37S","PT47M","PT14M26S","PT20M21S","PT8M40S","PT15M6S","PT20M6S","PT36M40S","PT14M19S","PT13M36S","PT15M46S","PT20M52S","PT23M11S","PT21M45S","PT12M4S","PT7M16S","PT20M48S","PT11M16S","PT29M52S","PT20M29S","PT26M56S","PT26M22S","PT18M21S","PT25M44S","PT22M48S","PT30M18S","PT9M50S","PT8M40S","PT17M38S","PT15M21S","PT17M16S","PT24M31S","PT19M38S","PT10M40S","PT7M22S","PT23M25S","PT16M16S","PT9M10S","PT10M36S","PT27M1S","PT4M6S","PT30M38S","PT15M2S","PT12M16S","PT12M11S","PT12M30S","PT49M48S","PT15M1S","PT19M","PT26M36S","PT13M26S","PT13M49S","PT12M29S","PT13M41S","PT22M1S","PT31M43S","PT14M39S","PT47M57S","PT23M49S","PT24M31S","PT25M22S","PT24M6S","PT28M50S","PT27M32S","PT5M35S","PT49M12S","PT29M26S","PT5M26S","PT6M22S","PT34M2S","PT9M58S","PT11M37S","PT20M57S","PT10M19S","PT16M44S","PT14M51S","PT38M15S","PT16M12S","PT52M25S","PT11M10S","PT33M25S","PT15M4S","PT15M19S","PT5M30S","PT29M12S","PT17M51S","PT20M33S","PT19M44S","PT30M21S","PT16M8S","PT11M28S","PT11M29S","PT24M25S","PT16M2S","PT17M54S","PT13M2S","PT55M51S","PT30M15S","PT14M47S","PT23M","PT16M20S","PT19M1S","PT24M","PT16M3S","PT26M15S","PT7M1S","PT27M18S","PT20M11S","PT14M43S","PT12M20S","PT42M9S","PT41M12S","PT15M39S","PT15M42S","PT18M18S","PT31M16S","PT22M18S","PT43M9S","PT23M45S","PT30M9S","PT26M44S","PT12M","PT31M","PT21M52S","PT6M46S","PT9M13S","PT14M40S","PT24M4S","PT17M29S","PT12M42S","PT36M37S","PT11M20S","PT8M36S","PT24M39S","PT19M33S","PT5M11S","PT11M13S","PT27M46S","PT17M17S","PT24M55S","PT3M52S","PT15M1S","PT21M30S","PT21M51S","PT11M32S","PT35M55S","PT33M46S","PT14M40S","PT14M4S","PT32M8S","PT14M50S","PT22M20S","PT5M5S","PT11M23S","PT40M","PT35M31S","PT19M55S","PT23M44S","PT19M11S","PT16M10S","PT20M37S","PT13M39S","PT30M43S","PT6M54S","PT31M20S","PT20M13S","PT7M17S","PT32M15S","PT25M42S","PT17M21S","PT23M36S","PT58M15S","PT8M33S","PT22M26S","PT12M57S","PT13M17S","PT13M50S","PT28M55S","PT22M29S","PT30M6S","PT13M9S","PT5M48S","PT26M6S","PT23M6S","PT26M28S","PT15M25S","PT1H3M55S","PT17M22S","PT22M45S","PT28M46S","PT24M6S","PT20M4S","PT30M35S","PT15M25S","PT20M41S","PT2M56S","PT31M25S","PT4M4S","PT15M38S","PT23M58S","PT19M14S","PT15M11S","PT20M3S","PT23M51S","PT29M37S","PT7M22S","PT26M28S","PT18M23S","PT38M40S","PT11M36S","PT8M33S","PT24M34S","PT26M54S","PT19M8S","PT24M48S","PT16M27S","PT19M49S","PT16M6S","PT24M20S","PT14M25S","PT18M18S","PT26M6S","PT22M34S","PT1H20M37S","PT18M38S","PT47M52S","PT12M42S","PT10M28S","PT21M44S","PT21M50S","PT22M22S","PT40M2S","PT31M51S","PT20M40S","PT13M52S","PT22M53S","PT33M11S","PT9M44S","PT23M37S","PT8M30S","PT26M8S","PT28M25S","PT20M3S","PT19M47S","PT24M51S","PT21M2S","PT34M55S","PT24M45S","PT11M18S","PT19M13S","PT14M20S","PT18M53S","PT24M1S","PT33M59S","PT9M","PT14M27S","PT17M51S","PT17M4S","PT27M46S","PT1H35M16S","PT27M3S","PT26M21S","PT8M35S","PT25M52S","PT12M40S","PT5M52S","PT1H19M52S","PT30M19S","PT24M28S","PT29M54S","PT44M36S","PT19M47S","PT6M39S","PT33M28S","PT15M29S","PT15M23S","PT28M22S","PT32M9S","PT36M25S","PT18M36S","PT47M2S","PT41M8S","PT30M43S","PT34M9S","PT35M21S","PT24M41S","PT28M54S","PT40M2S","PT5M41S","PT25M52S","PT3M39S","PT15M28S","PT15M49S","PT27M2S","PT33M29S","PT2M34S","PT16M32S","PT5M4S","PT27M54S","PT18M","PT28M9S","PT22M51S","PT16M3S","PT15M35S","PT16M21S","PT1H24M43S","PT14M59S","PT57M53S","PT15M9S","PT30M16S","PT27M14S","PT3M53S","PT35M35S","PT22M32S","PT15M17S","PT22M","PT1H34M15S","PT1H2M40S","PT22M57S","PT32M12S","PT21M5S","PT32M28S","PT16M","PT33M35S","PT1H42M27S","PT29M39S","PT13M43S","PT13M34S","PT2H4M19S","PT18M46S","PT12M22S","PT1H20M59S","PT16M50S","PT36M28S","PT3H20M46S","PT29M13S","PT3H27M48S","PT44M42S","PT16M44S","PT17M29S","PT26M29S","PT1H29M55S","PT22M31S","PT2M59S","PT8M52S","PT11M37S","PT30M10S","PT21M7S","PT47M18S","PT28M59S","PT12M7S","PT8M57S","PT1M","PT36M22S","PT15M36S","PT26M25S","PT26M23S","PT6M30S","PT29M52S","PT2M38S","PT15M19S","PT1H25M17S","PT22M3S","PT31M46S","PT30M53S","PT19M10S","PT6M31S","PT41M28S","PT41M15S","PT4M45S","PT1M57S","PT1H15M13S","PT48M29S","PT22M21S","PT57M18S","PT2H6M35S","PT1M31S","PT6M47S","PT9M47S","PT18M32S","PT21M25S","PT14M26S","PT8M40S","PT23M11S","PT26M22S"],["214479","113210","268656","569462","47067","266645","129429","192126","158481","186213","382358","105374","77628","228170","542222","127658","75935","105380","98171","136170","165706","44781","153421","71326","111769","231559","191274","101492","92530","75897","152002","137893","349289","116136","689817","383506","344287","323599","173774","183951","90280","113252","171024","107049","159019","202155","104859","102782","241680","42859","97064","107534","67402","152177","167153","121760","85929","658746","379121","252830","435152","207194","116818","162003","105530","95315","171162","44516","345726","82909","101246","46045","52097","71248","137733","70134","132637","118362","27022","92951","97354","117951","225574","146745","211669","65856","203780","84533","176995","108317","518300","140396","188267","175619","149505","93111","121714","125385","602671","57122","162926","161177","78369","37403","192693","87068","58231","147965","159728","152678","201727","105320","91337","294309","120272","190920","319939","190964","106758","200231","168121","90329","117532","231565","80644","71738","103971","72916","338358","110787","110307","471349","138257","112664","224073","83153","263606","48373","175883","84683","52094","53853","146993","59174","182344","95511","269669","43326","164126","73784","255770","176407","143140","191221","212752","350097","112349","76189","68832","65128","54384","44973","290009","38715","21424","363840","27268","28168","167203","194324","40235","47269","27488","305080","124214","79715","42257","126465","130796","52676","673125","91724","144745","442203","164198","173018","105762","66969","122768","39776","170428","90884","103187","203764","70734","152403","173018","474052","98601","104946","83344","144829","193579","427814","161601","88893","48802","100790","166363","66242","132053","75704","66560","222004","173626","91932","125915","169299","81978","441897","28800","127946","140783","104380","383943","86384","119110","114304","62941","121586","88830","909628","84574","376057","228545","144748","86211","462133","90946","88812","99262","137095","71696","207230","79856","26820","129409","118468","141593","77510","324289","101427","73143","240293","152383","119627","155249","230677","75258","99585","169633","123448","90102","368516","90632","69919","328105","43963","139359","177590","195230","371155","110006","56235","76257","579838","173284","138786","28633","32282","90022","110112","145506","109405","51544","54311","67311","134861","347281","106001","99229","134132","62555","33539","64804","111413","252315","11862","176666","83285","175822","218426","223948","192782","153782","70510","72607","180431","266870","169442","35706","35618","84256","133198","170702","153008","96943","790489","319840","261217","121493","78187","103432","18109","346686","50476","145457","327330","160241","293534","40022","100886","36708","79628","84683","41254","380083","303521","125009","290683","70341","25498","70897","26727","105402","165576","43676","22915","65251","47977","77386","41015","102058","211032","215002","104516","210353","425650","209842","40609","663799","77770","292333","80594","165461","39719","54368","282201","287643","100962","448977","134967","121863","675818","185035","926185","64243","99230","26150","14126","179898","80832","205107","216170","178426","146830","113689","82598","345992","378091","234733","99368","364753","168627","13742","203227","464637","192311","273141","134584","140286","61722","175517","22624","17892","38632","369648","515109","311962","412759","86139","20327","11088","97184","98192","349302","323617","183956","102782","85943"],["11838","12125","24762","34970","10686","16599","15989","15569","16705","32230","27580","9992","9697","18325","53723","7569","10228","14471","13692","13871","10585","4902","15114","9540","13858","20971","13061","8784","14778","4480","25808","8365","28400","14832","42740","53581","23016","28256","15508","11685","10981","13571","21947","14515","19258","11099","13549","12722","16779","4805","5646","12620","4702","10268","19904","18857","10830","42196","29210","18689","19360","28408","5835","8952","9450","13392","13515","5794","17671","10200","13850","4127","7329","5801","15081","6859","14214","8352","2594","6242","10399","15603","30994","7896","19622","5184","15878","5883","17249","12278","31359","13477","20719","12065","15202","10789","14438","5263","33106","3954","19059","17720","3294","3047","18218","5587","5394","10715","14680","16746","11047","9862","10213","25840","13428","17222","20149","17029","15640","18516","16418","6698","6841","25302","9714","5982","12415","5700","53338","10966","8921","43598","12835","14435","16427","3717","25965","3583","30766","9738","6800","5487","16108","4527","11491","8635","27421","7201","17041","8779","20168","20083","14666","19145","11863","39990","14370","7941","14164","4882","7553","2166","20539","6123","2065","36030","1896","1628","15111","19000","2316","2730","3755","31488","13796","11119","5586","9213","13733","7255","37188","8841","13726","27613","12551","14830","11327","10199","13752","3514","20995","10838","13413","27220","6020","17387","12416","25756","8373","5856","5112","6515","15981","22599","10146","10167","4937","12178","16951","6196","16789","8060","9572","19295","20202","17632","7207","17910","4044","29317","2056","12187","12344","12764","30496","6768","6896","10270","8935","13988","11244","52989","9567","39096","20085","15468","10613","51561","11034","7657","13118","12918","10617","22929","9171","1632","13561","13260","16834","10373","29042","10325","5965","17711","15768","12546","17410","13741","7216","4817","17891","16006","16340","31733","11270","8605","33879","3062","13090","17735","10957","44870","16454","3875","10058","45161","14846","16216","1587","1882","9031","13564","12466","14338","3771","4437","10994","10441","14411","6045","10075","13729","6970","2333","6829","11995","20032","839","17295","8295",null,"15385","22387","28380","19850","7448","7349","11394","21071","19712","2684","2765","8282","12439","20830","12182","10910","54736","19059","17094","15350","8244","12371","738","25271","4562","14361","18847","13543","23358","3682","9486","4095","9218","5056","2773","26547","24863","10657","26695","5689","1700","6246","774","10712","13971","5275","2671","5181","5425","3422","3912","8857","25967","18413","8001","26757","22894","18077","3873","37617","8219","16642","6586","18297","6650","4234","25859","33960","7696","24902","7834","12016","31534","24997","68459","7696","8341","3093","1784","13614","7523","21384","15687","17561","14400","12130","6207","23677","28090","23403","10037","18497","8814","1017","21485","22253","7123","24406","11366","12915","10662","12259","3582","1850","4345","26297","28625","19857",null,"5082","1378","805","18262","13693","28401","28257","11686","12723","10831"],["246","263","297","1223","170","338","139","150","316","307","516","206","230","475","949","99","181","131","206","184","212","64","169","124","151","308","323","120","226","116","590","162","956","213","1312","714","302","519","369","386","158","357","399","544","836","236","100","236","513","136","180","340","124","127","326","512","167","1971","667","578","625","871","119","118","166","140","314","59","827","323","112","32","74","56","230","175","407","139","59","201","188","173","1289","200","123","37","741","76","258","180","747","314","267","336","376","161","355","106","1430","60","447","580","40","77","695","187","105","166","510","256","578","231","415","797","875","682","1659","377","334","1021","266","88","102","596","398","279","400","169","1736","226","656","1037","400","705","516","232","1417","45","1384","142","66","72","266","35","246","95","964","86","1035","264","352","392","271","1187","502","2400","586","201","95","134","174","68","983","108","17","2121","19","23","429","394","106","70","50","2259","425","460","79","102","450","94","2287","106","862","2471","286","321","393","193","413","34","603","168","459","1690","341","843","352","1626","742","52","126","408","314","1906","336","268","62","374","1207","84","390","180","179","2012","1068","299","674","608","71","1878","23","399","280","236","1603","141","266","636","104","264","359","3991","252","2222","1243","611","134","3491","192","175","546","448","245","671","171","18","240","339","791","357","3376","264","279","1381","692","555","562","582","197","291","169","259","463","3479","144","50","1667","31","496","686","696","3715","381","64","178","4519","640","683","21","29","439","171","127","754","50","292","117","609","783","572","178","861","131","31","144","155","2310","20","1325","197",null,"636","1975","2313","241","179","301","1642","1723","427","34","29","245","616","1868","791","161","6666","803","1560","447","165","481","21","2466","190","1252","2439","1784","2442","90","1143","109","229","152","48","2454","3504","985","2813","97","32","252","18","303","1409","49","53","390","93","164","136","317","3415","2564","330","3027","4150","2953","110","7064","1095","2009","219","2989","142","231","5062","3283","701","4390","375","1291","5281","2957","14446","241","1632","92","81","1868","652","4767","2768","2808","2966","2747","364","4990","7818","4988","1608","9276","1906","17","8021","4266","1851","5093","3386","4007","258","6057","17","92","137","13568","19129","19808",null,"2870","90","11","10070","206","956","519","386","236","167"],["0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0","0"],["1315","1134","1742","5374","962","1578","1294","2254","1666","2","2886","1111",null,"2634","4330","1169","1021","862","1442","851","2092","867","956","922","1121","1896","2184","988","1592","609","2880","1107","3963","1066","3025","1544","3341","1451","1402","3001","1860","1024","1578","2272","2726","2311","769","1129","2804","510","513","1316","840","1276","2272","1514","2024","3573","2893","3311","3173","1831","1677","1308","1236","902","1765","495","5421","596","1137","432","432","544","1545","566","1342","1176","246","1337","626","1525","4531","2547","3235","513","2649","506","755","666","2379","1828","1518","1380","1677","895","1826","457","8650","318","3529","2539","391","834","2656","588","358","2444","1034","1035","2749","1197","928","3449","1375","4012","7455","1781","825","2694","1311","459","1898","2652","628","1574","989","1304","1730","1004","1168","3283","1525","1928","1868","810","4382","526","2915","866","359","617","1168","598","1216","670","2432","514","1558","997","1864","1651","1215","2419","1941","3719","1286","529","1738","1009","545","527","3089","579","126","6039","202","228","2027","793","286","353","289","5698","1713","953","638","1089","2386","455","4849","596","851","9536","4305","1138","1205","692","1706","681","1981","554","1891","1875","822","2291","960","6559","3298","876","1362","2128","1256","4221","579","837","384","810","2621","908","1655","757","805","5534","2827","1016","1396","1553","494","4345","195","1400","537","1802","2646","640","2646","933","731","961","3921","17995","763","4641","2317","1245","477","5408","1260","598","1033","1826","484","2531","767","122","905","1602","2828","1118","2938","803","976","2330","2240","1239","1628","2964","426","1475","908","597","1638","6460","1195","647","2466","595","1946","2727","3199","5190","1734","701","1037","6988","1664","1990","239","321","1189","1041","1400","1202","569","979","619","1014","1759","947","1109","2213","632","376","336","1119","4654","30","1714","800",null,"2408","3683","2977","987","585","1451","3046","2437","1359","501","387","1329","2114","3057","1683","1022","10330","2265","2224","2971","988","980","46","4169","538","1419","5594","637","3823","426","2479","311","1163","1505","321","3556","2864","2116","4610","817","287","538","75","1303","3411","363","158","1268","353","639","283","1352","4828","3485","933","4889","4666","4193","250","7984","1470","10311","595","3534","770","534","4195","4029","1073","4716","932","4642","4732","2341","6503","325","2584","472","165","3659","1144","3417","2668","2971","2600","2080","496","6295","5360","3561","1262","5725","2083","59","6350","6292","1562","3562","1248","3033","969","4018","318","321","447","4117","10013","8370","4875","647","115","68","2652","1442","3963","1451","3001","1129","2024"]],"container":"<table class=\"display\">\n  <thead>\n    <tr>\n      <th> <\/th>\n      <th>title<\/th>\n      <th>time<\/th>\n      <th>duration<\/th>\n      <th>view<\/th>\n      <th>like<\/th>\n      <th>dislike<\/th>\n      <th>favorite<\/th>\n      <th>comment<\/th>\n    <\/tr>\n  <\/thead>\n<\/table>","options":{"pageLength":4,"order":[[2,"desc"]],"autoWidth":false,"orderClasses":false,"columnDefs":[{"orderable":false,"targets":0}],"lengthMenu":[4,10,25,50,100]}},"evals":[],"jsHooks":[]}</script><!--/html_preserve-->

Para trabalhar com a data de publicação (`time`) e a duração do vídeo (`duration`) convertemos a primeira na classe dttm (datatime) e extraimos via regex o tempo em segundos da segunda variável.


```r
dados_prep <- dados %>% 
  dplyr::as_tibble() %>% 
  purrr::set_names(~sub('Count', '', .x)) %>% 
  mutate(time = lubridate::ymd(time)) %>% 
  mutate(hora = if_else(grepl("H", duration, fixed=TRUE), 
                        true = as.numeric(sub(".*PT *(.*?) *H.*", "\\1", duration)), 
                        false = 0)) %>% 
  mutate(minuto = if_else(grepl("M", duration, fixed=TRUE), 
                        true = as.numeric(sub(".*H *(.*?) *M.*", "\\1", duration)), 
                        false = 0)) %>% 
  mutate(minuto = if_else(is.na(minuto), 
                        true = as.numeric(sub(".*PT *(.*?) *M.*", "\\1", duration)), 
                        false = as.numeric(minuto))) %>% 
  mutate(segundo = if_else(grepl("S", duration, fixed=TRUE), 
                        true = as.numeric(sub(".*M *(.*?) *S.*", "\\1", duration)), 
                        false = 0)) %>% 
  mutate(total = segundo/60 + minuto + hora*60)
```

```
## Warning in if_else(grepl("H", duration, fixed = TRUE), true =
## as.numeric(sub(".*PT *(.*?) *H.*", : NAs introduzidos por coerção
```

```
## Warning in if_else(grepl("M", duration, fixed = TRUE), true =
## as.numeric(sub(".*H *(.*?) *M.*", : NAs introduzidos por coerção
```

```
## Warning in if_else(is.na(minuto), true = as.numeric(sub(".*PT *(.*?)
## *M.*", : NAs introduzidos por coerção
```

```
## Warning in if_else(grepl("S", duration, fixed = TRUE), true =
## as.numeric(sub(".*M *(.*?) *S.*", : NAs introduzidos por coerção
```

Finalmente, para termos uma conjunto de dados representativa do pirula-normal (CNTP!) removemos os vídeos de Live e com participação de convidados


```r
dados_clean <- dplyr::filter(dados_prep, !grepl('feat|LIVE|Live|live', title))
```

### Análise Estatística

O histograma dos tempos apresentado na imagem abaixo nos dá a perspectiva de que a distribuição é assimétrica. Tomar a média dos dados como valor representativo acabaria levando a interpretaçõs incorretas pela sua sensibilidade aos valores da cauda à direita da distribuição.


```r
dados_clean %>% 
  ggplot(aes(x = total)) +
  geom_histogram(col = "white", fill = "black") + 
  xlab("Duração do Vídeo (minutos)") +
  ylab("Quantidade")
```

```
## `stat_bin()` using `bins = 30`. Pick better value with `binwidth`.
```

<img src="post_files/figure-html/unnamed-chunk-12-1.png" style="display: block; margin: auto;" />

O uso da mediana, pode, neste caso, ser uma alternativa razoável e que nos levaria ao pirula de duração igual a 19.7833333 minutos. Valor próximo ao registrado na literatura (vide Twitter hahahhahaha).

<blockquote class="twitter-tweet" data-lang="en"><p lang="pt" dir="ltr">Até onde fizeram o cálculo de todos os meus vídeos, 1 pirula = 20 minutos de duração, e não meia hora como dizem por aí. Calúnias!!</p>&mdash; Pirula #120K (@Pirulla25) <a href="https://twitter.com/Pirulla25/status/836207898527150080?ref_src=twsrc%5Etfw">February 27, 2017</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

A série temporal da duração do vídeo, contudo, revela uma tendência crescente, com o passar do tempo, da duração dos vídeos e da variabilidade deles.


```r
dados_clean %>% 
  ggplot(aes(x = time, y = total)) + 
  geom_line() + geom_point(size = 0.3) + 
  xlab("Ano") + 
  ylab("Duração do Vídeo (minutos")
```

<img src="post_files/figure-html/unnamed-chunk-13-1.png" style="display: block; margin: auto;" />

É interessante notar no gráfico de correlação abaixo que o número de views do vídeo possui baixa correlação a duração do vídeo - o que pode indicar que a audiência não treme quando vê aquele video de +45 minutos. A correlação positiva e alta (algo discutível) das visualizações com o número de likes e comentários pode estar relacionada apenas ao maior número de pessoas que consumiu o vídeo e procurou interagir ou ao fato de que vídeos maiores despertam maior interação do público - Pontos a serem investigados!


```r
dados_clean %>% 
  select(total, view, like, dislike, comment) %>% 
  mutate_all(as.numeric) %>% 
  GGally::ggpairs()
```

<img src="post_files/figure-html/unnamed-chunk-14-1.png" style="display: block; margin: auto;" />

Finalmente, avaliamos a hipótese temporal mais detalhadamente. A imagem abaixo apresenta a mediana da duração dos vídeos agregada por ano. Podemos observar, mais claramente, como o pirula de duração varia ao longo dos anos e seus valores em anos anteriores não são mais representativos.


```r
dados_clean %>% 
  mutate(year = lubridate::year(time)) %>% 
  group_by(year) %>% 
  summarise(media = median(total)) %>% 
  ggplot(aes(x = year, y = media)) + 
  geom_line() + geom_point(size = 0.3) + geom_label(aes(label = round(media, 1)))
```

<img src="post_files/figure-html/unnamed-chunk-15-1.png" style="display: block; margin: auto;" />

Por esta razão, o pirula de duração não deve ser encarado com uma constante física e, por ainda não estar estabilizado, seu valor está atrelado ao ano de publicação dos vídeos. 

Chega-se, assim, a um valor de 24 minutos para o pirula de duração em 2018!

Obrigado por chegar até aqui guerreirinho!

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
  tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>
