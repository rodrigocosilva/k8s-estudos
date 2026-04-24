# CLAUDE.md

## Contexto do Projeto

Este projeto é um laboratório local de observabilidade rodando em um Mac Mini M4 com 16GB de RAM.
O ambiente roda em Kubernetes local usando OrbStack.
O objetivo é estudar, testar e evoluir uma stack de observabilidade moderna, combinando monitoramento, métricas, tracing e visualização.

---

## Stack Atual

Componentes implantados no Kubernetes (namespace `monitoring-lab`):

| Componente | Versão | Função |
|---|---|---|
| Grafana | 11.1.4 | Dashboards e visualização |
| Jaeger | 1.58 | Tracing distribuído |
| OpenTelemetry Collector | 0.100.0 | Ingestão e roteamento de telemetria |
| PostgreSQL | 16.4-alpine | Backend do Zabbix |
| Prometheus | v2.54.1 | Coleta e consulta de métricas |
| Zabbix Server | 7.0.13 | Monitoramento tradicional |
| Zabbix Web | 7.0.13 | Interface web do Zabbix |
| Zabbix Agent 2 | alpine-7.0 | Agente de monitoramento K8s-nativo |
| Demo App (Flask) | Python 3.11 | App de exemplo com OpenTelemetry |

---

## Plataforma

- Host: Mac Mini M4
- Memória: 16GB
- Runtime/Kubernetes: OrbStack
- Arquitetura: ARM64 / Apple Silicon
- Ambiente: laboratório local

---

## Estrutura de Diretórios

```
base/         → manifests principais (Zabbix, Prometheus, Grafana, Jaeger, PostgreSQL)
apps/         → aplicações instrumentadas (ex: demo-frontend Flask + OpenTelemetry)
observability/ → configurações do OpenTelemetry Collector
legacy/       → manifests antigos, não aplicar
```

Ao criar novos manifests:
- Componentes de infraestrutura de observabilidade → `base/`
- Aplicações de exemplo ou instrumentadas → `apps/`
- Configs de coleta de telemetria → `observability/`

---

## Como Aplicar Manifests

Este projeto usa **YAML puro com kubectl**. Não usa Helm, Kustomize nem Docker Compose.

```bash
# Aplicar um manifest
kubectl apply -f base/lab-observabilidade-v2.yaml

# Verificar recursos no namespace
kubectl get all -n monitoring-lab
```

A IA não deve sugerir Helm charts nem Kustomize a menos que o usuário solicite explicitamente.

---

## Diretrizes Gerais

A IA deve considerar que este ambiente é um laboratório local, mas deve sugerir práticas próximas de produção sempre que possível.
Prioridades:

1. Simplicidade operacional
2. Clareza arquitetural
3. Baixo consumo de recursos
4. Compatibilidade com ARM64
5. Facilidade de troubleshooting
6. Evolução gradual para práticas de produção

---

## Regras para Kubernetes

- Usar manifests YAML claros e bem indentados.
- Separar recursos por componente sempre que fizer sentido.
- Preferir namespaces dedicados.
- Evitar configurações excessivamente complexas para um lab local.
- Sempre considerar `resources.requests` e `resources.limits`.
- **LoadBalancer é aceito neste ambiente**: OrbStack suporta LoadBalancer nativamente no Mac. Preferir LoadBalancer para UIs (Grafana, Prometheus, Jaeger, Zabbix Web) e ClusterIP para serviços internos.
- Validar comandos com `kubectl`.
- Sempre especificar o namespace nos comandos.

---

## Namespace

O namespace principal do projeto é:
```
monitoring-lab
```

---

## Componentes

### Grafana

Usado para visualização de métricas e dashboards.

Integrações prioritárias:
- Prometheus (datasource principal)
- Zabbix (via plugin alexanderzobnin-zabbix-app 6.1.1)
- Jaeger (quando aplicável)

Boas práticas:
- Dashboards claros e reutilizáveis
- Evitar excesso de complexidade visual

---

### Prometheus

Usado para coleta e consulta de métricas.

Boas práticas:
- Validar targets em `/targets`
- Validar endpoint `/metrics`
- Evitar alta cardinalidade
- Manter configuração simples
- Usar labels com critério

---

### Jaeger

Usado para tracing distribuído.

- Modo all-in-one (adequado para lab)
- Integrado com OpenTelemetry Collector
- Focado em análise de traces da demo app e serviços futuros

---

### OpenTelemetry Collector

Componente central de ingestão e roteamento de telemetria.

Pipeline atual:
- **Receivers**: OTLP gRPC (4317) + HTTP (4318)
- **Processors**: batch
- **Exporters**: Jaeger (traces), Prometheus (métricas), debug

