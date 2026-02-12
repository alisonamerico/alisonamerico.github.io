---
title: Testando APIs do GitHub com Pytest — Prática
date: 2026-02-12
draft: false
tags: ["python", "testes", "pytest", "github", "mock", "flask"]
categories: ["testing"]
description: "Guia completo e prático para testar APIs do GitHub com Pytest. Domine testes unitários, de integração, mocking, fixtures e testes de performance com exemplos do mundo real. Aprenda padrões profissionais de teste e construa suítes robustas para suas aplicações."
cover: testing-with-pytest-practice.png
---

<!-- ![](testing-with-pytest-practice.png) -->

## Introdução

No [post anterior]({{< relref "posts/post-1-testing-with-pytest-fundamentals-best-practices-and-strategy/index.md" >}}), cobrimos os fundamentos teóricos do Pytest.

Agora vamos aplicar tudo na prática usando um exemplo real que consome a API pública do GitHub:

- Construir um pequeno projeto do mundo real
- Consumir a API pública do GitHub
- Aplicar testes unitários
- Aplicar testes de integração
- Usar fixtures
- Usar mocks
- Usar marcadores
- Aplicar o padrão AAA
- Usar plugins pytest

Este guia é intencionalmente muito detalhado. Cada linha importante será explicada para que até iniciantes possam seguir.


------------------------------------------------------------------------

## Estrutura do Projeto

``` text
github_api_tests/
├── app.py                     # Aplicação Flask
├── services/
│   └── github_service.py      # Camada de serviço da API GitHub
├── tests/
│   ├── conftest.py            # Fixtures compartilhadas
│   ├── test_users_unit.py     # Testes unitários com mocks
│   ├── test_users_integration.py  # Testes de integração (API real)
│   ├── test_parametrize.py    # Testes parametrizados
│   ├── test_skip.py           # Exemplos de marcador skip
│   ├── test_fail.py           # Exemplos de marcador xfail
│   ├── test_errors.py         # Testes de tratamento de erros
│   ├── test_functional.py     # Testes funcionais (rotas Flask)
│   ├── test_performace.py     # Benchmarks de performance
│   └── test_regression.py     # Testes de regressão
├── pytest.ini                 # Configuração do pytest
└── requirements.txt           # Dependências do projeto
```

Organizamos o projeto seguindo as melhores práticas:

- **`app.py`** - Aplicação Flask principal (ponto de entrada)
- **`services/`** - Camada de lógica de negócios (separa responsabilidades)
- **`tests/`** - Todos os arquivos de teste organizados por tipo
- **`pytest.ini`** - Configuração global de testes
- **`requirements.txt`** - Gerenciamento de dependências

Esta arquitetura proporciona:
- Separação clara de responsabilidades
- Fácil manutenção e escalabilidade
- Estrutura padrão da indústria
- Melhor organização de testes

---

## Configuração do Projeto

Antes de executar os testes, precisamos preparar nosso ambiente de desenvolvimento local adequadamente.

---

### Criando um Ambiente Virtual

Um ambiente virtual nos permite isolar as dependências do projeto da instalação global do Python.

Isso evita conflitos de versão entre diferentes projetos e garante reproduibilidade.

```bash
python -m venv .venv
```


#### O que este comando faz:

`python` → Executa o interpretador Python.

`-m venv` → Executa o módulo embutido venv.

`.venv` → Cria uma nova pasta de ambiente virtual chamada `.venv`.

Após executar este comando, um diretório chamado .venv/ será criado contendo:

- Um interpretador Python local

- Seu próprio `pip`

- Um diretório site-packages isolado

Isso garante que qualquer pacote instalado dentro deste ambiente não afetará outros projetos.

---

### Ativando o Ambiente Virtual

Uma vez criado, o ambiente virtual deve ser ativado:

```bash
source .venv/bin/activate
```

#### O que isto faz:

`source` → Executa o script no shell atual.

`.venv/bin/activate` → Ativa o ambiente virtual.

Após ativação:

Seu terminal geralmente muda (ex., `(.venv)` aparece).

`python` e `pip` agora apontam para as versões do ambiente virtual.

Todos os pacotes instalados serão isolados dentro do `.venv`.

---

### Dependências

Todas as dependências do projeto estão listadas em um arquivo `requirements.txt`:

```
flask
requests
pytest
pytest-mock
pytest-cov
pytest-benchmark
```

O que cada dependência faz:

