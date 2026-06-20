# Revisão do Helm Chart `base-template-chart`

## 1. Bugs / Problemas nos Templates Atuais

### 1.1 deployment.yaml — `serviceAccountName` hardcoded
```yaml
serviceAccountName: {{ include "base-template-chart.fullname" . }}
```
**Problema:** Ignora o helper `base-template-chart.serviceAccountName`, que respeita `.Values.serviceAccount.create` e `.Values.serviceAccount.name`.  
**Correção:**
```yaml
serviceAccountName: {{ include "base-template-chart.serviceAccountName" . }}
```

### 1.2 deployment.yaml — Bloco `envFrom` vazio renderizado
O template sempre imprime `envFrom:` mesmo quando não há ConfigMaps, Secrets nem ExternalSecrets configurados, gerando YAML inválido ou vazio.

**Correção:** Envolver todo o bloco `envFrom` em uma condição:
```yaml
{{- if or .Values.envFromConfigMaps .Values.externalSecrets.secrets .Values.envFromSecrets }}
envFrom:
  ...
{{- end }}
```

### 1.3 configmap.yaml — Contexto `.` perdido no `range`
Dentro do `range $name, $cm := .Values.configMaps`, o ponto (`.`) deixa de ser o root do release e vira o valor do ConfigMap (`$cm`).
```yaml
{{- include "base-template-chart.labels" . | nindent 4 }}
```
Isso pode quebrar o helper porque `.Chart`, `.Release`, etc. não estarão disponíveis.

**Correção:** Salvar o root antes do range:
```yaml
{{- $root := . -}}
{{- range $name, $cm := .Values.configMaps }}
...
  labels:
    {{- include "base-template-chart.labels" $root | nindent 4 }}
```

### 1.4 external-secret.yaml — Versão da API
```yaml
apiVersion: external-secrets.io/v1
```
O External Secrets Operator normalmente usa `external-secrets.io/v1beta1`. A `v1` ainda não é a API padrão na maioria das instalações.  
**Sugestão:** Mudar para `external-secrets.io/v1beta1` (ou parametrizar via values).

### 1.5 pvc.yaml — Ausência de labels e annotations
O PVC não herda labels padrão nem suporta annotations, dificultando rastreamento e automações (ex.: Velero, backup).

**Correção sugerida:**
```yaml
metadata:
  name: ...
  labels:
    {{- include "base-template-chart.labels" . | nindent 4 }}
  {{- with .Values.persistence.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
```

### 1.6 service.yaml — Sem suporte a múltiplas portas
O Service só expõe uma porta HTTP. Muitas aplicações precisam de porta de métricas (Prometheus), gRPC, etc.

### 1.7 httproute.yaml — `parentRefs` obrigatório pode sumir
O bloco `parentRefs` está dentro de `{{- with }}`, o que o remove se não houver valores. Para `HTTPRoute`, `parentRefs` é obrigatório no spec.

---

## 2. Melhorias nos Templates Existentes

### 2.1 Adicionar `topologySpreadConstraints`
Atualmente só há `affinity`, `tolerations` e `nodeSelector`. `topologySpreadConstraints` é essencial para HA real em múltiplas zonas.

### 2.2 HPA com `behavior`
O HPA só configura CPU/memory. Adicionar `behavior` permite controlar velocidade de scale-up/down (ex.: não derrubar tudo de uma vez).

### 2.3 Deployment com `startupProbe`
Hoje há liveness e readiness, mas não `startupProbe`. Sem ele, aplicações de inicialização lenta ficam em loop de liveness.

### 2.4 ServiceAccount — `automountServiceAccountToken` parametrizável
Está hardcoded como `true`. Deveria ser controlável via values (boa prática de segurança).

### 2.5 Adicionar `revisionHistoryLimit` no Deployment
Controlar quantas ReplicaSets antigas manter (default do k8s é 10, muitas vezes excessivo).

### 2.6 Ingress — Suporte a múltiplos paths com backend separado
Hoje o backend aponta sempre para o mesmo service. Para APIs com múltiplos serviços sob o mesmo host, seria útil.

### 2.7 Padronizar uso de `nindent` vs `indent`
Alguns templates usam `indent` onde deveria ser `nindent`, ou vice-versa, causando YAML mal formatado. Revisar todos os `toYaml` com consistência.

---

## 3. Templates Sugeridos para Adicionar

