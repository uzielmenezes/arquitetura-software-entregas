# Unidade 3 - Oficina: dois serviços, dois bancos e uma falha parcial

## Resumo do que foi executado e o que ficou limitado

Esta oficina tem dois percursos previstos pelo roteiro: a demonstração com
contêineres (Docker Compose) e a verificação por testes Python. **Nesta máquina
não há Docker instalado** (nem `docker.exe` no PATH, nem no `Program Files`,
nem em `LocalAppData`), portanto segui o roteiro de contingência do próprio
documento:

> Se o daemon não respondeu, faça somente `config --quiet` e os testes Python
> e registre: "Compose validado estaticamente; execução de contêineres não
> realizada porque o daemon não respondeu". Não afirme que observou health
> checks sem uma execução real.

**Registro formal:** Compose validado estaticamente; execução de contêineres
não realizada porque o daemon não respondeu (Docker não instalado). Não houve
observação de health checks em contêineres reais.

O que foi efetivamente executado:

- Validação estática de `infra/compose.servicos.yml` (YAML parseado, 4 serviços,
  3 redes, 2 volumes, health checks e dependências conferidos) - arquivo
  `02-compose-validacao-estatica.txt`.
- Testes de fronteira `tests/test_service_boundaries.py` - **4 passed** -
  arquivo `03-testes-fronteiras.txt`. Eles cobrem o contrato HTTP (201),
  a falha parcial (503 `dependencia_indisponivel`), a falha da base própria
  (503 `banco_indisponivel`) e a ausência de SQL do Exames contra a tabela de
  Elegibilidade.
- Demonstração em nível de aplicação (TestClient + `httpx.MockTransport`) da
  tabela de tradução de falhas de `exames.py` - arquivos
  `04-demonstracao-tabela-falhas.txt` e `05-tabela-falhas.json`. O 201 esperado
  foi reproduzido simulando a gravação no banco (como o próprio teste faz), pois
  sem Docker não há PostgreSQL local.
- Prova da fronteira no código-fonte - arquivo `06-fronteira-no-codigo.txt`.

## A fronteira declarada no Compose (leitura estática)

As evidências de versões/ferramentas estão em `01-versoes.txt`.

O Compose declara **três redes**, e não uma:

| Rede | `internal` | Quem participa |
|---|---|---|
| `application-net` | `false` | elegibilidade, exames |
| `elegibilidade-db-net` | `true` | db_elegibilidade (alias `elegibilidade-db`), elegibilidade |
| `exames-db-net` | `true` | db_exames (alias `exames-db`), exames |

Consequências declaradas na topologia:

- **exames** só enxerga **db_exames**. Não há rota de rede até `db_elegibilidade`;
- **elegibilidade** só enxerga **db_elegibilidade**;
- os dois bancos não publicam porta (apenas `5432/tcp`, sem `0.0.0.0:porta`),
  ficando inalcançáveis a partir do host;
- `depends_on` com `condition: service_healthy` ordena a inicialização: bancos
  primeiro, depois aplicações; exames só sobe com banco e vizinho saudáveis;
- os health checks chamam apenas `/health` do próprio serviço - ou seja, o
  health check de Exames não verifica a saúde de Elegibilidade.

Isso concretiza a tese do módulo: **banco por serviço é uma decisão de
autoridade (quem pode alcançar o quê), não de tecnologia** - os dois bancos
rodam o mesmo PostgreSQL 16.

## Questões exploratórias

### 1. Qual dependência permanece saudável e qual capacidade deixa de ser concluída?

Pelo roteiro, ao parar `elegibilidade` com `docker compose stop elegibilidade`:

- **Dependências que permanecem saudáveis:** o processo de Exames (o health
  check `/health` continua 200 enquanto a base própria responde) e o banco
  próprio `db_exames`. Para o Docker, o contêiner `exames` continua `healthy`.
- **Capacidade que deixa de ser concluída:** o `POST /exames`, porque a sua
  sequência é **chamar Elegibilidade por HTTP e só depois gravar** na base. Sem
  o vizinho, a operação não é concluída e responde **503
  `dependencia_indisponivel`**.
- A contradição observável é a do mesmo instante: `/health` do Exames respondendo
  `200 OK` enquanto `POST /exames` responde `503`. A capacidade degrada sem que o
  processo caia - é a **falha parcial**.

Tal falha parcial é possível porque `exames.py` trata a chamada ao vizinho
(incluindo o `timeout=2.0s` do cliente HTTP), não deixa a requisição pendurar e
traduz cada falha do vizinho num código próprio (tabela em `05-tabela-falhas.json`).
Sem esse prazo de espera, a indisponibilidade de um serviço viraria a
indisponibilidade dos dois (ausência de timeouts não é um cenário em chamadas
locais dentro do mesmo processo - é o custo da fronteira física).

## O que NÃO foi observado nesta execução

Por ausência do Docker, **não** foi executado nesta roda: `up -d --build --wait`,
`docker compose ps` com os quatro `healthy`, `curl` para `/health`, o `201` real
via rede, `stop elegibilidade` + `503` real, nem `down -v`. Esses são exatamente
os momentos em que o roteiro pede para comparar `ps` com resposta da API. Em uma
execução com Docker disponível, repita os comandos da seção "Execução" do
documento e guarde as saídas nesta pasta.