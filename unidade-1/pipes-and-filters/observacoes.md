# Observações — Pipes and Filters

## Condição alterada

No arquivo `main.py`, a vaga foi alterada de `experiencia_minima=3` para `experiencia_minima=1`. O objetivo foi testar como a etapa de filtro de experiência muda o fluxo completo do pipeline sem mexer na ordem dos filtros.

## Evidência

- `saida-antes.txt`: três candidatos aprovados e um ranking final com 3 encaminhamentos.
- `saida-depois.txt`: Bruno Rocha passa no filtro de experiência e o pipeline conclui com 4 candidatos aprovados. A mudança foi visível antes do report final, confirmando que o filtro de experiência faz descarte antes do cálculo de score.

## Responsabilidade arquitetural

A regra de negócio está no filtro `FiltroPorExperienciaMinima`, enquanto o `Pipeline` apenas conecta etapas; a apresentação final é responsabilidade do `RelatorioDeTriagem` no fim do fluxo.

## Ambiente

O erro de encodificação em Windows foi resolvido com `chcp 65001` e `PYTHONUTF8=1`, porque os caracteres `≥` e `─` exigem UTF-8 em console.
