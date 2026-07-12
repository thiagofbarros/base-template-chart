# base-template-chart

Helm chart **base/template** para aplicações no Kubernetes. Serve como ponto de
partida padronizado para expor um workload HTTP (imagem de container única) com
os recursos mais comuns de plataforma já embutidos: `Deployment`, `Service`,
`ServiceAccount`, `ConfigMaps`, variáveis de ambiente (`env`/`envFrom`),
exposição por `Ingress` **ou** Gateway API (`HTTPRoute`), autoscaling (`HPA`),
armazenamento persistente (`PVC`), integração com **ExternalSecrets** e
_hardening_ de `securityContext` habilitado por padrão.

O chart é publicado como artefato **OCI** (Amazon ECR), assinado com
**cosign keyless** e acompanhado de um **SBOM (SPDX)** anexado como attestation.

| Item | Valor |
| --- | --- |
| Nome do chart | `base-template-chart` |
| Versão do chart | `1.2.2` |
| `appVersion` | `1.0.0` |
| Tipo | `application` (Helm `apiVersion: v2`) |
| Versionamento | SemVer (obrigatório incrementar a cada mudança) |

> Nota sobre "stateful": o único workload gerado é um `Deployment`. O suporte a
> estado é oferecido via `PersistentVolumeClaim` autônomo montado no Deployment
> (`persistence.enabled`). Não há template de `StatefulSet` — o
> `values.schema.json` inclusive restringe `application.type` a `deployment`.

---

## Sumário