- **flask** → Framework web para construir endpoints de API.
- **requests** → Cliente HTTP usado para interagir com a API do GitHub.
- **pytest** → Framework de testes.
- **pytest-mock** → Fornece integração entre pytest e `unittest.mock`.
- **pytest-cov** → Adiciona relatório de cobertura de testes.
- **pytest-benchmark** → Ferramentas de teste de performance e benchmarking.

---

### Instalando Dependências

Para instalar todos os pacotes necessários:

```bash
pip install -r requirements.txt
```

O que este comando faz:

- `pip` → Instalador de pacotes Python.
- `-r requirements.txt` → Lê a lista de dependências do arquivo.
- Instala todos os pacotes listados dentro do ambiente virtual ativo.

---

### Configuração do Pytest

O projeto inclui um arquivo pytest.ini para configurar o comportamento do pytest globalmente.

`pytest.ini`

```ini
[pytest]
addopts = -ra -q --cov=app --cov-report=term-missing
markers =
    unit: marca testes como testes unitários
    integration: marca testes como testes de integração
    slow: marca testes como testes lentos
    regression: marca testes como testes de regressão
```

---

#### Explicação linha por linha

`[pytest]`

Declara que este arquivo contém configuração para o pytest.

`addopts = -ra -q --cov=app --cov-report=term-missing`

Define opções de linha de comando padrão que sempre serão aplicadas ao executar `pytest`.

Quebrando em partes:

- `-r a` → Mostra um resumo para todos os resultados dos testes (passed, skipped, xfailed, etc.).
- `-q` → Executa pytest em modo silencioso (menos output detalhado).
- `--cov=app` → Mede cobertura de testes para o pacote app.
- `--cov-report=term-missing` → Exibe linhas faltando diretamente no relatório de cobertura do terminal.

Isso garante que toda execução de teste inclua automaticamente análise de cobertura.

`markers =`

Registra marcadores de teste personalizados para evitar avisos como:

```makefile
PytestUnknownMarkWarning: Unknown pytest.mark.integration
```

Cada marcador deve ser declarado explicitamente.

---

`unit`

```ini
unit: marca testes como testes unitários
```

Marca testes como testes unitários.

Estes testes:

- Validam pequenos pedaços de lógica
- Evitam dependências externas
- Executam muito rápido

Você pode executá-los usando:

```bash
pytest -m unit
```

`integration`

```ini
integration: marca testes como testes de integração
```

Marca testes de integração.

Estes testes:

- Validam interação entre componentes
- Podem chamar serviços reais ou APIs externas
- São geralmente mais lentos que testes unitários

Execute apenas testes de integração:

```bash
pytest -m integration
```

`slow`

```ini
slow: marca testes como testes lentos
```

Marca testes mais lentos.

Você pode excluí-los:

```bash
pytest -m "not slow"
```


## Código de Produção

### Aplicação Flask (app.py)

```python
# app.py

from flask import Flask, jsonify
from services.github_service import fetch_users

app = Flask(__name__)

@app.route('/users')
def github_users():
    try:
        users = fetch_users()
        return jsonify(users)
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    app.run(debug=True)
```


### Camada de Serviço (services/github_service.py)

```python
# services/github_service.py

import requests

GITHUB_API_URL = 'https://api.github.com/users'

def fetch_users(per_page=10):
    """
    Busca usuários da API pública do GitHub.
    """
    response = requests.get(
        GITHUB_API_URL, params={'per_page': per_page}, timeout=5
    )
    response.raise_for_status()
    return response.json()
```


### Código de Produção (Explicação Linha por Linha)

#### Aplicação Flask (app.py)

``` python
# app.py

from flask import Flask, jsonify
from services.github_service import fetch_users
```

Nós importamos:
- `Flask` - Framework web para criar nossa API
- `jsonify` - Auxiliar Flask para respostas JSON
- `fetch_users` - Função de serviço que lida com chamadas da API GitHub

------------------------------------------------------------------------

``` python
app = Flask(__name__)
```

Cria a instância da aplicação Flask.

------------------------------------------------------------------------

``` python
@app.route('/users')
def github_users():
    try:
        users = fetch_users()
        return jsonify(users)
    except Exception as e:
        return jsonify({"error": str(e)}), 500
```

