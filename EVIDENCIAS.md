# Evidências de funcionamento

Este documento mapeia **cada requisito do projeto** para a implementação correspondente, o
comando que o valida e o arquivo de evidência gerado.

Todas as evidências em [`docs/evidencias/`](docs/evidencias/) foram produzidas por execução
real da stack, pelo script [`scripts/capture-evidence.sh`](scripts/capture-evidence.sh).
**Nenhuma captura ou saída foi fabricada.**

Reprodução completa:

```bash
docker compose down -v
docker compose up --build -d
./scripts/wait-stack.sh
./scripts/smoke-test.sh          # 28/28
./scripts/generate-traffic.sh 3
./scripts/verify-stack.sh        # 83/83
./scripts/capture-evidence.sh
```

---

## 1. Matriz de requisitos

| # | Requisito | Implementado | Validação executada | Evidência |
|---|---|---|---|---|
| 0 | **Multistage Build** (eliminatório) | **Sim** | `docker build` + inspeção da imagem final: sem `mvn`/`javac`/`jar`, 0 arquivos `.java`/`pom.xml`, 800MB → 398MB | [`18-multistage-inspecao.txt`](docs/evidencias/18-multistage-inspecao.txt) · [`Dockerfile`](Dockerfile) |
| 1 | **Pipeline CI/CD automatizada** | **Sim** | GitHub Actions: `mvn test`, `mvn package`, build multistage, validação da imagem, push no GHCR | [`.github/workflows/ci-cd.yml`](.github/workflows/ci-cd.yml) · seção 4 deste documento |
| 2 | **Publicação automática no registry** | **Sim** | Job `docker` publica em `ghcr.io` com `GITHUB_TOKEN` (sem PAT) em push na `main` | seção 4 deste documento |
| 3 | **Orquestração Docker Compose** | **Sim** | `docker compose up --build` sobe 7 serviços; todos `healthy`, sem passo manual | [`10-docker-compose-ps.txt`](docs/evidencias/10-docker-compose-ps.txt) |
| 4 | **Testes automatizados** | **Sim** | `mvn test` → 22 testes, 0 falhas; também no build da imagem e na pipeline | [`19-mvn-test.txt`](docs/evidencias/19-mvn-test.txt) |
| 5 | **Interface web** (não só API) | **Sim** | `GET /` entrega painel Thymeleaf com CSS/JS próprios, sem CDN | [`01-interface-aplicacao.png`](docs/evidencias/01-interface-aplicacao.png) |
| 6 | **Endpoints mínimos** | **Sim** | 200 / 400 / 500, 4 endpoints de log, gerador de tráfego, status e histórico | [`20-smoke-test.txt`](docs/evidencias/20-smoke-test.txt) · [`21-verify-stack.txt`](docs/evidencias/21-verify-stack.txt) |
| 7 | **Actuator + Micrometer** | **Sim** | `/actuator/health`, `/actuator/info` e `/actuator/prometheus` respondendo; tags `application`/`environment`/`version` em todas as séries | [`13-actuator-prometheus.txt`](docs/evidencias/13-actuator-prometheus.txt) |
| 8 | **Prometheus coletando (target UP)** | **Sim** | Target `spring-boot-app` = **UP** em `http://app:8080/actuator/prometheus` | [`02-prometheus-targets.png`](docs/evidencias/02-prometheus-targets.png) · [`12-prometheus-targets.json`](docs/evidencias/12-prometheus-targets.json) |
| 9 | **Métricas obrigatórias com dados reais** | **Sim** | 15 consultas PromQL executadas, todas com valor: heap, CPU, RPS, 2xx/4xx/5xx, latência, p95, threads, uptime | [`14-prometheus-metricas-consultadas.txt`](docs/evidencias/14-prometheus-metricas-consultadas.txt) · [`03-prometheus-grafico-rps.png`](docs/evidencias/03-prometheus-grafico-rps.png) |
| 10 | **Grafana com data source provisionado** | **Sim** | `GET /api/datasources/uid/prometheus` → provisionado, health `OK` | [`15-grafana-provisionamento.txt`](docs/evidencias/15-grafana-provisionamento.txt) |
| 11 | **Dashboard provisionado automaticamente** | **Sim** | `GET /api/dashboards/uid/spring-observability` → *Spring Boot — Application Observability*, 14 painéis + 3 linhas | [`05-grafana-dashboard.png`](docs/evidencias/05-grafana-dashboard.png) · [`15-grafana-provisionamento.txt`](docs/evidencias/15-grafana-provisionamento.txt) |
| 12 | **Dashboard com métricas reais** | **Sim** | `POST /api/ds/query` pelo próprio Grafana devolve séries com dados | [`15-grafana-provisionamento.txt`](docs/evidencias/15-grafana-provisionamento.txt) |
| 13 | **Graylog + MongoDB + OpenSearch** | **Sim** | Três serviços `healthy`, versões fixas e compatíveis (5.2.11 / 6.0.19 / 2.15.0) | [`10-docker-compose-ps.txt`](docs/evidencias/10-docker-compose-ps.txt) |
| 14 | **Input GELF criado automaticamente** | **Sim** | `graylog-init` exit 0; Input `RUNNING`; segunda execução detecta o existente (idempotente) | [`11-graylog-init-logs.txt`](docs/evidencias/11-graylog-init-logs.txt) · [`16-graylog-logs-por-nivel.txt`](docs/evidencias/16-graylog-logs-por-nivel.txt) |
| 15 | **Logs pesquisáveis no Graylog** | **Sim** | Busca por API: `application:spring-devops-observability` retorna centenas de mensagens | [`16-graylog-logs-por-nivel.txt`](docs/evidencias/16-graylog-logs-por-nivel.txt) |
| 16 | **DEBUG aparece** | **Sim** | `level_name:DEBUG` → `total_results > 0` | [`16-graylog-logs-por-nivel.txt`](docs/evidencias/16-graylog-logs-por-nivel.txt) |
| 17 | **INFO aparece** | **Sim** | `level_name:INFO` → `total_results > 0` | [`16-graylog-logs-por-nivel.txt`](docs/evidencias/16-graylog-logs-por-nivel.txt) |
| 18 | **WARN aparece** | **Sim** | `level_name:WARN` → `total_results > 0` | [`16-graylog-logs-por-nivel.txt`](docs/evidencias/16-graylog-logs-por-nivel.txt) |
| 19 | **ERROR aparece** | **Sim** | `level_name:ERROR` → `total_results > 0`, com stack trace em `full_message` | [`17-graylog-mensagem-exemplo.txt`](docs/evidencias/17-graylog-mensagem-exemplo.txt) |
| 20 | **Logs com campos úteis** | **Sim** | `timestamp`, `level_name`, `logger_name`, `message`, `application`, `environment`, `version`, `correlation_id`, `thread_name`, `source` | [`17-graylog-mensagem-exemplo.txt`](docs/evidencias/17-graylog-mensagem-exemplo.txt) |
| 21 | **Healthchecks e `depends_on`** | **Sim** | 6 serviços com healthcheck; ordem mongodb/opensearch → graylog → app → prometheus → grafana | [`docker-compose.yml`](docker-compose.yml) · [`10-docker-compose-ps.txt`](docs/evidencias/10-docker-compose-ps.txt) |
| 22 | **Networks e named volumes** | **Sim** | Rede `observability-net`; 6 volumes nomeados | [`docker-compose.yml`](docker-compose.yml) |
| 23 | **Sem tags `latest`** | **Sim** | Todas as imagens com versão fixa; a pipeline **falha** se aparecer `:latest` | [`docker-compose.yml`](docker-compose.yml) · job `compose-lint` |
| 24 | **Scripts de validação com exit code** | **Sim** | `wait-stack`, `generate-traffic`, `smoke-test`, `verify-stack`, `capture-evidence` | [`20-smoke-test.txt`](docs/evidencias/20-smoke-test.txt) · [`21-verify-stack.txt`](docs/evidencias/21-verify-stack.txt) |
| 25 | **Sem segredos versionados** | **Sim** | `.env` no `.gitignore`; só `.env.example` versionado; nenhum PAT/token no repositório; Trivy com scanner de segredos na pipeline | [`.gitignore`](.gitignore) · [`.env.example`](.env.example) |
| 26 | **README completo** | **Sim** | Visão geral, arquitetura Mermaid, execução, URLs, credenciais, validação de cada componente, pipeline, multistage, estrutura, decisões técnicas, troubleshooting | [`README.md`](README.md) |
| 27 | **DevSecOps (extras)** | **Sim** | Usuário não-root na imagem, superfície mínima do Actuator (verificada por teste), Trivy (dependências, imagem e segredos) | [`.github/workflows/ci-cd.yml`](.github/workflows/ci-cd.yml) |

