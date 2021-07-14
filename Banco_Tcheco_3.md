---
title: "Modelo de Classificação de Bons e Maus Pagadores"
author: "Fabio Ribeiro Leal"
date: "29 de setembro de 2020"
output:
  html_document:
    keep_md: yes
---




## Descrição do projeto
Projeto realizado utilizando o banco de dados de um suposto banco tcheco. O objetivo é classificar os clientes do banco entre *bons* e *maus pagadores* para avaliação de concessão de crédito.

O modelo proposto utilizou os dados dos clientes objetivando prever se eles pagarão os empréstimos da tabela *loan*, para tanto, foram utilizadas variáveis das outras tabelas(*trans*, *account*, *client*, *order*, *disp*, *district* e *card*), assim como, na tentativa de captar melhor a tendência de pagamento para cada cliente, foram criadas novas variáveis com base nas já existentes no banco original. O algoritmo de classificação escolhido para o projeto foi a regressão logística. A tabela *loan*, que possui a variável resposta, contém 682 observações, assim a divisão dos dados entre treino e teste deu-se da seguinte forma: foram considerados 70% dos dados da tabela *loan* para a base de treino e 30% para a base de teste. Contudo esta divisão não se deu de forma aleatória, foi estabelecida uma data de corte para os contratos de empréstimos, utilizando como limite a data do primeiro pagamento do empréstimo, já que precisamos de pelo menos um pagamento para formar a nossa variável resposta. Com base na proporcionalidade acima citada, foi fixada a data de corte no dia 31/10/1997, utilizando esse limite para a base treino e os dados com datas posteriores para base teste. A decisão de dividir a base respeitando a temporalidade, e não fazendo de forma aleatória, deve-se ao fato de respeitarmos a utilização de dados passados para prever o futuro, assim foram utilizados os dados das outras tabelas até essa data de corte(31/10/1997), portanto foi simulada a execução do modelo como se estivéssemos em novembro de 1997. O *training set* foi composto pelos clientes da tabela *loan* que tinham algum vencimento de parcela até 31/10/1997 e seus respectivos dados derivados das outras tabelas. O *test set* foi formado pelos clientes da tabela *loan* que não possuiam vencimentos até 31/10/1997 e portanto foi utilizado o dado futuro de pagamento ou não para a aferição da acurácia do modelo. Despois de treinado, o modelo de classificação foi avaliado comparando a predição de pagamento para os contratos feitos a partir de outubro de 1997 com o respectivo dado verdadeiro. Dessa forma, o projeto tornou-se mais realista e foi diminuido o risco de contaminação do *training set*, ou seja, vazamento de informação do *test set* para o *training set*.


### Estrutura do dataset e relacionamentos.


![Caption for the picture.](C:/Users/FABIO/Desktop/Projetos/Czech/czech1.jpg)

#### Descrição das tabelas.
#### client: descreve características dos clientes.
#### account: descreve caracteísticas das contas.
#### disposition: relaciona as contas a cada cliente.
#### card: descreve os cartões emitidos para cada conta.
#### district: descreve características demográficas de cada distrito.
#### order: descreve características das ordens de pagamento.
#### trans: descreve as transações realizadas por cada conta.
#### loan: descreve empréstimos concedidos para cada conta.


### Importação dos arquivos.

```r
setwd("C:/Users/FABIO/Desktop/Projetos/Czech/Czech_data")
getwd()
```

```
## [1] "C:/Users/FABIO/Desktop/Projetos/Czech/Czech_data"
```

```r
data.table::fread("loan.asc", sep = ";") -> loan
data.table::fread("trans.asc", sep = ";") -> trans
data.table::fread("card.asc", sep = ";") -> card
data.table::fread("client.asc", sep = ";") -> client
data.table::fread("account.asc", sep = ";") -> account
data.table::fread("disp.asc", sep = ";") -> disp
data.table::fread("district.asc", sep = ";") -> district
data.table::fread("order.asc", sep = ";") -> order
```


```r
head(loan)
```

```
##    loan_id account_id   date amount duration payments status
## 1:    5314       1787 930705  96396       12     8033      B
## 2:    5316       1801 930711 165960       36     4610      A
## 3:    6863       9188 930728 127080       60     2118      A
## 4:    5325       1843 930803 105804       36     2939      A
## 5:    7240      11013 930906 274740       60     4579      A
## 6:    6687       8261 930913  87840       24     3660      A
```


```r
str(loan)
```

```
## Classes 'data.table' and 'data.frame':	682 obs. of  7 variables:
##  $ loan_id   : int  5314 5316 6863 5325 7240 6687 7284 6111 7235 5997 ...
##  $ account_id: int  1787 1801 9188 1843 11013 8261 11265 5428 10973 4894 ...
##  $ date      : int  930705 930711 930728 930803 930906 930913 930915 930924 931013 931104 ...
##  $ amount    : int  96396 165960 127080 105804 274740 87840 52788 174744 154416 117024 ...
##  $ duration  : int  12 36 60 36 60 24 12 24 48 24 ...
##  $ payments  : num  8033 4610 2118 2939 4579 ...
##  $ status    : chr  "B" "A" "A" "A" ...
##  - attr(*, ".internal.selfref")=<externalptr>
```


```r
head(trans)
```

```
##    trans_id account_id   date   type operation amount balance k_symbol
## 1:   695247       2378 930101 PRIJEM     VKLAD    700     700         
## 2:   171812        576 930101 PRIJEM     VKLAD    900     900         
## 3:   207264        704 930101 PRIJEM     VKLAD   1000    1000         
## 4:  1117247       3818 930101 PRIJEM     VKLAD    600     600         
## 5:   579373       1972 930102 PRIJEM     VKLAD    400     400         
## 6:   771035       2632 930102 PRIJEM     VKLAD   1100    1100         
##    bank account
## 1:           NA
## 2:           NA
## 3:           NA
## 4:           NA
## 5:           NA
## 6:           NA
```


```r
str(trans)
```

```
## Classes 'data.table' and 'data.frame':	1056320 obs. of  10 variables:
##  $ trans_id  : int  695247 171812 207264 1117247 579373 771035 452728 725751 497211 232960 ...
##  $ account_id: int  2378 576 704 3818 1972 2632 1539 2484 1695 793 ...
##  $ date      : int  930101 930101 930101 930101 930102 930102 930103 930103 930103 930103 ...
##  $ type      : chr  "PRIJEM" "PRIJEM" "PRIJEM" "PRIJEM" ...
##  $ operation : chr  "VKLAD" "VKLAD" "VKLAD" "VKLAD" ...
##  $ amount    : num  700 900 1000 600 400 1100 600 1100 200 800 ...
##  $ balance   : num  700 900 1000 600 400 1100 600 1100 200 800 ...
##  $ k_symbol  : chr  "" "" "" "" ...
##  $ bank      : chr  "" "" "" "" ...
##  $ account   : int  NA NA NA NA NA NA NA NA NA NA ...
##  - attr(*, ".internal.selfref")=<externalptr>
```


```r
head(card)
```

```
##    card_id disp_id    type          issued
## 1:    1005    9285 classic 931107 00:00:00
## 2:     104     588 classic 940119 00:00:00
## 3:     747    4915 classic 940205 00:00:00
## 4:      70     439 classic 940208 00:00:00
## 5:     577    3687 classic 940215 00:00:00
## 6:     377    2429 classic 940303 00:00:00
```


```r
str(card)
```

```
## Classes 'data.table' and 'data.frame':	892 obs. of  4 variables:
##  $ card_id: int  1005 104 747 70 577 377 721 437 188 13 ...
##  $ disp_id: int  9285 588 4915 439 3687 2429 4680 2762 1146 87 ...
##  $ type   : chr  "classic" "classic" "classic" "classic" ...
##  $ issued : chr  "931107 00:00:00" "940119 00:00:00" "940205 00:00:00" "940208 00:00:00" ...
##  - attr(*, ".internal.selfref")=<externalptr>
```


```r
head(client)
```

```
##    client_id birth_number district_id
## 1:         1       706213          18
## 2:         2       450204           1
## 3:         3       406009           1
## 4:         4       561201           5
## 5:         5       605703           5
## 6:         6       190922          12
```


```r
str(client)
```

```
## Classes 'data.table' and 'data.frame':	5369 obs. of  3 variables:
##  $ client_id   : int  1 2 3 4 5 6 7 8 9 10 ...
##  $ birth_number: int  706213 450204 406009 561201 605703 190922 290125 385221 351016 430501 ...
##  $ district_id : int  18 1 1 5 5 12 15 51 60 57 ...
##  - attr(*, ".internal.selfref")=<externalptr>
```


```r
head(account)
```

```
##    account_id district_id        frequency   date
## 1:        576          55 POPLATEK MESICNE 930101
## 2:       3818          74 POPLATEK MESICNE 930101
## 3:        704          55 POPLATEK MESICNE 930101
## 4:       2378          16 POPLATEK MESICNE 930101
## 5:       2632          24 POPLATEK MESICNE 930102
## 6:       1972          77 POPLATEK MESICNE 930102
```


```r
str(account)
```

