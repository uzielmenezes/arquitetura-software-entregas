# Observações — Estilo em Camadas

## Condição alterada

No arquivo `servicos.py`, a regra de conflito de agenda foi enfraquecida para evidenciar a responsabilidade da camada de negócio. A condição original `if horario.conflita_com(existente.horario):` foi substituída por `if True:` em uma cópia de entrega, sem alterar o fluxo de apresentação.

## Evidência

- `saida-antes.txt`: três agendamentos válidos e um conflito esperado com HTTP 409.
- `saida-depois.txt`: a segunda e a terceira consultas do mesmo médico passam a falhar imediatamente por conflito, mostrando que a regra da invariância de agenda está centralizada no serviço e não no controller.

## Ambiente

A execução em `PowerShell` sem UTF-8 falhava com `UnicodeEncodeError` por causa do code page padrão do Windows (`cp1252`). A solução aplicada para registrar a evidência foi:

```powershell
$env:PYTHONUTF8 = '1'
chcp 65001
py main.py
```

Isso preserva o comportamento do código e evita o erro de encoding.
