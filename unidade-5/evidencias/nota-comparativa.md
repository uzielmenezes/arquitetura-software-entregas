# Nota comparativa: contrato, implementação e execução

## Contrato explícito
O arquivo [contratos/openapi.yaml](../contratos/openapi.yaml) descreve as operações `POST /elegibilidades` e `GET /elegibilidades/{protocolo}` com os schemas `PedidoElegibilidade`, `ElegibilidadeAceita` e `ErroAPI`. Ele também exige `cpf` com 11 dígitos, `202 Accepted` e o cabeçalho `Location` na resposta do POST.

## Contrato gerado
A aplicação FastAPI gera o OpenAPI em tempo de execução através de `app.openapi()`. Os testes validam que os caminhos, schemas e campos obrigatórios do contrato gerado coincidem com o YAML explícito. Isso mostra que a implementação e a descrição do contrato têm forte sobreposição, mas não são equivalentes.

## Execução real
Ao chamar a API em execução, observou-se:
- `POST /elegibilidades` respondeu `202 Accepted`.
- O corpo continha `protocolo`, `situacao` e `criado_em`.
- O cabeçalho `Location` apontava para `/elegibilidades/{protocolo}`.
- `GET /elegibilidades/{protocolo}` respondeu `200 OK` com o mesmo payload.
- Quando o campo `cpf` foi omitido, a API respondeu `422 Unprocessable Entity` com `codigo: dados_invalidos` e `campo: body.cpf`.

## Conclusão
O contrato comunica a intenção; os testes confirmam o comportamento-chave; e a execução HTTP valida a experiência real do consumidor. Cada uma dessas perspectivas observa um aspecto diferente da API, e todas devem convergir para o mesmo resultado.