```
## Classes 'data.table' and 'data.frame':	4500 obs. of  4 variables:
##  $ account_id : int  576 3818 704 2378 2632 1972 1539 793 2484 1695 ...
##  $ district_id: int  55 74 55 16 24 77 1 47 74 76 ...
##  $ frequency  : chr  "POPLATEK MESICNE" "POPLATEK MESICNE" "POPLATEK MESICNE" "POPLATEK MESICNE" ...
##  $ date       : int  930101 930101 930101 930101 930102 930102 930103 930103 930103 930103 ...
##  - attr(*, ".internal.selfref")=<externalptr>
```


```r
head(disp)
```

```
##    disp_id client_id account_id      type
## 1:       1         1          1     OWNER
## 2:       2         2          2     OWNER
## 3:       3         3          2 DISPONENT
## 4:       4         4          3     OWNER
## 5:       5         5          3 DISPONENT
## 6:       6         6          4     OWNER
```


```r
str(disp)
```

```
## Classes 'data.table' and 'data.frame':	5369 obs. of  4 variables:
##  $ disp_id   : int  1 2 3 4 5 6 7 8 9 10 ...
##  $ client_id : int  1 2 3 4 5 6 7 8 9 10 ...
##  $ account_id: int  1 2 2 3 3 4 5 6 7 8 ...
##  $ type      : chr  "OWNER" "OWNER" "DISPONENT" "OWNER" ...
##  - attr(*, ".internal.selfref")=<externalptr>
```


```r
head(district)
```

```
##    A1          A2              A3      A4 A5 A6 A7 A8 A9   A10   A11  A12
## 1:  1 Hl.m. Praha          Prague 1204953  0  0  0  1  1 100.0 12541 0.29
## 2:  2     Benesov central Bohemia   88884 80 26  6  2  5  46.7  8507 1.67
## 3:  3      Beroun central Bohemia   75232 55 26  4  1  5  41.7  8980 1.95
## 4:  4      Kladno central Bohemia  149893 63 29  6  2  6  67.4  9753 4.64
## 5:  5       Kolin central Bohemia   95616 65 30  4  1  6  51.4  9307 3.85
## 6:  6  Kutna Hora central Bohemia   77963 60 23  4  2  4  51.5  8546 2.95
##     A13 A14   A15   A16
## 1: 0.43 167 85677 99107
## 2: 1.85 132  2159  2674
## 3: 2.21 111  2824  2813
## 4: 5.05 109  5244  5892
## 5: 4.43 118  2616  3040
## 6: 4.02 126  2640  3120
```


```r
str(district)
```

```
## Classes 'data.table' and 'data.frame':	77 obs. of  16 variables:
##  $ A1 : int  1 2 3 4 5 6 7 8 9 10 ...
##  $ A2 : chr  "Hl.m. Praha" "Benesov" "Beroun" "Kladno" ...
##  $ A3 : chr  "Prague" "central Bohemia" "central Bohemia" "central Bohemia" ...
##  $ A4 : int  1204953 88884 75232 149893 95616 77963 94725 112065 81344 92084 ...
##  $ A5 : int  0 80 55 63 65 60 38 95 61 55 ...
##  $ A6 : int  0 26 26 29 30 23 28 19 23 29 ...
##  $ A7 : int  0 6 4 6 4 4 1 7 4 4 ...
##  $ A8 : int  1 2 1 2 1 2 3 1 2 3 ...
##  $ A9 : int  1 5 5 6 6 4 6 8 6 5 ...
##  $ A10: num  100 46.7 41.7 67.4 51.4 51.5 63.4 69.4 55.3 46.7 ...
##  $ A11: int  12541 8507 8980 9753 9307 8546 9920 11277 8899 10124 ...
##  $ A12: chr  "0.29" "1.67" "1.95" "4.64" ...
##  $ A13: num  0.43 1.85 2.21 5.05 4.43 4.02 2.87 1.44 3.97 0.54 ...
##  $ A14: int  167 132 111 109 118 126 130 127 149 141 ...
##  $ A15: chr  "85677" "2159" "2824" "5244" ...
##  $ A16: int  99107 2674 2813 5892 3040 3120 4846 4987 2487 4316 ...
##  - attr(*, ".internal.selfref")=<externalptr>
```


```r
head(order)
```

```
##    order_id account_id bank_to account_to amount k_symbol
## 1:    29401          1      YZ   87144583 2452.0     SIPO
## 2:    29402          2      ST   89597016 3372.7     UVER
## 3:    29403          2      QR   13943797 7266.0     SIPO
## 4:    29404          3      WX   83084338 1135.0     SIPO
## 5:    29405          3      CD   24485939  327.0         
## 6:    29406          3      AB   59972357 3539.0 POJISTNE
```


```r
str(order)
```

```
## Classes 'data.table' and 'data.frame':	6471 obs. of  6 variables:
##  $ order_id  : int  29401 29402 29403 29404 29405 29406 29407 29408 29409 29410 ...
##  $ account_id: int  1 2 2 3 3 3 4 4 5 6 ...
##  $ bank_to   : chr  "YZ" "ST" "QR" "WX" ...
##  $ account_to: int  87144583 89597016 13943797 83084338 24485939 59972357 26693541 5848086 37390208 44486999 ...
##  $ amount    : num  2452 3373 7266 1135 327 ...
##  $ k_symbol  : chr  "SIPO" "UVER" "SIPO" "SIPO" ...
##  - attr(*, ".internal.selfref")=<externalptr>
```

### Ajustes na tabela *loan*.


```r
library(lubridate)
```

```
## 
## Attaching package: 'lubridate'
```

```
## The following object is masked from 'package:base':
## 
##     date
```

```r
loan$date <- ymd(loan$date)
head(loan$date)
```

```
## [1] "1993-07-05" "1993-07-11" "1993-07-28" "1993-08-03" "1993-09-06"
## [6] "1993-09-13"
```


```r
loan$first_pay <- loan$date + days(30)
head(loan)
```

```
##    loan_id account_id       date amount duration payments status
## 1:    5314       1787 1993-07-05  96396       12     8033      B
## 2:    5316       1801 1993-07-11 165960       36     4610      A
## 3:    6863       9188 1993-07-28 127080       60     2118      A
## 4:    5325       1843 1993-08-03 105804       36     2939      A
## 5:    7240      11013 1993-09-06 274740       60     4579      A
## 6:    6687       8261 1993-09-13  87840       24     3660      A
##     first_pay
## 1: 1993-08-04
## 2: 1993-08-10
## 3: 1993-08-27
## 4: 1993-09-02
## 5: 1993-10-06
## 6: 1993-10-13
```


```r
loan$last_pay <- loan$date + months(loan$duration)
head(loan)
```

```
##    loan_id account_id       date amount duration payments status
## 1:    5314       1787 1993-07-05  96396       12     8033      B
## 2:    5316       1801 1993-07-11 165960       36     4610      A
## 3:    6863       9188 1993-07-28 127080       60     2118      A
## 4:    5325       1843 1993-08-03 105804       36     2939      A
## 5:    7240      11013 1993-09-06 274740       60     4579      A
## 6:    6687       8261 1993-09-13  87840       24     3660      A
##     first_pay   last_pay
## 1: 1993-08-04 1994-07-05
## 2: 1993-08-10 1996-07-11
## 3: 1993-08-27 1998-07-28
## 4: 1993-09-02 1996-08-03
## 5: 1993-10-06 1998-09-06
## 6: 1993-10-13 1995-09-13
```

#### Transformação da variável resposta de A, B, C e D para 0 ou 1.
### 0 <- pagou o empréstimo
### 1 <- não pagou o empréstimo


```r
loan$target <- ifelse(loan$status=="A",0,
                         ifelse(loan$status=="B",1,
                                ifelse(loan$status=="C",0,
                                       ifelse(loan$status=="D",1,"NA"))))
head(loan)
```

```
##    loan_id account_id       date amount duration payments status
## 1:    5314       1787 1993-07-05  96396       12     8033      B
## 2:    5316       1801 1993-07-11 165960       36     4610      A
## 3:    6863       9188 1993-07-28 127080       60     2118      A
## 4:    5325       1843 1993-08-03 105804       36     2939      A
## 5:    7240      11013 1993-09-06 274740       60     4579      A
## 6:    6687       8261 1993-09-13  87840       24     3660      A
##     first_pay   last_pay target
## 1: 1993-08-04 1994-07-05      1
## 2: 1993-08-10 1996-07-11      0
## 3: 1993-08-27 1998-07-28      0
## 4: 1993-09-02 1996-08-03      0
## 5: 1993-10-06 1998-09-06      0
## 6: 1993-10-13 1995-09-13      0
```

![Caption for the picture.](C:/Users/FABIO/Desktop/Projetos/Czech/czech2.jpg)


```r
682*0.7 -> cut_off_date
loan$first_pay <- sort(loan$first_pay)
loan$first_pay[cut_off_date]
```

```
## [1] "1997-11-06"
```



```r
nrow(loan[first_pay<='1997-10-31'])
```

```
## [1] 474
```

```r
nrow(loan[first_pay>'1997-10-31'])
```

```
## [1] 208
```