Esta é uma rota Flask com tratamento de erros:
- `@app.route('/users')` → Mapeia HTTP GET `/users` para esta função
- `try/except` → Lida com falhas de API elegantemente
- `jsonify(users)` → Retorna resposta JSON para o cliente
- `return jsonify({"error": str(e)}), 500` → Retorna resposta de erro quando a API GitHub falha

------------------------------------------------------------------------

``` python
if __name__ == '__main__':
    app.run(debug=True)
```

Permite executar o aplicativo diretamente com `python app.py`.

#### Camada de Serviço (services/github_service.py)

``` python
# services/github_service.py

import requests

GITHUB_API_URL = 'https://api.github.com/users'
```

- Import requests para chamadas HTTP
- Define constante para URL base da API GitHub

------------------------------------------------------------------------

``` python
def fetch_users(per_page=10):
```

Parâmetro da função:
- `per_page=10` → Valor padrão, permite personalização

------------------------------------------------------------------------

``` python
response = requests.get(
    GITHUB_API_URL, params={'per_page': per_page}, timeout=5
)
```

Detalhes da requisição HTTP:
- `GITHUB_API_URL` → Endpoint base
- `params={'per_page': per_page}` → Parâmetros de query
- `timeout=5` → Previne requisições pendentes

------------------------------------------------------------------------

``` python
response.raise_for_status()
return response.json()
```

- `raise_for_status()` → Levanta exceção para erros HTTP
- `response.json()` → Analisa resposta JSON para lista Python

Esta arquitetura segue o **Padrão de Camada de Serviço**, separando preocupações web da lógica de negócios.

------------------------------------------------------------------------

## Entendendo o Padrão AAA

AAA significa:

-   Arrange → Preparar dados
-   Act → Executar a função
-   Assert → Validar o resultado

Exemplo:

``` python
def test_simple_math():
    # Arrange
    number = 5

    # Act
    result = number + 2

    # Assert
    assert result == 7
```

Esta estrutura melhora a legibilidade e profissionalismo.

------------------------------------------------------------------------
## Fixtures

```python
# tests/conftest.py

import pytest
from app import app

@pytest.fixture
def client():
    """
    Fixture que cria um cliente HTTP de teste para o Flask.
    """
    with app.test_client() as client:
        yield client

@pytest.fixture
def sample_username():
    return 'octocat'
```

Esta fixture fornece dados de teste reutilizáveis.

### Fixtures (Explicação)

``` python
# tests/conftest.py

import pytest
from app import app
```

Nós importamos pytest e o aplicativo Flask.

------------------------------------------------------------------------

#### Fixture de Cliente de Teste Flask

``` python
@pytest.fixture
def client():
    """
    Fixture que cria um cliente HTTP de teste para o Flask.
    """
    with app.test_client() as client:
        yield client
```

Análise linha por linha:
- `@pytest.fixture` → Registra esta função como uma fixture
- `client()` → O nome da função se torna injetável
- `app.test_client()` → Cria cliente de teste Flask
- `with ... as client:` → Gerenciador de contexto para limpeza
- `yield client` → Fornece cliente aos testes, depois limpa

Esta fixture permite testar rotas Flask sem iniciar um servidor real.

------------------------------------------------------------------------

#### Fixture de Dados Amostra

``` python
@pytest.fixture
def sample_username():
    return 'octocat'
```

Análise linha por linha:
- `@pytest.fixture` → Registra como fixture
- `sample_username` → se torna parâmetro injetável
- `return 'octocat'` → Fornece dados de teste

Quando os testes incluem estes parâmetros:
``` python
def test_example(client, sample_username):
```

O Pytest injeta automaticamente ambas as fixtures. Isso é **injeção de dependência**.

------------------------------------------------------------------------

## Teste Unitário com Mock

```python
# tests/test_users_unit.py

from services.github_service import fetch_users

def test_fetch_users_with_pytest_mock(mocker):
    """
    Mesmo teste unitário, usando pytest-mock.
    """
    # Arrange
    fake_users = [{'login': 'pytest-mock'}]

    mocker.patch(
        'services.github_service.requests.get',
        return_value=mocker.Mock(
            json=lambda: fake_users, raise_for_status=lambda: None
        ),
    )

    # Act
    users = fetch_users()

    # Assert
    assert users == fake_users
```

### Teste Unitário (Explicação Linha por Linha)

``` python
# tests/test_users_unit.py

from services.github_service import fetch_users
```

Nós importamos a função de serviço para testá-la em isolamento.

------------------------------------------------------------------------

