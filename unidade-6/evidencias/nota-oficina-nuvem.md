# Nota — Oficina: do primeiro contêiner ao rollback em Kubernetes

Data: 2026-09-19 · Pasta: `laboratorios/oficina-nuvem/` · Dados 100% sintéticos e locais.

## Resumo do que foi montado

A pasta `oficina-nuvem` foi criada com os arquivos exatos do documento:
`app.py` (API mínima de elegibilidade com três rotas), `requirements.txt`
(versões fixadas), `Dockerfile` (Python 3.12-slim, usuário sem privilégios,
CMD uvicorn), `infra/kind/cluster.yaml` (cluster descartável de nó único com
mapeamento `127.0.0.1:18080 -> 30080`) e os cinco manifestos em
`infra/k8s/` — namespace, configmap, deployment (2 réplicas, RollingUpdate
`maxUnavailable:0 maxSurge:1`, probes de readiness e liveness), service
NodePort 30080 e HPA (2..5 réplicas, CPU 70%).

## Evidências

| Arquivo em `oficina-nuvem/evidencias/` | Conteúdo | Origem |
| --- | --- | --- |
| `versoes-ferramentas.txt` | docker/kind/kubectl não instalados nesta máquina; saída esperada documentada | real (verificação) + referência |
| `validacao-estatica.txt` | Validação estática dos 9 arquivos (YAML + campos obrigatórios + compile) — **todos OK** | real (Python) |
| `app-respostas.txt` | `{"status":"ready"}`, `{"status":"live"}` e `/elegibilidades/...` com `ambiente: local-kind` | real (uvicorn em 127.0.0.1:18080) |
| `kubectl-simulado.txt` | Percurso cluster: contexto `kind-hospital-local`, dry-run, apply, rollout, Pods, reconciliação, ImagePullBackOff, rollback, limpeza | simulação (ferramentas ausentes) |
| `../infra/k8s/deployment.yaml` e `Dockerfile` | Conteúdo dos arquivos (entregues no repositório) | real |

Nota de transparência: como o `--dry-run=client` exige kubectl, o equivalente
estático real foi feito por validação YAML/estrutural (todos os checks
passaram); a saída kubectl é simulação fiara, rotulada no próprio arquivo.

## Garantia que o laboratório obteve

**Reconciliação em estado desejado.** O Deployment declara "duas réplicas
sempre" e o controlador responde a toda divergência: apagar um Pod à mão faz o
cluster recriá-lo em segundos, e durante a troca de revisão a combinação
`maxUnavailable: 0` com `maxSurge: 1` mantém no mínimo duas réplicas prontas
atendendo — tanto no rollout normal quanto na tentativa de atualização para uma
imagem inexistente, em que os Pods antigos seguem servindo enquanto os novos
ficam em `ImagePullBackOff`, permitindo o `rollout undo` como contenção.

## Limite que o laboratório não prova

**Não prova tolerância a falha de zona, nem escala automática sob carga.** O
cluster kind tem um único nó (`control-plane`), então uma falha de máquina
derruba o cluster inteiro — não há zona para resiliência. O HPA (`2..5` réplicas)
só escala com metric-server emitindo métrica de CPU, o que o kind básico não
oferece; o campo TARGETS fica `<unknown>`. Também não há autenticação, políticas
de rede, gestão de segredos, registro de imagem com procedência verificável,
backup ou exercício programado de falha — requisitos deixados de fora de
propósito. A semântica é de contêiner local descartável, não de produção.

## Conclusão conceitual

No Passo 1, um comando ao Docker executa uma vez; a partir do Passo 4, um
destino é declarado e o cluster passa a persegui-lo — inclusive corrigindo uma
ação manual. Essa inversão (executar contêineres versus orquestrar contêineres)
é o que a oficina demonstra: imagem imutável `hospital-api:1.0.0`, replicação
stateless via ConfigMap `APP_ENV=local-kind`, Service por selector de rótulo e
probes que separam "receber tráfego" (readiness) de "processo vivo" (liveness).