#### A data de corte foi estipulada no dia 31/10/1997.

### Ajustes na tabela *account*.

#### Tradução para o português dos termos em tcheco da coluna frequency.

#### Verificação dos termos a serem traduzidos.

```r
table(account$frequency)
```

```
## 
##   POPLATEK MESICNE POPLATEK PO OBRATU     POPLATEK TYDNE 
##               4167                 93                240
```

#### Sentença IF para a tradução.


```r
account$frequency <- ifelse(account$frequency=="POPLATEK MESICNE", "EXTRATO MENSAL",
                            ifelse(account$frequency=="POPLATEK TYDNE", "EXTRATO SEMANAL",
                                   ifelse(account$frequency=="POPLATEK PO OBRATU","EXTRATO DEPOIS DE TRANSAÇÃO",NA)))
```


```r
head(account)
```

```
##    account_id district_id      frequency   date
## 1:        576          55 EXTRATO MENSAL 930101
## 2:       3818          74 EXTRATO MENSAL 930101
## 3:        704          55 EXTRATO MENSAL 930101
## 4:       2378          16 EXTRATO MENSAL 930101
## 5:       2632          24 EXTRATO MENSAL 930102
## 6:       1972          77 EXTRATO MENSAL 930102
```

#### Ajuste da data.


```r
library(lubridate)
account$date <- lubridate::ymd(account$date)
```


```r
min(account$date)
```

```
## [1] "1993-01-01"
```

```r
max(account$date)
```

```
## [1] "1997-12-29"
```


```r
head(account)
```

```
##    account_id district_id      frequency       date
## 1:        576          55 EXTRATO MENSAL 1993-01-01
## 2:       3818          74 EXTRATO MENSAL 1993-01-01
## 3:        704          55 EXTRATO MENSAL 1993-01-01
## 4:       2378          16 EXTRATO MENSAL 1993-01-01
## 5:       2632          24 EXTRATO MENSAL 1993-01-02
## 6:       1972          77 EXTRATO MENSAL 1993-01-02
```

#### Filtragem das contas abertas até 31/10/1997 para respeitar a data de corte.


```r
library(dplyr)
```

```
## 
## Attaching package: 'dplyr'
```

```
## The following objects are masked from 'package:lubridate':
## 
##     intersect, setdiff, union
```

```
## The following objects are masked from 'package:stats':
## 
##     filter, lag
```

```
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
```

```r
account_10_97 <- filter(account, date <= 19971031)
```


#### Criação da variável tempo de conta, que corresponde a quantidade de meses a partir da abertura da conta até a data de corte.


```r
account_10_97$account_time<-interval(account_10_97$date, ymd(19971101)) %/% months(1)
```

```
## Note: method with signature 'Timespan#Timespan' chosen for function '%/%',
##  target signature 'Interval#Period'.
##  "Interval#ANY", "ANY#Period" would also be valid
```


```r
head(account_10_97)
```

```
##   account_id district_id      frequency       date account_time
## 1        576          55 EXTRATO MENSAL 1993-01-01           58
## 2       3818          74 EXTRATO MENSAL 1993-01-01           58
## 3        704          55 EXTRATO MENSAL 1993-01-01           58
## 4       2378          16 EXTRATO MENSAL 1993-01-01           58
## 5       2632          24 EXTRATO MENSAL 1993-01-02           57
## 6       1972          77 EXTRATO MENSAL 1993-01-02           57
```

### Ajustes na tabela *trans*.

#### Ajuste da data.


```r
trans$date <- ymd(trans$date)
```

#### Tradução para o português dos termos da coluna type.

#### Sentença IF para a tradução.


```r
trans$type <- ifelse(trans$type=="PRIJEM","CRÉDITO",
                     ifelse(trans$type==" VYDAJ","SAQUE","SAQUE EM DINHEIRO"))
```

#### Tradução para o português dos termos da coluna operation.

#### Sentença IF para a tradução.


```r
trans$operation <- ifelse(trans$operation=="PREVOD NA UCET","ENVIO PARA OUTRO BANCO",
                          ifelse(trans$operation=="PREVOD Z UCTU","RECEBIMENTO DE OUTRO BANCO",
                                 ifelse(trans$operation=="VKLAD","CRÉDITO EM DINHEIRO",
                                        ifelse(trans$operation=="VYBER","SAQUE EM DINHEIRO",
                                               ifelse(trans$operation=="VYBER KARTOU","SAQUE DO CARTÃO DE CRÉDITO",NA)))))
```

#### Tradução para o português dos termos da coluna k_symbol.

#### Sentença IF para a tradução.


```r
trans$k_symbol <- ifelse(trans$k_symbol=="DUCHOD","PENSÃO POR IDADE",
                         ifelse(trans$k_symbol=="POJISTNE","PAGAMENTO DE SEGURO",
                                ifelse(trans$k_symbol=="SANKC. UROK","PENALIZAÇÃO DE JUROS SE CONTA NEGATIVA",
                                       ifelse(trans$k_symbol=="SIPO","DOMÉSTICO",
                                              ifelse(trans$k_symbol=="SLUZBY","PAGAMENTO POR EXTRATO",
                                                     ifelse(trans$k_symbol=="UROK","JUROS CREDITADOS",
                                                            ifelse(trans$k_symbol=="UVER","PAGAMENTO DE EMPRÉSTIMO",NA)))))))
```


```r
head(trans)
```

```
##    trans_id account_id       date    type           operation amount
## 1:   695247       2378 1993-01-01 CRÉDITO CRÉDITO EM DINHEIRO    700
## 2:   171812        576 1993-01-01 CRÉDITO CRÉDITO EM DINHEIRO    900
## 3:   207264        704 1993-01-01 CRÉDITO CRÉDITO EM DINHEIRO   1000
## 4:  1117247       3818 1993-01-01 CRÉDITO CRÉDITO EM DINHEIRO    600
## 5:   579373       1972 1993-01-02 CRÉDITO CRÉDITO EM DINHEIRO    400
## 6:   771035       2632 1993-01-02 CRÉDITO CRÉDITO EM DINHEIRO   1100
##    balance k_symbol bank account
## 1:     700     <NA>           NA
## 2:     900     <NA>           NA
## 3:    1000     <NA>           NA
## 4:     600     <NA>           NA
## 5:     400     <NA>           NA
## 6:    1100     <NA>           NA
```

#### Exploração das transações da tabela para tentar criar variáveis que possam explicar o comportamento dos clientes com relação ao pagamento de dívida.

#### Ajuste do período da tabela *trans* de acordo com a data de corte.


```r
library(dplyr)
trans_10_97 <- filter(trans, date <= as.Date("1997-10-31"))
```


```r
str(trans_10_97)
```

```
## 'data.frame':	681606 obs. of  10 variables:
##  $ trans_id  : int  695247 171812 207264 1117247 579373 771035 452728 725751 497211 232960 ...
##  $ account_id: int  2378 576 704 3818 1972 2632 1539 2484 1695 793 ...
##  $ date      : Date, format: "1993-01-01" "1993-01-01" ...
##  $ type      : chr  "CRÉDITO" "CRÉDITO" "CRÉDITO" "CRÉDITO" ...
##  $ operation : chr  "CRÉDITO EM DINHEIRO" "CRÉDITO EM DINHEIRO" "CRÉDITO EM DINHEIRO" "CRÉDITO EM DINHEIRO" ...
##  $ amount    : num  700 900 1000 600 400 1100 600 1100 200 800 ...
##  $ balance   : num  700 900 1000 600 400 1100 600 1100 200 800 ...
##  $ k_symbol  : chr  NA NA NA NA ...
##  $ bank      : chr  "" "" "" "" ...
##  $ account   : int  NA NA NA NA NA NA NA NA NA NA ...
##  - attr(*, ".internal.selfref")=<externalptr>
```

#### Totalização do total movimentado por conta.


```r
trans_amount <- group_by(trans_10_97,account_id) %>% summarise(total_amount=sum(amount))
head(trans_amount)
```

```
## # A tibble: 6 x 2
##   account_id total_amount
##        <int>        <dbl>
## 1          1      264789.
## 2          2     2504014.
## 3          3       48447.
## 4          4      205792.
## 5          5       25703.
## 6          6      462430.
```

#### Calculo do saldo médio das contas.


```r
trans_balance <- group_by(trans_10_97,account_id) %>% summarise(balance_mean=mean(balance))
head(trans_balance)
```

```
## # A tibble: 6 x 2
##   account_id balance_mean
##        <int>        <dbl>
## 1          1       16351.
## 2          2       35265.
## 3          3       20293.
## 4          4       21295.
## 5          5       16277.
## 6          6       31324.
```

#### Join das tabelas *trans_amount* e *trans_balance*.


```r
trans_summary <- merge(trans_amount,trans_balance, by = "account_id", all=F)
head(trans_summary)
```

```
##   account_id total_amount balance_mean
## 1          1     264789.4     16351.10
## 2          2    2504014.3     35264.81
## 3          3      48447.3     20293.35
## 4          4     205791.8     21294.78
## 5          5      25703.1     16277.16
## 6          6     462429.5     31323.81
```