``` python
def test_fetch_users_with_pytest_mock(mocker):
```

- `mocker` vem do plugin pytest-mock
- Nenhum marcador explícito necessário, mas poderia usar `@pytest.mark.unit`

------------------------------------------------------------------------

#### Seção Arrange

``` python
fake_users = [{'login': 'pytest-mock'}]
```

Cria dados de teste esperados.

------------------------------------------------------------------------

``` python
mocker.patch(
    'services.github_service.requests.get',
    return_value=mocker.Mock(
        json=lambda: fake_users, 
        raise_for_status=lambda: None
    ),
)
```

Esta é a configuração de mocking:
- `mocker.patch()` → Substitui `requests.get` em nosso serviço
- `return_value=mocker.Mock()` → Cria resposta mock
- `json=lambda: fake_users` → Método mock que retorna nossos dados falsos
- `raise_for_status=lambda: None` → Mock que não faz nada (sem exceção)

**Ponto chave**: Nenhuma chamada HTTP real é feita!

------------------------------------------------------------------------

#### Seção Act

``` python
users = fetch_users()
```

Executa a função de serviço, que usa o `requests.get` mockado.

------------------------------------------------------------------------

#### Seção Assert

``` python
assert users == fake_users
```

Verifica que o serviço retorna exatamente o que nosso mock forneceu.

------------------------------------------------------------------------
## Teste de Integração (Chamada Real de API)

```python
# tests/test_users_integration.py

import pytest
from services.github_service import fetch_users

@pytest.mark.integration
@pytest.mark.skip(reason="GitHub API rate limit exceeded - skip for now")
def test_fetch_users_integration():
    """
    Chama API REAL do GitHub
    """
    # Act
    users = fetch_users(5)

    # Assert
    assert isinstance(users, list)
    assert len(users) > 0
    assert 'login' in users[0]
```

Este teste faz uma chamada HTTP real para o GitHub.

### Teste de Integração (Explicação Linha por Linha)

``` python
@pytest.mark.integration
def test_fetch_users_integration():
```

- Marcador `integration` categoriza como teste de integração
- Nenhum marcador `slow` aqui, mas poderia ser adicionado

------------------------------------------------------------------------

``` python
users = fetch_users(5)
```

Isto realiza uma requisição real para a API GitHub, pedindo por 5 usuários.

------------------------------------------------------------------------

``` python
assert isinstance(users, list)
assert len(users) > 0
assert 'login' in users[0]
```

Múltiplas asserções validam:
- Resposta é uma lista
- Lista não está vazia
- Primeiro usuário tem estrutura esperada

**Importante**: Este teste requer conexão com internet e pode ser lento.

------------------------------------------------------------------------
## Parametrizar

```python
# tests/test_parametrize.py

import pytest
from services.github_service import fetch_users

@pytest.mark.parametrize('qty', [1, 3, 5])
def test_fetch_users_parametrize(qty, mocker):
    """
    Mesmo teste com múltiplos valores.
    Usando requests mockados para evitar rate limiting.
    """
    # Arrange - mockar resposta para evitar rate limiting
    fake_users = [{'login': f'user{i}'} for i in range(qty)]
    
    mocker.patch(
        'services.github_service.requests.get',
        return_value=mocker.Mock(
            json=lambda: fake_users, 
            raise_for_status=lambda: None
        ),
    )
    
    # Act
    users = fetch_users(qty)
    
    # Assert
    assert len(users) <= qty
    assert len(users) == qty  # Deve corresponder exatamente com nosso mock
```

O Pytest executa este teste três vezes com diferentes quantidades.

## Parametrizar (Detalhado)

``` python
@pytest.mark.parametrize('qty', [1, 3, 5])
```

O Pytest irá:
- Executar o teste com `qty = 1`
- Executar com `qty = 3`
- Executar com `qty = 5`

------------------------------------------------------------------------

``` python
def test_fetch_users_parametrize(qty, mocker):
    fake_users = [{'login': f'user{i}'} for i in range(qty)]
    mocker.patch('services.github_service.requests.get', ...)
    users = fetch_users(qty)
    assert len(users) == qty
```

Os parâmetros são automaticamente injetados na função de teste.

Esta abordagem:
- Elimina duplicação de código
- Testa casos extremos (1, 3, 5 usuários)
- Usa mocking para evitar rate limiting de API
- Torna testes mais mantíveis

