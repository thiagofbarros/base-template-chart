# Contribuindo

Obrigado por contribuir com o `base-template-chart`. Este guia é curto e direto;
a documentação completa está no [README](./README.md).

## Fluxo

1. Crie uma branch a partir de `main` (não commite direto na `main`).
2. Faça a mudança nos `templates/` e/ou `values.yaml`, mantendo o estilo,
   indentação e nomes de helpers já existentes.
3. **Incremente a `version` no `Chart.yaml`** (SemVer). Isto é **obrigatório**:
   o CI roda `ct lint --check-version-increment` e falha PRs sem bump.
4. Se adicionar/alterar chaves de `values`, atualize também o
   `values.schema.json` e a tabela de values no README.
5. Abra um Pull Request para `main`.

## Validação local antes do PR

```bash
helm lint .
helm template t .                          # defaults
helm template t . -f ci/hpa-values.yaml    # cenários
```

Se tiver `kubeconform` instalado, valide os manifests renderizados (comando no
README, seção "Desenvolvimento e CI").

## Fixtures de teste

- `ci/*-values.yaml`: cenários instaláveis num cluster `kind` (usados por
  `ct install`) e também validados por kubeconform. Use imagens de longa
  duração (ex.: `registry.k8s.io/pause`) compatíveis com o `securityContext`
  restritivo.
- `test/render/*-values.yaml`: cenários que dependem de CRDs ausentes no kind
  (HTTPRoute, ExternalSecret) — validados **apenas** estaticamente por
  kubeconform.

## O que o CI valida (em PR)

- `ct lint` (+ `check-version-increment`) e `ct install` em `kind`.
- `kubeconform` sobre defaults e todos os fixtures (`ci/` e `test/render/`).

## Release

Releases são cortadas por maintainers via tag `v*.*.*` (a versão da tag deve
casar com `Chart.yaml`). Veja a seção "Release e publicação" no README.

## Commits

Mensagens em pt-BR são bem-vindas (o histórico do repo é em pt-BR). Prefira
mensagens descritivas e escopadas à mudança.