- [Recursos suportados](#recursos-suportados)
- [Pré-requisitos](#pré-requisitos)
- [Instalação](#instalação)
- [Configuração (values)](#configuração-values)
- [Exemplos por cenário](#exemplos-por-cenário)
- [Desenvolvimento e CI](#desenvolvimento-e-ci)
- [Release e publicação](#release-e-publicação)
- [Verificando assinatura e SBOM](#verificando-assinatura-e-sbom)

---

## Recursos suportados

| Recurso | Template | Condição de renderização |
| --- | --- | --- |
| `Deployment` | `templates/deployment.yaml` | `application.type == "deployment"` (default) |
| `Service` (ClusterIP) | `templates/service.yaml` | sempre |
| `ServiceAccount` | `templates/serviceaccount.yaml` | `serviceAccount.create == true` (default) |
| `ConfigMap` (0..N) | `templates/configmap.yaml` | uma entrada por chave em `configMaps` |
| `Ingress` | `templates/ingress.yaml` | `ingress.enabled == true` |
| `HTTPRoute` (Gateway API) | `templates/httproute.yaml` | `httpRoute.enabled == true` |
| `HorizontalPodAutoscaler` | `templates/hpa.yaml` | `deployment.hpa.enabled == true` |
| `PersistentVolumeClaim` | `templates/pvc.yaml` | `persistence.enabled == true` |
| `ExternalSecret` (0..N) | `templates/external-secret.yaml` | `externalSecrets` definido e com `secrets` |
| Helpers de nome/labels | `templates/_helpers.tpl` | — |

Detalhes de comportamento relevantes:

- **Porta única acoplada**: o `containerPort` do Deployment e a porta do Service
  usam o mesmo valor `service.port` (porta nomeada `http`). O `targetPort` do
  Service aponta para a porta nomeada `http`.
- **`securityContext` restritivo por padrão** (nível pod e container):
  `runAsNonRoot: true`, `runAsUser: 1000`, `readOnlyRootFilesystem: true`,
  `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]` e
  `seccompProfile: RuntimeDefault`.
- **HPA e réplicas**: quando `deployment.hpa.enabled == true`, o campo
  `replicas` é omitido do Deployment (deixando o HPA no controle).
- **`envFrom` agregado**: o Deployment monta em `envFrom` (nesta ordem) os
  ConfigMaps marcados com `envFrom: true` em `configMaps`, os de
  `envFromConfigMaps`, os Secrets criados por `externalSecrets.secrets` e os de
  `envFromSecrets`.
- **PVC**: o nome do PVC é sempre `<fullname>-data`; `persistence.name` afeta
  apenas o nome do volume/mount interno.

---

## Pré-requisitos

| Componente | Versão / requisito |
| --- | --- |
| Helm | **v3.8+** (suporte a registries OCI). CI/release usam Helm v4.1.0 |
| Kubernetes | 1.25+ recomendado (CI valida contra schemas do **1.31.0**) |
| Gateway API CRDs | Necessárias **apenas** se `httpRoute.enabled` (`gateway.networking.k8s.io/v1`) |
| ExternalSecrets Operator | Necessário **apenas** se usar `externalSecrets` (`external-secrets.io/v1`) |
| Ingress Controller | Necessário **apenas** se `ingress.enabled` |
| metrics-server | Necessário **apenas** se `deployment.hpa.enabled` |

As CRDs de Gateway API e ExternalSecrets **não** são empacotadas pelo chart;
devem existir previamente no cluster (ex.: instaladas via GitOps/Argo CD).

---

## Instalação

### A partir do registry OCI (Amazon ECR)

```bash
# <registry> = ex.: 123456789012.dkr.ecr.us-east-1.amazonaws.com
# <prefix>   = valor de ECR_REPOSITORY (opcional); omita se não houver prefixo
helm install minha-app \
  oci://<registry>/<prefix>/base-template-chart \
  --version 1.2.2 \
  --namespace minha-app --create-namespace \
  -f meus-values.yaml
```

Se o ECR for privado, autentique o Helm antes:

```bash
aws ecr get-login-password --region <region> \
  | helm registry login -u AWS --password-stdin <registry>
```

### A partir do source local

```bash
git clone https://github.com/thiagofbarros/base-template-chart.git
cd base-template-chart

helm install minha-app . -f meus-values.yaml
# ou empacotando
helm package .
helm install minha-app base-template-chart-1.2.2.tgz
```

---

## Configuração (values)

Os defaults abaixo refletem exatamente o `values.yaml`. Tipos e campos
obrigatórios são validados por `values.schema.json` no momento do
`install`/`template`.

### Aplicação e imagem

| Chave | Default | Descrição |
| --- | --- | --- |
| `application.name` | `app-name` | Nome lógico da aplicação; usado como `fullname` e nome do container. |
| `application.environment` | `dev` | Rótulo informativo de ambiente. |
| `application.type` | `deployment` | Tipo de workload. Schema aceita apenas `deployment`. |
| `nameOverride` | `""` | Sobrescreve o nome do chart em nomes de recursos/labels (default: nome do chart). `application.name`, quando definido, tem precedência para o `fullname`. |
| `image.repository` | `hello-world` | Repositório da imagem. |
| `image.pullPolicy` | `IfNotPresent` | `Always` \| `IfNotPresent` \| `Never`. |
| `image.tag` | `latest` | Tag da imagem (fallback: `.Chart.AppVersion`). |
| `imagePullSecrets` | `[]` | Lista de secrets de pull. |

### Deployment / rollout / autoscaling

| Chave | Default | Descrição |
| --- | --- | --- |
| `deployment.replicaCount` | `1` | Réplicas (ignorado quando HPA ativo). |
| `deployment.strategy.type` | `RollingUpdate` | Estratégia de rollout. |
| `deployment.strategy.rollingUpdate.maxSurge` | `25%` | — |
| `deployment.strategy.rollingUpdate.maxUnavailable` | `25%` | — |
| `deployment.hpa.enabled` | `false` | Habilita o HorizontalPodAutoscaler. |
| `deployment.hpa.minReplicas` | `1` | Mínimo de réplicas do HPA. |
| `deployment.hpa.maxReplicas` | `10` | Máximo de réplicas do HPA. |
| `deployment.hpa.targetCPUUtilizationPercentage` | `80` | Alvo de CPU (%); métrica omitida se nulo. |
| `deployment.hpa.targetMemoryUtilizationPercentage` | _(comentado)_ | Alvo de memória (%); métrica só é criada se definido. |

### Configuração / variáveis de ambiente

| Chave | Default | Descrição |
| --- | --- | --- |
| `env` | `[]` | Lista de env vars no formato core do Kubernetes (`name`/`value`/`valueFrom`). |
| `envFromConfigMaps` | `[]` | Lista de `{ name: <configmap> }` injetados via `envFrom`. |
| `envFromSecrets` | `[]` | Lista de `{ name: <secret> }` injetados via `envFrom`. |
| `configMaps` | `{}` | Mapa `nome -> { data, binaryData, labels, annotations, immutable, envFrom }`. Cria um ConfigMap por chave. |
| `configMaps.<n>.envFrom` | — | Se `true`, o ConfigMap é montado automaticamente no `envFrom` do workload. |
| `externalSecrets` | _(nil)_ | Bloco de ExternalSecrets (ver seção dedicada). |

### Persistência e volumes

| Chave | Default | Descrição |
| --- | --- | --- |
| `persistence.enabled` | `false` | Cria um PVC `<fullname>-data` e o monta no container. |
| `persistence.name` | `data` | Nome do volume/mount interno. |
| `persistence.accessMode` | `ReadWriteOnce` | `ReadWriteOnce` \| `ReadOnlyMany` \| `ReadWriteMany` \| `ReadWriteOncePod`. |
| `persistence.size` | _(obrigatório se enabled)_ | Tamanho solicitado (ex.: `5Gi`). Falha o render se ausente. |
| `persistence.mountPath` | _(vazio)_ | Caminho de montagem no container. |
| `persistence.storageClass` | _(vazio)_ | StorageClass; omitido se vazio. |
| `volumes` | `[]` | Volumes adicionais (suporta `configMap`/`secret`). |
| `volumeMounts` | `[]` | Mounts adicionais (`name`/`mountPath`/`subPath`). |

### Segurança, agendamento e ServiceAccount

| Chave | Default | Descrição |
| --- | --- | --- |
| `serviceAccount.create` | `true` | Cria um ServiceAccount dedicado. |
| `serviceAccount.name` | `""` | Nome do ServiceAccount. Vazio: usa o `fullname` (quando `create: true`) ou `default` (quando `create: false`). |
| `serviceAccount.annotations` | `{}` | Annotations (ex.: IRSA/role ARN). |
| `serviceAccount.automountServiceAccountToken` | `false` | Só habilite se o workload acessa a API do Kubernetes. |
| `podSecurityContext.runAsNonRoot` | `true` | — |
| `podSecurityContext.seccompProfile.type` | `RuntimeDefault` | — |
| `securityContext.allowPrivilegeEscalation` | `false` | — |
| `securityContext.capabilities.drop` | `[ALL]` | — |
| `securityContext.readOnlyRootFilesystem` | `true` | — |
| `securityContext.runAsNonRoot` | `true` | — |
| `securityContext.runAsUser` | `1000` | — |
| `securityContext.seccompProfile.type` | `RuntimeDefault` | — |
| `resources` | `{}` | Requests/limits (recomendado definir em produção). |
| `livenessProbe` | `{}` | Probe de liveness (só renderiza se definido). |
| `readinessProbe` | `{}` | Probe de readiness (só renderiza se definido). |
| `nodeSelector` | `{}` | — |
| `tolerations` | `[]` | — |
| `affinity` | `{}` | — |
| `podAnnotations` | `{}` | — |
| `podLabels` | `{}` | — |

### Exposição de rede

| Chave | Default | Descrição |
| --- | --- | --- |
| `service.type` | `ClusterIP` | `ClusterIP` \| `NodePort` \| `LoadBalancer` \| `ExternalName`. |
| `service.port` | `80` | Porta do Service **e** `containerPort` do pod. |
| `ingress.enabled` | `false` | Cria um Ingress (`networking.k8s.io/v1`). |
| `ingress.className` | _(vazio)_ | `ingressClassName`. |
| `ingress.annotations` | `{}` | — |
| `ingress.hosts` | _(vazio)_ | Lista de `{ host, paths: [{ path, pathType }] }`. |
| `ingress.tls` | _(vazio)_ | Lista de `{ secretName, hosts: [] }`. |
| `httpRoute.enabled` | `false` | Cria um HTTPRoute (`gateway.networking.k8s.io/v1`). |
| `httpRoute.annotations` | `{}` | — |
| `httpRoute.parentRefs` | `[{ name: gateway, sectionName: http }]` | Gateways aos quais o Route se conecta. |
| `httpRoute.hostnames` | `[chart-example.local]` | Hostnames do match. |
| `httpRoute.rules` | `[{ matches: [path PathPrefix /headers] }]` | Regras/filtros; `backendRefs` aponta para o Service. |

> `httpRoute` traz `parentRefs`/`hostnames`/`rules` de exemplo já preenchidos no
> default, mas nada é renderizado enquanto `httpRoute.enabled` for `false`.

---

## Exemplos por cenário

### Ingress habilitado

```yaml
image:
  repository: nginx
  tag: "1.27"
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: app.exemplo.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: app-tls
      hosts:
        - app.exemplo.com
```

### Gateway API (HTTPRoute) habilitado

```yaml
httpRoute:
  enabled: true
  parentRefs:
    - name: gateway
      sectionName: http
      # namespace: infra
  hostnames:
    - app.exemplo.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
```

### HPA habilitado

```yaml
deployment:
  hpa:
    enabled: true
    minReplicas: 2
    maxReplicas: 8
    targetCPUUtilizationPercentage: 70
    # targetMemoryUtilizationPercentage: 80
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

### Persistência habilitada

```yaml
persistence:
  enabled: true
  size: 5Gi
  mountPath: /data
  accessMode: ReadWriteOnce
  # storageClass: standard
```

### ExternalSecrets

```yaml
externalSecrets:
  # Opcionais (defaults mostrados):
  # apiVersion: external-secrets.io/v1
  # secretStoreKind: ClusterSecretStore
  secretStoreKind: ClusterSecretStore
  secrets:
    - name: my-external-secret
      key: cluster/namespace/my-external-secret   # (ou 'path')
      refreshInterval: 1h
      secretStore: vault-backend
```

Cada item em `secrets` gera um `ExternalSecret` que usa `dataFrom.extract` na
`key`/`path` informada, e o Secret resultante (mesmo `name`) é injetado no
`envFrom` do Deployment.

### ConfigMap com injeção em env

```yaml
configMaps:
  app-config:
    envFrom: true          # monta todo o ConfigMap no envFrom do pod
    data:
      LOG_LEVEL: info
    annotations:
      description: "Config da app"
```

---

## Desenvolvimento e CI

### Validação local

```bash
helm lint .
helm template minha-app .                       # render com defaults
helm template minha-app . -f ci/hpa-values.yaml # render de um cenário

# Render + validação de schema (opcional, requer kubeconform)
helm template t . | kubeconform -strict -summary \
  -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  -ignore-missing-schemas
```

### Pipeline de CI (`.github/workflows/ci.yaml`)

Dispara em **pull request** para `main`, com dois jobs:

1. **Lint & install (chart-testing)** — roda `ct lint` (com
   `--check-version-increment`) e, se o chart mudou, `ct install` num cluster
   **kind** para um smoke test real. Config em `ct.yaml` (chart na raiz,
   `check-version-increment: true`).
2. **Validate manifests (kubeconform)** — renderiza os cenários (defaults +
   todos os `ci/*-values.yaml` e `test/render/*-values.yaml`) e valida contra os
   schemas do Kubernetes (`1.31.0`) e das CRDs (catálogo
   [datreeio/CRDs-catalog](https://github.com/datreeio/CRDs-catalog), com
   `-ignore-missing-schemas`).

Todas as GitHub Actions estão **fixadas por commit SHA (pin)**. O
[Dependabot](.github/dependabot.yml) atualiza as Actions semanalmente.

### Convenção de bump de versão (obrigatória)

Como o CI roda `ct lint --check-version-increment`, **todo PR que altera o chart
precisa incrementar `version` no `Chart.yaml`** (SemVer). PRs sem bump falham no
job de chart-testing.

### Fixtures

| Diretório | Uso | Imagem |
| --- | --- | --- |
| `ci/*-values.yaml` | `ct install` real no kind **e** kubeconform | `registry.k8s.io/pause:3.10` |
| `test/render/*-values.yaml` | **Somente** kubeconform (dependem de CRD ausente no kind) | `registry.k8s.io/pause:3.10` |

Os fixtures usam `registry.k8s.io/pause` porque a imagem default (`hello-world`)
encerra imediatamente e nunca fica `Ready` — o `pause` é um processo de longa
duração que satisfaz o `securityContext` restritivo (non-root, rootfs read-only,
todas as capabilities dropadas). Os cenários de `test/render/` (HTTPRoute e
ExternalSecret) só são validados estaticamente porque suas CRDs não existem num
kind padrão.

### Empacotamento

O `.helmignore` garante que o `.tgz` publicado contenha apenas o essencial:
`Chart.yaml`, `values.yaml`, `values.schema.json` e `templates/`. Diretórios de
CI/test/docs (`.github/`, `.claude/`, `ci/`, `test/`, `ct.yaml`, `*.md`) ficam
de fora do artefato.

---

## Release e publicação

O workflow `.github/workflows/create-chart-release.yaml` dispara ao criar uma
**tag `v*.*.*`** e publica o chart como artefato OCI no Amazon ECR.

### Como cortar uma release

```bash
# 1. Atualize a versão no Chart.yaml (deve casar com a tag, sem o 'v')
#    version: 1.2.2  ->  tag v1.2.2
# 2. Faça o commit/PR e merge na main
# 3. Crie e envie a tag
git tag v1.2.2
git push origin v1.2.2
```

### O que o workflow faz

1. Extrai a versão da tag (`v1.2.2` → `1.2.2`).
2. `helm lint .` (bloqueante).
3. **Valida que `Chart.yaml` version == tag** (falha se divergir).
4. `helm dependency update` + `helm package` (bloqueantes).
5. **Garante o repositório ECR** de forma idempotente
   (`describe-repositories || create-repository`) — o ECR não cria o repo
   automaticamente no `helm push`.
6. Login OCI e `helm push` para `oci://$ECR_REGISTRY[/$ECR_REPOSITORY]`,
   capturando o **digest** do artefato empurrado.
7. **Assina** o chart com **cosign keyless (OIDC)** apontando para o digest.
8. Gera um **SBOM SPDX** (anchore/sbom-action) e o **anexa como attestation**
   (`cosign attest --type spdxjson`) ao mesmo digest.

O caminho de destino é `<ECR_REGISTRY>/<ECR_REPOSITORY>/base-template-chart`
(o `helm push` acrescenta o nome do chart ao path). Sem `ECR_REPOSITORY`, o
repositório é apenas `base-template-chart`.

### Variáveis e segredos necessários

Configure em **Settings → Secrets and variables → Actions** do repositório:

| Tipo | Nome | Uso |
| --- | --- | --- |
| Variable | `AWS_REGION` | Região do ECR. |
| Variable | `ECR_REGISTRY` | Host do registry (ex.: `<acct>.dkr.ecr.<region>.amazonaws.com`). |
| Variable | `ECR_REPOSITORY` | **Opcional**. Prefixo de path do repositório no ECR. |
| Secret | `AWS_ROLE_TO_ASSUME` | Role IAM assumida via **OIDC**. |

A role IAM precisa de permissões de ECR
(`ecr:DescribeRepositories`, `ecr:CreateRepository`, `ecr:GetAuthorizationToken`
e push de camadas) e de push das assinaturas/attestations do cosign no mesmo
repositório. A assinatura keyless usa **Fulcio/Rekor públicos** e exige
`id-token: write` (já concedido no workflow).

---

## Verificando assinatura e SBOM

Substitua `<ref>` pelo endereço completo do chart no ECR
(`<registry>/<prefix>/base-template-chart:1.2.2` ou por digest `...@sha256:...`).

Verificar a **assinatura keyless** (ajuste identidade/issuer conforme sua
policy):

```bash
cosign verify "<ref>" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp 'https://github.com/thiagofbarros/base-template-chart/.+'
```

Verificar a **attestation do SBOM (SPDX)**:

```bash
cosign verify-attestation "<ref>" \
  --type spdxjson \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp 'https://github.com/thiagofbarros/base-template-chart/.+'
```

Baixar o SBOM anexado (via attestation SPDX):

```bash
# Extrai o predicate SPDX da attestation
cosign verify-attestation "<ref>" --type spdxjson \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp 'https://github.com/thiagofbarros/base-template-chart/.+' \
  | jq -r '.payload' | base64 -d | jq '.predicate'
```

> O SBOM é publicado como **attestation** (`cosign attest`), não como um artefato
> `.sbom` separado — por isso a recuperação é feita via `verify-attestation`, e
> não por `cosign download sbom`.

---

## Licença / mantenedor

Mantenedor: **Thiago Barros** — thiagofbarros@outlook.com
Repositório: <https://github.com/thiagofbarros/base-template-chart>