**Dica profissional**: Você também pode combinar múltiplos parâmetros:
```python
@pytest.mark.parametrize('qty,expected', [(1, 1), (5, 5), (100, 100)])
```


------------------------------------------------------------------------

## Testando Erros

```python
# tests/test_errors.py

import pytest
import requests
from services.github_service import fetch_users

def test_timeout_error(mocker):
    """
    Testa como o serviço lida com timeouts.
    """
    # Arrange
    mocker.patch(
        'services.github_service.requests.get',
        side_effect=requests.Timeout("Request timed out")
    )

    # Act & Assert
    with pytest.raises(requests.Timeout):
        fetch_users()

def test_http_error_handling(mocker):
    """
    Testa tratamento de erros HTTP.
    """
    # Arrange
    mocker.patch(
        'services.github_service.requests.get',
        side_effect=requests.HTTPError("404 Not Found")
    )

    # Act & Assert
    with pytest.raises(requests.HTTPError):
        fetch_users()
```

Isso garante que erros são tratados corretamente.

### Testando Erros (Explicação Linha por Linha)

#### Teste de Erro de Timeout

``` python
mocker.patch(
    'services.github_service.requests.get',
    side_effect=requests.Timeout("Request timed out")
)
```

- `side_effect` → Em vez de retornar, levanta exceção
- `requests.Timeout` → Exceção específica de timeout
- Testa como o serviço lida com timeouts de rede

------------------------------------------------------------------------

``` python
with pytest.raises(requests.Timeout):
    fetch_users()
```

- Afirma que `fetch_users()` levanta `Timeout`
- Teste passa se a exceção correta for levantada
- Teste falha se não houver exceção ou exceção errada

#### Teste de Erro HTTP

``` python
side_effect=requests.HTTPError("404 Not Found")
```

Simula erros HTTP como 404, 500, etc.

**Benefícios chave**:
- Testa tratamento de erros sem falhas reais
- Garante propagação adequada de exceções
- Verifica robustez do serviço

------------------------------------------------------------------------

## Skip vs Xfail

```python
# tests/test_skip.py

import pytest

@pytest.mark.skip(reason='Feature em desenvolvimento')
def test_future_feature():
    assert True
```

```python
# tests/test_fail.py

import pytest
from services.github_service import fetch_users

@pytest.mark.xfail(reason='Bug conhecido')
def test_known_bug():
    assert False

@pytest.mark.xfail(reason='Bug conhecido quando per_page > 100')
def test_expected_failure():
    fetch_users(200)
```

### Skip vs Xfail (Detalhado)

#### Marcador Skip

``` python
@pytest.mark.skip(reason='Feature em desenvolvimento')
```

- **Teste não executa de todo**
- Mostra como "skipped" no relatório de testes
- Útil para funcionalidades incompletas
- Economiza tempo de execução

#### Marcador Xfail

``` python
@pytest.mark.xfail(reason='Bug conhecido')
```

- **Teste executa mas falha é esperada**
- Mostra como "xfail" (falha esperada) se falhar
- Mostra como "xpass" (passado inesperado) se passar
- Útil para bugs conhecidos ou casos extremos

**Quando usar cada**:

- `@pytest.mark.skip`: Funcionalidade não pronta, problemas de ambiente
- `@pytest.mark.xfail`: Limitações conhecidas, falhas esperadas

**Pulando condicional**:
```python
@pytest.mark.skipif(sys.version_info < (3, 8), reason="Requer Python 3.8+")
def test_python_38_feature():
    pass
```

------------------------------------------------------------------------

## Testes Funcionais (Rotas Flask)

```python
# tests/test_functional.py

from http import HTTPStatus

def test_users_page(client):
    """
    Teste funcional:
    - Simula acesso à rota Flask
    """
    # Act
    response = client.get('/users')

    # Assert - verificar por resposta bem-sucedida ou erro de rate limit
    assert response.status_code in [HTTPStatus.OK, HTTPStatus.INTERNAL_SERVER_ERROR]
    
    # Verificar se resposta é JSON válido
    try:
        data = json.loads(response.data)
        if response.status_code == HTTPStatus.OK:
            assert isinstance(data, list)
        else:
            # Deve ser uma resposta de erro
            assert isinstance(data, dict)
            assert 'error' in data
    except json.JSONDecodeError:
        assert False, "Resposta não é JSON válido"
```

### Testes Funcionais (Explicação Linha por Linha)

