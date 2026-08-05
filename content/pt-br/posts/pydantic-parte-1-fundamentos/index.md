---
title: "Pydantic na Prática (Parte 1) - Fundamentos e Validação de Dados em Python"
date: 2026-08-05
draft: false
tags: ["python", "pydantic", "validação-de-dados", "backend", "fastapi"]
categories: ["Python"]
description: "Uma introdução ao Pydantic V2, abordando seus principais conceitos, instalação com uv, modelos básicos, validação automática de tipos e um exemplo prático de validação de payload de API."
series: ["Pydantic na Prática"]
cover: pydantic-fundamentals.png
---

## Introdução

Se você desenvolve APIs, consome dados externos ou já perdeu tempo debugando um `TypeError` que só apareceu em produção porque um campo chegou como string em vez de número, este post é para você. Vamos falar sobre **Pydantic**, uma das bibliotecas mais usadas do ecossistema Python moderno - e que provavelmente já está rodando por baixo dos panos em algum projeto seu, mesmo que você nunca a tenha importado diretamente (o FastAPI, por exemplo, é construído em cima dela).

Esta é a **Parte 1** de uma série de dois posts. Aqui vamos cobrir os fundamentos: o que é o Pydantic, como instalar, como criar seus primeiros modelos e como validar dados na prática, com um caso de uso real ao final. Na **Parte 2**, vamos avançar para tópicos intermediários/avançados: validators customizados, `model_config`, tipos aninhados, `Annotated`, Pydantic Settings, performance e mais.

## Quem é o autor do Pydantic?

O Pydantic foi criado por **Samuel Colvin**, engenheiro britânico que começou o projeto em 2017 para resolver um problema recorrente no próprio dia a dia: validar dados de entrada em aplicações Python sem escrever uma montanha de código repetitivo de `if`/`raise`. Colvin fundou posteriormente a **Pydantic Services Inc.**, empresa que hoje mantém o projeto e também desenvolve ferramentas relacionadas, como o **Pydantic AI** (framework para construção de agentes de IA) e o **Pydantic Logfire** (observabilidade). O projeto é open source, mantido por uma equipe e uma comunidade ativa no GitHub, e é usado por empresas como Amazon, Microsoft, NASA, NVIDIA, Netflix e a própria Anthropic.

Hoje o Pydantic está na sua segunda geração de arquitetura: a partir da versão 2.0, o núcleo de validação foi reescrito em **Rust** (o pacote `pydantic-core`), o que trouxe ganhos de performance expressivos em relação à V1 - a documentação oficial cita a validação como uma das mais rápidas entre as bibliotecas Python do gênero. Neste post estamos usando a versão mais recente da série 2.x.

## O que é o Pydantic e por que usá-lo?

Pydantic é uma **biblioteca de validação e modelagem de dados** para Python. Na prática, você declara a "forma" dos seus dados usando **type hints** nativos do Python (`str`, `int`, `list[str]`, `datetime`, etc.), e o Pydantic cuida de três coisas para você:

1. **Validação** - garante que os dados recebidos realmente correspondem aos tipos declarados, e rejeita (com um erro claro) o que não corresponde.
2. **Conversão (coerção)** - quando possível e seguro, converte tipos compatíveis automaticamente (por exemplo, a string `"25"` vira o inteiro `25`).
3. **Serialização** - transforma os objetos Python de volta em dicionários ou JSON, prontos para trafegar em uma API, ser salvos em banco ou logados.

A grande sacada do Pydantic é que ele usa a sintaxe de tipagem que você já escreve (ou deveria escrever) em Python moderno. Não é uma DSL nova para aprender - é `class Usuario(BaseModel)` com anotações de tipo, e o resto é automático.

### Que problema o Pydantic resolve?

Todo backend que recebe dados de fora (requisição HTTP, mensagem de fila, variável de ambiente, arquivo de configuração, resposta de outra API) enfrenta o mesmo problema: **você não pode confiar cegamente no formato dessas informações**. Sem uma camada de validação, é comum escrever código assim:

```python
def criar_usuario(dados: dict):
    if "nome" not in dados or not isinstance(dados["nome"], str):
        raise ValueError("nome inválido")
    if "idade" not in dados or not isinstance(dados["idade"], int):
        raise ValueError("idade inválida")
    # ... e assim por diante, para cada campo
```

