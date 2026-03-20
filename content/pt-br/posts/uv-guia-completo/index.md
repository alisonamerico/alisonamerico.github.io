---
title: uv - O Gerenciador de Pacotes Python que Você Precisava
date: 2026-03-20
draft: false
tags: ["python", "uv", "packaging", "poetry", "pip", "fastapi"]
categories: ["python"]
description: "Guia completo sobre o uv, a ferramenta que substitui pip, poetry, pyenv e pipx em um único comando. Aprenda na prática como criar projetos, gerenciar dependências, versões do Python e ferramentas globais com muito mais velocidade."
cover: uv-python-package-manager.png
---

> **Nível:** iniciante a intermediário  
> **Pré-requisito:** conhecimento básico de Python e linha de comando

---

## O Problema que o uv Resolve

Se você já trabalhou com Python por algum tempo, provavelmente se deparou com um cenário parecido com este:

```
# Para começar um projeto "simples", você precisava de:
pip install pipx          # para instalar ferramentas globais com segurança
pipx install poetry       # para gerenciar dependências
poetry self add poetry-plugin-shell  # para ativar o venv pelo poetry
poetry python install 3.12           # para instalar a versão do Python
poetry new meu-projeto    # para criar o projeto
poetry shell              # para ativar o ambiente
poetry add fastapi        # finalmente, instalar o que você queria
```

Antes de escrever uma linha de código, você já precisou instalar **3 ferramentas separadas** (`pip`, `pipx`, `poetry`) e executar **7 comandos**. E isso sem contar o `pyenv` para quem gerencia múltiplas versões do Python, o `pip-tools` para travar dependências, ou o `virtualenv` para criar ambientes virtuais.

O **uv** chegou para resolver isso. Uma única ferramenta, escrita em Rust, que substitui tudo isso com muito mais velocidade.

---

## O que é o uv?

