Introdução
==========

Nosso objetivo neste primeiro tutorial da série é desenvolver um modelo que avalia a qualidade das defesas e ataques dos times que disputam o Campeonato Brasileiro.

Bem, qual o impacto dos confrontos entre os times para a escolha dos jogadores do Cartola?

1.  Goleiros e defensores e técnicos ganham 5 pontos por não tomarem gol. Então, estamos falando de 25 pontos caso escalemos defesas que não tomem gols.
2.  Meias e atacantes possuem melhores chances contra defesas fracas.

Por que você deveria usar nosso método?
---------------------------------------

Tradicionalmente, os cartoleiros e cartoleiras acessam somente a tabela simples do campeonato brasileiro com saldo de gols, gols marcados e sofridos para estimar as chances de ganhar saldo de gol. Cartoleiros e cartoleiras mais avançadas separam as três estatísticas ainda quando o time joga dentro de casa e fora. A realidade é mais complexa que isso.

No entanto, nós do Cartola PFC podemos ajudamos você a medir a incerteza usando estatística. A literatura científica sobre previsão de partidas é [vasta](https://scholar.google.com.br/scholar?hl=pt-BR&as_sdt=0%2C5&q=soccer+match+prediction&btnG=&oq=soccer+match). Vamos nos concentrar no trabalho de [Dixon e Coles (1997)](https://rss.onlinelibrary.wiley.com/doi/abs/10.1111/1467-9876.00065), que analisou 6629 partidas da Premier League para criar um modelo estatístico de previsão. O modelo de previsão dos autores têm quatro vantagens ao assumir as características:

1.  As habilidades de cada time são uma função do seu desempenho ;
2.  O efeito "mando de campo" é levado em consideração para a estimação dos parâmetros;
3.  A habilidade de um time deve levar em consideração resultados mais recentes;
4.  Devido à natureza do futebol, um time pode ser resumido em duas habilidades: habilidade de ataque (capacidade de fazer gols) e defesa (habilidade de não levar gols);

Além de estimar a qualidade das defesas e dos ataques, para nós interessados no Cartola FC, há uma curiosidade extra. Qual a probabilidade de um time ficar sem levar gols? Para isso iremos avançar no processo de modelagem.

Preparar dados!
===============

Vamos nos aproveitar do repositório caRtola no [Github](https://github.com/henriquepgomide/caRtola) para baixar os dados atualizados sobre as partidas. Os códigos abaixo foram escrito em [R](https://www.r-project.org).

``` r
# Carregar pacotes
library(tidyverse) # Manipulação dos dados
library(fbRanks)   # Uso do modelo de Dixon e Coles para estimativa da força de um time

# Abrir dados dos confrontos
matches <- read.csv("~/caRtola/data/2018/2018_partidas.csv", stringsAsFactors = FALSE)
```

``` r
# Visualizar banco de dados
glimpse(matches)
```

    ## Observations: 220
    ## Variables: 9
    ## $ X.1       <int> 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 1...
    ## $ game      <int> 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 1...
    ## $ round     <int> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 2, 2, 2, 2, 2, 2, 2...
    ## $ date      <chr> "14/04/2018 - 16:00", "15/04/2018 - 19:00", "15/04/2...
    ## $ home_team <chr> "Cruzeiro - MG", "Atlético - PR", "América - MG", "V...
    ## $ score     <chr> "0 x 1", "5 x 1", "3 x 0", "2 x 2", "2 x 1", "1 x 1"...
    ## $ away_team <chr> "Grêmio - RS", "Chapecoense - SC", "Sport - PE", "Fl...
    ## $ arena     <chr> "Mineirão - Belo Horizonte - MG", "Arena da Baixada ...
    ## $ X         <lgl> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, ...

Etapas de preparação dos dados
------------------------------

O trabalho em análise de dados é basicamente limpeza e preparo. Aqui não será diferente. O que precisamos fazer:

1.  Transformar o campo de datas "date" para um tipo 'Date' no R;
2.  Separar os resultados dos confrontos manipulando a coluna "score";
3.  Selecionar somente datas necessárias para criação de nosso modelo.

``` r
# Limpeza de dados

# 1. Transformar coluna date em 'Date'
matches$date <- as.Date(matches$date, format = "%d/%m/%Y - %H:%M")

# 2. Separar dados da coluna "score"
matches            <- separate(matches, score, c("home.score","vs","away.score"), convert = TRUE)
matches$home.score <- as.integer(matches$home.score)
matches$away.score <- as.integer(matches$away.score)

# 3. Retirar dados até a data 03/09/2018
matches <- filter(matches, date <= "2018-09-03")

# 4. Adicional - renomear colunas do banco de dados de acordo com estilo R
matches <- 
  matches %>%
  dplyr::select(game,round,date,
         home_team,home.score, away.score,
         away_team)

names(matches)[c(4,7)] <- c("home.team", "away.team")
```

Modelagem
---------

Pronto. Depois da limpeza, a modelagem. Para facilitar nossa vida, vamos usar o pacote fbRanks, especificamente a função rank.teams, que usará o modelo especificado no início do artigo.

``` r
# Criar sistema de pontuação dos times
team_features <- rank.teams(scores          = matches, 
                            family          = "poisson",
                            fun             = "speedglm",
                            max.date        = Sys.Date(),
                            time.weight.eta = 0.01)
```

Como interpretar os resultados acima?
-------------------------------------

Você deverá olhar as colunas ataque e defesa. Quanto maior for o valor, ataque do São Paulo é igual a 1.59 contra o ataque do Paraná que é igual a 0.45. Isso nos ajuda a avaliar os times e principalmente possibilita avaliar os resultados dos próximos confrontos. Para avaliar, é só usar o código abaixo.

Explicação
----------

Se você gosta e entende de estatística, estes valores de defesa e ataque são simplesmente os valores lambda da distribuição de [Poisson](https://en.wikipedia.org/wiki/Poisson_distribution). Para avaliar um determinado confronto entre o time A vs. time B, a quantidade de gols esperada de um time é estimada através da razão da habilidade de ataque de um time A e a defesa do time B. Para estimar então os resultados da partida, podemos simular n encontros dos times e calcularmos as chances de vitória, derrota, empate como também as chances do time não ser vazado. O modelo que estamos trabalhando em questão possibilita ainda um ajuste para estimação das habilidades de um time ao atribuir impacto maior aos jogos mais recentes do campeonato.

Prever resultados de jogos no futuro
====================================

``` r
teamPredictions <- predict.fbRanks(team_features, 
                                   newdata  = matches[, c(3,4,7)],
                                   min.date = "2018-09-01",
                                   max.date = "2018-09-03")
```

Como interpretar os resultados acima?
-------------------------------------

Estes são os resultados para a rodada 22 do campeonato brasileiro. Como você deve interpretar: No confronto entre Grêmio e Botafogo, o time da casa (HW) tem 65% de chances de ganhar, o time fora de casa (AW) 10%, e 25% (T) de chances de um empate acontecer. O resultado previsto é Grêmio 2, Botafogo 0.

Além disso, precisamos estimar as chances de um time não levar gols. Para isso usamos o código abaixo:

``` r
home.team.names   <- names(teamPredictions$home.score)
away.team.names   <- names(teamPredictions$away.score)
home.goals.vector <- rep(NA,10)
away.goals.vector <- rep(NA,10)

for (i in 1:10){
  home.goals.vector[i] <- round(prop.table(table(teamPredictions$home.goals[i,] == 0))[1],2)
  away.goals.vector[i] <- round(prop.table(table(teamPredictions$away.goals[i,] == 0))[1],2)
}

team.names       <- c(home.team.names, away.team.names)
clean.sheet.odds <- c(home.goals.vector, away.goals.vector)

clean.sheet.df <- data.frame(team.names = team.names, clean.sheet.odds = clean.sheet.odds*100)
clean.sheet.df <- arrange(clean.sheet.df, desc(clean.sheet.odds))

print(clean.sheet.df)
```

    ##            team.names clean.sheet.odds
    ## 1      São Paulo - SP               84
    ## 2      Palmeiras - SP               84
    ## 3        América - MG               80
    ## 4         Grêmio - RS               79
    ## 5       Atlético - PR               76
    ## 6         Santos - SP               73
    ## 7       Atlético - MG               71
    ## 8       Flamengo - RJ               70
    ## 9  Internacional - RS               70
    ## 10 Vasco da Gama - RJ               68
    ## 11   Corinthians - SP               67
    ## 12         Bahia - BA               61
    ## 13       Vitória - BA               58
    ## 14         Sport - PE               55
    ## 15        Paraná - PR               54
    ## 16    Fluminense - RJ               52
    ## 17   Chapecoense - SC               41
    ## 18         Ceará - CE               39
    ## 19      Cruzeiro - MG               36
    ## 20      Botafogo - RJ               35

Isso significa que o São Paulo possui 84% de chances de não levar gol na rodada 22. Lembrando que o modelo leva em conta os gols marcados pelo São Paulo quando joga em casa e gols levados pela defesa do Fluminense quando joga fora de casa.

Concluindo, se queremos montar uma defesa com potencial ganho de pontos por saldo de gols para a rodada 22, devemos escolher jogadores de times com alto valores na coluna acima como São Paulo, Palmeiras, América e Grêmio.

Considerações
=============

Estas informações podem nos ajudar a escalar melhor os jogadores para compor as defesas dos nossos times do cartola. Algumas limitações devem ser observadas: 1. Precisamos ainda validar o modelo usando séries históricas. 2. Precisamos otimizar o ponto ótimo do peso dos últimos confrontos para determinar as probabilidades 3. Existem outros modelos que podem nos gerar estimativas mais precisas

Quer saber quais serão os resultados para as próximas rodadas?
==============================================================

Boa parte do código para análise está aí para quem quiser aprimorar e estudar. Se quiser acompanhar nossa série de previsões, acompanhe nosso site [Cartola PFC](https://www.cartolapfc.com.br). Para mais dicas, faça seu cadastro gratuito. É de graça. :)