#### Cálculo de quantas vezes a conta de cada cliente ficou negativa.


```r
arrange(trans_10_97, account_id) -> trans_10_97_sorted
```


```r
trans_10_97_sorted$balance_neg<-ifelse(trans_10_97_sorted$balance<0,1,0)
```


```r
head(filter(trans_10_97_sorted, balance_neg>0))
```

```
##   trans_id account_id       date              type         operation
## 1     4654         19 1997-04-03 SAQUE EM DINHEIRO SAQUE EM DINHEIRO
## 2     4656         19 1997-06-01 SAQUE EM DINHEIRO SAQUE EM DINHEIRO
## 3     4659         19 1997-09-01 SAQUE EM DINHEIRO SAQUE EM DINHEIRO
## 4    53949        180 1994-12-25 SAQUE EM DINHEIRO SAQUE EM DINHEIRO
## 5    54091        180 1994-12-31 SAQUE EM DINHEIRO SAQUE EM DINHEIRO
## 6  3445167        180 1994-12-31 SAQUE EM DINHEIRO SAQUE EM DINHEIRO
##    amount  balance                               k_symbol bank account
## 1 15000.0 -10385.3                              DOMÉSTICO            0
## 2 18500.0 -10604.7                              DOMÉSTICO            0
## 3 17000.0  -9029.3                              DOMÉSTICO            0
## 4 18948.0  -2335.4                                   <NA>           NA
## 5    30.0  -2294.2                  PAGAMENTO POR EXTRATO           NA
## 6     5.3  -2299.5 PENALIZAÇÃO DE JUROS SE CONTA NEGATIVA           NA
##   balance_neg
## 1           1
## 2           1
## 3           1
## 4           1
## 5           1
## 6           1
```


```r
summarise(group_by(trans_10_97_sorted,account_id),times_neg_balance=sum(balance_neg)) -> times_neg_balance_summary
head(times_neg_balance_summary)
```

```
## # A tibble: 6 x 2
##   account_id times_neg_balance
##        <int>             <dbl>
## 1          1                 0
## 2          2                 0
## 3          3                 0
## 4          4                 0
## 5          5                 0
## 6          6                 0
```

#### Cálculo de quanto em valores cada cliente deixou a conta negativa.


```r
summarise(group_by(trans_10_97_sorted,account_id),amount_neg_balance=sum(-balance[balance<0])) -> amount_neg_balance_summary
head(amount_neg_balance_summary)
```

```
## # A tibble: 6 x 2
##   account_id amount_neg_balance
##        <int>              <dbl>
## 1          1                  0
## 2          2                  0
## 3          3                  0
## 4          4                  0
## 5          5                  0
## 6          6                  0
```

#### Join das tabelas *trans_summary* e *times_neg_balance_summary*.


```r
trans_summary_1 <- merge(trans_summary, times_neg_balance_summary, by = "account_id", all=F)
```

#### Join das tabelas *trans_summary_1* e *amount_neg_balance_summary*.


```r
trans_summary_2 <- merge(trans_summary_1, amount_neg_balance_summary, by = "account_id", all = F)
head(trans_summary_2)
```

```
##   account_id total_amount balance_mean times_neg_balance
## 1          1     264789.4     16351.10                 0
## 2          2    2504014.3     35264.81                 0
## 3          3      48447.3     20293.35                 0
## 4          4     205791.8     21294.78                 0
## 5          5      25703.1     16277.16                 0
## 6          6     462429.5     31323.81                 0
##   amount_neg_balance
## 1                  0
## 2                  0
## 3                  0
## 4                  0
## 5                  0
## 6                  0
```

### Ajustes da tabela *card*.
#### Ajuste da data.


```r
card$issued <- substr(card$issued,1,6) %>% ymd()
str(card)
```

```
## Classes 'data.table' and 'data.frame':	892 obs. of  4 variables:
##  $ card_id: int  1005 104 747 70 577 377 721 437 188 13 ...
##  $ disp_id: int  9285 588 4915 439 3687 2429 4680 2762 1146 87 ...
##  $ type   : chr  "classic" "classic" "classic" "classic" ...
##  $ issued : Date, format: "1993-11-07" "1994-01-19" ...
##  - attr(*, ".internal.selfref")=<externalptr>
```


```r
card_10_97 <- filter(card,issued<= as.Date("1997-10-31"))
```

### Ajustes na tabela *client*.

#### Identificação do sexo dos clientes.
#### A coluna birth_number identifica a data de nascinemto e o sexo. As datas estão no formato yymmdd, porém no caso do sexo feminino, o número 50 foi adicionado aos meses(mm).
#### Criação da coluna sex através da identificação dos meses nas datas.


```r
client$sex <- ifelse(substr(client$birth_number, 3, 4)>12,"F","M")
```


```r
head(client)
```

```
##    client_id birth_number district_id sex
## 1:         1       706213          18   F
## 2:         2       450204           1   M
## 3:         3       406009           1   F
## 4:         4       561201           5   M
## 5:         5       605703           5   F
## 6:         6       190922          12   M
```

#### Ajuste da data.


```r
mm_client <-substr(client$birth_number, 3, 4)
subtract = 50
mm_client<- as.numeric(mm_client)
mm_client<- ifelse(mm_client>12,mm_client-subtract,mm_client)
mm_client <- ifelse(mm_client<10,paste(0,mm_client,sep = ""),mm_client)
yy_client <- substr(client$birth_number, 1, 2)
dd_client <- substr(client$birth_number, 5, 6)
client$birthday<-as.numeric(paste(yy_client,mm_client,dd_client, sep = ""))
client$birthday <- ymd(client$birthday)
client$birthday <-format(as.Date(client$birthday, "%y-%m-%d"), "19%y-%m-%d")
```

#### Criação da coluna idade do cliente através da coluna birthday. A idade foi calculada considerando que estamos em outubro de 1997.


```r
interval(client$birthday, ymd(19971101)) %/% months(1) %/% 12  -> client$age_97
```

```
## Note: method with signature 'Timespan#Timespan' chosen for function '%/%',
##  target signature 'Interval#Period'.
##  "Interval#ANY", "ANY#Period" would also be valid
```



```r
head(client)
```

```
##    client_id birth_number district_id sex   birthday age_97
## 1:         1       706213          18   F 1970-12-13     26
## 2:         2       450204           1   M 1945-02-04     52
## 3:         3       406009           1   F 1940-10-09     57
## 4:         4       561201           5   M 1956-12-01     40
## 5:         5       605703           5   F 1960-07-03     37
## 6:         6       190922          12   M 1919-09-22     78
```

### Ajuste na tabela *order*.

#### Tradução para o português dos termos da coluna k_symbol.


```r
order$k_symbol <- ifelse(order$k_symbol == "POJISTNE","PAGAMENTO DE SEGURO", 
                        ifelse(order$k_symbol=="SIPO","DOMÉSTICO",
                              ifelse(order$k_symbol=="UVER","PAGAMENTO DE EMPRÉSTIMO",NA)))
```


```r
head(order)
```

```
##    order_id account_id bank_to account_to amount                k_symbol
## 1:    29401          1      YZ   87144583 2452.0               DOMÉSTICO
## 2:    29402          2      ST   89597016 3372.7 PAGAMENTO DE EMPRÉSTIMO
## 3:    29403          2      QR   13943797 7266.0               DOMÉSTICO
## 4:    29404          3      WX   83084338 1135.0               DOMÉSTICO
## 5:    29405          3      CD   24485939  327.0                    <NA>
## 6:    29406          3      AB   59972357 3539.0     PAGAMENTO DE SEGURO
```

### Ajustes na tabela *dist*.

#### Nomeação das colunas.


```r
names(district)<-c("district_id","name","region","inhabitants","municipalities<499",
                   "municipalities_500_1999","municipalities_2000_9999","municipalities>10000",
                   "cities","ratio_urbans","average_salary","unemploymant_rate_95",
                   "unemploymant_rate_96","enterpreneurs","crimes_95","crimes_96")
```


```r
head(district)
```

```
##    district_id        name          region inhabitants municipalities<499
## 1:           1 Hl.m. Praha          Prague     1204953                  0
## 2:           2     Benesov central Bohemia       88884                 80
## 3:           3      Beroun central Bohemia       75232                 55
## 4:           4      Kladno central Bohemia      149893                 63
## 5:           5       Kolin central Bohemia       95616                 65
## 6:           6  Kutna Hora central Bohemia       77963                 60
##    municipalities_500_1999 municipalities_2000_9999 municipalities>10000
## 1:                       0                        0                    1
## 2:                      26                        6                    2
## 3:                      26                        4                    1
## 4:                      29                        6                    2
## 5:                      30                        4                    1
## 6:                      23                        4                    2
##    cities ratio_urbans average_salary unemploymant_rate_95
## 1:      1        100.0          12541                 0.29
## 2:      5         46.7           8507                 1.67
## 3:      5         41.7           8980                 1.95
## 4:      6         67.4           9753                 4.64
## 5:      6         51.4           9307                 3.85
## 6:      4         51.5           8546                 2.95
##    unemploymant_rate_96 enterpreneurs crimes_95 crimes_96
## 1:                 0.43           167     85677     99107
## 2:                 1.85           132      2159      2674
## 3:                 2.21           111      2824      2813
## 4:                 5.05           109      5244      5892
## 5:                 4.43           118      2616      3040
## 6:                 4.02           126      2640      3120
```

