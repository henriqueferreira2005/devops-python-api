# API Python para demonstrar DevOps

Repositório didático para apresentar, em uma única aplicação, três ideias que se conectam:

1. uma API HTTP escrita em Python;
2. integração contínua com GitHub Actions;
3. execução reproduzível com Docker.

Neste trabalho final da disciplina **Integração DevOps**, o projeto foi estendido com **entrega contínua**: a cada merge na `main`, a aplicação é testada, empacotada em uma imagem Docker, publicada no **Amazon ECR** e implantada automaticamente em uma instância **Amazon EC2**.

```
Alteração no código → Branch → Pull Request → Testes automatizados → Merge na main
→ Build Docker → Publicação no ECR → Deploy na EC2 → Aplicação atualizada
```

O projeto utiliza FastAPI, pytest, Ruff, GitHub Actions, Docker, Amazon ECR e Amazon EC2. Os dados ficam em memória para manter o foco no fluxo DevOps, sem banco de dados.

## Estrutura do repositório

```
.
├── .github/workflows/
│   ├── ci.yml                 # integração contínua (Pull Requests)
│   └── deploy.yml             # build, publicação no ECR e deploy na EC2
├── app/
│   ├── __init__.py
│   └── main.py                # API FastAPI
├── docs/
│   ├── API.md
│   ├── DOCKER.md
│   ├── GITHUB-ACTIONS.md
│   ├── ROTEIRO-AULA.md
│   └── evidencias/            # prints da execução das pipelines
├── tests/test_api.py          # testes automatizados
├── .dockerignore
├── .python-version
├── compose.yaml
├── Dockerfile
├── pyproject.toml             # configuração de pytest, cobertura e Ruff
├── requirements.txt           # dependências de execução
└── requirements-dev.txt       # dependências de desenvolvimento e CI
```

## Tecnologias

- Python 3.13, FastAPI e Uvicorn
- pytest, pytest-cov e Ruff
- Docker e Docker Compose
- GitHub Actions
- Amazon ECR (registro privado de imagens)
- Amazon EC2 (Ubuntu) com IAM Role para leitura do ECR

## 1. Executar localmente

Pré-requisitos: Git e Python 3.13.

### Linux e macOS

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --requirement requirements-dev.txt
uvicorn app.main:app --reload
```

### Windows PowerShell

```powershell
py -3.13 -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --requirement requirements-dev.txt
uvicorn app.main:app --reload
```

Acesse:

- API: http://localhost:8000
- documentação Swagger: http://localhost:8000/docs
- verificação de saúde: http://localhost:8000/health

## 2. Experimentar a API

```bash
curl http://localhost:8000/health

curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Estudar integração contínua"}'

curl http://localhost:8000/tasks