---

## 2. Resultado da auditoria automatizada

`./scripts/verify-stack.sh` → **83/83 verificações passaram** (exit code 0).

| Seção | Verificações | Resultado |
|---|---|---|
| 1. Arquivos obrigatórios do repositório | 21 | todas OK |
| 2. Dockerfile multistage (eliminatório) | 6 | todas OK |
| 2b. Inspeção da imagem construída | 4 | todas OK |
| 3. Containers da stack | 7 | todas OK |
| 4. Endpoints da aplicação | 14 | todas OK |
| 5. Prometheus: target UP e métricas | 15 | todas OK |
| 6. Grafana: provisionamento e dados | 5 | todas OK |
| 7. Graylog: Input GELF e logs por nível | 10 | todas OK |
| 8. Suíte de testes (`mvn test`) | 1 | 22 testes, 0 falhas |

`./scripts/smoke-test.sh` → **28/28 verificações passaram** (exit code 0).

Saídas completas: [`21-verify-stack.txt`](docs/evidencias/21-verify-stack.txt) e
[`20-smoke-test.txt`](docs/evidencias/20-smoke-test.txt).

---

## 3. Evidências do requisito eliminatório (multistage)

Medido na imagem realmente construída
([`18-multistage-inspecao.txt`](docs/evidencias/18-multistage-inspecao.txt)):

