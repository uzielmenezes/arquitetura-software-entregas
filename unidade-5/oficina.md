# Oficina de Ferramentas - Arquitetura de Software

## 🎯 Objetivo

- Executar e observar uma API didática (FastAPI).
- Validar o contrato HTTP com diferentes ferramentas.
- Comparar contrato, testes e implementação.
- Entender limites de armazenamento em memória e comportamento da API.

---

## 🛠️ Ferramentas Utilizadas

- **Python 3.11+** → execução da aplicação e testes.
- **FastAPI + Uvicorn** → implementação e servidor HTTP local.
- **OpenAPI 3.1** → contrato explícito (`openapi.yaml`).
- **Bruno** → consumidor manual de requisições.
- **Spectral CLI 6.16.1** → validação de regras do contrato.
- **TestClient (pytest)** → verificação de comportamento da implementação.

---

## 📋 Pré-requisitos

- Repositório da disciplina clonado.
- Editor de texto disponível.
- Instalação de Python, Node.js e Bruno.
- Criação de ambiente virtual (`.venv`) e instalação das dependências.

---

## 🔑 Operações da API

- **POST /elegibilidades**
  - Recebe CPF sintético, código de operadora e matrícula.
  - Retorna `202 Accepted` com protocolo e cabeçalho `Location`.

- **GET /elegibilidades/{protocolo}**
  - Recupera pedido aceito pelo protocolo.
  - Retorna `200 OK` ou `404 Not Found`.

---

## 📂 Evidências a Entregar

- Arquivo `spectral-valido.txt` (lint do contrato).
- Arquivo `testes-contrato.txt` (execução dos testes).
- Coleção Bruno importada.
- Respostas de requisições (POST, GET e erro 422).
- Nota comparativa entre contrato explícito, contrato gerado e execução.
- Evidência da falha deliberada (exemplo inválido detectado).
- **Opcional (extensão Ocelot)**: saída das consultas via gateway e arquivo `ocelot.json`.

---

## 🔍 Questões Explorativas

1. O que o status **202 Accepted** permite ao provedor mudar sem quebrar o consumidor?
2. Por que o cabeçalho **Location** é preferível a convenções de URL?
3. Qual divergência entre OpenAPI e aplicação os testes ainda não detectam?
4. Quando uma chave de idempotência seria necessária?
5. O que deixaria de funcionar com múltiplas instâncias e memória separada?

---

## 📌 Resultado Esperado

- API funcionando localmente em `http://127.0.0.1`.
- Testes de contrato aprovados (`6 passed`).
- Contrato validado com Spectral sem erros.
- Evidências coletadas e organizadas.
- Compreensão clara de como cada ferramenta examina documento, implementação e experiência do consumidor.