curl -X PATCH http://localhost:8000/tasks/1/done
```

| Método | Caminho | Objetivo | Resposta esperada |
|---|---|---|---|
| GET | `/` | Apresentar a API | 200 |
| GET | `/health` | Informar saúde e versão | 200 |
| GET | `/tasks` | Listar tarefas | 200 |
| POST | `/tasks` | Criar uma tarefa | 201 |
| PATCH | `/tasks/{id}/done` | Concluir uma tarefa | 200 ou 404 |

Veja a explicação do código em [docs/API.md](docs/API.md).

## 3. Executar as verificações locais

```bash
ruff check .
ruff format --check .
pytest --cov=app --cov-report=term-missing
```

Esses comandos são os mesmos usados no GitHub Actions. Para aplicar a formatação automaticamente: `ruff format .`

## 4. Executar com Docker

```bash
docker build -t devops-python-api:v1 .
docker run --rm -p 8000:8000 --name devops-python-api devops-python-api:v1
```

Em outro terminal:

```bash
curl http://localhost:8000/health
docker ps
docker logs devops-python-api
```

Com Docker Compose:

```bash
docker compose up --build
docker compose down
```

O `Dockerfile` instala as dependências Python durante o build e inicia a API com o Uvicorn na porta 8000. Veja [docs/DOCKER.md](docs/DOCKER.md).

## 5. Pipeline de integração contínua (`.github/workflows/ci.yml`)

Executa em **Pull Requests para a `main`** (abertura, atualização e reabertura) e também manualmente pela aba Actions:

1. Baixa o código (`actions/checkout`)
2. Configura o Python 3.13 com cache do pip
3. Instala as dependências de `requirements-dev.txt`
4. Valida com `ruff check` e `ruff format --check`
5. Executa os testes com `pytest` e cobertura
6. Informa o resultado no PR e no resumo da execução

Se qualquer passo falhar, o PR fica reprovado e a entrega não acontece. Veja [docs/GITHUB-ACTIONS.md](docs/GITHUB-ACTIONS.md).

## 6. Pipeline de entrega contínua (`.github/workflows/deploy.yml`)

Executa a cada push (merge) na `main`, em três jobs encadeados por `needs`:

1. **Testes**: repete lint e pytest. Se falhar, os jobs seguintes não executam.
2. **Build e push no ECR**: autentica na AWS com GitHub Secrets, faz login no ECR, constrói a imagem e publica com duas tags: o **SHA do commit** e `latest`.
3. **Deploy na EC2**: conecta por SSH, autentica o Docker da instância no ECR (com a IAM Role), baixa a imagem do commit, remove o contêiner anterior, inicia o novo (`-p 80:8000`, `--restart unless-stopped`) e verifica o `/health`. Se a verificação falhar, o job falha.

### Credenciais

Nada sensível é versionado. Os valores ficam em **GitHub Secrets**: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `AWS_REGION`, `ECR_REPOSITORY`, `EC2_HOST`, `EC2_USER` e `EC2_SSH_KEY_B64` (chave SSH em Base64). A instância lê o ECR por **IAM Role**, sem chaves.

## 7. Arquitetura de implantação

```
┌──────────────┐   PR / merge   ┌───────────────────┐
│ Desenvolvedor│ ─────────────▶ │  GitHub (main)    │
└──────────────┘                └─────────┬─────────┘
                                          │ push na main
                                          ▼
                              ┌─────────────────────────┐
                              │ GitHub Actions          │
                              │ deploy.yml              │
                              │ 1. testes               │
                              │ 2. build + push (ECR)   │
                              │ 3. deploy via SSH       │
                              └───────┬───────────┬─────┘
                       push da imagem │           │ SSH
                       (tag = SHA)    ▼           ▼
                              ┌────────────┐  ┌──────────────────────────┐
                              │ Amazon ECR │◀─│ EC2 (Ubuntu + Docker)    │
                              │ (privado)  │  │ IAM Role lê o ECR        │
                              └────────────┘  │ docker pull + run :80    │
                                              └────────────┬─────────────┘
                                                           │ HTTP :80
                                                           ▼
                                                     Usuário / navegador