| Verificação | Resultado |
|---|---|
| Estágios `FROM` declarados | 2 (`build` e `runtime`) |
| Estágio 1 | `maven:3.9.9-eclipse-temurin-21` (JDK 21 + Maven) |
| Estágio 1 executa os testes | `mvn -B -ntp clean package` |
| Estágio 2 | `eclipse-temurin:21.0.12_8-jre-alpine` (apenas JRE) |
| Copiado do estágio 1 para o final | apenas `/app/app.jar` (29 MB) |
| Tamanho da imagem de build | 800 MB |
| **Tamanho da imagem final** | **398 MB** |
| `mvn` na imagem final | **AUSENTE** |
| `javac` na imagem final | **AUSENTE** |
| `jar` na imagem final | **AUSENTE** |
| Arquivos `.java` na imagem final | **0** |
| Arquivos `pom.xml` na imagem final | **0** |
| Diretório `/root/.m2` na imagem final | **ausente** |
| Usuário do processo | `uid=100(spring) gid=101(spring)` — não-root |

As mesmas verificações rodam automaticamente no `verify-stack.sh` e no job `docker` da
pipeline, que **falha** se qualquer ferramenta de build vazar para o runtime.

---

## 4. Pipeline CI/CD e GHCR

Preenchido após o push para o GitHub — veja a seção
[Status da pipeline](#status-da-pipeline) no final deste documento.

---

## 5. Índice dos arquivos de evidência

### Capturas de tela

| Arquivo | Conteúdo |
|---|---|
| `01-interface-aplicacao.png` | Painel da aplicação em execução: cards de estado com dados reais, seções de tráfego e logs, indicador do GELF ativo e histórico com status 200/400/500 |
| `02-prometheus-targets.png` | *Status → Targets* do Prometheus com `spring-boot-app (1/1 up)` em estado **UP** |
| `03-prometheus-grafico-rps.png` | Gráfico de requisições por status HTTP ao longo do tempo, no próprio Prometheus |
| `04-graylog-login.png` | Interface web do Graylog respondendo em `localhost:9000` |
| `05-grafana-dashboard.png` | Dashboard *Spring Boot — Application Observability* com os painéis preenchidos |

### Saídas de comandos e respostas de API

| Arquivo | Conteúdo |
|---|---|
| `10-docker-compose-ps.txt` | Estado dos 7 serviços da stack (6 `healthy` + `graylog-init` em `Exited (0)`) |
| `11-graylog-init-logs.txt` | Log do provisionamento do Input GELF, incluindo a execução idempotente |
| `12-prometheus-targets.json` | Resposta crua de `/api/v1/targets` com `"health":"up"` |
| `13-actuator-prometheus.txt` | Trecho real de `/actuator/prometheus` com as métricas exigidas e as tags comuns |
| `14-prometheus-metricas-consultadas.txt` | 15 consultas PromQL com os valores retornados |
| `15-grafana-provisionamento.txt` | Data source, health, título/painéis do dashboard e consulta devolvendo dados |
| `16-graylog-logs-por-nivel.txt` | Input GELF, estado `RUNNING` e contagem de mensagens por nível e por campo |
| `17-graylog-mensagem-exemplo.txt` | Campos de uma mensagem ERROR realmente indexada |
| `18-multistage-inspecao.txt` | Prova completa da separação dos estágios |
| `19-mvn-test.txt` | Saída de `mvn test` (22 testes, `BUILD SUCCESS`) |
| `20-smoke-test.txt` | Saída do smoke test (28/28) |
| `21-verify-stack.txt` | Saída da auditoria completa (83/83) |

---

## 6. Checklist do critério final de "pronto"

| Item | Status |
|---|---|
| Aplicação Spring funciona | ✅ |
| Interface funciona | ✅ |
| Endpoints funcionam | ✅ |
| Testes passam (22/22) | ✅ |
| Dockerfile é multistage | ✅ |
| Imagem Docker compila | ✅ |
| Compose sobe a stack | ✅ |
| App fica `healthy` | ✅ |
| Prometheus coleta a aplicação | ✅ |
| Target está `UP` | ✅ |
| Grafana possui dashboard provisionado | ✅ |
| Dashboard apresenta métricas reais | ✅ |
| Graylog inicia | ✅ |
| Input é configurado automaticamente | ✅ |
| Logs chegam ao Graylog | ✅ |
| DEBUG aparece | ✅ |
| INFO aparece | ✅ |
| WARN aparece | ✅ |
| ERROR aparece | ✅ |
| README está completo | ✅ |
| Evidências estão organizadas | ✅ |
| Git está limpo | ✅ |
| Repositório GitHub foi criado | ✅ |
| Código foi enviado | ✅ |
| GitHub Actions foi executado | ver [Status da pipeline](#status-da-pipeline) |
| Pipeline ficou verde | ver [Status da pipeline](#status-da-pipeline) |
| Imagem publicada no GHCR | ver [Status da pipeline](#status-da-pipeline) |

---

## Status da pipeline

*Esta seção é preenchida com o resultado real da execução do GitHub Actions após o push.*