Esse código cresce, fica repetitivo, é fácil esquecer uma verificação e difícil de manter conforme o modelo de dados evolui. O Pydantic resolve isso substituindo essas verificações manuais por uma **declaração única e tipada**, validada automaticamente toda vez que um objeto é criado. O resultado é menos código boilerplate, mensagens de erro padronizadas e muito mais confiança em qualquer fronteira onde dados "não confiáveis" entram no seu sistema.

### Quando usar o Pydantic?

- Validar o corpo (body) de requisições em APIs (é literalmente o motor de validação do FastAPI).
- Validar e tipar variáveis de ambiente e arquivos de configuração (via `pydantic-settings`).
- Garantir a integridade de mensagens recebidas de filas (RabbitMQ, Celery, Kafka).
- Modelar dados de domínio de forma explícita, evitando o uso de `dict` "solto" pelo código.
- Serializar/desserializar dados para logs estruturados, cache (Redis) ou respostas de API.

## Instalação e configuração inicial (com uv)

Vamos usar o **uv** para o setup, já que é o gerenciador de pacotes/ambientes mais rápido e prático do ecossistema Python hoje.

```bash
# Cria um novo projeto (se ainda não tiver um)
uv init pydantic-lab
cd pydantic-lab

# Adiciona o Pydantic como dependência do projeto
uv add pydantic

# (opcional) Adiciona o pydantic-settings, usado para configs via env vars
uv add pydantic-settings
```

O `uv add` já resolve a versão, atualiza o `pyproject.toml` e cria/atualiza o `uv.lock`, sem você precisar gerenciar `venv` manualmente. Para rodar um script do projeto, use:

```bash
uv run python meu_script.py
```

Se preferir o fluxo tradicional com `pip`, também funciona normalmente:

```bash
pip install pydantic
```

Para conferir a versão instalada:

```bash
uv run python -c "import pydantic; print(pydantic.VERSION)"
```

## Introdução aos modelos básicos (`BaseModel`)

O ponto de partida de praticamente tudo no Pydantic é a classe `BaseModel`. Você a herda para criar seus próprios modelos, declarando os campos como atributos de classe com type hints:

```python
from pydantic import BaseModel


class Usuario(BaseModel):
    nome: str
    idade: int
    email: str
    ativo: bool = True  # campo com valor padrão -> não é obrigatório
```

Alguns pontos importantes já nesse exemplo simples:

- `nome`, `idade` e `email` são **obrigatórios**, pois não têm valor padrão.
- `ativo` é **opcional**, porque já tem um valor padrão (`True`). Se não for informado na criação, assume esse valor.
- A ordem dos campos não importa para a criação (o Pydantic aceita `**kwargs`), mas importa para leitura/documentação do código.

Criando uma instância:

```python
usuario = Usuario(nome="Alison", idade=34, email="alison@example.com")
print(usuario)
#> nome='Alison' idade=34 email='alison@example.com' ativo=True
```

## Validação automática de tipos com Pydantic

A cada vez que você instancia um modelo, o Pydantic roda a validação de todos os campos automaticamente - sem você precisar chamar nenhum método explícito para isso. Vamos ver isso na prática:

```python
from pydantic import BaseModel, ValidationError


class Usuario(BaseModel):
    nome: str
    idade: int
    email: str


# Dados corretos
usuario_ok = Usuario(nome="Maria", idade=30, email="maria@example.com")
print(usuario_ok)
#> nome='Maria' idade=30 email='maria@example.com'

# Dados incorretos
try:
    usuario_invalido = Usuario(nome="João", idade="trinta", email="joao@example.com")
except ValidationError as e:
    print(e)
```

Saída do erro (resumida):

```text
1 validation error for Usuario
idade
  Input should be a valid integer, unable to parse string as an integer
  [type=int_parsing, input_value='trinta', input_type=str]
```

Repare em dois detalhes importantes:

1. O Pydantic levanta **`ValidationError`** (não um `ValueError` genérico) - uma exceção própria que carrega uma lista estruturada de todos os erros encontrados, campo a campo.
2. A mensagem já indica exatamente **qual campo falhou**, **por quê** e **qual foi o valor recebido** - ótimo tanto para debugging quanto para devolver como resposta de erro em uma API.

## Trabalhando com dados válidos e inválidos

### Coerção de tipos (dados "válidos" com conversão)