O **uv** é um gerenciador de pacotes e projetos Python desenvolvido pela [Astral](https://astral.sh) — a mesma empresa por trás do [Ruff](https://github.com/astral-sh/ruff), o linter/formatter Python mais rápido do mercado.

Em uma frase: **uv é para Python o que o Cargo é para Rust** — uma ferramenta unificada que cuida de tudo no ciclo de vida do projeto.

### O que ele substitui

| Ferramenta antiga | Função | Equivalente no uv |
|---|---|---|
| `pip` | Instalar pacotes | `uv pip install` |
| `pip-tools` | Travar dependências | `uv pip compile` |
| `virtualenv` / `venv` | Criar ambientes virtuais | `uv venv` |
| `pyenv` | Gerenciar versões do Python | `uv python install` |
| `poetry` | Gerenciar projetos e deps | `uv init` + `uv add` |
| `pipx` | Ferramentas globais isoladas | `uv tool install` / `uvx` |
| `twine` | Publicar pacotes no PyPI | `uv publish` |

---

## Por que é tão rápido?

O uv é escrito em **Rust** e utiliza algumas técnicas que o tornam 10 a 100x mais rápido que o `pip`:

- **Cache global de pacotes**: pacotes baixados uma vez ficam em cache e são reutilizados em todos os projetos via hard links — sem copiar arquivos.
- **Resolução paralela de dependências**: resolve o grafo de dependências em paralelo, não sequencialmente.
- **Sem overhead de Python**: por ser escrito em Rust, não precisa do Python para iniciar — o processo começa em milissegundos.

Para ter uma ideia concreta: instalar as dependências do [Trio](https://trio.readthedocs.io/) leva **~11ms** com cache quente no uv, versus **~5s** no pip.

---

## Instalação

```bash
# macOS e Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# Via pip (se preferir)
pip install uv

# Via Homebrew (macOS)
brew install uv
```

Após instalar, recarregue o shell:

```bash
source ~/.bashrc   # ou ~/.zshrc
```

Verifique a instalação:

```bash
uv --version
# uv 0.5.x (...)
```

---

## Conceitos Fundamentais

Antes de ver os comandos, vale entender dois conceitos centrais do uv:

### 1. pyproject.toml — o coração do projeto

O `pyproject.toml` é o arquivo padrão moderno do Python (PEP 517/518) para configurar projetos. O uv lê e escreve nele automaticamente. É onde ficam registradas suas dependências, versão do Python, scripts, etc.

```toml
[project]
name = "meu-projeto"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi[standard]>=0.115.0",
]

[dependency-groups]
dev = [
    "pytest>=8.0.0",
    "ruff>=0.9.0",
]
```

### 2. uv.lock — o arquivo de lockfile

O `uv.lock` é gerado automaticamente e registra as versões **exatas** de todas as dependências (incluindo as transitivas). Isso garante que todo membro do time ou servidor de CI instale exatamente os mesmos pacotes.

> **Dica:** commit o `uv.lock` no git. Nunca o edite manualmente.

---

## Guia Prático: Do Zero ao Projeto

### Criando um projeto novo

```bash
# Opção 1: cria a pasta e inicializa
uv init meu-projeto
cd meu-projeto

# Opção 2: inicializa na pasta atual
mkdir meu-projeto && cd meu-projeto
uv init
```

Estrutura gerada:

```
meu-projeto/
├── main.py           ← ponto de entrada de exemplo
├── pyproject.toml    ← configuração do projeto
├── README.md
└── .python-version   ← versão do Python fixada
```

### Instalando uma versão específica do Python

```bash
# Listar versões disponíveis
uv python list

# Instalar uma versão
uv python install 3.13

# Fixar a versão no projeto (cria/atualiza .python-version)
uv python pin 3.13
```

### Adicionando dependências

```bash
# Dependência principal
uv add fastapi

# Com extras (equivale a pip install fastapi[standard])
uv add "fastapi[standard]"

# Dependência de desenvolvimento
uv add --dev pytest ruff

# Versão específica
uv add "sqlalchemy>=2.0,<3.0"
```

O uv atualiza o `pyproject.toml` e o `uv.lock` automaticamente.

### Instalando dependências existentes

```bash
# Sincroniza o ambiente com o pyproject.toml / uv.lock
uv sync

# Sem dependências de dev (útil em produção)
uv sync --no-dev
```

### Removendo dependências

```bash
uv remove fastapi
```

### Executando comandos no ambiente virtual

```bash
# Rodar a aplicação
uv run fastapi dev meu_projeto/app.py

# Rodar testes
uv run pytest

# Rodar um script
uv run python script.py

# Abrir o interpretador interativo
uv run python
```

> **Você não precisa ativar o venv manualmente.** O `uv run` cuida disso automaticamente.

Se preferir ativar o venv para uma sessão do terminal:

```bash
source .venv/bin/activate   # Linux/macOS
.venv\Scripts\activate      # Windows
```

---

## Exemplo Completo: Projeto FastAPI

Vamos montar um projeto FastAPI do zero com uv:

```bash
# 1. Criar o projeto
uv init fast-zero
cd fast-zero

# 2. Criar o pacote Python
mkdir fast_zero
touch fast_zero/__init__.py

# 3. Instalar dependências
uv add "fastapi[standard]"
uv add --dev pytest pytest-cov ruff taskipy

# 4. Rodar
uv run fastapi dev fast_zero/app.py
```

Antes de rodar, crie o arquivo `fast_zero/app.py` com o seguinte conteúdo:

```python
from fastapi import FastAPI

app = FastAPI()


@app.get('/')
def read_root():
    return {'message': 'Olá Mundo!'}
```

Configurando o `pyproject.toml`:

```toml
# ─────────────────────────────────────────────
# [project] — metadados e dependências principais
# Informações essenciais do projeto e pacotes
# necessários para rodar em produção.
# ─────────────────────────────────────────────
[project]
name = "fast-zero"          # nome do projeto (kebab-case por convenção)
version = "0.1.0"           # versão atual do projeto
requires-python = ">=3.12"  # versão mínima do Python aceita
dependencies = [
    # fastapi com o grupo [standard], que inclui uvicorn, httpx e outros
    # extras úteis para desenvolvimento e produção
    "fastapi[standard]>=0.115.0",
]

# ─────────────────────────────────────────────
# [dependency-groups] — dependências de desenvolvimento
# Pacotes usados apenas durante o desenvolvimento
# (testes, linting, formatação). Não vão para produção.
# ─────────────────────────────────────────────
[dependency-groups]
dev = [
    "pytest>=8.0.0",      # framework de testes
    "pytest-cov>=6.0.0",  # plugin para medir cobertura de testes
    "ruff>=0.9.0",        # linter e formatador de código
    "taskipy>=1.14.0",    # task runner — atalhos para comandos do projeto
]

# ─────────────────────────────────────────────
# [tool.ruff] — configuração global do Ruff
# Define comportamentos que se aplicam tanto
# ao linter quanto ao formatador.
# ─────────────────────────────────────────────
[tool.ruff]
line-length = 79             # limite de caracteres por linha (padrão PEP 8)
extend-exclude = ['migrations']  # ignora a pasta de migrações (código gerado
                                 # automaticamente — não queremos que o ruff altere)

# ─────────────────────────────────────────────
# [tool.ruff.lint] — configuração do linter
# Define quais regras de boas práticas o ruff
# vai checar no código.
# ─────────────────────────────────────────────
[tool.ruff.lint]
preview = true  # habilita regras ainda em fase experimental
select = [
    'I',   # isort — verifica ordenação alfabética dos imports
    'F',   # pyflakes — detecta erros comuns (variáveis não usadas, imports desnecessários)
    'E',   # pycodestyle errors — erros de estilo de código
    'W',   # pycodestyle warnings — avisos de estilo de código
    'PL',  # pylint — boas práticas gerais de Python
    'PT',  # flake8-pytest — boas práticas específicas para testes com pytest
]

# ─────────────────────────────────────────────
# [tool.ruff.format] — configuração do formatador
# Define o estilo de formatação automática do código.
# ─────────────────────────────────────────────
[tool.ruff.format]
preview = true          # habilita comportamentos de formatação experimentais
quote-style = 'single'  # usa aspas simples em vez de duplas (escolha pessoal/de equipe)

# ─────────────────────────────────────────────
# [tool.pytest.ini_options] — configuração do pytest
# Define como o pytest se comporta ao rodar os testes.
# ─────────────────────────────────────────────
[tool.pytest.ini_options]
pythonpath = "."           # adiciona a raiz do projeto ao PYTHONPATH, permitindo
                           # imports como "from fast_zero.app import app"
addopts = '-p no:warnings' # suprime warnings de bibliotecas externas para
                           # manter a saída dos testes mais limpa

# ─────────────────────────────────────────────
# [tool.taskipy.tasks] — atalhos de comandos
# Define tarefas executáveis com "uv run task <nome>".
# Evita ter que lembrar flags e caminhos longos.
# ─────────────────────────────────────────────
[tool.taskipy.tasks]
lint   = 'ruff check'                        # verifica boas práticas no código
format = 'ruff format'                       # formata o código automaticamente
run    = 'fastapi dev fast_zero/app.py'      # inicia o servidor de desenvolvimento
test   = 'pytest -s -x --cov=fast_zero -vv' # roda os testes com:
                                             #   -s: exibe prints durante os testes
                                             #   -x: para na primeira falha
                                             #   --cov=fast_zero: mede cobertura do pacote
                                             #   -vv: saída detalhada
```

Rodando as tasks:

```bash
uv run task test
uv run task lint
uv run task run
```

---

## Ferramentas Globais (substitui o pipx)

O uv também gerencia ferramentas globais — aquelas que você usa no terminal mas não são parte de nenhum projeto específico.

```bash
# Instalar uma ferramenta globalmente
uv tool install ruff
uv tool install httpie
uv tool install black

# Usar sem instalar (ambiente temporário, equivale a pipx run)
uvx ruff check .
uvx black --check .

# Listar ferramentas instaladas
uv tool list

# Atualizar
uv tool upgrade ruff

# Remover
uv tool uninstall black
```

---

## Scripts com Dependências Inline

Um recurso menos conhecido mas muito útil: o uv pode rodar **scripts Python com dependências declaradas no próprio arquivo**, sem precisar de projeto.

```python
# script.py
# /// script
# requires-python = ">=3.12"
# dependencies = [
#   "httpx",
#   "rich",
# ]
# ///

import httpx
from rich import print

response = httpx.get("https://api.github.com/repos/astral-sh/uv")
print(response.json()["stargazers_count"])
```

```bash
uv run script.py
# Instala as deps automaticamente em um ambiente isolado e executa
```

Isso é excelente para scripts utilitários, automações e compartilhar código sem precisar criar um projeto inteiro.

---

## Tabela de Referência de Comandos

| Comando | O que faz |
|---|---|
| **Projeto** | |
| `uv init meu-projeto` | Cria pasta e inicializa o projeto |
| `uv init` | Inicializa projeto na pasta atual |
| `uv init --lib` | Cria no formato de biblioteca (com `src/`) |
| **Dependências** | |
| `uv add fastapi` | Adiciona dependência principal |
| `uv add "fastapi[standard]"` | Adiciona com extras |
| `uv add --dev pytest` | Adiciona como dep. de desenvolvimento |
| `uv remove fastapi` | Remove dependência |
| `uv sync` | Instala tudo do `pyproject.toml` (= `poetry install`) |
| `uv sync --no-dev` | Instala apenas dependências de produção |
| `uv lock` | Gera/atualiza `uv.lock` sem instalar |
| **Execução** | |
| `uv run <comando>` | Executa no ambiente virtual do projeto |
| `uv run --with httpx script.py` | Executa com pacote extra temporário |
| **Ferramentas globais** | |
| `uvx ruff check .` | Executa ferramenta sem instalar (= `pipx run`) |
| `uv tool install ruff` | Instala ferramenta globalmente (= `pipx install`) |
| `uv tool uninstall ruff` | Remove ferramenta global |
| `uv tool list` | Lista ferramentas instaladas |
| `uv tool upgrade ruff` | Atualiza ferramenta global |
| **Python** | |
| `uv python install 3.13` | Baixa e instala versão do Python |
| `uv python list` | Lista versões disponíveis/instaladas |
| `uv python pin 3.12` | Fixa versão no `.python-version` |
| `uv python uninstall 3.11` | Remove versão instalada |
| **Ambiente virtual e pip** | |
| `uv venv` | Cria `.venv/` na pasta atual |
| `uv venv --python 3.12` | Cria venv com versão específica |
| `uv pip install fastapi` | Instala pacote (interface pip) |
| `uv pip freeze` | Lista pacotes instalados |
| `uv pip compile pyproject.toml -o requirements.txt` | Gera `requirements.txt` travado |

---

## uv vs. Alternativas

### uv vs. pip + venv

| | pip + venv | uv |
|---|---|---|
| Velocidade | Lento | 10–100x mais rápido |
| Lockfile | Não tem (precisa pip-tools) | `uv.lock` nativo |
| Gerencia Python | Não | Sim |
| Ferramenta única | Não (múltiplas) | Sim |
| Curva de aprendizado | Baixa | Baixa |

**Quando usar pip + venv:** projetos legados, ambientes onde uv não pode ser instalado.

---

### uv vs. Poetry

| | Poetry | uv |
|---|---|---|
| Velocidade | Moderado | Muito mais rápido |
| Gerencia Python | Sim (via plugin) | Sim (nativo) |
| Formato do lockfile | `poetry.lock` (próprio) | `uv.lock` (padrão PEP) |
| Publicação no PyPI | Sim | Sim (`uv publish`) |
| Plugin shell | Precisa instalar | `uv run` dispensa |
| Maturidade | Alta (desde 2018) | Crescendo rápido (2024+) |
| Compatibilidade pip | Parcial | Total (interface pip) |

**Quando usar Poetry:** times com muito histórico em Poetry, projetos que dependem de plugins específicos do Poetry.

---

### uv vs. Conda

| | Conda | uv |
|---|---|---|
| Pacotes não-Python | Sim (C, Fortran, CUDA) | Não |
| Velocidade | Lento | Muito mais rápido |
| Uso em data science | Excelente | Adequado |
| Tamanho de instalação | Grande | Pequeno |
| Foco | Ciência de dados / ML | Desenvolvimento Python geral |

**Quando usar Conda:** data science com dependências nativas pesadas (TensorFlow com GPU, CUDA, pacotes geoespaciais).

---

## Vantagens do uv

- **Velocidade absurda**: resolução e instalação em frações de segundo
- **Ferramenta única**: substitui pip, poetry, pyenv, pipx e mais
- **Cache global inteligente**: economiza espaço em disco
- **Lockfile padrão**: compatível com o ecossistema Python moderno
- **Sem dependência circular**: não precisa de Python para funcionar
- **Compatibilidade total com pip**: você pode migrar gradualmente
- **Ativo e bem mantido**: pela Astral, com suporte corporativo

---

## Desvantagens e Limitações

- **Relativamente novo**: surgiu em 2024; menos histórico que pip/poetry
- **Não gerencia pacotes não-Python**: sem suporte a C, CUDA, etc. (domínio do Conda)
- **Mudanças frequentes**: a API ainda evolui rapidamente; alguns comandos mudaram entre versões
- **uv.lock não é compatível com poetry.lock**: migrações exigem conversão
- **Menor ecossistema de plugins**: o Poetry tem plugins maduros que o uv ainda não replica

---

## Migrando de Poetry para uv

Se você tem um projeto em Poetry e quer migrar:

```bash
# 1. Instale o uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. No diretório do projeto, sincronize
# O uv lê o pyproject.toml do Poetry normalmente
uv sync

# 3. Troque os comandos no dia a dia
# poetry run pytest   →   uv run pytest
# poetry add X        →   uv add X
# poetry shell        →   source .venv/bin/activate  (ou só use uv run)

# 4. (Opcional) Gere o uv.lock e delete o poetry.lock
uv lock
git rm poetry.lock
git add uv.lock
```

> O `uv sync` consegue ler o `pyproject.toml` gerado pelo Poetry. A migração costuma ser direta para a maioria dos projetos.

---

## Dicas do Dia a Dia

### Adicionar variável de ambiente ao rodar

```bash
DATABASE_URL=postgresql://localhost/dev uv run python manage.py migrate
```

### Usar com arquivos .env

```bash
# uv não lê .env automaticamente, use python-dotenv ou direnv
uv add --dev python-dotenv
```

### Ver o que está instalado

```bash
uv pip list
uv pip show fastapi
```

### Atualizar todas as dependências

```bash
uv lock --upgrade
uv sync
```

### Exportar requirements.txt (para deploy legado)

```bash
uv pip compile pyproject.toml -o requirements.txt
# ou, para incluir só produção:
uv export --no-dev -o requirements.txt
```

### Executar sem criar projeto (script isolado)

```bash
uv run --with fastapi --with uvicorn python -c "
import uvicorn
from fastapi import FastAPI
app = FastAPI()

@app.get('/')
def root(): return {'ok': True}

uvicorn.run(app)
"
```

---

## Integração com CI/CD

### GitHub Actions

```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          version: "latest"

      - name: Install dependencies
        run: uv sync --frozen

      - name: Run tests
        run: uv run pytest
```

O flag `--frozen` garante que o `uv.lock` não será atualizado durante o CI — se houver divergência, o build falha, o que é o comportamento desejado.

### Docker

```dockerfile
FROM python:3.13-slim

# Instalar uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app

# Copiar lockfile e instalar dependências (camada cacheável)
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

# Copiar código
COPY . .

CMD ["uv", "run", "fastapi", "run", "fast_zero/app.py", "--port", "8000"]
```

---

## Conclusão

O uv é provavelmente a melhora mais significativa na experiência de desenvolvimento Python dos últimos anos. Ele não reinventa a roda — ele pega todas as ferramentas que já usávamos e as unifica em uma interface coerente, muito mais rápida e fácil de usar.

Se você está começando um projeto novo hoje, **use uv**. Se você tem projetos existentes com Poetry ou pip, a migração é simples e vale o esforço.

O ecossistema Python finalmente tem a ferramenta que merecia.

---

## Recursos Adicionais

- [Documentação oficial do uv](https://docs.astral.sh/uv/)
- [Guia de migração Poetry → uv](https://docs.astral.sh/uv/guides/migration/)
- [Integração com FastAPI](https://docs.astral.sh/uv/guides/integration/fastapi/)
- [Repositório no GitHub](https://github.com/astral-sh/uv)
- [Benchmarks de velocidade](https://github.com/astral-sh/uv/blob/main/BENCHMARKS.md)
