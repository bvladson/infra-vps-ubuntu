# Modelo de release: publicar sem apostar

Como uma release entra em produção nesta VPS. O modelo vale para qualquer um dos sistemas;
os nomes de script abaixo são os do BIOMIE v2, que é onde ele está mais completo.

A ideia central é simples: **a publicação não é o momento da decisão.** Quando o comando de
ativação roda, tudo já foi decidido e medido. Ativar é só apertar o botão de algo que já se
sabe que funciona.

## As seis etapas

### 1. Saber o que está no ar

```
49_release_ativa.py
```

Primeira pergunta de qualquer sessão de trabalho, sempre. Responde qual release está ativa,
desde quando e com qual migration. Sem isso, qualquer plano é chute — especialmente quando
mais de uma pessoa (ou mais de um agente) opera o mesmo servidor.

### 2. Subir a cópia e validar ali

```
41_homolog.py        clona a produção para a cópia de homologação
22_test_database.py  suíte de banco contra a cópia
29_smoke.py          teste de fumaça com navegador real na cópia
```

A cópia não é "um ambiente parecido": é a produção clonada na hora. Uma suíte que passa num
banco com dados de exemplo não prova nada sobre um banco com 2,6 milhões de linhas e os casos
estranhos que a vida real produziu.

### 3. Tirar a foto de antes

```
50_foto_producao.py
```

Um retrato dos números que a produção mostra agora: faturamento, metas, contagens por
empresa, por tela. É o portão. Não é um teste que diz "funcionou"; é um registro que diz
"estes eram os números".

### 4. Esperar a janela

```
janela_testes.py
```

Build, clone e carga competem por 2 vCPUs com a operação real. A janela é verificada por
código: fora do expediente, longe das cargas recorrentes do worker, sem leitura de estoque
na fila, com o servidor folgado. Fora disso o comando é **recusado** — e a recusa sai com
sucesso, explicando o motivo, para não disparar alarme falso em agendamento.

### 5. Ativar

```
21_activate_release.py <release> --start-worker
```

Um comando. A volta já está escrita, com o identificador da release anterior, **antes** de
começar:

```
21_activate_release.py <release-anterior> --start-worker
25_load.py resume
```

### 6. Tirar a foto de depois e comparar

```
50_foto_producao.py --anterior=<release-anterior>
```

Se os números não baterem com a foto de antes, volta. Não há discussão, não há "deve ser só
o cache": o portão é numérico.

## O que esse modelo evita

**O deploy que "deu certo" e quebrou no dia seguinte.** A foto pega diferença de número, que
é o que o usuário percebe — não diferença de pixel, que é o que um comparador visual pega. Um
comparador de páginas chegou a reprovar uma publicação perfeitamente boa; virou aviso, não
portão.

**A volta improvisada.** Escrever o comando de retorno *antes* obriga a responder "isso é
reversível?" enquanto ainda dá tempo de mudar de ideia. Uma migration nova muda a resposta:
quando há migration, o retorno deixa de ser só trocar a release.

**O pisão em outro operador.** Mais de um agente trabalhou nesta VPS no mesmo dia. A regra
foi: perguntar o que está no ar, combinar a janela por escrito antes, avisar ao terminar.
Uma vez duas releases candidatas existiam ao mesmo tempo e só uma continha a funcionalidade
combinada — foi o `49_release_ativa.py` que pegou, antes da ativação.

## O que não tem portão automático

Honestidade sobre os limites do modelo:

- **Carga do ERP.** O Omie é instável em alguns horários e tem limites de chamada próprios
  (mesma consulta uma vez por minuto; 10 erros derrubam a chave por 30 minutos). Falha de
  leitura no fechamento já deixou duas empresas sem curva nova. O modelo detecta, não evita.
- **Dado que entra errado na origem.** Se o cadastro no ERP está errado, a foto bate e o
  número continua errado. Por isso existe uma tela de Qualidade dos Dados.
- **Primeira execução de um agendamento.** Um cron novo só prova que funciona na primeira vez
  que roda sozinho. Até lá, é acompanhado à mão.
