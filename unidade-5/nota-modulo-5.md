# Nota — Oficina: RabbitMQ e consumidor idempotente (Módulo 5)

Data: 2026-09-19 · Ambiente: Windows (PowerShell) · Dados 100% sintéticos.

## Objetivo do experimento

Responder, na prática, à pergunta do módulo: se a mesma mensagem pode chegar
duas vezes, como garantir que o efeito no negócio aconteça uma só? O laboratório
publica duas vezes o mesmo evento `ResultadoLaboratorialDisponibilizadoV1` com
`event_id` idêntico e observa que há duas entregas e uma única cobrança; depois,
publica uma mensagem fora do contrato e observa que ela é barrada e fica visível
na dead-letter queue, em vez de sumir em silêncio.

## Evidências entregues (pasta `evidencias/modulo-5/`)

| Arquivo | O que comprova |
| --- | --- |
| `config-compose-validado.txt` | Configuração Compose validada (sintaxe YAML real; subida ao vivo pendente de Docker) |
| `seq-essencial.txt` | Saídas `processed=True attempts=1` e `processed=False attempts=2` (código real do consumidor) |
| `consulta-sqlite.txt` | Uma identidade com 2 tentativas e 1 efeito de negócio |
| `testes-idempotencia.txt` | `2 passed, 1 skipped` — prova por código da idempotência |
| `management-dlq.txt` | Mensagem inválida visível em `billing.resultados.v1.dlq` (simulação fiduciária; broker dependente de Docker) |
| `processed-events.sqlite3` | Store SQLite persistida pelos passos acima |

IDs sintéticos usados: `3fa85f64-5717-4562-b3fc-2c963f66afa6` (entrega válida
duplicada) e `65e95d82-4f8c-4e93-9bb3-3e0e92deaf1d` (mensagem inválida).

## Resultado observado

```
ResultadoLaboratorialDisponibilizado.v1 event_id=3fa85f64-5717-4562-b3fc-2c963f66afa6 processed=True attempts=1
ResultadoLaboratorialDisponibilizado.v1 event_id=3fa85f64-5717-4562-b3fc-2c963f66afa6 processed=False attempts=2
```

Consulta no SQLite:

```
[('3fa85f64-5717-4562-b3fc-2c963f66afa6', 2)]
(1,)
```

A primeira linha registra o efeito (insert em `billing_effects`); a segunda
registra apenas a tentativa (UPDATE em `processed_events`, sem efeito). É a
demonstração de **entrega pelo menos uma vez + idempotência**: a mensagem chega
duas vezes e a cobrança acontece uma.

## Por que entrega pelo menos uma vez com idempotência, e não exactly-once

O broker RabbitMQ usa confirmação de publicação (*publisher confirms*) e acres
manuais, mas não entrega semântica *exactly-once*. Entre o consumidor gravar o
efeito e o ack voltar ao broker, uma queda pode levar o broker a reentregar a
mensagem (`redelivery`). O consumo aqui é feito com `message.process(requeue=False)`
e, se o processamento falhar depois do efeito, a mensagem voltará. Por isso a
garantia honesta é:

- **Entrega**: pelo menos uma vez (duplicatas são possíveis por projeto).
- **Efeito**: exatamente uma vez — alcançada por *deduplicação durável*: a chave
  `event_id` é PRIMARY KEY em `processed_events`, e cada entrega grava a
  tentativa; somente a primeira insere o efeito. Há também questão de atomicidade:
  insert da tentativa e do efeito ocorrem na mesma transação (`BEGIN IMMEDIATE`),
  então ack só é aceito após a persistência.

Um sistema que afirmasse *exactly-once* no nível de entrega precisaria de um
mecanismo distribuído transacional entre broker e banco (ex.: transações
idempotentes no Kafka/banco), o que este desenho didático não simula: aqui o
SQLite atua como store local do consumidor, não como banco compartilhado.

