# aula09-gc — Pipeline de CI/CD com GitHub Actions e Netlify

Atividade prática da **Aula 09 — Integração Contínua e Pipelines**
ENS4000 Gerência de Configuração · Universidade de Caxias do Sul · Prof. Dr. Leonardo Pellizzoni
Aluno: Gianluca Debastiani

**Sistema em produção:** https://aula09-gc.netlify.app

---

## Objetivo

Construir, como equipe de **engenharia de plataforma**, o caminho automatizado do código até o usuário final:
todo merge na `main` publica o sistema em produção sem nenhuma ação humana além do próprio merge.

O "sistema" é o arquivo `index.html`. Ele é deliberadamente trivial: no lugar dele poderia estar uma API em Java,
um app Flutter ou qualquer outro serviço. A estrutura do pipeline seria a mesma; mudariam apenas os steps internos
(compilar, testar, empacotar).

## Requisitos atendidos

| Requisito | Como foi atendido |
|---|---|
| Pipeline versionado no repositório | `.github/workflows/deploy.yml` |
| Publicação no Netlify sem usar o CI/CD do próprio Netlify | Projeto criado via CLI (`netlify sites:create`), sem vínculo com o Git; o deploy é feito pelo GitHub Actions com `netlify deploy --prod` |
| Credenciais fora do código | Token e ID do projeto guardados como *secrets* do GitHub |
| Deploy automático a cada alteração na `main` | Trigger `push` na `main` executa build + deploy |
| Commits direto na `main` bloqueados | Branch protection: PR obrigatória + status check `build` obrigatório |
| Rastreabilidade a partir da produção | Versão (commit + execução do pipeline) gravada no próprio site e em `/version.json` |
| Repositório público | ✔ |

## Estrutura do repositório

```
aula09-gc/
├── .github/
│   └── workflows/
│       └── deploy.yml     # pipeline de CI/CD
├── .gitignore             # ignora a pasta local .netlify
├── index.html             # o "sistema"
└── README.md
```

## Arquitetura do pipeline

```
issue → branch → Pull Request ──(job build = status check)──> merge na main
                                                                  │
                                                                  ▼
                                    push na main → job build → job deploy → Netlify (produção)
```

| Componente | Onde está no `deploy.yml` |
|---|---|
| **Evento (trigger)** | `pull_request` na `main` (apenas verifica), `push` na `main` (verifica e publica), `workflow_dispatch` (execução manual) |
| **Verificação** | Job `build`: checa se o `index.html` existe e é um HTML válido. É o status check exigido para o merge |
| **Build** | Job `build`: gera a pasta `dist/` e grava no HTML o SHA do commit, o ID da execução e a data; gera `version.json`; publica `dist/` como artefato da execução |
| **Publicação** | Job `deploy` (depende do `build`, nunca roda em PR): baixa o artefato e executa `netlify deploy --prod` autenticado pelos secrets |
| **Registro** | Histórico na aba *Actions*, ambiente `production` no GitHub, resumo da execução com URL e commit, e mensagem de cada deploy no Netlify (`commit \| run \| título`) |

## Secrets

Cadastrados em *Settings → Secrets and variables → Actions*:

| Secret | Conteúdo |
|---|---|
| `NETLIFY_AUTH_TOKEN` | Personal access token gerado em *Netlify → User settings → Applications* |
| `NETLIFY_SITE_ID` | Project ID do projeto no Netlify |

O Netlify CLI lê essas variáveis de ambiente automaticamente, então nenhuma credencial aparece no código nem no log.

## Proteção da branch `main`

Configurada em *Settings → Branches*:

- **Require a pull request before merging**: nenhum commit entra na `main` sem PR;
- **Require status checks to pass before merging**: o job `build` precisa passar;
- **Do not allow bypassing the above settings**: a regra vale também para administradores.

## Fluxo de trabalho do time de desenvolvimento

1. Criar uma issue descrevendo a alteração (ex.: `#1`).
2. Criar uma branch a partir da `main`:
   ```bash
   git checkout -b feature/1-descricao
   ```
3. Alterar o `index.html`, commitar e enviar a branch:
   ```bash
   git commit -am "Descrição da alteração"
   git push origin feature/1-descricao
   ```
4. Abrir uma Pull Request para a `main` com `Closes #1` na descrição.
5. Aguardar o check `build` passar e fazer o merge.
6. O pipeline publica automaticamente; conferir em https://aula09-gc.netlify.app.

Nenhum contato com a equipe de plataforma é necessário.

## Rastreabilidade: da produção até a issue

O rodapé do site mostra a versão publicada:

- **Versão** → link para o commit no GitHub → o commit mostra a PR que o levou à `main` → a PR mostra a issue vinculada (`Closes #N`);
- **Build** → link para a execução do pipeline no GitHub Actions (logs, artefato e deploy).

Os mesmos dados estão disponíveis em formato de máquina em https://aula09-gc.netlify.app/version.json.

## Conceitos aplicados

- **Integração Contínua (CI):** cada PR é construída e verificada automaticamente antes do merge.
- **Entrega Contínua (Continuous Delivery):** o artefato gerado está sempre pronto para produção.
- **Implantação Contínua (Continuous Deployment):** o que passa na verificação vai para produção automaticamente, sem intervenção manual. É o modelo implementado aqui.
- **Secrets / DevSecOps:** credenciais geridas pela plataforma, fora do código.
- **Gerência de configuração:** pipeline versionado junto com o código; cada versão em produção é identificável e rastreável.

## Execução manual (referência)

Deploy equivalente ao do pipeline, feito pelo terminal:

```bash
npm install -g netlify-cli
netlify login
netlify status
netlify deploy --site="ID_DO_PROJETO" --auth="TOKEN" --dir="." --prod
```

O pipeline automatiza exatamente esse passo, com as credenciais vindas dos secrets.