Ao sugerir mudanças:
- Explicar o pipeline antes (receivers → processors → exporters)
- Evitar pipelines complexos sem necessidade
- Garantir compatibilidade com Jaeger e Prometheus

---

### PostgreSQL

Banco utilizado principalmente pelo Zabbix.

**Atenção — storage atual**: O lab usa `emptyDir` para o PostgreSQL. Isso significa que os dados do Zabbix são **perdidos ao reiniciar o pod**. Isso é intencional para simplicidade do lab.

Se o usuário quiser persistência real, sugerir migração para PVC com `hostPath` ou `local` StorageClass.

Cuidados:
- Avisar explicitamente antes de qualquer ação que cause perda de dados
- Evitar queries pesadas desnecessárias

---

### Zabbix Server

Monitoramento tradicional (SNMP, agentes, etc).

Boas práticas:
- Preferir Zabbix Agent 2 quando aplicável
- Evitar polling agressivo
- Validar impacto no banco
- Usar `zabbix_get` para testes

---

### Zabbix Web

Interface web do Zabbix (nginx).

Cuidados:
- Validar variáveis de ambiente de conexão com PostgreSQL
- Checar logs antes de mudanças
- Confirmar que o Zabbix Server está healthy antes de troubleshoot o Web

---

### Zabbix Agent 2 (K8s-nativo)

Agente com monitoramento específico de Kubernetes.

Características:
- Usa ServiceAccount + Role + RoleBinding para acessar a API do Kubernetes
- Injeta o binário `kubectl` via init container (bitnami/kubectl)
- Monitora: status dos pods, restarts, readiness
- Usa `UserParameter` customizados no ConfigMap

Ao modificar:
- Manter o RBAC mínimo necessário
- Testar user parameters com `zabbix_agent2 -t <key>`

---

### Demo App (Flask + OpenTelemetry)

Aplicação de exemplo para validar o pipeline de observabilidade.

- Python 3.11, Flask
- Exporta **traces** para o OTEL Collector via HTTP (OTLP)
- Exporta **métricas** customizadas: `demo_requests_total`, `demo_work_duration_ms`
- Rotas: `/` (hello), `/health`, `/work` (simula latência)
- Disponível em NodePort 30080

---

## Credenciais e Secrets

As credenciais atuais estão em texto claro nos manifests (padrão de lab). Para qualquer nova credencial:
- Em lab: pode usar `env.value` diretamente nos manifests, mas documentar
- Para evolução: migrar para `kind: Secret` com `valueFrom.secretKeyRef`
- **Nunca commitar senhas reais** no repositório

---

## Troubleshooting

Ordem obrigatória:

1. Verificar pods
```
kubectl get pods -n monitoring-lab
```

2. Verificar services
```
kubectl get svc -n monitoring-lab
```

3. Verificar eventos
```
kubectl get events -n monitoring-lab --sort-by=.lastTimestamp
```

4. Ver logs
```
kubectl logs -n monitoring-lab <pod>
```

5. Descrever recurso
```
kubectl describe pod -n monitoring-lab <pod>
```

6. Testar conectividade interna
```
kubectl exec -n monitoring-lab -it <pod> -- sh
```

---

## Comportamento Esperado da IA

A IA deve:

- Explicar antes de implementar
- Priorizar soluções simples
- Evitar overengineering
- Sugerir melhorias quando identificar riscos
- Avisar quando houver risco de perda de dados (especialmente com emptyDir no PostgreSQL)
- Considerar limitações de hardware (16GB RAM)
- Considerar arquitetura ARM64
- Gerar YAML válido e bem indentado

---

## Restrições

A IA NÃO deve:

- Assumir arquitetura x86_64
- Assumir uso de Docker Compose ou Helm (a menos que solicitado)
- Sugerir tags `latest` sem justificativa — preferir tags fixas e específicas
- Ignorar namespace
- Remover volumes persistentes sem aviso explícito
- Propor HA complexa desnecessária
- Aplicar manifests do diretório `legacy/`

---

## Evolução Esperada

Já implementado:
- ✅ Instrumentação de aplicações (demo Flask + OpenTelemetry)
- ✅ Pipelines OpenTelemetry (traces → Jaeger, métricas → Prometheus)
- ✅ Integração Zabbix + Grafana
- ✅ Integração Prometheus + Grafana
- ✅ Monitoramento K8s-nativo com Zabbix Agent 2

Próximos passos possíveis:
- Observabilidade de logs (Loki ou similar)
- Dashboards operacionais com SLO/SLI
- Alertas no Prometheus (AlertManager)
- Mais aplicações instrumentadas
- AIOps

---

## Filosofia do Projeto

Este ambiente existe para aprendizado profundo e prática real.
A IA deve evitar respostas genéricas e sempre adaptar soluções ao contexto deste laboratório.
