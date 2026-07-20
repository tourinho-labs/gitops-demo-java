# demo-java — deploy (gitops)

Repo de **deploy** do `demo-java`, **gerenciado por máquina** — par do
repo de **código** `tourinho-labs/demo-java`. **O dev não edita aqui**:
ele mexe em `tools/microsservico/` no repo de código, e o CI (config-sync) copia
pra cá. Descoberta e Apps são da **plataforma** (ApplicationSet `scaffolded-apps-helm`),
não deste repo.

O app consome o **arquétipo Helm central `backend`**
(`oci://ghcr.io/tourinhom/charts`) por versão, sem copiar o chart.

## Estrutura

```
Chart.yaml                 dep <arquétipo>@versão (Renovate bumpa)
values.yaml                BASE — params comuns (config-sync copia do dev)
values/
  <env>.yaml               delta do dev por env (config-sync)
  <env>.image.yaml         image por env — DONO: Kargo (nasce na promoção)
image.seed.yaml            semente que o Kargo copia p/ criar <env>.image.yaml
kargo/                     Project + Warehouse + Stages (dev→hml→prod)
renovate.json              bump do arquétipo
```

## Quem escreve o quê (ninguém à mão)

| Camada | Onde | Quem |
|--------|------|------|
| base + delta por env | `values.yaml`, `values/<env>.yaml` | **config-sync** (dev edita `tools/microsservico/` no código → push) |
| host (`ambiente`/domínio) | `global.*` via ApplicationSet | **plataforma** (não vive no repo) |
| image por env | `values/<env>.image.yaml` | **Kargo** (pin por digest) |
| versão do arquétipo | `Chart.yaml` | **Renovate** |

Camadas no Argo (last-wins): `values.yaml` → `values/<env>.yaml` → `values/<env>.image.yaml`
(+ `global.*` injetado pelo ApplicationSet).

## Promoção de ambiente (Kargo)

O **Warehouse** observa a image no ghcr e cria Freight (imutável, por digest). Os
**Stages** promovem: **dev** (auto) → **hml** (auto, após verificação) → **prod**
(**manual**). Cada promoção cria/atualiza `values/<env>.image.yaml`; o ApplicationSet
`scaffolded-apps-helm-env` descobre o arquivo e **o ambiente passa a existir**.

Ou seja: um app recém-scaffoldado **não nasce com os 3 ambientes** — cada env
nasce quando é promovido (dev na 1ª image do CI; hml/prod nos gates seguintes).