#### Estrutura das tabelas com as variáveis que foram criadas.

![Caption for the picture.](C:/Users/FABIO/Desktop/Projetos/Czech/czech3.jpg)


### Join das tabelas.
#### O ojetivo foi juntar todas as variáveis que pudessem ajudar a classificar os clientes para formar o *training set*.

![Caption for the picture.](C:/Users/FABIO/Desktop/Projetos/Czech/czech4.jpg)

#### Join das tabelas *client* e *district*.


```r
client_district <- merge(client, district, by = "district_id", all.x = TRUE)
```


```r
head(client_district)
```

```
##    district_id client_id birth_number sex   birthday age_97        name
## 1:           1         2       450204   M 1945-02-04     52 Hl.m. Praha
## 2:           1         3       406009   F 1940-10-09     57 Hl.m. Praha
## 3:           1        22       696011   F 1969-10-11     28 Hl.m. Praha
## 4:           1        23       730529   M 1973-05-29     24 Hl.m. Praha
## 5:           1        28       450929   M 1945-09-29     52 Hl.m. Praha
## 6:           1        58       275429   F 1927-04-29     70 Hl.m. Praha
##    region inhabitants municipalities<499 municipalities_500_1999
## 1: Prague     1204953                  0                       0
## 2: Prague     1204953                  0                       0
## 3: Prague     1204953                  0                       0
## 4: Prague     1204953                  0                       0
## 5: Prague     1204953                  0                       0
## 6: Prague     1204953                  0                       0
##    municipalities_2000_9999 municipalities>10000 cities ratio_urbans
## 1:                        0                    1      1          100
## 2:                        0                    1      1          100
## 3:                        0                    1      1          100
## 4:                        0                    1      1          100
## 5:                        0                    1      1          100
## 6:                        0                    1      1          100
##    average_salary unemploymant_rate_95 unemploymant_rate_96 enterpreneurs
## 1:          12541                 0.29                 0.43           167
## 2:          12541                 0.29                 0.43           167
## 3:          12541                 0.29                 0.43           167
## 4:          12541                 0.29                 0.43           167
## 5:          12541                 0.29                 0.43           167
## 6:          12541                 0.29                 0.43           167
##    crimes_95 crimes_96
## 1:     85677     99107
## 2:     85677     99107
## 3:     85677     99107
## 4:     85677     99107
## 5:     85677     99107
## 6:     85677     99107
```

#### Join das tabelas *client_district* e *disp*.


```r
client_district_disp <- merge(client_district, disp, by = "client_id", all.x = TRUE)
```


```r
head(client_district_disp)
```

```
##    client_id district_id birth_number sex   birthday age_97        name
## 1:         1          18       706213   F 1970-12-13     26       Pisek
## 2:         2           1       450204   M 1945-02-04     52 Hl.m. Praha
## 3:         3           1       406009   F 1940-10-09     57 Hl.m. Praha
## 4:         4           5       561201   M 1956-12-01     40       Kolin
## 5:         5           5       605703   F 1960-07-03     37       Kolin
## 6:         6          12       190922   M 1919-09-22     78     Pribram
##             region inhabitants municipalities<499 municipalities_500_1999
## 1:   south Bohemia       70699                 60                      13
## 2:          Prague     1204953                  0                       0
## 3:          Prague     1204953                  0                       0
## 4: central Bohemia       95616                 65                      30
## 5: central Bohemia       95616                 65                      30
## 6: central Bohemia      107870                 84                      29
##    municipalities_2000_9999 municipalities>10000 cities ratio_urbans
## 1:                        2                    1      4         65.3
## 2:                        0                    1      1        100.0
## 3:                        0                    1      1        100.0
## 4:                        4                    1      6         51.4
## 5:                        4                    1      6         51.4
## 6:                        6                    1      6         58.0
##    average_salary unemploymant_rate_95 unemploymant_rate_96 enterpreneurs
## 1:           8968                 2.83                 3.35           131
## 2:          12541                 0.29                 0.43           167
## 3:          12541                 0.29                 0.43           167
## 4:           9307                 3.85                 4.43           118
## 5:           9307                 3.85                 4.43           118
## 6:           8754                 3.83                 4.31           137
##    crimes_95 crimes_96 disp_id account_id      type
## 1:      1740      1910       1          1     OWNER
## 2:     85677     99107       2          2     OWNER
## 3:     85677     99107       3          2 DISPONENT
## 4:      2616      3040       4          3     OWNER
## 5:      2616      3040       5          3 DISPONENT
## 6:      3804      3868       6          4     OWNER
```

#### Join das tabelas *client_district_disp* e *account_97_10*.


```r
table(client_district_disp$type)
```

```
## 
## DISPONENT     OWNER 
##       869      4500
```


```r
client_district_disp_account <- merge(client_district_disp, account_10_97, by = "account_id", all.x = T)
```


#### Criação da variável dependent.

#### Somente os clientes com a classificação type=owner podem solicitar empréstimos, então foram identificados os clientes que possuem dependentes, pois isso pode ajudar a classificar o cliente.


```r
client_district_disp_account$dependent<- duplicated(client_district_disp_account$account_id, fromLast = T) 
```


```r
head(subset(client_district_disp_account, select=c("account_id","type","dependent")))
```

```
##    account_id      type dependent
## 1:          1     OWNER     FALSE
## 2:          2     OWNER      TRUE
## 3:          2 DISPONENT     FALSE
## 4:          3     OWNER      TRUE
## 5:          3 DISPONENT     FALSE
## 6:          4     OWNER     FALSE
```


```r
client_district_disp_account$dependent <- ifelse(client_district_disp_account$dependent==T,1,0)
```


```r
head(subset(client_district_disp_account, select=c("account_id","type","dependent")))
```

```
##    account_id      type dependent
## 1:          1     OWNER         0
## 2:          2     OWNER         1
## 3:          2 DISPONENT         0
## 4:          3     OWNER         1
## 5:          3 DISPONENT         0
## 6:          4     OWNER         0
```

#### Join das tabelas *client_district_disp_account* e *card_10_97*.


```r
client_district_disp_account_card <- merge(client_district_disp_account, card_10_97, by = "disp_id", all.x = T)
```

#### Adição da variável *card* à tabela *client_district_disp_account* para identifcar os clientes que possuem cartão.


```r
client_district_disp_account_card$card <- ifelse(is.na(client_district_disp_account_card$type.y),0,1)
```

#### Exclusão dos clientes *disponent* da base, pois eles não podem solicitar empréstimos.


```r
client_district_disp_account_card_owner <- dplyr::filter(client_district_disp_account_card, type.x=="OWNER")
```


```r
str(client_district_disp_account_card_owner)
```

```
## 'data.frame':	4500 obs. of  33 variables:
##  $ disp_id                 : int  1 2 4 6 7 8 9 10 12 13 ...
##  $ account_id              : int  1 2 3 4 5 6 7 8 9 10 ...
##  $ client_id               : int  1 2 4 6 7 8 9 10 12 13 ...
##  $ district_id.x           : int  18 1 5 12 15 51 60 57 40 54 ...
##  $ birth_number            : int  706213 450204 561201 190922 290125 385221 351016 430501 810220 745529 ...
##  $ sex                     : chr  "F" "M" "M" "M" ...
##  $ birthday                : chr  "1970-12-13" "1945-02-04" "1956-12-01" "1919-09-22" ...
##  $ age_97                  : num  26 52 40 78 68 59 62 54 16 23 ...
##  $ name                    : chr  "Pisek" "Hl.m. Praha" "Kolin" "Pribram" ...
##  $ region                  : chr  "south Bohemia" "Prague" "central Bohemia" "central Bohemia" ...
##  $ inhabitants             : int  70699 1204953 95616 107870 58796 121947 110643 161954 128118 387570 ...
##  $ municipalities<499      : int  60 0 65 84 22 37 49 21 9 0 ...
##  $ municipalities_500_1999 : int  13 0 30 29 16 28 41 37 16 0 ...
##  $ municipalities_2000_9999: int  2 0 4 6 7 7 4 20 6 0 ...
##  $ municipalities>10000    : int  1 1 1 1 1 3 1 3 3 1 ...
##  $ cities                  : int  4 1 6 6 5 11 4 8 8 1 ...
##  $ ratio_urbans            : num  65.3 100 51.4 58 51.9 70.5 51.9 48 85.3 100 ...
##  $ average_salary          : int  8968 12541 9307 8754 9045 8541 8441 8720 9317 9897 ...
##  $ unemploymant_rate_95    : chr  "2.83" "0.29" "3.85" "3.83" ...
##  $ unemploymant_rate_96    : num  3.35 0.43 4.43 4.31 3.6 2.97 4.48 4.5 7.07 1.96 ...
##  $ enterpreneurs           : int  131 167 118 137 124 131 115 116 97 140 ...
##  $ crimes_95               : chr  "1740" "85677" "2616" "3804" ...
##  $ crimes_96               : int  1910 99107 3040 3868 1879 3839 2252 3651 6872 18696 ...
##  $ type.x                  : chr  "OWNER" "OWNER" "OWNER" "OWNER" ...
##  $ district_id.y           : int  18 1 5 12 15 51 60 57 70 54 ...
##  $ frequency               : chr  "EXTRATO MENSAL" "EXTRATO MENSAL" "EXTRATO MENSAL" "EXTRATO MENSAL" ...
##  $ date                    : Date, format: "1995-03-24" "1993-02-26" ...
##  $ account_time            : num  31 56 3 20 5 37 11 25 57 14 ...
##  $ dependent               : num  0 1 1 0 0 0 0 1 0 0 ...
##  $ card_id                 : int  NA NA NA NA NA NA NA NA NA NA ...
##  $ type.y                  : chr  NA NA NA NA ...
##  $ issued                  : Date, format: NA NA ...
##  $ card                    : num  0 0 0 0 0 0 0 0 0 0 ...
##  - attr(*, ".internal.selfref")=<externalptr> 
##  - attr(*, "sorted")= chr "disp_id"
```

