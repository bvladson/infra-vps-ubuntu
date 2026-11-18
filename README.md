# Uma VPS Ubuntu, nove sistemas em produção

Relato técnico da infraestrutura que hospeda todos os projetos deste portfólio: **um único
servidor Ubuntu 24.04 LTS**, 2 vCPUs e 7,75 GiB de RAM, rodando em produção os sistemas de
duas empresas ao mesmo tempo — sem queda por falta de memória em nenhum dos períodos medidos.

Este repositório não tem código de aplicação. Tem o que normalmente não se escreve: **como se
opera um servidor compartilhado por nove sistemas sem derrubar nenhum.**

---

## O servidor

| Item | Valor |
| --- | --- |
| Máquina | Hostinger KVM 2 — 2 vCPUs AMD EPYC 9354P (Zen 4, AVX-512), 7,75 GiB RAM, 96 GB de disco |
| Sistema | Ubuntu 24.04 LTS |
| Containers | Docker 29 + Compose v5, cgroup v2, logs limitados a 10 MB × 3 |
| Exposição | **Somente as portas 22, 80 e 443.** Todo o resto escuta em `127.0.0.1` |
| Firewall | UFW ativo, entrada negada por padrão |
| Proxy | nginx no host, um *server block* por domínio, TLS por domínio |

Medição de 17 dias, amostras a cada 5 minutos: memória disponível **nunca abaixo de 3,5 GB**,
carga típica entre 0,4 e 0,65, CPU média de 13–20% das duas vCPUs. Nenhuma hora do dia sem
folga.

## O que roda ali

