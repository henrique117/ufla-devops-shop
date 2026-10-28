# Analise de access.log -- Henrique Assis Moreira (@henrique117)

**Linhas analisadas:** 516866 dados/access.log

## 1. Volume e falha

```bash
awk '{t++} $9 ~ /^4/ {c4++} $9 ~ /^5/ {c5++} END {printf "total: %d\n4xx: %d (%.2f%%)\n5xx: %d (%.2f%%)\n", t, c4, c4*100/t, c5, c5*100/t}' dados/access.log
```

```
total: 516866
4xx: 6162 (1.19%)
5xx: 11749 (2.27%)
```

**Leitura:** Cerca de 3,5% das requisições falharam. Os erros do servidor (5xx) são quase o dobro dos erros do cliente (4xx), o que aponta para um problema na aplicação e não em quem a acessa.

## 2. Os 10 IPs mais frequentes

```bash
awk -F'"' '{split($1, a, " "); ip = a[1]; m = substr(a[4], 2, 17); split($2, r, " "); c = r[2]; n[ip]++; if (++pm[ip, m] > pico[ip]) pico[ip] = pm[ip, m]; if (++pc[ip, c] == 1) nc[ip]++; if (pc[ip, c] > topn[ip]) {topn[ip] = pc[ip, c]; top[ip] = c}; if ($6 !~ /Mozilla/) bot[ip]++} END {for (ip in n) printf "%6d %-15s pico: %4d/min | %4d caminhos | mais pedido: %-24s | UA nao-navegador: %3.0f%% | %s\n", n[ip], ip, pico[ip], nc[ip], top[ip], bot[ip]*100/n[ip], (pico[ip] >= 100 || bot[ip] == n[ip]) ? "SUSPEITO" : "ok"}' dados/access.log | sort -rn | head -10
```

```
 88400 203.0.113.47    pico: 1021/min |    4 caminhos | mais pedido: /api/busca?q=mochila     | UA nao-navegador: 100% | SUSPEITO
  1788 192.0.2.245     pico:    7/min |   13 caminhos | mais pedido: /                        | UA nao-navegador:   0% | ok
  1772 192.0.2.171     pico:    7/min |   13 caminhos | mais pedido: /                        | UA nao-navegador:   0% | ok
  1771 192.0.2.81      pico:    8/min |   13 caminhos | mais pedido: /                        | UA nao-navegador:   0% | ok
  1771 192.0.2.225     pico:    8/min |   13 caminhos | mais pedido: /                        | UA nao-navegador:   0% | ok
  1771 192.0.2.16      pico:    7/min |   13 caminhos | mais pedido: /                        | UA nao-navegador:   0% | ok
  1767 192.0.2.138     pico:    7/min |   13 caminhos | mais pedido: /                        | UA nao-navegador:   0% | ok
  1762 192.0.2.222     pico:    9/min |   13 caminhos | mais pedido: /                        | UA nao-navegador:   0% | ok
  1757 192.0.2.45      pico:    8/min |   13 caminhos | mais pedido: /                        | UA nao-navegador:   0% | ok
  1753 192.0.2.166     pico:    7/min |   13 caminhos | mais pedido: /                        | UA nao-navegador:   0% | ok
```

**Leitura:** O `203.0.113.47` é suspeito, e não só pelo volume: ele fez 17% de todo o tráfego, mas o que pesa é o comportamento. Os outros nove IPs se parecem com usuários reais: navegam por 13 caminhos, usam navegador e não passam de 9 requisições por minuto. Ele, ao contrário, repete apenas 4 buscas, chega a 1021 requisições por minuto (mais de 100 vezes o ritmo humano) e nunca usa navegador. É um script automatizado.

## 3. O endpoint quebrado

```bash
awk '{split($7, a, "?"); p = a[1]; t[p]++; if ($9 == 500) e[p]++} END {for (p in e) printf "%6d erros 500 de %6d requisicoes (%5.1f%%)  %-24s %s\n", e[p], t[p], e[p]*100/t[p], p, (e[p] == t[p]) ? "sempre" : "as vezes"}' dados/access.log | sort -rn | head -5
```

```
  3620 erros 500 de  10400 requisicoes ( 34.8%)  /api/relatorio/gerar     as vezes
   228 erros 500 de  71682 requisicoes (  0.3%)  /                        as vezes
   167 erros 500 de  52100 requisicoes (  0.3%)  /api/produtos            as vezes
   160 erros 500 de  58793 requisicoes (  0.3%)  /produtos                as vezes
   150 erros 500 de  45541 requisicoes (  0.3%)  /produtos/detalhe        as vezes
```

**Leitura:** O `/api/relatorio/gerar` quebra às vezes, mas com muita frequência: falha em 34,8% das chamadas, cerca de uma a cada três. Os demais caminhos têm uma taxa de fundo de 0,3%, o que torna esse endpoint mais de 100 vezes pior que o resto do sistema. É ele o defeito real, e não uma instabilidade geral.

## 4. A hora do pico

```bash
awk '{print $4}' dados/access.log | cut -d: -f2 | sort | uniq -c
```

```
   3262 00
   1967 01
   1810 02
   1471 03
   1498 04
   1844 05
   3904 06
   9759 07
  19519 08
  27621 09
  30869 10
  32526 11
  31895 12
  29952 13
  31225 14
  32529 15
  30886 16
  28575 17
  25996 18
  22807 19
  18860 20
  15577 21
  43979 22
  68535 23
```

**Leitura:** O tráfego normal forma um platô de ~30 mil requisições por hora entre 10h e 16h, com queda gradual à noite (15577 às 21h). O pico absoluto, porém, é às 23h: 68535 requisições, mais que o dobro do horário comercial. Junto com o salto às 22h, esse pico quebra a curva natural e coincide com o horário de maior atividade do IP `203.0.113.47`.

## 5. Alguem batendo na porta

```bash
awk 'tolower($7) ~ /\/admin|\.env|\.git|wp-login|phpmyadmin/ {n++; if (!($1 in ip)) d++; ip[$1]++; st[$9]++} END {print "tentativas:", n; print "IPs distintos:", d; for (i in ip) print "  IP " i ":", ip[i], "tentativas"; for (s in st) print "  status " s ":", st[s]}' dados/access.log
```

```
tentativas: 2080
IPs distintos: 2
  IP 198.51.100.9: 1053 tentativas
  IP 198.51.100.23: 1027 tentativas
  status 404: 2080
```

**Leitura:** Foram 2080 tentativas de acesso a caminhos sensíveis, divididas quase igualmente entre dois IPs da mesma faixa (`198.51.100.x`), o que sugere uma única varredura automatizada. O servidor respondeu 404 a todas, então nenhum arquivo sensível foi exposto: é ruído de fundo, não emergência.

## Conclusao: minha primeira acao como operador de plantao

Minha primeira ação seria bloquear (ou limitar a taxa de) o IP `203.0.113.47` no Nginx. Os dados mostram que ele sozinho gerou 17% do tráfego, a até 1021 requisições por minuto, enquanto usuários reais não passam de 9. Ele usa só `curl` e se concentra às 23h, justamente o horário do pico anormal de 68535 requisições e do servidor lento. É a ação de maior efeito imediato e também a mais fácil de reverter: se for uma integração legítima, o bloqueio se desfaz em segundos e o responsável aparece. O `/api/relatorio/gerar`, com 34,8% de falha contra 0,3% do resto do sistema, é o segundo problema, mas pede correção no código e não um ajuste de madrugada. Por isso eu abriria um chamado para o time responsável. A varredura dos IPs `198.51.100.x` recebeu apenas 404 e não exige ação agora.
