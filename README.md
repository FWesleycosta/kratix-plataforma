# kratix-plataforma

Bootstrap da plataforma do ensaio pessoal Backstage (frontend) e Kratix (backend).
Um único cluster kind local, `plataforma` (contexto `kind-plataforma`), faz o papel de
plataforma e de destino dev. O cluster só fica ligado nas sessões de estudo.

## Estrutura

| Caminho | O que é |
|---|---|
| `kind/plataforma.yaml` | Configuração do cluster kind (um nó, API em `127.0.0.1:6443`) |
| `manifests/kratix/kratix.yaml` | Cópia versionada do manifesto de instalação do Kratix |

## Versões

Conferidas em 2026-11-12.

| Item | Versão | Fonte |
|---|---|---|
| kind | v0.33.0 | <https://github.com/kubernetes-sigs/kind/releases/tag/v0.33.0> |
| Imagem do nó | `kindest/node:v1.36.4@sha256:099e049362a1526b2db71494e1947aae99bd16290d7c895f2b7ea312e3cbfaed` | Notas da release v0.33.0 do kind |
| cert-manager | v1.21.3 | <https://github.com/cert-manager/cert-manager/releases/tag/v1.21.3> e <https://cert-manager.io/docs/releases/> |
| Kratix | release contínua `latest` de 2026-11-12 (cópia em `manifests/kratix/`) | <https://docs.kratix.io/main/guides/installing-kratix-others> |

Hashes (sha256):

- `manifests/kratix/kratix.yaml`: `5a966d6a9bc9de37f75bc8c6652f0a193caa2cdc4b05a66b064a34d7e7a2e66a`
  (imagem do controlador `syntasso/kratix-platform@sha256:9db2164a56d001214ae0565e9fa111f097f8a6997dafd946d941554e32a4b1ad`)
- `cert-manager.yaml` v1.21.3: `c04a51295adda2b1a775146751f973e787190778c7f35180861d9678876e5c74`

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

## Ligar e desligar entre sessões

- Desligar: parar o contêiner `plataforma-control-plane` no Docker.
- Religar: iniciar o mesmo contêiner. O kind mantém o estado do cluster no contêiner.
- Recriar do zero: apagar o cluster e criar de novo com `kind/plataforma.yaml`. A CA e os tokens
  mudam, então o acesso do Backstage ao cluster precisa ser refeito.

## Memória (Docker Desktop com 4 GB)

Estimativa do passo 3 (nó do kind, cert-manager e Kratix): de 0,8 a 1,2 GiB.
Com o Argo CD core do passo 4: de 1,1 a 1,7 GiB. Valores medidos: a preencher.