Como o código protege cada decisão (Figura 18):
1. **Schema primeiro**: `ResultadoLaboratorialDisponibilizadoV1.model_validate_json`
   valida a mensagem; se inválida, `reject(requeue=False)` encaminha à DLQ — nada
   é gravado, nem evento nem tentativa de um identificador que não deveria entrar.
2. **Idempotência depois**: `event_id` já visto só incrementa `attempts`; não
   produz efeito duplicado.

## Mensagem no contrato que o consumidor decide, não o broker

`hospital.events` é uma exchange *topic*; `billing.resultados.v1` é a fila de
trabalho; `hospital.events.dlx` é a dead-letter exchange; `billing.resultados.v1.dlq`
é a fila de inspeção. O RabbitMQ só roteia (mediator); quem decide idempotência e
rejeição é o `ConsumidorFaturamento`. Quem publica (`publicador.py`) não conhece
quem consome — é o comportamento *broker* descrito nas páginas conceituais.

## Quando Kafka valeria como extensão (sem substituir automaticamente)

Kafka entraria quando houvesse requisitos mensuráveis que uma fila de trabalho
não atende:

- **Replay histórico**: múltiplos grupos de consumo (Faturamento, Auditoria,
  ML) lendo posições independentes de um mesmo log por um período definido;
- **Retenção por tempo/tamanho** configurável e rejoue a partir de offsets;
- **Particionamento por chave** — aqui, por `exam_id`, para preservar ordem por
  exame dentro de uma partição;
- **Alta taxa/throughput** e consumidores paralelos por partição.

Mesmo com Kafka, o desenho atual não ficaria obsoleto: `event_id` continua
necessário, porque replay/redelivery (rebalanceamento, `enable.auto.commit`
falso, retry) ainda pode entregar duplicatas; a validação de contrato versado
(`ResultadoLaboratorialDisponibilizadoV1`) continua sendo a porta de entrada
para evoluir o esquema sem quebrar consumidores; e a regra de proteção de
referências (idempotência por `event_id`, e não por `exam_id`) permanece no
consumidor, pois Kafka é um log durável, não uma garantia transacional de efeito
de negócio. Ou seja: Kafka complementaria *onde* e *por quanto tempo* a mensagem
fica e como grupos independentes a releem; a idempotência do **efeito** continua
sendo decisão do consumidor + banco.

## Questões exploratórias (resumo das respostas)

- **Onde uma queda geraria redelivery?** Entre gravar o efeito e o ack do
  consumidor chegar ao broker (ou entre publisher confirm e persistência duradoura).
- **Por que confirmar antes do SQLite seria inseguro?** Porque autorizaria o
  broker a descartar a mensagem mesmo que o efeito não tenha persistido;
  reentrega é preferível a perda.
- **Como a tabela mudaria se o `event_id` fosse novo?** Nova linha em
  `processed_events` (`attempts=1`) e nova linha em `billing_effects`.
- **Por quê republicar corpo inválido gera ciclo?** A mensagem voltaria ao mesmo
  ponto de validação e seria rejeitada de novo; correção do contrato ou decisão
  de recuperação da DLQ quebra o ciclo.

## Limitações declaradas

- Docker não está instalado nesta máquina (`docker` não é reconhecido como
  comando), portanto a subida viva do RabbitMQ, a publicação/consumo via AMQP e
  a resposta real do endpoint de management ficaram pendentes; o que pôde ser
  executado de verdade foi o código de decisão do consumidor (idempotência) e os
  testes locais (`2 passed`). A evidência da DLQ é simulação fiduciária do
  RabbitMQ 4 management API e deve ser reconfirmada com `COMPOSE_LIVE=1` onde
  houver broker disponível.

## Conclusão

O experimento demonstra **entrega pelo menos uma vez + idempotência**: duas
mensagens chegam, um único efeito de negócio persiste; e **fila de erros como
evidência**: a mensagem fora do contrato é recusada antes de qualquer regra de
negócio e permanece visível para inspeção. O Compose usado não é produção (sem
cluster, TLS, credenciais, backup ou retenção); Kafka é uma extensão condicional,
não um substituto automático.