```

Infraestrutura na AWS:

- **ECR**: repositório privado `devops-python-api` (us-east-1)
- **EC2**: Ubuntu, `t3.micro`, Docker e AWS CLI instalados por *user data*, com perfil de instância que permite ler o ECR
- **Security Group**: porta 22 (SSH para o deploy) e porta 80 (HTTP)
- **Elastic IP**: endereço fixo para a API

## 8. API publicada

**http://54.161.242.182/**

- Saúde e versão: http://54.161.242.182/health
- Documentação: http://54.161.242.182/docs

> Os recursos da AWS são desligados após a avaliação, para evitar cobranças. Depois disso, o endereço deixa de responder.

### Verificar a versão implantada

```bash
curl http://54.161.242.182/health
```

Resposta esperada:

```json
{"status":"ok","version":"1.1.0"}
```

A imagem em execução na instância (`docker ps`) tem como tag o SHA do commit, o mesmo que aparece no ECR e no histórico do GitHub.

## 9. Fluxo de desenvolvimento

Todo o trabalho foi feito por branches e Pull Requests:

| PR | Branch | Conteúdo |
|---|---|---|
| [#1](https://github.com/henriqueferreira2005/devops-python-api/pull/1) | `feature/ci-cd-ec2` | Pipelines de CI e CD, formatação do código, `requirements-dev.txt` e remoção do workflow de exemplo do ECS |
| [#2](https://github.com/henriqueferreira2005/devops-python-api/pull/2) | `fix/ssh-key-deploy` | Ajuste do tratamento da chave SSH |
| [#3](https://github.com/henriqueferreira2005/devops-python-api/pull/3) | `fix/ssh-key-b64` | Chave SSH em Base64 |
| [#4](https://github.com/henriqueferreira2005/devops-python-api/pull/4) | `fix/deploy-image-name` | Nome da imagem montado na EC2 |
| [#5](https://github.com/henriqueferreira2005/devops-python-api/pull/5) | `feature/atualiza-versao` | Atualização da versão para 1.1.0 (demonstração do deploy automático) |

### Alterações em relação ao projeto original

- `ci.yml` passou a validar Pull Requests (antes também rodava em push e testava o contêiner).
- `deploy.yml` foi adicionado para publicar no ECR e implantar na EC2.
- O workflow de exemplo `aws.yml` (deploy no ECS) foi removido, pois esta solução usa EC2.
- `requirements-dev.txt` foi criado com as dependências de CI (pytest, pytest-cov, httpx e ruff).
- `app/main.py` recebeu ajuste de formatação exigido pelo Ruff e, na segunda entrega, a versão `1.1.0`.

### Problemas encontrados e soluções

- **Formatação reprovada no CI** (`ruff format`): corrigida em `app/main.py`; o CI barrou o PR até a correção.
- **Chave SSH com formato inválido no runner**: o secret passou a guardar a chave em Base64.
- **Nome da imagem entre jobs**: o GitHub omite *outputs* que contêm dados mascarados. A EC2 passou a montar o nome da imagem com o ID da conta e o SHA do commit.
- **Credenciais temporárias expiradas** (AWS Academy): os secrets `AWS_*` são renovados a cada sessão do laboratório.

## 10. Evidências

### Integração contínua

![CI aprovada no Pull Request #1](docs/evidencias/01-ci-pr-verde.png)
*Pull Request #1 com o check "CI / Qualidade e testes" aprovado.*

![Execução do CI no Actions](docs/evidencias/02-ci-actions-verde.png)
*Todos os passos verdes, incluindo "Testes com pytest".*

### Configuração da AWS

![EC2 configurada](docs/evidencias/03-ec2-configurada.png)
*Instância com Docker, AWS CLI e a role do laboratório (`LabRole`).*

![Secrets do repositório](docs/evidencias/04-secrets.png)
*Credenciais guardadas em GitHub Secrets (valores ocultos).*

### Publicação e deploy

![Deploy concluído](docs/evidencias/05-deploy-verde.png)
*Workflow "Deploy" com os três jobs concluídos.*

![Imagens no ECR](docs/evidencias/06-ecr-imagens.png)
*Repositório privado do ECR com as imagens identificadas pelo SHA do commit.*

![Contêiner em execução](docs/evidencias/07-docker-ps.png)
*Contêiner `devops-python-api` em execução na EC2, porta 80 mapeada para a 8000.*

### Atualização automática de versão

![API na versão 1.0.0](docs/evidencias/08-api-v1.png)
*Primeira versão publicada: `/health` retorna `1.0.0`.*

![API na versão 1.1.0](docs/evidencias/09-api-v2.png)
*Depois do merge do PR #5, o deploy automático atualizou a API para `1.1.0`.*

![CI do Pull Request #5](docs/evidencias/10-pr5-ci-verde.png)
*Pull Request #5 com a CI aprovada antes do merge.*

## Referências

- [GitHub Actions: criar e testar Python](https://docs.github.com/actions/use-cases-and-examples/building-and-testing/building-and-testing-python)
- [actions/checkout](https://github.com/actions/checkout)
- [actions/setup-python](https://github.com/actions/setup-python)
- [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)
- [aws-actions/amazon-ecr-login](https://github.com/aws-actions/amazon-ecr-login)
- [FastAPI](https://fastapi.tiangolo.com/)
- [Docker para Python](https://docs.docker.com/language/python/)