``` python
def test_users_page(client):
```

- `client` é a fixture de cliente de teste Flask
- Testa rota web ponta a ponta
- Lida com ambos cenários de sucesso e erro

------------------------------------------------------------------------

``` python
response = client.get('/users')
```

- Simula requisição HTTP GET para `/users`
- Nenhum servidor real necessário (cliente de teste)

------------------------------------------------------------------------

``` python
assert response.status_code in [HTTPStatus.OK, HTTPStatus.INTERNAL_SERVER_ERROR]
```

- Aceita tanto respostas de sucesso (200) quanto de erro (500)
- 500 ocorre quando a API GitHub limita a requisição

------------------------------------------------------------------------

``` python
data = json.loads(response.data)
if response.status_code == HTTPStatus.OK:
    assert isinstance(data, list)
else:
    assert isinstance(data, dict)
    assert 'error' in data
```

- Valida que a resposta é sempre JSON válido
- Sucesso: deve ser uma lista de usuários
- Erro: deve ser um dict com chave 'error'

## Testes de Performance

```python
# tests/test_performace.py

def test_users_endpoint_performance(benchmark, client):
    """
    Mede tempo de resposta da rota /users.
    """
    benchmark(lambda: client.get('/users'))
```

### Teste de Performance (Explicação Linha por Linha)

``` python
def test_users_endpoint_performance(benchmark, client):
```

- `benchmark` é a fixture do pytest-benchmark
- `client` é a fixture para testes Flask

------------------------------------------------------------------------

``` python
benchmark(lambda: client.get('/users'))
```

- Mede tempo de execução do lambda
- Fornece métricas detalhadas de performance
- Estabelece linha base para comparações futuras

**Resultados de benchmark atuais do projeto**:
```
------------------------------------------------------- benchmark: 1 tests ------------------------------------------------------
Name (time in ms)                        Min       Max      Mean  StdDev    Median      IQR  Outliers     OPS  Rounds  Iterations
---------------------------------------------------------------------------------------------------------------------------------
test_users_endpoint_performance     246.0832  264.0037  254.0807  8.1997  252.4519  15.2874       1;0  3.9358       5           1
---------------------------------------------------------------------------------------------------------------------------------

Legend:
  Outliers: 1 Standard Deviation from Mean; 1.5 IQR (InterQuartile Range) from 1st Quartile and 3rd Quartile.
  OPS: Operations Per Second, computed as 1 / Mean
```

**Métricas de benchmark explicadas**:
- **Min**: Tempo de execução mais rápido observado
- **Max**: Tempo de execução mais lento observado
- **Mean**: Tempo de execução médio através de todas as rodadas
- **StdDev**: Desvio padrão (medida de consistência)
- **Median**: Valor médio quando ordenado (menos afetado por outliers)
- **IQR**: Range interquartil (50% médio dos valores)
- **Outliers**: Valores que desviam significativamente da norma
- **OPS**: Operações por segundo (quantas vezes poderia executar em um segundo)
- **Rounds**: Número de iterações de teste realizadas
- **Iterations**: Número de execuções por rodada

**Execute com**: `pytest --benchmark-only`

## Testes de Regressão

```python
# tests/test_regression.py

from http import HTTPStatus
import pytest

@pytest.mark.regression
def test_users_endpoint_handles_gracefully(client):
    """
    Testa que a rota lida com erros elegantemente.
    Mesmo se a API externa falhar, a rota não deve travar.
    """
    # Act
    response = client.get('/users')

    # Deve retornar resposta JSON adequada mesmo quando a API GitHub falhar
    assert response.status_code in [HTTPStatus.OK, HTTPStatus.INTERNAL_SERVER_ERROR]
    
    # Resposta deve ser sempre JSON válido
    import json
    try:
        data = json.loads(response.data)
        assert isinstance(data, (list, dict))
    except json.JSONDecodeError:
        assert False, "Resposta não é JSON válido"
```

### Teste de Regressão (Explicação Linha por Linha)

``` python
@pytest.mark.regression
```

- Marcador personalizado para testes de regressão
- Pode ser executado com: `pytest -m regression`

------------------------------------------------------------------------

``` python
assert response.status_code in [HTTPStatus.OK, HTTPStatus.INTERNAL_SERVER_ERROR]
```

- Garante que o endpoint sempre retorna respostas adequadas
- Mesmo quando a API externa falha, retorna JSON estruturado
- Previne travamentos e exceções não tratadas