Por padrão (modo "lax"), o Pydantic tenta converter tipos compatíveis automaticamente, em vez de rejeitar de cara:

```python
usuario = Usuario(nome="Ana", idade="25", email="ana@example.com")
print(usuario)
#> nome='Ana' idade=25 email='ana@example.com'
print(type(usuario.idade))
#> <class 'int'>
```

Mesmo recebendo `"25"` como string, o campo `idade` é convertido para `int`. Isso é extremamente útil quando os dados vêm de fontes como formulários HTML, query strings ou variáveis de ambiente, onde tudo naturalmente chega como texto.

### Erros de validação (dados realmente inválidos)

Quando a conversão não é possível ou segura, o Pydantic recusa o dado e lista todos os problemas de uma vez:

```python
from pydantic import BaseModel, ValidationError


class Usuario(BaseModel):
    nome: str
    idade: int
    email: str


try:
    usuario = Usuario(nome=123, idade=25.5, email=None)
except ValidationError as e:
    print(e)
```

```text
3 validation errors for Usuario
nome
  Input should be a valid string [type=string_type, input_value=123, input_type=int]
idade
  Input should be a valid integer, got a number with a fractional part [type=int_from_float, input_value=25.5, input_type=float]
email
  Input should be a valid string [type=string_type, input_value=None, input_type=NoneType]
```

Note que o Pydantic **não para no primeiro erro** - ele coleta todos os erros de validação do modelo de uma vez, o que é bem melhor do que corrigir um campo por vez em ciclos de tentativa e erro.

### Modo estrito (`strict`)

Se você não quer nenhuma coerção automática - por exemplo, para não aceitar `"25"` como `idade` - o Pydantic também oferece um **modo estrito**, tanto por campo quanto por modelo inteiro. Isso é útil quando os dados já vêm de uma fonte 100% confiável (como outro serviço interno que também usa Pydantic) e qualquer divergência de tipo deve ser tratada como bug, não como algo a "consertar" silenciosamente.

## Serialização e desserialização de dados JSON

Essa é, na prática, uma das funcionalidades mais usadas no dia a dia de quem constrói APIs.

### Serialização (modelo → JSON)

```python
usuario = Usuario(nome="Carlos", idade=40, email="carlos@example.com")

# Para dicionário Python
print(usuario.model_dump())
#> {'nome': 'Carlos', 'idade': 40, 'email': 'carlos@example.com'}

# Direto para uma string JSON
print(usuario.model_dump_json())
#> {"nome":"Carlos","idade":40,"email":"carlos@example.com"}
```

`model_dump()` retorna um `dict` Python; `model_dump_json()` já retorna a string JSON pronta para a resposta HTTP.

### Desserialização (JSON → modelo)

```python
dados_json = '{"nome": "Luiza", "idade": 22, "email": "luiza@example.com"}'

usuario = Usuario.model_validate_json(dados_json)
print(usuario)
#> nome='Luiza' idade=22 email='luiza@example.com'
```

Se preferir partir de um `dict` já em memória (por exemplo, o corpo de uma requisição já parseado), use `model_validate()`:

```python
dados_dict = {"nome": "Pedro", "idade": 28, "email": "pedro@example.com"}
usuario = Usuario.model_validate(dados_dict)
```

> 💡 Nas versões antigas do Pydantic (V1) você provavelmente viu `parse_raw()` e `.dict()`. Esses métodos ainda existem por compatibilidade, mas estão **deprecados** - na V2, o padrão é `model_validate_json()`, `model_validate()`, `model_dump()` e `model_dump_json()`.

## Um pouco mais: `Field`, tipos opcionais e valores padrão

Antes de partir para o caso prático, vale conhecer o `Field`, usado para adicionar restrições e metadados aos campos:

```python
from typing import Optional
from pydantic import BaseModel, Field


class Produto(BaseModel):
    nome: str = Field(min_length=2, max_length=100)
    preco: float = Field(gt=0, description="Preço em reais, deve ser positivo")
    estoque: int = Field(default=0, ge=0)
    descricao: Optional[str] = None
```

- `min_length` / `max_length`: restrições de tamanho para strings.
- `gt` / `ge` / `lt` / `le`: restrições numéricas ("greater than", "greater or equal", etc.).
- `default`: valor padrão explícito (alternativa a `= valor` direto no atributo).
- `Optional[str] = None`: campo que aceita `None` e não é obrigatório.