#### Join das tabelas *trans_summary_2* e *client_district_disp_account_card_owner*.


```r
dataset_1 <- merge(client_district_disp_account_card_owner,trans_summary_2, by = "account_id", all.x = T)
```


```r
str(dataset_1)
```

```
## 'data.frame':	4500 obs. of  37 variables:
##  $ account_id              : int  1 2 3 4 5 6 7 8 9 10 ...
##  $ disp_id                 : int  1 2 4 6 7 8 9 10 12 13 ...
##  $ client_id               : int  1 2 4 6 7 8 9 10 12 13 ...
##  $ district_id.x           : int  18 1 5 12 15 51 60 57 40 54 ...
##  $ birth_number            : int  706213 450204 561201 190922 290125 385221 351016 430501 810220 745529 ...
##  $ sex                     : chr  "F" "M" "M" "M" ...
##  $ birthday                : chr  "1970-12-13" "1945-02-04" "1956-12-01" "1919-09-22" ...
##  $ age_97                  : num  26 52 40 78 68 59 62 54 16 23 ...
##  $ name                    : chr  "Pisek" "Hl.m. Praha" "Kolin" "Pribram" ...
##  $ region                  : chr  "south Bohemia" "Prague" "central Bohemia" "central Bohemia" ...
##  $ inhabitants             : int  70699 1204953 95616 107870 58796 121947 110643 161954 128118 387570 ...
##  $ municipalities<499      : int  60 0 65 84 22 37 49 21 9 0 ...
##  $ municipalities_500_1999 : int  13 0 30 29 16 28 41 37 16 0 ...
##  $ municipalities_2000_9999: int  2 0 4 6 7 7 4 20 6 0 ...
##  $ municipalities>10000    : int  1 1 1 1 1 3 1 3 3 1 ...
##  $ cities                  : int  4 1 6 6 5 11 4 8 8 1 ...
##  $ ratio_urbans            : num  65.3 100 51.4 58 51.9 70.5 51.9 48 85.3 100 ...
##  $ average_salary          : int  8968 12541 9307 8754 9045 8541 8441 8720 9317 9897 ...
##  $ unemploymant_rate_95    : chr  "2.83" "0.29" "3.85" "3.83" ...
##  $ unemploymant_rate_96    : num  3.35 0.43 4.43 4.31 3.6 2.97 4.48 4.5 7.07 1.96 ...
##  $ enterpreneurs           : int  131 167 118 137 124 131 115 116 97 140 ...
##  $ crimes_95               : chr  "1740" "85677" "2616" "3804" ...
##  $ crimes_96               : int  1910 99107 3040 3868 1879 3839 2252 3651 6872 18696 ...
##  $ type.x                  : chr  "OWNER" "OWNER" "OWNER" "OWNER" ...
##  $ district_id.y           : int  18 1 5 12 15 51 60 57 70 54 ...
##  $ frequency               : chr  "EXTRATO MENSAL" "EXTRATO MENSAL" "EXTRATO MENSAL" "EXTRATO MENSAL" ...
##  $ date                    : Date, format: "1995-03-24" "1993-02-26" ...
##  $ account_time            : num  31 56 3 20 5 37 11 25 57 14 ...
##  $ dependent               : num  0 1 1 0 0 0 0 1 0 0 ...
##  $ card_id                 : int  NA NA NA NA NA NA NA NA NA NA ...
##  $ type.y                  : chr  NA NA NA NA ...
##  $ issued                  : Date, format: NA NA ...
##  $ card                    : num  0 0 0 0 0 0 0 0 0 0 ...
##  $ total_amount            : num  264789 2504014 48447 205792 25703 ...
##  $ balance_mean            : num  16351 35265 20293 21295 16277 ...
##  $ times_neg_balance       : num  0 0 0 0 0 0 0 0 0 0 ...
##  $ amount_neg_balance      : num  0 0 0 0 0 0 0 0 0 0 ...
```

![Caption for the picture.](C:/Users/FABIO/Desktop/Projetos/Czech/czech5.jpg)

#### Adicão da variável resposta da tabela *loan* ao *dataset_1*.


```r
dataset_2 <- merge(dataset_1, loan[,c("account_id","target","first_pay","amount")], by = "account_id", all = F)
```


```r
length(dataset_2$client_id)
```

```
## [1] 682
```

#### Adicão da variável ratio. Ela representa a proporção entre o valor do empréstimo e a média do saldo em conta corrente para cada cliente.


```r
dataset_2$ratio<-dataset_2$amount/dataset_2$balance_mean
```


```r
head(dataset_2$ratio)
```

```
## [1] 2.2955459 1.6155159 0.4824654 6.0170935 3.7924991 3.4208924
```

#### Verificação dos clientes no cadastro que não tiveram movimentação da conta.


```r
dataset_1 %>% summarise(count = sum(is.na(total_amount)))
```

```
##   count
## 1   126
```

#### Verificação de quantos desses clientes estão na tabela *loan*.


```r
filter(dataset_1, is.na(total_amount)) -> account_No_tans
```


```r
nrow(merge(account_No_tans, loan, by = "account_id", all =F))
```

```
## [1] 6
```

#### Foram identificados somente 6 clientes que não tiveram movimentação, portanto eles foram excluídos da base sem grande prejuízo de perda de dados.

#### Exclusão das linhas com missing value de contas que não tiveram movimentção.


```r
dataset_3 <- filter(dataset_2, total_amount != is.na(total_amount))
```


```r
length(dataset_3$client_id)
```

```
## [1] 676
```

![Caption for the picture.](C:/Users/FABIO/Desktop/Projetos/Czech/czech6.jpg)

### Divisão dos dados em treino e teste.


```r
training <- filter(dataset_3, first_pay<=as.Date("1997-10-31"))
```


```r
test <- filter(dataset_3, first_pay>as.Date("1997-10-31"))
```


### Análise exploratória.


```r
library(ggplot2)
ggplot(training, aes(age_97)) + geom_histogram(color="darkblue", fill="skyblue2",binwidth = 2) + xlab("Idade") + ylab("Contagem") + ggtitle("Histograma da Idade")
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-88-1.png)<!-- -->



```r
ggplot(training, aes(target, fill=target)) + geom_bar() + scale_fill_manual(values=c("skyblue4","powderblue")) + xlab("Target") + ylab("Contagem") + ggtitle("Distribuição da Variável Target") + theme(legend.position="none")
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-89-1.png)<!-- -->

#### O dataset está desbalanceado, um tratamento se faz necessário.


```r
ggplot(training, aes(target, age_97, fill=target)) + geom_boxplot() + scale_fill_manual(values=c("skyblue4","powderblue")) + xlab("Target") + ylab("Idade") + ggtitle("Boxplot da Idade vs Target") + theme(legend.position="none")
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-90-1.png)<!-- -->


```r
ggplot(training, aes(target, balance_mean, fill=target)) + geom_boxplot() +scale_fill_manual(values=c("skyblue4","powderblue")) + xlab("Target") + ylab("Saldo médio por cliente") + ggtitle("Boxplot do Saldo Médio vs Target") + theme(legend.position="none")
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-91-1.png)<!-- -->


```r
ggplot(training, aes(target, total_amount, fill=target)) + geom_boxplot() +
scale_fill_manual(values = c("skyblue4","powderblue")) + xlab("Target") + ylab("Total movimentado por cliente") + ggtitle("Boxplot do Total Movimentado vs Target") + theme(legend.position="none")
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-92-1.png)<!-- -->

#### Foram identificados outliers.

#### Tratamento dos outliers. Foi utilizada a técnica de Winsorizing para limitar os valores acima do percentil 95%. Esses valores foram subistituidos pelo valor do percentil limite.

```r
out1<-sort(training$total_amount)[95*length(training$total_amount)/100]
```


```r
training$total_amount<-ifelse(training$total_amount>out1,out1,training$total_amount)
```


```r
ggplot(training, aes(target, total_amount, fill=target)) + geom_boxplot() +
scale_fill_manual(values = c("skyblue4","powderblue")) + xlab("Target") + ylab("Total movimentado por cliente") + ggtitle("Boxplot do Total Movimentado vs Target") + labs(subtitle="Com Tratamento de Outliers") + theme(legend.position="none")
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-95-1.png)<!-- -->