**Execute testes de regressão apenas**: `pytest -m regression`

## Cobertura

```bash
pytest --cov=app --cov-report=term-missing
```

Isto:

- Rastreia linhas executadas
- Mostra linhas faltando no relatório de cobertura
- Integrado ao pytest.ini para cobertura automática

**Relatório de cobertura atual do projeto**:
```
==================== tests coverage ====================
Name     Stmts   Miss  Cover   Missing
--------------------------------------
app.py      12      2    83%   10, 15
--------------------------------------
TOTAL       12      2    83%
```

**Métricas de cobertura explicadas**:
- **Name**: Nome do arquivo sendo medido
- **Stmts**: Total de declarações (linhas de código) no arquivo
- **Miss**: Número de declarações não executadas pelos testes
- **Cover**: Percentual de código coberto pelos testes
- **Missing**: Números das linhas específicas não cobertas

⚠️ **Importante**: Alta cobertura não significa testes de alta qualidade. Foque em testar comportamento, não apenas linhas.

------------------------------------------------------------------------

## Executando Diferentes Tipos de Teste

```bash
# Executar apenas testes unitários
pytest -m unit

# Executar apenas testes de integração
pytest -m integration

# Executar testes de regressão
pytest -m regression

# Pular testes lentos
pytest -m "not slow"

# Executar benchmarks de performance
pytest --benchmark-only

# Executar com cobertura (configurado em pytest.ini)
pytest

# Executar com output detalhado
pytest -v

# Executar arquivo de teste específico
pytest tests/test_users_unit.py
```

## Resumo das Melhores Práticas

1. **Organização de Testes**
   - Separar código de produção de testes
   - Usar nomes de testes descritivos
   - Agrupar testes relacionados em arquivos

2. **Padrão AAA**
   - Arrange: Preparar dados de teste e mocks
   - Act: Executar a função sob teste
   - Assert: Verificar resultado

3. **Estratégia de Mocking**
   - Mockar dependências externas em testes unitários
   - Usar chamadas reais em testes de integração
   - Mockar apenas o que você precisa (métodos específicos)

4. **Uso de Fixtures**
   - Reutilizar configuração de teste comum
   - Manter fixtures focadas e simples
   - Usar `yield` para operações de limpeza

5. **Categorias de Teste**
   - Unit: Rápidos, isolados, lógica de negócios
   - Integration: Chamadas externas reais
   - Functional: Cenários completos do usuário
   - Performance: Benchmark de caminhos críticos
   - Regression: Prevenir recorrência de bugs

## Dicas Avançadas

### Marcadores Personalizados
```python
# pytest.ini
markers =
    unit: marca testes como testes unitários
    integration: marca testes como testes de integração
    regression: marca testes como testes de regressão
    slow: marca testes como testes lentos
    network: marca testes requerendo internet
```

### Configuração de Teste
```python
# conftest.py
@pytest.fixture(scope="session")
def api_client():
    """Cliente compartilhado entre todos os testes"""
    return SomeApiClient()

@pytest.fixture(autouse=True)
def setup_test_environment():
    """Fixture auto-usada para todos os testes"""
    # Código de configuração aqui
    yield
    # Código de limpeza aqui
```

## Conclusão

Você agora entende:

- **Organização de Testes**: Estrutura adequada e separação
- **Testes Unitários**: Testes isolados com mocks
- **Testes de Integração**: Chamadas reais de API
- **Testes Funcionais**: Cenários ponta a ponta
- **Testes de Performance**: Benchmarking com pytest-benchmark
- **Testes de Regressão**: Prevenindo recorrência de bugs
- **Estratégia de Mocking**: Quando e como mockar
- **Fixtures**: Componentes de teste reutilizáveis
- **Marcadores**: Categorizando e filtrando testes
- **Parametrizar**: Reduzindo duplicação de testes
- **Padrão AAA**: Estrutura clara de teste
- **Cobertura**: Medindo completude de testes

Isto é **arquitetura de teste de nível profissional** que escalará com seu projeto.

### Próximos Passos

1. Adicionar mais testes de casos extremos
2. Implementar fábricas de dados de teste
3. Configurar CI/CD com testes automatizados
4. Explorar teste baseado em propriedades com hypothesis
5. Considerar teste de contrato para APIs

Projeto completo no **[Github](https://github.com/alisonamerico/github_api_tests)**