## Caso prático (cenário real): validando o payload de um webhook

Um cenário extremamente comum no dia a dia de backend: você expõe um endpoint que recebe **webhooks** de um serviço externo (gateway de pagamento, CI/CD, sistema de terceiros) e precisa validar o payload antes de processar qualquer coisa. Vamos montar esse cenário com FastAPI + Pydantic, validando o evento de "pedido criado" de um sistema de e-commerce:

```python
from datetime import datetime
from typing import Literal

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, EmailStr, Field, ValidationError


class ItemPedido(BaseModel):
    sku: str = Field(min_length=1)
    quantidade: int = Field(gt=0)
    preco_unitario: float = Field(gt=0)


class WebhookPedidoCriado(BaseModel):
    evento: Literal["pedido.criado"]
    pedido_id: str
    cliente_email: EmailStr
    itens: list[ItemPedido]
    criado_em: datetime
    valor_total: float = Field(gt=0)


app = FastAPI()


@app.post("/webhooks/pedidos")
def receber_webhook(payload: WebhookPedidoCriado):
    # Se chegou até aqui, o Pydantic já validou:
    # - o tipo de cada campo
    # - que "evento" é exatamente "pedido.criado"
    # - que o email é um email válido (EmailStr)
    # - que cada item tem quantidade e preço positivos
    # - que a data está em formato ISO 8601 válido
    total_calculado = sum(item.quantidade * item.preco_unitario for item in payload.itens)

    if abs(total_calculado - payload.valor_total) > 0.01:
        raise HTTPException(status_code=422, detail="valor_total não confere com os itens")

    return {"status": "processado", "pedido_id": payload.pedido_id}
```

O que ganhamos aqui, na prática:

- **Zero validação manual** de tipo, formato de e-mail ou de data no corpo da função - o FastAPI já rejeita automaticamente (com `422 Unprocessable Entity`) qualquer payload que não bata com `WebhookPedidoCriado`, antes mesmo de `receber_webhook` ser chamada.
- **`Literal["pedido.criado"]`** garante que só esse tipo específico de evento seja aceito nesse endpoint - útil quando o mesmo webhook pode enviar eventos diferentes.
- **`EmailStr`** valida o formato de e-mail automaticamente (requer o pacote extra `email-validator`, instalável com `uv add pydantic[email]`).
- A regra de negócio (**conferir se o total bate com a soma dos itens**) fica isolada, depois que já temos a garantia de que os tipos estão corretos - sem misturar validação estrutural com regra de negócio.

Testando localmente com `curl`:

```bash
curl -X POST http://localhost:8000/webhooks/pedidos \
  -H "Content-Type: application/json" \
  -d '{
    "evento": "pedido.criado",
    "pedido_id": "PED-001",
    "cliente_email": "cliente@example.com",
    "itens": [{"sku": "ABC123", "quantidade": 2, "preco_unitario": 49.9}],
    "criado_em": "2026-08-04T10:00:00",
    "valor_total": 99.8
  }'
```

## Conclusão

Nesta primeira parte você viu:

- Quem criou o Pydantic e por que ele existe.
- Que problema real ele resolve (validação de dados de fontes não confiáveis).
- Como instalar e configurar o projeto usando `uv`.
- Como criar modelos básicos com `BaseModel`.
- Como funciona a validação e a coerção automática de tipos.
- Como lidar com dados válidos e inválidos, incluindo a leitura de um `ValidationError`.
- Como serializar e desserializar JSON com `model_dump_json()` e `model_validate_json()`.
- Um caso prático completo validando um webhook real com FastAPI.

Na **Parte 2** desta série vamos aprofundar em tópicos intermediários e avançados: validators customizados (`field_validator` e `model_validator`), `model_config`, modelos aninhados e recursivos, `Annotated` com validação customizada, `pydantic-settings` para configuração via variáveis de ambiente, e algumas notas sobre performance - sempre com outro caso prático real ao final.

## Links

- [Documentação oficial do Pydantic - Get Started](https://pydantic.dev/docs/validation/latest/get-started/)
- [Repositório oficial no GitHub](https://github.com/pydantic/pydantic)
- [Documentação do FastAPI (usa Pydantic internamente)](https://fastapi.tiangolo.com/)
- [uv - gerenciador de pacotes Python](https://docs.astral.sh/uv/)
