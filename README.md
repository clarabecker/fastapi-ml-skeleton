# Upgrade de Versão: Python, FastAPI e Pydantic no fastapi-ml-skeleton

**Cenário de migração:** Upgrade de Versão de Linguagem/Framework (Python 3.11 → 3.13, FastAPI e Pydantic para as versões estáveis mais recentes).

---

## 1. Contextualização e Motivação

### Repositório de origem

- **Link:** https://github.com/eightBEC/fastapi-ml-skeleton
- **Readme original:** [README.md](https://github.com/eightBEC/fastapi-ml-skeleton/blob/main/README.md)
- **Projeto:** *FastAPI Model Server Skeleton* — esqueleto de aplicação para servir modelos de Machine Learning como API REST, de autoria de eightBEC. Licença Apache 2.0, 604 estrelas. Inclui um modelo de regressão de exemplo (previsão de preço de imóveis) persistido via `joblib`/scikit-learn e carregado **em memória** na inicialização do servidor — **não há nenhum banco de dados envolvido**, o que torna este repositório um bom candidato para uma migração de versão "pura", sem a complexidade adicional de estado persistido externamente.

### Cenário Atual

O próprio changelog do projeto documenta que a versão vigente (`v1.1.0`, publicada em 28/12/2023) já foi fruto de uma migração anterior ("Update to Python 3.11, FastAPI 0.108.0, Pydantic 2.x, Poetry"). É esse estado — o mais recente disponível no repositório — que serve de ponto de partida para a nova rodada de upgrade proposta aqui:

| Componente | Situação atual (`pyproject.toml`, branch `main`) |
|---|---|
| Linguagem | Python `^3.11`, gerenciado via Poetry |
| Framework web | FastAPI `^0.109.1` |
| Validação de dados | Pydantic `^2.5.3` |
| Servidor ASGI | Uvicorn `^0.25.0` |
| ML / inferência | scikit-learn `^1.3.2`, numpy `^1.26.2`, joblib `^1.3.2` (carregamento do modelo `.joblib` em `sample_model/`) |
| Logging | loguru `^0.7.2` |
| HTTP client (uso interno/testes) | requests `^2.31.0` |
| Ferramentas de dev | isort `^5.13.2`, mypy `^1.8.0`, black `^24.3.0`, flake8 `^6.1.0`, bandit `^1.7.6`, pytest `^7.4.3`, httpx `^0.26.0`, pytest-cov `^4.1.0` |
| Persistência | **Nenhuma** — o único "estado" da aplicação é o arquivo do modelo treinado (`sample_model/*.joblib`), lido uma única vez no `startup` do servidor (`fastapi_skeleton/core/event_handlers.py`) |
| Autenticação | Chave de API simples validada via header, comparada com o valor de `API_KEY` definido em `.env` (`fastapi_skeleton/core/security.py`) |
| Estrutura do módulo | `fastapi_skeleton/{api/routes, core, models, services}` — routers em `api/routes/` (`heartbeat.py`, `prediction.py`), configuração em `core/config.py`, schemas Pydantic em `models/` (`payload.py`, `prediction.py`, `heartbeat.py`), lógica de carregamento/inferência do modelo em `services/` |

Como não existe camada de banco de dados, toda a superfície de risco da migração se concentra em três pontos: (1) compatibilidade das APIs do FastAPI/Pydantic usadas nos routers e schemas, (2) compatibilidade binária do arquivo `.joblib` do modelo com a nova versão do scikit-learn, e (3) compatibilidade da suíte de lint/testes com o novo interpretador Python.

### Cenário Alvo

| Componente | Situação alvo |
|---|---|
| Linguagem | Python `^3.13` — versão estável mais recente com suporte ativo de segurança, ganhos de performance do interpretador (especializações de bytecode introduzidas a partir do 3.11/3.12) e horizonte de suporte mais longo que o 3.11 |
| Framework web | FastAPI atualizado para a última versão estável publicada no PyPI no momento da migração (`≥ 0.136`, linha ativa confirmada em 2026) |
| Validação de dados | Pydantic mantido na série 2.x, atualizado para a última *minor* estável compatível com o FastAPI escolhido |
| Servidor ASGI | Uvicorn atualizado para a última versão compatível com Python 3.13 |
| ML / inferência | numpy e scikit-learn atualizados para as versões com *wheels* pré-compilados para Python 3.13; **modelo `.joblib` re-serializado** com a versão nova do scikit-learn (ver Passo 7) |
| Ferramentas de dev | isort, mypy, black, flake8, bandit, pytest, httpx e pytest-cov atualizados para as versões compatíveis com Python 3.13 |
| Estrutura do módulo | Sem alterações estruturais — o upgrade é de versões de dependência, não de arquitetura |

### Justificativa técnica e benefícios esperados

- **Ciclo de vida e segurança:** manter a aplicação em uma versão de Python e de FastAPI ativamente mantidas reduz a exposição a CVEs não corrigidos em versões antigas do interpretador e das bibliotecas de rede (Starlette/Uvicorn).
- **Evitar uma migração "big bang" futura:** o próprio ecossistema FastAPI/Pydantic vem, ano a ano, descontinuando compatibilidade com versões anteriores (o suporte à *shim* `pydantic.v1` dentro do FastAPI já está em processo de depreciação nas versões mais recentes do framework). Atualizar em incrementos menores e frequentes é mais barato do que acumular débito técnico e ser forçado a um salto grande no futuro.
- **Performance:** Python 3.12/3.13 trazem ganhos de desempenho relevantes no interpretador em relação ao 3.11, o que beneficia diretamente uma aplicação de inferência de ML sensível a latência.
- **Compatibilidade de ecossistema de dados:** numpy e scikit-learn recentes só publicam *wheels* pré-compilados para versões de Python também recentes; permanecer em Python 3.11 eventualmente força o uso de versões cada vez mais antigas (e não mais corrigidas) dessas bibliotecas.
- **Superfície de mudança pequena e bem isolada:** como a aplicação não depende de banco de dados, a migração se resume a atualizar o `pyproject.toml`, resolver eventuais *breaking changes* de API do FastAPI/Pydantic nos poucos módulos existentes, e revalidar o artefato do modelo — um cenário de baixo risco, ideal para praticar uma rotina de upgrade de versão.

---

## 2. Ambiente e Pré-requisitos

| Ferramenta | Versão mínima | Verificação |
|---|---|---|
| Python | 3.13.x | `python3.13 --version` |
| pyenv (recomendado, para instalar o 3.13 sem afetar o Python do sistema) | qualquer versão recente | `pyenv --version` |
| Poetry | ≥ 1.8 | `poetry --version` |
| Git | qualquer versão recente | `git --version` |

Credenciais e valores necessários (já usados hoje pela aplicação, apenas reaproveitados no `.env` durante a validação):

- `API_KEY` — string qualquer usada para autenticar as chamadas à API; pode ser gerada localmente com `python -c "import uuid; print(uuid.uuid4())"`, conforme o próprio `README.md` do projeto já instrui.

Não há credenciais de banco de dados, filas ou serviços externos a configurar — a aplicação é *stateless* e autocontida, dependendo apenas do arquivo do modelo versionado em `sample_model/`.

---

## 3. Passo a Passo Executável (Roteiro de Migração)

Todos os comandos abaixo assumem que o terminal está na raiz do repositório clonado (`fastapi-ml-skeleton/`).

### Passo 1 — Confirmar o estado do repositório e estabelecer uma linha de base

```bash
git clone https://github.com/eightBEC/fastapi-ml-skeleton.git
cd fastapi-ml-skeleton
git status
```

Crie uma branch dedicada para a migração, preservando a `main` intacta para rollback:

```bash
git checkout -b feature/upgrade-python313-fastapi
```

Antes de tocar em qualquer versão, rode a suíte de testes e o lint **na configuração atual**, para ter uma linha de base confiável de comparação:

```bash
poetry install
./scripts/test.sh
./scripts/linting.sh
```

### Passo 2 — Instalar o Python 3.13 e apontar o Poetry para ele

```bash
pyenv install 3.13.2
pyenv local 3.13.2
poetry env use $(pyenv which python)
```

### Passo 3 — Atualizar as versões no `pyproject.toml`

Seção `[tool.poetry.dependencies]` — atual:

```toml
[tool.poetry.dependencies]
python = "^3.11"
requests = "^2.31.0"
uvicorn = "^0.25.0"
fastapi = "^0.109.1"
numpy = "^1.26.2"
joblib = "^1.3.2"
loguru = "^0.7.2"
pydantic = "^2.5.3"
scikit-learn = "^1.3.2"
```

Alvo:

```diff
  [tool.poetry.dependencies]
- python = "^3.11"
+ python = "^3.13"
  requests = "^2.31.0"
- uvicorn = "^0.25.0"
+ uvicorn = "^0.34.0"
- fastapi = "^0.109.1"
+ fastapi = "^0.136.0"
- numpy = "^1.26.2"
+ numpy = "^2.1.0"
- joblib = "^1.3.2"
+ joblib = "^1.4.0"
  loguru = "^0.7.2"
- pydantic = "^2.5.3"
+ pydantic = "^2.10.0"
- scikit-learn = "^1.3.2"
+ scikit-learn = "^1.5.0"
```

Seção `[tool.poetry.group.dev.dependencies]` — atualize na mesma linha (mantendo compatibilidade com Python 3.13):

```diff
  [tool.poetry.group.dev.dependencies]
- isort = "^5.13.2"
+ isort = "^5.13.2"   # já compatível, sem alteração necessária
- mypy = "^1.8.0"
+ mypy = "^1.13.0"
- black = "^24.3.0"
+ black = "^24.10.0"
- flake8 = "^6.1.0"
+ flake8 = "^7.1.0"
- bandit = "^1.7.6"
+ bandit = "^1.8.0"
- pytest = "^7.4.3"
+ pytest = "^8.3.0"
- httpx = "^0.26.0"
+ httpx = "^0.28.0"
- pytest-cov = "^4.1.0"
+ pytest-cov = "^6.0.0"
```

Ajuste também o alvo de formatação do Black, hoje fixado em Python 3.11:

```diff
  [tool.black]
  line-length = 88
- target-version = ['py311']
+ target-version = ['py313']
```

> As faixas de versão acima devem ser conferidas contra o PyPI no momento real da execução da migração (`poetry show --outdated` ajuda a listar o que está disponível); o importante é fixar `python = "^3.13"` e deixar o `poetry lock` resolver as versões mais recentes compatíveis dentro de cada faixa.

### Passo 4 — Regerar o lockfile e reinstalar as dependências

```bash
poetry lock
poetry install
```

Se o `poetry lock` reportar conflitos de resolução (comum quando numpy/scikit-learn sobem de major version), resolva-os incrementando as faixas de versão dos pacotes apontados no erro, um de cada vez, e repetindo o comando.

### Passo 5 — Revisar o código-fonte quanto a APIs depreciadas do FastAPI/Pydantic

Mesmo já estando em Pydantic 2.x, vale rodar uma checagem por padrões que podem ter sido depreciados entre a versão atual e a alvo:

```bash
grep -rn "pydantic.v1\|\.dict(\|\.json(\|orm_mode" fastapi_skeleton/
```

- Ocorrências de `.dict()` / `.json()` em modelos Pydantic devem migrar para `.model_dump()` / `.model_dump_json()`.
- Ocorrências de `orm_mode = True` em `class Config` devem migrar para `model_config = ConfigDict(from_attributes=True)`.
- Qualquer `import` de `pydantic.v1` (shim de compatibilidade) deve ser eliminado, pois versões recentes do FastAPI vêm descontinuando esse caminho de compatibilidade.

Neste projeto especificamente, os schemas em `fastapi_skeleton/models/` (`payload.py`, `prediction.py`, `heartbeat.py`) são o único lugar onde esse tipo de padrão poderia aparecer — revise-os manualmente além do `grep`, já que buscas textuais não cobrem 100% dos casos (ex.: heranças indiretas de `BaseModel`).

### Passo 6 — Verificar a rota de autenticação e configuração

`fastapi_skeleton/core/config.py` normalmente usa `pydantic-settings`/`BaseSettings` para carregar o `.env`. Confirme que o pacote `pydantic-settings` está declarado explicitamente como dependência (no Pydantic 2.x, `BaseSettings` foi extraído do pacote principal):

```bash
grep -n "BaseSettings" fastapi_skeleton/core/config.py
poetry show pydantic-settings || poetry add pydantic-settings
```

### Passo 7 — Re-serializar o modelo de ML com a nova versão do scikit-learn

Como não há banco de dados, o artefato `sample_model/*.joblib` é o único "estado" persistido da aplicação — e arquivos `pickle`/`joblib` gerados por uma versão do scikit-learn **não têm garantia de compatibilidade binária** com versões futuras (major ou até minor) da biblioteca. Isso equivale, neste cenário, ao que seria uma migração de dados em um projeto com banco de dados.

```bash
poetry run python scripts/train_model.py   # ou o script de treino equivalente do projeto
```

Se o repositório não versionar um script de treino dedicado, recrie o modelo a partir do mesmo dataset e hiperparâmetros documentados, já dentro do ambiente com o scikit-learn atualizado, e substitua o arquivo em `sample_model/`. Ao carregar o modelo antigo sob a nova versão sem re-treinar, o scikit-learn normalmente emite um `InconsistentVersionWarning` — trate esse aviso como bloqueante, não apenas informativo.

### Passo 8 — Rodar lint e testes na nova versão

```bash
./scripts/linting.sh
./scripts/test.sh
```

### Passo 9 — Subir a aplicação e testar manualmente

```bash
set -a
source .env
set +a
poetry run uvicorn fastapi_skeleton.main:app
```

Acesse `http://localhost:8000/docs`, clique em **Authorize**, informe o `API_KEY` do `.env` e envie o payload de exemplo em `docs/sample_payload.json` contra o endpoint de predição.

---

## 4. Plano de Rollback e Testes de Validação

### Validação

Checklist para confirmar que a aplicação migrada está funcional e sem regressões em relação à versão anterior (Python 3.11 / FastAPI 0.109.1 / Pydantic 2.5.3):

1. **Ambiente correto**
   ```bash
   poetry run python --version
   ```
   Deve reportar Python 3.13.x.

2. **Suíte de testes automatizados**
   ```bash
   ./scripts/test.sh
   ```
   Todos os testes que passavam na linha de base do Passo 1 devem continuar passando, com a mesma cobertura mínima configurada em `tool.coverage.report` (`fail_under = 90`).

3. **Lint e análise estática sem novos erros**
   ```bash
   ./scripts/linting.sh
   ```
   `mypy`, `flake8` e `bandit` não devem reportar problemas novos introduzidos pelas APIs mais recentes do FastAPI/Pydantic.

4. **Heartbeat da API**
   ```bash
   curl -H "Token: $API_KEY" http://localhost:8000/api/heartbeat
   ```
   Deve retornar `200 OK`, confirmando que o app subiu e a autenticação por API key continua funcionando.

5. **Predição funcional e numericamente equivalente**
   ```bash
   curl -X POST http://localhost:8000/api/predict \
     -H "Token: $API_KEY" -H "Content-Type: application/json" \
     -d @docs/sample_payload.json
   ```
   O valor predito deve ser **idêntico (ou compatível dentro de uma tolerância numérica desprezível)** ao valor obtido com a mesma requisição antes da migração — essa comparação é o teste mais importante deste cenário, pois confirma que o re-treino/re-serialização do modelo no Passo 7 não alterou o comportamento do modelo, apenas sua compatibilidade binária.

6. **Documentação interativa (`/docs`) carregando sem erros de schema** — o Swagger UI gerado a partir dos schemas Pydantic deve renderizar normalmente, sem exceções de serialização do OpenAPI.

7. **Ausência de warnings de depreciação no console** durante a subida e o uso da aplicação (`DeprecationWarning`, `InconsistentVersionWarning` do scikit-learn, avisos de `pydantic.v1`).

Somente considerar a migração bem-sucedida quando **todos** os itens acima passarem.

### Rollback

Procedimento para reverter ao estado original (Python 3.11 / FastAPI 0.109.1) em caso de falha durante a validação:

1. **Reverter as alterações de código e de dependências**
   ```bash
   git checkout main
   git branch -D feature/upgrade-python313-fastapi   # opcional, apaga a branch de migração
   ```
   Como todas as alterações (Passos 2–7) foram feitas em uma branch isolada, a branch principal mantém o `pyproject.toml`/`poetry.lock` originais, sem necessidade de reverter diffs manualmente.

2. **Restaurar o ambiente Python original**
   ```bash
   pyenv local 3.11.9
   poetry env use $(pyenv which python)
   poetry install
   ```

3. **Restaurar o artefato do modelo original**, caso o Passo 7 tenha sobrescrito `sample_model/*.joblib`:
   ```bash
   git checkout main -- sample_model/
   ```

4. **Validar que o rollback restaurou o comportamento anterior**
   ```bash
   poetry run python --version   # deve voltar a reportar 3.11.x
   ./scripts/test.sh
   ```

**Critério de decisão para rollback:** acionar este procedimento se, após o Passo 9, qualquer item do checklist de Validação falhar de forma não corrigível em até uma iteração de ajuste nas faixas de versão do `pyproject.toml`, ou se a predição do modelo re-serializado divergir do resultado original além de uma tolerância numérica aceitável e a causa raiz não puder ser isolada em tempo hábil.