```r
ggplot(training, aes(target, times_neg_balance, fill=target)) + geom_boxplot() +
scale_fill_manual(values = c("skyblue4","powderblue")) + xlab("Target") + ylab("vezes que a conta ficou com saldo negativo") + ggtitle("Boxplot da Quantidade de Vezes que a Conta Ficou Negativa vs Target") + theme(legend.position="none")
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-96-1.png)<!-- -->

#### Foram identificados outliers nessa feature também.

#### Tratamento dos outliers.


```r
out2<-sort(training$times_neg_balance)[95*length(training$times_neg_balance)/100]
```


```r
training$times_neg_balance<-ifelse(training$times_neg_balance>out2,out2,training$times_neg_balance)
```


```r
ggplot(training, aes(target, times_neg_balance, fill=target)) + geom_boxplot() +
scale_fill_manual(values = c("skyblue4","powderblue")) + xlab("Target") + ylab("vezes que a conta ficou com saldo negativo") + ggtitle("Boxplot da Quantidade de Vezes que a Conta Ficou Negativa vs Target") + labs(subtitle="Com Tratamento de Outliers") + theme(legend.position="none")
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-99-1.png)<!-- -->


```r
training %>% group_by(target) %>% summarise(times_neg_balance_median= median(times_neg_balance))
```

```
## # A tibble: 2 x 2
##   target times_neg_balance_median
##   <chr>                     <dbl>
## 1 0                             0
## 2 1                             5
```

#### É importante destacar que, analisando o gráfico, é possível identificar que existe uma diferença entre os grupos com relação à variável neg_negative balance.


```r
ggplot(training, aes(target, amount_neg_balance, fill=target)) + geom_boxplot() +
scale_fill_manual(values = c("skyblue4","powderblue")) + xlab("Target") + ylab("Soma do valor negativo") + ggtitle("Boxplot do Valor Total que  Cada Conta Ficou Negativa vs Target") + theme(legend.position="none")
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-101-1.png)<!-- -->

#### Foram identificados outliers nessa feature também.

#### Tratamento dos outliers.


```r
out3<-sort(training$amount_neg_balance)[99*length(training$amount_neg_balance)/100]
```


```r
training$amount_neg_balance<-ifelse(training$amount_neg_balance>out3,out3,training$amount_neg_balance)
```


```r
ggplot(training, aes(target, amount_neg_balance, fill=target)) + geom_boxplot() +
scale_fill_manual(values = c("skyblue4","powderblue")) + xlab("Target") + ylab("Soma do valor negativo") + ggtitle("Boxplot do Valor Total que Cada Conta Ficou Negativa vs Target") + 
labs(subtitle="Com Tratamento de Outliers") + theme(legend.position="none")
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-104-1.png)<!-- -->



```r
training %>% group_by(target) %>% summarise(neg_balance_amount = median(amount_neg_balance))
```

```
## # A tibble: 2 x 2
##   target neg_balance_amount
##   <chr>               <dbl>
## 1 0                      0 
## 2 1                   9598.
```

#### É válido destacar que, analisando o gráfico, é possível identificar que também existe uma diferença entre os grupos com relação à variável amount_neg_balance.


```r
ggplot(training, aes(target, account_time, fill=target)) + geom_boxplot() +
scale_fill_manual(values=c("skyblue4","powderblue")) + xlab("Target") + ylab("Tempo de conta") + ggtitle("Boxplot do Tempo de Conta vs Target") + theme(legend.position="none")  
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-106-1.png)<!-- -->


```r
ggplot(training, aes(target, card, fill=target)) + geom_bar(stat = 'identity') +
scale_fill_manual(values=c("skyblue4","powderblue")) + xlab("Target") + ylab("Possui cartão") + ggtitle("Distribuição dos Clientes que Possuem Cartão vs Target") + theme(legend.position="none")
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-107-1.png)<!-- -->


```r
ggplot(training, aes(target, dependent, fill=target)) + geom_bar(stat = 'identity') +
scale_fill_manual(values=c("skyblue4","powderblue")) + xlab("Target") + ylab("Possui dependente") + ggtitle("Distribuição dos Clientes que Possuem Dependente vs Target") + theme(legend.position="none")
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-108-1.png)<!-- -->

Analisando o gráfico, é possível identificar que também existe uma diferença entre os grupos com relação à variável dependent.


```r
ggplot(training, aes(target, unemploymant_rate_96, fill=target)) + geom_boxplot() + 
scale_fill_manual(values=c("skyblue4","powderblue")) + xlab("Target") + ylab("Taxa de desemprego da região do cliente") + ggtitle("Boxplot do Desemprego vs Target") + theme(legend.position="none")  
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-109-1.png)<!-- -->



```r
table(training$target, training$district_id.x)
```

```
##    
##      1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23
##   0 48  3  2  4  8  2  1  3  6  4  7  3  5  6  4  6  4  2  5  2  4  0  3
##   1  6  0  1  1  1  2  0  1  0  0  1  0  1  1  1  1  0  0  0  2  2  1  0
##    
##     24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46
##   0  3  1  5  4  5  4  0  9  4  3  4  2  3  5  9  4  4  2  3  5  3  5  4
##   1  2  0  1  0  0  1  1  2  0  0  0  0  0  0  0  1  0  0  1  1  1  0  0
##    
##     47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69
##   0  7  3  3  6  6  9  4 14  7  1  6  4  7  5  4  7  6 13  5  6  2  7  2
##   1  0  0  0  1  0  1  1  5  0  1  0  0  2  2  1  0  0  1  1  0  2  1  2
##    
##     70 71 72 73 74 75 76 77
##   0 16  2 10  1 16  2  3  3
##   1  2  0  2  2  3  0  0  1
```

#### Não existe uma ou algumas regiões que concentrem os clientes que não pagaram os empréstimos.


```r
ggplot(training, aes(target, ratio, fill=target)) + geom_boxplot() +
scale_fill_manual(values=c("skyblue4","powderblue")) + xlab("Target") + ylab("Proporção entre o valor do empréstimo e o saldo médio") + ggtitle("Boxplot da Proporção entre o Valor do Empréstimo e o Saldo Médio vs Target") + theme(legend.position="none")
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-111-1.png)<!-- -->

#### Identificação de outliers e tratamento.


```r
out4<-sort(training$ratio)[95*length(training$ratio)/100]
```


```r
training$ratio<-ifelse(training$ratio>out4,out4,training$ratio)
```


```r
ggplot(training, aes(target, ratio, fill=target)) + geom_boxplot() +
scale_fill_manual(values=c("skyblue4","powderblue")) + xlab("Target") + ylab("Proporção entre o valor do empréstimo e o saldo médio") + ggtitle("Boxplot da Proporção entre o Valor do Empréstimo e o Saldo Médio vs Target") + theme(legend.position="none") + labs(subtitle="Com Tratamento de Outliers")
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-114-1.png)<!-- -->


### Testes de hipóteses.


```r
training$target <- as.factor(training$target)
```


```r
t_test <- t.test(times_neg_balance ~ target, training, var.equal=F)
```


```r
t_test
```

```
## 
## 	Welch Two Sample t-test
## 
## data:  times_neg_balance by target
## t = -11.851, df = 63, p-value < 2.2e-16
## alternative hypothesis: true difference in means is not equal to 0
## 95 percent confidence interval:
##  -4.930116 -3.507384
## sample estimates:
## mean in group 0 mean in group 1 
##         0.00000         4.21875
```

#### A diferença entre as médias dos dois grupos tem significância.



```r
t_test_2 <- t.test(ratio ~ target, training, var.equal=F)
```


```r
t_test_2
```

```
## 
## 	Welch Two Sample t-test
## 
## data:  ratio by target
## t = -4.8309, df = 76.319, p-value = 6.859e-06
## alternative hypothesis: true difference in means is not equal to 0
## 95 percent confidence interval:
##  -2.574894 -1.071609
## sample estimates:
## mean in group 0 mean in group 1 
##        3.259685        5.082937
```

#### A diferença entre as médias dos dois grupos tem significância.


### Seleção das variáveis do modelo.


```r
training_sel <- dplyr::select(training,account_time,dependent,card,times_neg_balance,amount_neg_balance,balance_mean,ratio,first_pay,target)
```


```r
test_sel <- dplyr::select(test,account_time,dependent,card,times_neg_balance,amount_neg_balance,balance_mean,ratio,first_pay,target)
```


### Scaling
#### Normalização do training datase utilizando o método do mínimo e máximo.


```r
training_sel$account_time <- (training_sel$account_time-min(training_sel$account_time))/(max(training_sel$account_time)-min(training_sel$account_time))
```


```r
training_sel$times_neg_balance <- (training_sel$times_neg_balance-min(training_sel$times_neg_balance))/(max(training_sel$times_neg_balance)-min(training_sel$times_neg_balance))
```


