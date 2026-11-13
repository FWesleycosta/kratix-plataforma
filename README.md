# kratix-plataforma

Bootstrap da plataforma do ensaio pessoal Backstage (frontend) e Kratix (backend).
Um único cluster kind local, `plataforma` (contexto `kind-plataforma`), faz o papel de
plataforma e de destino dev. O cluster só fica ligado nas sessões de estudo.

## Estrutura

| Caminho | O que é |
|---|---|
| `kind/plataforma.yaml` | Configuração do cluster kind (um nó, API em `127.0.0.1:6443`) |
| `manifests/kratix/kratix.yaml` | Cópia versionada do manifesto de instalação do Kratix |
| `manifests/kratix/state-store.yaml` | GitStateStore `github` e Destination `dev` |
| `manifests/argocd/install/` | Argo CD em modo core: cópia do upstream, Namespace e kustomization (digests e réplicas) |
| `manifests/argocd/apps/` | AppProjects e Applications que ligam os repositórios de pedidos e de State Store ao cluster |

## Como as peças se ligam

```text
kratix-pedidos (Git) → Argo CD → pedido no cluster → Kratix (Pipeline do Promise)
  → kratix-state-store (Git) → Argo CD → recursos no cluster
```

| Repositório | Quem escreve | Quem lê | Aplicado por |
|---|---|---|---|
| [kratix-pedidos](https://github.com/FWesleycosta/kratix-pedidos) | Scaffolder do Backstage (passo 6); até lá, à mão | Argo CD, por HTTPS, sem credencial | Application `kratix-pedidos` (AppProject `kratix-pedidos`) |
| [kratix-state-store](https://github.com/FWesleycosta/kratix-state-store) | Só o Kratix, por deploy key SSH com escrita só nesse repositório | Argo CD, por HTTPS, sem credencial | Applications `kratix-dev-dependencies` e `kratix-dev-resources` (AppProject `kratix-state-store`) |

Os dois são públicos, com um ruleset na `main` que só bloqueia force push e exclusão: o Kratix
e o Scaffolder fazem commit direto, sem PR e sem assinatura.

- O AppProject `kratix-pedidos` só aceita recursos com namespace, no namespace `default`.
  Recurso de cluster vindo dos pedidos é recusado.
- O AppProject `kratix-state-store` libera recursos de cluster, porque as dependências de um
  Promise trazem CRD e Namespace.
- **Nenhum Secret vai para o State Store nem para os pedidos:** os repositórios são públicos.

### Credenciais

| Quem | Credencial | Onde fica |
|---|---|---|
| Kratix → `kratix-state-store` | Deploy key SSH (ed25519, sem senha) com escrita | Secret `kratix-state-store-git` no namespace `kratix-platform-system`, com as chaves `sshPrivateKey` e `knownHosts`. Criado à mão no cluster, nunca no Git |
| Argo CD → os dois repositórios | Nenhuma (repositórios públicos) | — |

A linha `knownHosts` do `github.com` é pública: confira com as impressões digitais em
<https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints>.
Para girar a chave: gere outra, troque a deploy key no repositório e recrie o Secret.

`insecure: false` fica explícito no GitStateStore: a CRD assume `true` (sem verificar TLS)
quando o campo está vazio.

## Versões

Conferidas em 2026-11-12.

| Item | Versão | Fonte |
|---|---|---|
| kind | v0.33.0 | <https://github.com/kubernetes-sigs/kind/releases/tag/v0.33.0> |
| Imagem do nó | `kindest/node:v1.36.4@sha256:099e049362a1526b2db71494e1947aae99bd16290d7c895f2b7ea312e3cbfaed` | Notas da release v0.33.0 do kind |
| cert-manager | v1.21.3 | <https://github.com/cert-manager/cert-manager/releases/tag/v1.21.3> e <https://cert-manager.io/docs/releases/> |
| Kratix | release contínua `latest` de 2026-11-12 (cópia em `manifests/kratix/`) | <https://docs.kratix.io/main/guides/installing-kratix-others> |
| Argo CD | v3.6.1, modo core (conferido em 2026-11-13) | <https://github.com/argoproj/argo-cd/releases/tag/v3.6.1> e <https://argo-cd.readthedocs.io/en/stable/operator-manual/core/> |

Hashes (sha256):

- `manifests/kratix/kratix.yaml`: `5a966d6a9bc9de37f75bc8c6652f0a193caa2cdc4b05a66b064a34d7e7a2e66a`
  (imagem do controlador `syntasso/kratix-platform@sha256:9db2164a56d001214ae0565e9fa111f097f8a6997dafd946d941554e32a4b1ad`)
- `cert-manager.yaml` v1.21.3: `c04a51295adda2b1a775146751f973e787190778c7f35180861d9678876e5c74`
- `manifests/argocd/install/core-install.yaml` (v3.6.1): `0b38189d6e1fd76ef5c3db10355026c48a02020d82bf21a1a1b5ab16dffe4f3c`
  - imagem `quay.io/argoproj/argocd:v3.6.1@sha256:296e3d74f449b2ce45265613fcb68c18ed7366488360c8ea32f5f2a93d2e5c54`
  - imagem `public.ecr.aws/docker/library/redis:8.10.1-alpine@sha256:ef216d85ec44f4fdf7c0e6df2d1757a4734ed32fbb278cabd3181b5a27353b79`
  - os digests são dos índices multiarquitetura (amd64 e arm64) e ficam no `kustomization.yaml`, não na cópia

### Por que Kubernetes 1.36

- O cert-manager 1.21 suporta Kubernetes de 1.33 a 1.36.
- O Argo CD 3.6 (próximo passo) é testado de 1.34 a 1.37.
- A 1.36 é a mais nova que atende aos dois. O kubectl 1.37 fala com ela (diferença de uma versão é suportada).

### Por que uma cópia do Kratix

O Kratix publica o manifesto numa release contínua (`latest`), sem versão numerada, e ela é
republicada sem aviso. Aplicar direto da URL dá um resultado diferente a cada recriação do
cluster. A cópia versionada deixa a instalação repetível e a atualização vira uma troca de
arquivo revisada em PR.

Não usamos o `kratix-quick-start-installer.yaml`: ele instala Flux, SeaweedFS e uma
Destination, e aqui o GitOps é o Argo CD com State Store num repositório do GitHub.

## Atualizar a cópia do Kratix

1. Baixe `https://github.com/syntasso/kratix/releases/download/latest/kratix.yaml` para uma pasta temporária.
2. Compare com a cópia atual (`diff`) e confira a imagem nova (`grep 'image:'`).
3. Substitua `manifests/kratix/kratix.yaml`, atualize o sha256 e a data neste README e abra um PR.

## Argo CD em modo core

Só o application-controller, o repo-server e o redis: sem servidor de API, sem interface web,
sem dex e sem notifications. O sync aparece pelo `kubectl` (`applications` no namespace
`argocd`) ou pelo `argocd` CLI com `--core`. O status chega ao Backstage pelo plugin Kubernetes.
O controlador de ApplicationSet fica com 0 réplicas (o ensaio não usa ApplicationSet).

Instalação com server-side apply, como pede a documentação do modo core (os CRDs são grandes
demais para o apply comum): `kubectl apply -k manifests/argocd/install --server-side --force-conflicts`.

### Atualizar a cópia do Argo CD

1. Baixe `https://raw.githubusercontent.com/argoproj/argo-cd/<versão>/manifests/core-install.yaml`
   para uma pasta temporária e compare com a cópia atual (`diff`).
2. Confira as imagens novas (`grep 'image:'`) e pegue o digest do índice de cada uma
   (`docker buildx imagetools inspect <imagem>:<tag>`).
3. Confira se a versão nova é testada na versão do Kubernetes do nó
   (`docs/operator-manual/tested-kubernetes-versions.md` na tag).
4. Substitua `manifests/argocd/install/core-install.yaml`, atualize os digests no
   `kustomization.yaml` e o sha256 e a data neste README e abra um PR.

## Ligar e desligar entre sessões

- Desligar: parar o contêiner `plataforma-control-plane` no Docker.
- Religar: iniciar o mesmo contêiner. O kind mantém o estado do cluster no contêiner.
- Recriar do zero: apagar o cluster e criar de novo com `kind/plataforma.yaml`. A CA e os tokens
  mudam, então o acesso do Backstage ao cluster precisa ser refeito.

## Memória (Docker Desktop com 4 GB)

Estimativa do passo 3 (nó do kind, cert-manager e Kratix): de 0,8 a 1,2 GiB.
Com o Argo CD core do passo 4 (3 pods, ApplicationSet em 0): de 1,3 a 1,55 GiB.
Valores medidos: a preencher.
