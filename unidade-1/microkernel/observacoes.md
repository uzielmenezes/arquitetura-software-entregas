# Observações — Microkernel

## Condição alterada

No plugin de frete, o limiar de isenção foi rebaixado de `5_000.00` para `1_000.00` em `plugins/frete.py`. A mudança deixa o frete gratuito para praticamente todas as faturas, alterando uma regra do plugin sem tocar no núcleo.

## Evidência

- `saida-antes.txt`: apenas a fatura maior que R$ 5.000,00 tinha frete zero.
- `saida-depois.txt`: todas as faturas passam a exibir `Frete: R$0.00`, demonstrando que o núcleo apenas orquestra e que a decisão real sobre a regra está no plugin.

## Responsabilidade arquitetural

O `CoreFaturamento` conhece apenas o contrato do plugin e a ordem das categorias (`impostos`, `frete`, `notificacao`). O comportamento concreto é implementado pelo plugin de frete, que reage ao contexto da fatura e altera o total final.

## Ambiente

O erro de execução em PowerShell foi causado pelo encoding do console. O ajuste `chcp 65001` + `PYTHONUTF8=1` corrigiu a falha sem mudar a lógica do programa.