```r
training_sel$amount_neg_balance <- (training_sel$amount_neg_balance-min(training_sel$amount_neg_balance))/(max(training_sel$amount_neg_balance)-min(training_sel$amount_neg_balance))
```


```r
training_sel$balance_mean <- (training_sel$balance_mean-min(training_sel$balance_mean))/(max(training_sel$balance_mean)-min(training_sel$balance_mean))
```


```r
training_sel$ratio <- (training_sel$ratio-min(training_sel$ratio))/(max(training_sel$ratio)-min(training_sel$ratio))
```


```r
head(training_sel)
```

```
##   account_time dependent card times_neg_balance amount_neg_balance
## 1    0.9803922         1    0         0.0000000          0.0000000
## 2    0.4705882         0    0         0.4285714          0.7393425
## 3    0.5882353         0    0         0.0000000          0.0000000
## 4    0.2156863         1    0         0.0000000          0.0000000
## 5    0.1764706         0    0         0.0000000          0.0000000
## 6    0.2156863         1    0         0.0000000          0.0000000
##   balance_mean     ratio  first_pay target
## 1   0.35555648 0.2316052 1994-02-04      0
## 2   0.09603884 0.1575157 1996-05-29      1
## 3   0.56363549 0.3542120 1996-06-01      0
## 4   0.34496005 0.3055387 1997-09-09      0
## 5   0.53376934 0.3615078 1997-10-08      0
## 6   0.66645052 0.1565056 1996-12-06      0
```


```r
str(training_sel)
```

```
## 'data.frame':	474 obs. of  9 variables:
##  $ account_time      : num  0.98 0.471 0.588 0.216 0.176 ...
##  $ dependent         : num  1 0 0 1 0 1 1 0 0 1 ...
##  $ card              : num  0 0 0 0 0 0 0 0 0 0 ...
##  $ times_neg_balance : num  0 0.429 0 0 0 ...
##  $ amount_neg_balance: num  0 0.739 0 0 0 ...
##  $ balance_mean      : num  0.356 0.096 0.564 0.345 0.534 ...
##  $ ratio             : num  0.232 0.158 0.354 0.306 0.362 ...
##  $ first_pay         : Date, format: "1994-02-04" "1996-05-29" ...
##  $ target            : Factor w/ 2 levels "0","1": 1 2 1 1 1 1 1 1 1 1 ...
```


#### Normalizando o test dataset utilizando os mínimo e máximo do training set.


```r
test_sel$account_time<-(test_sel$account_time-min(training$account_time))/(max(training$account_time)-min(training$account_time))
```


```r
test_sel$times_neg_balance <- (test_sel$times_neg_balance-min(training$times_neg_balance))/(max(training$times_neg_balance)-min(training$times_neg_balance))
```


```r
test_sel$amount_neg_balance <- (test_sel$amount_neg_balance-min(training$amount_neg_balance))/(max(training$amount_neg_balance)-min(training$amount_neg_balance))
```


```r
test_sel$balance_mean <- (test_sel$balance_mean-min(training$balance_mean))/(max(training$balance_mean)-min(training$balance_mean))
```


```r
test_sel$ratio <- (test_sel$ratio-min(training$ratio))/(max(training$ratio)-min(training$ratio))
```

#### Ajuste do tipo das variáveis.


```r
training_sel$dependent <- as.factor(training_sel$dependent)
training_sel$card <- as.factor(training_sel$card)
training_sel$target <- as.factor(training_sel$target)
test_sel$dependent <- as.factor(test_sel$dependent)
test_sel$card <- as.factor(test_sel$card)
test_sel$target <- as.factor(test_sel$target)
```


```r
str(training_sel)
```

```
## 'data.frame':	474 obs. of  9 variables:
##  $ account_time      : num  0.98 0.471 0.588 0.216 0.176 ...
##  $ dependent         : Factor w/ 2 levels "0","1": 2 1 1 2 1 2 2 1 1 2 ...
##  $ card              : Factor w/ 2 levels "0","1": 1 1 1 1 1 1 1 1 1 1 ...
##  $ times_neg_balance : num  0 0.429 0 0 0 ...
##  $ amount_neg_balance: num  0 0.739 0 0 0 ...
##  $ balance_mean      : num  0.356 0.096 0.564 0.345 0.534 ...
##  $ ratio             : num  0.232 0.158 0.354 0.306 0.362 ...
##  $ first_pay         : Date, format: "1994-02-04" "1996-05-29" ...
##  $ target            : Factor w/ 2 levels "0","1": 1 2 1 1 1 1 1 1 1 1 ...
```


#### Tratamento dos dados desbalanceados.


```r
training_sel$first_pay <- NULL
```


```r
test_sel$first_pay <- NULL
```


```r
ggplot(training, aes(target, fill=target)) + geom_bar() + scale_fill_manual(values=c("skyblue4","powderblue")) + xlab("Target") + ylab("Contagem") + ggtitle("Distribuição da Variável Target") + theme(legend.position="none")
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-138-1.png)<!-- -->


```r
library(DMwR)
```

```
## Loading required package: lattice
```

```
## Loading required package: grid
```

```r
training_bal <- SMOTE(target~.,training_sel,perc.over = 10000, perc.under=100)
```


```r
ggplot(training_bal, aes(target, fill=target)) + geom_bar() + scale_fill_manual(values=c("skyblue4","powderblue")) + xlab("Target") + ylab("Contagem") + ggtitle("Distribuição da Variável Target") + theme(legend.position="none") + labs(subtitle="Com Balanceamento")
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-140-1.png)<!-- -->


### Modelo de Regressão Logística.


```r
fit<-glm(data=training_bal, target~card+dependent+account_time+ratio+balance_mean+times_neg_balance,family=binomial())
```


### Predição das probabilidades.


```r
test_sel$pred=predict(fit, newdata = test_sel, type = "response")
```


```r
pc=.15
test_sel$class<-ifelse(test_sel$pred>pc,1,0)
```


### Matriz de confusão.


```r
library(caret)
test_sel$class<-as.factor(test_sel$class)
```


```r
cm <- confusionMatrix(data = test_sel$class, reference = test_sel$target, positive='1')
```


```r
draw_confusion_matrix <- function(cm) {

  layout(matrix(c(1,1,2)))
  par(mar=c(2,2,2,2))
  plot(c(100, 345), c(300, 450), type = "n", xlab="", ylab="", xaxt='n', yaxt='n')
  title('CONFUSION MATRIX', cex.main=2)

  # create the matrix 
  rect(150, 430, 240, 370, col='seagreen3')
  text(195, 435, '1', cex=1.2)
  rect(250, 430, 340, 370, col='tomato2')
  text(295, 435, '0', cex=1.2)
  text(125, 370, 'Predicted', cex=1.3, srt=90, font=2)
  text(245, 450, 'Actual', cex=1.3, font=2)
  rect(150, 305, 240, 365, col='tomato2')
  rect(250, 305, 340, 365, col='seagreen3')
  text(140, 400, '1', cex=1.2, srt=90)
  text(140, 335, '0', cex=1.2, srt=90)

  # add in the cm results 
  res <- as.numeric(cm$table)
  text(195, 400, res[4], cex=1.6, font=2, col='white')
  text(195, 335, res[3], cex=1.6, font=2, col='white')
  text(295, 400, res[2], cex=1.6, font=2, col='white')
  text(295, 335, res[1], cex=1.6, font=2, col='white')

  # add in the specifics 
  plot(c(100, 0), c(100, 0), type = "n", xlab="", ylab="", main = "DETAILS", xaxt='n', yaxt='n')
  text(10, 85, names(cm$byClass[1]), cex=1.2, font=2)
  text(10, 70, round(as.numeric(cm$byClass[1]), 3), cex=1.2)
  text(30, 85, names(cm$byClass[2]), cex=1.2, font=2)
  text(30, 70, round(as.numeric(cm$byClass[2]), 3), cex=1.2)
  text(50, 85, names(cm$byClass[5]), cex=1.2, font=2)
  text(50, 70, round(as.numeric(cm$byClass[5]), 3), cex=1.2)
  text(70, 85, names(cm$byClass[6]), cex=1.2, font=2)
  text(70, 70, round(as.numeric(cm$byClass[6]), 3), cex=1.2)
  text(90, 85, names(cm$byClass[7]), cex=1.2, font=2)
  text(90, 70, round(as.numeric(cm$byClass[7]), 3), cex=1.2)

  # add in the accuracy information 
  text(50, 35, names(cm$overall[1]), cex=1.5, font=2)
  text(50, 20, round(as.numeric(cm$overall[1]), 3), cex=1.4)
  
  
}  
```


```r
draw_confusion_matrix(cm)
```

![](Banco_Tcheco_3_files/figure-html/unnamed-chunk-147-1.png)<!-- -->

### Para predições de *default* de empréstimos, deve-se destacar que a métrica mais importante é o recall, pois nesse caso não devem haver falsos negativos, ou seja maus pagadores classificados como bons. Essa classificação errada, quando resulta em uma concessão de crédito equivocada, pode gerar mais prejuízo do que a rejeição de crédito para um bom pagador. 