| Template | Descrição | Motivação |
|----------|-----------|-----------|
| `poddisruptionbudget.yaml` | PDB com `minAvailable` ou `maxUnavailable` | Evita indisponibilidade total durante upgrades ou drains de nó |
| `networkpolicy.yaml` | NetworkPolicy para restringir tráfego de/para o pod | Segurança — princípio do menor privilégio de rede |
| `serviceMonitor.yaml` | ServiceMonitor do Prometheus Operator | Métricas — padrão de facto em Kubernetes |
| `podMonitor.yaml` | PodMonitor (alternativa ao ServiceMonitor) | Quando as métricas não passam pelo Service |
| `cronjob.yaml` | CronJob para tarefas agendadas | Muitos apps precisam de jobs rotineiros (cleanup, reports) |
| `job.yaml` | Job para execução one-off (migrations, seeds) | Executar antes/depois do deploy via Helm hooks |
| `secret.yaml` | Secret nativo do Kubernetes (opcional) | Fallback para quem não usa External Secrets |
| `rbac.yaml` | Roles, RoleBindings, ClusterRoles | Quando a aplicação precisa acessar a API do k8s |
| `vpa.yaml` | VerticalPodAutoscaler | Recomendações/ajuste automático de requests/limits |
| `certificate.yaml` | Certificate do cert-manager | TLS automatizado (complemento ao Ingress) |
| `rollout.yaml` | Argo Rollouts | Canary/Blue-green deployments avançados |
| `keda.yaml` / `scaledobject.yaml` | ScaledObject do KEDA | Scaling baseado em eventos (fila, Kafka, etc.) |

---

## 4. Melhorias Estruturais e Boas Práticas

### 4.1 `values.yaml` — `env` como lista, não mapa
```yaml
# Atualmente:
env: {}
# Mas os comentários mostram lista. Melhor:
env: []
```

### 4.2 Criar seção `global`
```yaml
global:
  imageRegistry: ""
  imagePullSecrets: []
  storageClass: ""
```
Facilita charts-filho (subcharts) e overrides em ambientes corporativos.

### 4.3 Separar `values.yaml` em arquivos de ambiente
Criar `values-production.yaml`, `values-staging.yaml` com overrides documentados.

### 4.4 Adicionar `schema	values.schema.json`
JSON Schema para validação automática de valores pelo Helm (`helm lint`, IDEs).

### 4.5 Renomear helpers para não hardcodar o nome do chart
Hoje o helper segue `<chart-name>.fullname`. Se o chart for renomeado (fork), todos os templates quebram. Usar um prefixo curto ou `.Chart.Name` no define — mas aí fica hardcoded no nome da função. Padrão do `helm create` já faz isso e é aceitável.

### 4.6 Adicionar testes de Helm (`helm test`)
Criar `templates/tests/test-connection.yaml` com um pod simples que valida se a aplicação responde.

### 4.7 `NOTES.txt`
Criar `templates/NOTES.txt` para exibir instruções pós-install (`helm install`).

### 4.8 `checksum/config` para restart automático do Deployment
Adicionar annotation `checksum/configmap` e `checksum/secret` no `template.metadata.annotations` do Deployment para forçar rollout quando configs mudam.

---

## 5. Checklist de Segurança Recomendada

- [ ] Definir `securityContext` e `podSecurityContext` por padrão (não deixar vazio)
- [ ] `readOnlyRootFilesystem: true`
- [ ] `runAsNonRoot: true` + `runAsUser` / `runAsGroup`
- [ ] `allowPrivilegeEscalation: false`
- [ ] `seccompProfile: { type: RuntimeDefault }`
- [ ] `capabilities: { drop: [ALL] }`
- [ ] Containers sem privilégios (`privileged: false`)
- [ ] NetworkPolicy default-deny + regras explícitas de egress/ingress

---

## 6. Ordem de Prioridade para Implementar

1. **Corrigir bugs** (seção 1) — rápidos e de baixo risco
2. **Adicionar `PodDisruptionBudget`** — essencial para produção
3. **Adicionar `startupProbe`** — melhora estabilidade
4. **Adicionar `NetworkPolicy`** — segurança
5. **Adicionar `ServiceMonitor`** — observabilidade
6. **Criar `NOTES.txt` e testes de Helm** — UX
7. **Adicionar `values.schema.json`** — qualidade/dev-experience
8. **Templates avançados** (KEDA, VPA, Argo Rollouts) — conforme necessidade
