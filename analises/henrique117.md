# Autopsia: AWS S3 – US-EAST-1 (28/02/2017)

**Autor:** Henrique Assis Moreira (@henrique117)
**Fonte primaria:** https://aws.amazon.com/message/41926/
**Data de acesso:** 27/10/2026

## 1. O que aconteceu

- O S3 é o "disco na nuvem" da Amazon: guarda arquivos de milhares de sites e de outros serviços da própria AWS.
- **9h37** (horário do Pacífico; 14h37 em Brasília): numa manutenção de rotina, um engenheiro digitou um valor errado e desligou servidores demais.
- Entre eles estavam os que mantêm o índice que diz onde fica cada arquivo; sem índice, o S3 da região parou de ler e de gravar.
- **Detecção:** o relatório não diz quando a equipe percebeu; o painel público de status só pôde ser atualizado às **11h37**.
- **Restauração:** leitura de volta às **12h26**, tudo normal às **13h54** — cerca de 4h17 fora do ar, levando junto serviços que dependiam do S3.

## 2. Qual das Tres Vias falhou

**Feedback (Segunda Via).**

A Segunda Via pede que o sistema mostre o problema no instante em que ele acontece, para quem o está causando. No S3, a ferramenta de remoção de capacidade executou a ordem sem consultar o estado do sistema: nada avisou nem interrompeu a operação quando ela passou a derrubar o subsistema de índice abaixo do mínimo necessário para funcionar. A principal ação corretiva do relatório é justamente criar esse retorno — uma checagem que bloqueia qualquer remoção que leve um subsistema abaixo da capacidade mínima.

O feedback para fora também falhou: o console de administração do painel de status (Service Health Dashboard) dependia do próprio S3. Por duas horas, das 9h37 às 11h37, o instrumento de monitoramento caiu junto com o que deveria monitorar, e a AWS teve de avisar os clientes pelo Twitter.

Não escolhi Fluxo porque a operação não era uma entrega nova nem um lote grande de mudanças: era rotineira, documentada e feita por alguém autorizado. O que faltou foi o sistema responder ao operador.

## 3. Quais metricas DORA teriam denunciado antes

O relatório não publica métricas; a questão é quais números, acompanhados antes de 28/02, já mostrariam o risco.

**Tempo de restauração (MTTR).** A AWS admite que os subsistemas de índice e de posicionamento não passavam por um reinício completo nas regiões grandes havia muitos anos. Ou seja, o tempo de restauração da pior falha possível não era medido — o último valor conhecido vinha de um S3 muito menor. Como reiniciar o índice exige validar a integridade de todos os metadados, esse tempo cresce junto com o volume armazenado, e o próprio relatório reconhece o crescimento massivo do serviço. Um indicador simples, como "tempo do último ensaio de reinício completo" ou "dias desde o último ensaio", já denunciaria que a organização não sabia quanto tempo levaria para voltar. O mesmo relatório mostra que a divisão do índice em células menores, que reduz esse tempo, estava agendada só para mais tarde naquele ano.

**Taxa de falha de mudanças.** Remover capacidade é, nas palavras da AWS, um dos processos operacionais centrais do S3, feito por comando com parâmetros digitados à mão. Cada execução é uma mudança em produção. Se essas execuções fossem contadas como mudanças, com registro das que precisaram de correção (parâmetro errado, servidor errado removido e reposto), a taxa mostraria a probabilidade de uma digitação errada causar dano — dano que a ferramenta não limitava. O fato de a AWS ter passado a auditar todas as outras ferramentas operacionais depois do incidente indica que ela não tinha esse número.

## 4. Qual pratica do semestre teria evitado -- e em que semana

**Prática:** Infraestrutura como Código com validação automática no pipeline (plan → revisão → apply), com regras de política como código. Acredito que seja da Semana 7.

Em vez de um operador digitar no terminal quais servidores remover, a capacidade de cada subsistema ficaria declarada em um arquivo versionado, e toda mudança passaria por pull request com o pipeline calculando a diferença entre o estado atual e o desejado antes de aplicar. Isso teria barrado o incidente em dois pontos:

1. **Na revisão:** o plano exibiria que a mudança removeria servidores dos subsistemas de índice e de posicionamento — e não só do subsistema de cobrança. Um erro de digitação vira um diff grande e visível, revisado por outra pessoa.
2. **Na validação automática:** uma regra de política rejeitaria qualquer plano que deixasse um subsistema abaixo da capacidade mínima, sem depender de alguém notar. É exatamente a proteção que a AWS adicionou à ferramenta depois do incidente.

## 5. A cultura do relatorio: generativa ou patologica?

**Generativa.**

O trecho central do relatório atribui o problema à ferramenta, não à pessoa:

> "the tool used allowed too much capacity to be removed too quickly"

*(em tradução livre: "a ferramenta utilizada permitiu que capacidade demais fosse removida rápido demais")*

O sujeito da frase é a ferramenta. O operador não é nomeado nem punido; o relatório o descreve como um membro autorizado da equipe que seguia um procedimento estabelecido. Todas as ações corretivas miram o sistema: travas de capacidade mínima, auditoria das demais ferramentas, antecipação da divisão em células e migração do painel de status para várias regiões. Na tipologia de Westrum, isso é o comportamento generativo: a falha leva à investigação do sistema, não à busca de um culpado.

A ressalva é a transparência: o relatório é curto e omite justamente o intervalo entre o início do incidente e a detecção pela equipe, que seria o dado mais revelador da linha do tempo.