| Sistema | Como | Repositório |
| --- | --- | --- |
| BIOMIE v2 — BI comercial | Docker: Postgres 17, PostgREST, GoTrue, nginx, web e worker | [biomie-v2](https://github.com/bvladson/biomie-v2) |
| BIOMIE v1 — BI em PHP | php-fpm + MariaDB (desativado em 23/10/2026) | [biomie-v1](https://github.com/bvladson/biomie-v1) |
| Site MAG + CRM comercial | Docker, 3 containers, SQLite | [mag-site-crm](https://github.com/bvladson/mag-site-crm) |
| MAG AI — LLM local | Docker: Ollama + gateway, isolado da rede | [mag-ai](https://github.com/bvladson/mag-ai) |
| Base de tarefas MAG | php-fpm + instância MariaDB exclusiva | [mag-base](https://github.com/bvladson/mag-base) |
| ERP da VSB (ERPNext) | Docker, 9 containers | [vsb-360-integrations](https://github.com/bvladson/vsb-360-integrations) |
| n8n da VSB | Docker, 3 containers | [vsb-360-integrations](https://github.com/bvladson/vsb-360-integrations) |
| Site e CMS da VSB | Docker + WordPress | [vsb-site-institucional](https://github.com/bvladson/vsb-site-institucional) |
| SCF — RH e folha | Docker, 2 containers | — |

Mais o conector de WhatsApp (WAHA) e o *builder* da homologação do BIOMIE.

---

## As cinco regras

Nenhuma delas é teoria: cada uma nasceu de um problema concreto.

### 1. Todo serviço tem um manifesto

Antes de um sistema entrar na VPS, ele ganha um `MANIFESTO.md` instalado em `/srv/<serviço>/`
que responde, linha por linha: quais containers são meus, em que porta local, sob qual
usuário, com qual limite de memória e CPU, onde ficam meus dados, quais arquivos têm segredo
e qual o modo deles, que cron é meu, e **o que no servidor não me pertence**.

Sem isso, em um servidor com nove sistemas, ninguém sabe o que pode reiniciar.
→ Exemplos reais: [site MAG](https://github.com/bvladson/mag-site-crm/blob/main/deploy/srv/MANIFESTO.md)
· [MAG AI](https://github.com/bvladson/mag-ai/blob/main/srv/MANIFESTO.md)

### 2. Nada entra em produção sem os quatro itens

**Diagnóstico · impacto medido · teste na cópia · caminho de volta escrito antes.**

A cópia de homologação não é um ambiente parecido: é a produção clonada, atualizada na hora
(`41_homolog.py`). O que não passa lá não chega aqui.

O portão é uma **foto da produção**: `50_foto_producao.py` tira um retrato dos números antes
da mudança e outro depois. Se não baterem, a ativação não acontece — e a volta é um comando
único já escrito antes de começar.

### 3. Operação pesada tem hora

Build, clone de banco, carga de ERP e teste de navegador competem por 2 vCPUs com a operação
real das empresas. A janela é verificada por código, não por disciplina:

```
Testes pesados só rodam:
  • fora do expediente — dias úteis antes das 07:30 ou a partir das 18:30;
    sábado a partir das 12:30; domingo livre
  • longe das cargas do worker (:21 e :51) — só nos minutos :00–:12 e :30–:42
  • sem leitura de estoque em andamento ou na fila
  • com o servidor folgado — carga de 1 min abaixo de 1,0 nas 2 CPUs
Fora disso o comando é RECUSADO.
```
→ [`janela_testes.py`](https://github.com/bvladson/biomie-v2/blob/main/ops/janela_testes.py)

Recusa não é falha: o script sai com sucesso e explica por que não rodou. Isso permite
agendar sem medo de alarme falso.

### 4. Backup só existe se a restauração foi testada

`vps-backup-run` roda para todos os projetos de madrugada, cifra o pacote, copia para fora do
servidor e **confere o hash** na origem e no destino. Cada pacote carrega dentro o seu próprio
`RESTORE.md`.

E a restauração foi exercitada de verdade, não presumida: a do banco do BIOMIE v1 foi
reconstruída do arquivo cifrado e conferida tabela a tabela — **119 de 119 tabelas,
2.670.587 linhas idênticas**. Só depois disso o desligamento do sistema antigo foi autorizado.
→ [procedimento de restauração](https://github.com/bvladson/mag-site-crm/blob/main/deploy/srv/RESTORE.md)

### 5. Segredo não passa pela máquina de desenvolvimento

Chaves de API são gravadas direto no runtime do servidor, em arquivo `0600` do root, por um
script que lê de `stdin` sem eco e **nunca imprime nem grava o valor localmente**. O
desenvolvimento confere o `stat` do arquivo, não o conteúdo.

Para o banco de produção, a máquina local conecta por túnel SSH com usuário **somente
leitura**, e uma barreira no código (`ProductionDatabaseGuard`) aborta qualquer migration
apontada para produção — porque isso já aconteceu uma vez.

---

## Isolamento entre sistemas

Quando o LLM local (MAG AI) foi instalado, a pergunta não foi "funciona?", foi **"ele alcança
o banco do BIOMIE?"**. A resposta virou evidência arquivada: uma sonda a partir do container
tentou todas as portas internas do servidor e de outros containers.

```
bloqueado 203.0.113.10:5432 (ENETUNREACH)    ← Postgres do BIOMIE
bloqueado 203.0.113.10:3306 (ENETUNREACH)    ← MariaDB
bloqueado 172.16.14.1:22    (ECONNREFUSED)   ← SSH do host
...
```

Todas as linhas dizem "bloqueado". Essa é a prova de que a rede do serviço novo não alcança
nada — e ela fica no repositório, datada.
→ [evidências de isolamento](https://github.com/bvladson/mag-ai/tree/main/docs/evidencias-20261107)

Cada sistema roda sob **usuário de sistema próprio, sem login**, com raiz do container
somente leitura, sem capabilities, e limites de memória e CPU declarados no compose.

---

## Lições que custaram caro

| O que aconteceu | O que mudou |
| --- | --- |
| Uma migration apontada para o banco de produção | Barreira estrutural no código, com lista explícita de alvos de produção; o teste falha se a barreira sumir |
| Um deploy abortado por `__pycache__` sujando a árvore | O `.gitignore` passou a ser parte do contrato de deploy; a publicação exige árvore limpa |
| Script `.sh` gerado no Windows quebrou na VPS com "required file not found" | CRLF. `.gitattributes` com `eol=lf` em todo script, e conferência com `file` antes de rodar |
| Comparação de horário entre MariaDB (UTC) e aplicação (America/Fortaleza) deu falso positivo | Todo portão passou a comparar em UTC explicitamente |
| Um comparador de páginas reprovou uma publicação boa | O comparador virou aviso, não portão; o portão passou a ser a foto dos números |
| Imagens Docker encheram 35,8 GB do disco | Limpeza diária de imagens sem uso às 05:01 |
| Dois agentes publicando no mesmo servidor no mesmo dia | Combinação de janelas por escrito antes, e `49_release_ativa.py` responde "o que está no ar agora?" antes de qualquer ação |

---

## Sobre esta cópia pública

O endereço real da VPS foi trocado pelo IP de documentação `203.0.113.10` (RFC 5737) — o
endereço verdadeiro é, de qualquer forma, o registro DNS público dos domínios. O
identificador da instância foi trocado por `srv-vps`. Nenhuma credencial, chave ou dado de
cliente aparece aqui.
