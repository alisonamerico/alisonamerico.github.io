---
title: "Pydantic na Prática (Parte 2) - Tópicos Intermediários, Avançados e um Projeto Real"
date: 2026-08-12
draft: false
tags: ["python", "pydantic", "validação-de-dados", "backend", "fastapi", "rabbitmq", "celery"]
categories: ["Python"]
description: "Explore validators customizados, model_config, modelos aninhados, Annotated, computed_field, discriminated unions, aliases, Generics e Pydantic Settings por meio de um projeto prático de processamento de pedidos baseado em filas."
series: ["Pydantic na Prática"]
cover: pydantic-advanced.png
---

## Recapitulando a Parte 1

Na [Parte 1]({{< relref "posts/pydantic-parte-1-fundamentos/index.md" >}}) desta série vimos os fundamentos: o que é o Pydantic, quem o criou, instalação com `uv`, criação de modelos com `BaseModel`, validação automática de tipos, a diferença entre dados válidos/inválidos e serialização/desserialização de JSON. Se você ainda não leu, vale a pena começar por lá - este post assume que você já sabe criar um modelo simples e entende o que é um `ValidationError`.

Aqui vamos avançar para o que realmente diferencia um uso "básico" de um uso "profissional" do Pydantic em produção: regras de validação customizadas, configuração fina do comportamento dos modelos, composição de modelos complexos, tipos reutilizáveis, e gerenciamento de configuração de aplicação. Fechamos com um **projeto prático completo**, simulando um serviço real de processamento de pedidos consumidos de uma fila.

## Validators customizados: `field_validator` e `model_validator`

Nem toda regra de negócio cabe em um `Field(gt=0)`. Às vezes você precisa de lógica customizada - validar um CPF, normalizar um texto, ou comparar dois campos entre si.

### `field_validator`: validando (ou transformando) um campo específico

```python
from pydantic import BaseModel, field_validator


class Cliente(BaseModel):
    nome: str
    cpf: str

    @field_validator("cpf")
    @classmethod
    def validar_cpf(cls, valor: str) -> str:
        digitos = "".join(filter(str.isdigit, valor))
        if len(digitos) != 11:
            raise ValueError("CPF deve conter 11 dígitos")
        return digitos  # normaliza, removendo pontuação
```

Repare que o validator, além de validar, também **normaliza** o dado - o `cpf` que sai do modelo já vem sem pontuação, independente de como foi digitado na entrada (`"123.456.789-00"` ou `"12345678900"`).

O parâmetro `mode` controla **quando** o validator roda em relação à validação padrão do tipo:

- `mode="after"` (padrão): roda depois que o Pydantic já validou/converteu o tipo básico. Você recebe o valor já no tipo Python esperado.
- `mode="before"`: roda antes, com o valor "cru" (como ele veio, por exemplo ainda como string ou dict). Útil para pré-processar dados em formatos inesperados antes da validação de tipo.

```python
from pydantic import BaseModel, field_validator


class Produto(BaseModel):
    preco: float

    @field_validator("preco", mode="before")
    @classmethod
    def limpar_preco(cls, valor):
        # aceita "R$ 49,90" e converte para 49.90 antes da validação de tipo
        if isinstance(valor, str):
            valor = valor.replace("R$", "").replace(".", "").replace(",", ".").strip()
        return valor
```

### `model_validator`: validação cruzada entre campos

Quando a regra depende de **mais de um campo**, o `field_validator` não é suficiente - para isso existe o `model_validator`, que recebe o modelo inteiro (ou os dados brutos, dependendo do modo):

```python
from datetime import date
from pydantic import BaseModel, model_validator


class Reserva(BaseModel):
    check_in: date
    check_out: date

    @model_validator(mode="after")
    def validar_datas(self) -> "Reserva":
        if self.check_out <= self.check_in:
            raise ValueError("check_out deve ser posterior ao check_in")
        return self
```

Com `mode="after"`, o `self` já é uma instância validada do modelo - você retorna `self` (possivelmente modificado) ao final. Com `mode="before"`, você recebe o `dict` bruto de entrada, antes de qualquer validação de campo, o que é útil para renomear ou combinar chaves antes da validação padrão acontecer.

## `model_config` / `ConfigDict`: configurando o comportamento do modelo

Todo `BaseModel` pode ser configurado com um `model_config`, usando `ConfigDict`:

```python
from pydantic import BaseModel, ConfigDict


class Configuracao(BaseModel):
    model_config = ConfigDict(
        extra="forbid",             # rejeita campos não declarados no modelo
        frozen=True,                # torna a instância imutável após criada
        str_strip_whitespace=True,  # remove espaços extras de strings automaticamente
        populate_by_name=True,      # permite popular tanto pelo nome do campo quanto por alias
    )

    ambiente: str
    debug: bool = False
```

Algumas opções que mais fazem diferença no dia a dia:

| Opção | O que faz |
|---|---|
| `extra="forbid"` | Levanta erro se o payload trouxer campos desconhecidos (ótimo para pegar erros de digitação em nomes de campo). O padrão é `"ignore"`. |
| `extra="allow"` | Aceita e mantém campos extras não declarados, acessíveis via `model_extra`. |
| `frozen=True` | Torna instâncias imutáveis (como um `dataclass(frozen=True)`) - útil para objetos de valor. |
| `str_strip_whitespace=True` | Remove espaços em branco no início/fim de toda string automaticamente. |
| `validate_assignment=True` | Revalida os campos toda vez que você faz `instancia.campo = novo_valor`, e não só na criação. |

## Modelos aninhados e recursivos

Modelos Pydantic podem ser compostos livremente - um campo pode ser outro `BaseModel`, uma lista de modelos, ou até referenciar o próprio tipo (recursão):

```python
from pydantic import BaseModel


class Endereco(BaseModel):
    rua: str
    cidade: str
    cep: str


class Cliente(BaseModel):
    nome: str
    endereco: Endereco          # modelo aninhado
    enderecos_extras: list[Endereco] = []  # lista de modelos aninhados
```

```python
cliente = Cliente(
    nome="Ana",
    endereco={"rua": "Av. Boa Viagem", "cidade": "Recife", "cep": "51021-000"},
)
print(cliente.endereco.cidade)
#> Recife
```

Note que passamos um `dict` para `endereco` e o Pydantic converteu automaticamente para `Endereco`, validando recursivamente cada campo aninhado - inclusive gerando um `ValidationError` detalhado com o **caminho completo** até o campo problemático (ex.: `endereco.cep`), caso algo esteja errado.

Modelos recursivos (uma árvore de comentários, por exemplo) também são possíveis usando `list["Comentario"]` com `from __future__ import annotations` ou aspas no type hint:

```python
from __future__ import annotations
from pydantic import BaseModel


class Comentario(BaseModel):
    texto: str
    respostas: list[Comentario] = []
```

## `Annotated`: tipos reutilizáveis com validação embutida

Quando a mesma regra de validação se repete em vários modelos (por exemplo, "CPF válido" ou "string não vazia"), vale a pena extrair isso para um **tipo reutilizável** usando `Annotated`:

```python
from typing import Annotated
from pydantic import AfterValidator, BaseModel


def validar_cpf(valor: str) -> str:
    digitos = "".join(filter(str.isdigit, valor))
    if len(digitos) != 11:
        raise ValueError("CPF deve conter 11 dígitos")
    return digitos


CPF = Annotated[str, AfterValidator(validar_cpf)]


class Cliente(BaseModel):
    nome: str
    cpf: CPF


class Funcionario(BaseModel):
    nome: str
    cpf: CPF  # mesma validação, reaproveitada, sem repetir o @field_validator
```

Essa abordagem é a forma "moderna" (e recomendada pela documentação oficial) de compartilhar lógica de validação entre modelos diferentes, em vez de copiar o mesmo `field_validator` em cada classe.

## `computed_field`: campos derivados que entram na serialização

Às vezes você quer expor um valor calculado a partir de outros campos, mas que também apareça no `model_dump()`/`model_dump_json()` - para isso existe o `@computed_field`:

```python
from pydantic import BaseModel, computed_field


class ItemPedido(BaseModel):
    preco_unitario: float
    quantidade: int

    @computed_field
    @property
    def subtotal(self) -> float:
        return round(self.preco_unitario * self.quantidade, 2)
```

```python
item = ItemPedido(preco_unitario=49.9, quantidade=3)
print(item.model_dump())
#> {'preco_unitario': 49.9, 'quantidade': 3, 'subtotal': 149.7}
```

Diferente de uma `@property` comum (que não aparece em `model_dump()`), o `computed_field` é tratado como parte do "contrato" de saída do modelo - útil para expor totais, idades calculadas a partir de data de nascimento, ou campos formatados sem duplicar dado no banco.

## Discriminated Unions: validando "um entre vários formatos"

Um problema comum: um payload pode representar **formas diferentes de um mesmo conceito** - por exemplo, um pagamento pode ser via cartão ou via Pix, cada um com campos totalmente diferentes. O Pydantic resolve isso com **discriminated unions**, usando um campo em comum (o "discriminador") para decidir qual modelo aplicar:

```python
from typing import Literal, Union, Annotated
from pydantic import BaseModel, Field


class PagamentoCartao(BaseModel):
    tipo: Literal["cartao"]
    numero_final: str
    parcelas: int = Field(ge=1, le=12)


class PagamentoPix(BaseModel):
    tipo: Literal["pix"]
    chave: str


Pagamento = Annotated[
    Union[PagamentoCartao, PagamentoPix],
    Field(discriminator="tipo"),
]


class Pedido(BaseModel):
    id: str
    pagamento: Pagamento
```

```python
pedido = Pedido(id="1", pagamento={"tipo": "pix", "chave": "cliente@example.com"})
print(type(pedido.pagamento))
#> <class '__main__.PagamentoPix'>
```

A vantagem do discriminador explícito (`Field(discriminator="tipo")`) sobre uma `Union` "simples" é performance e clareza de erro: o Pydantic olha só o campo `tipo` para saber qual modelo validar, em vez de tentar validar contra cada opção da união até uma "encaixar" - e o erro, se `tipo` for inválido, aponta exatamente para os valores aceitos.

## Aliases: compatibilizando nomes de campo com APIs externas

É muito comum consumir uma API que usa `camelCase` (padrão em JavaScript/JSON de muitos serviços) enquanto seu código Python segue `snake_case`. Os aliases resolvem isso sem forçar você a "sujar" os nomes dos seus campos:

```python
from pydantic import BaseModel, ConfigDict, Field


class Cliente(BaseModel):
    model_config = ConfigDict(populate_by_name=True)

    nome_completo: str = Field(alias="fullName")
    data_nascimento: str = Field(alias="birthDate")


# Populando a partir do payload externo (camelCase)
cliente = Cliente.model_validate({"fullName": "Alison Lira", "birthDate": "1990-01-01"})
print(cliente.nome_completo)
#> Alison Lira

# Também funciona pelo nome do campo em Python, graças a populate_by_name=True
cliente2 = Cliente(nome_completo="Ana Silva", data_nascimento="1995-05-05")
```

Também é possível ter aliases diferentes para entrada e saída (`validation_alias` e `serialization_alias`), quando o formato de entrada e o de saída da sua API precisam divergir.

## Generics: modelos reutilizáveis para padrões repetidos

Um padrão extremamente comum em APIs é o **envelope de resposta paginada** - a mesma estrutura (`items`, `total`, `pagina`), mudando apenas o tipo do item. Em vez de duplicar isso para cada entidade, usamos `Generic`:

```python
from typing import Generic, TypeVar
from pydantic import BaseModel

T = TypeVar("T")


class RespostaPaginada(BaseModel, Generic[T]):
    items: list[T]
    total: int
    pagina: int


class Produto(BaseModel):
    id: int
    nome: str


resposta = RespostaPaginada[Produto](
    items=[{"id": 1, "nome": "Teclado"}, {"id": 2, "nome": "Mouse"}],
    total=2,
    pagina=1,
)
print(resposta.items[0].nome)
#> Teclado
```

`RespostaPaginada[Produto]` funciona como um "molde" reaproveitável - o mesmo modelo genérico serve para paginar clientes, pedidos, produtos, etc., sem reescrever a estrutura toda vez.

## Pydantic Settings: configuração de aplicação tipada

O pacote `pydantic-settings` (instalado separadamente, como vimos na Parte 1) aplica a mesma filosofia de validação do Pydantic para **configuração de aplicação**, lendo automaticamente de variáveis de ambiente e/ou arquivos `.env`:

```python
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class ConfiguracaoApp(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="APP_")

    ambiente: str = "development"
    debug: bool = False
    database_url: str
    rabbitmq_url: str = "amqp://guest:guest@localhost:5672/"
    max_conexoes: int = Field(default=10, ge=1)


config = ConfiguracaoApp()
```

Com `env_prefix="APP_"`, a variável de ambiente `APP_DATABASE_URL` popula `database_url` automaticamente. Isso já traz, de graça, tudo que vimos até aqui: validação de tipo (`max_conexoes` tem que ser um inteiro ≥ 1), conversão automática de tipos (variáveis de ambiente sempre chegam como string, e o Pydantic converte para `bool`/`int` conforme declarado), e erros claros na inicialização da aplicação caso falte uma variável obrigatória - muito melhor do que descobrir isso em runtime, no meio de uma requisição.

## Notas de performance

- O núcleo de validação do Pydantic V2 (`pydantic-core`) é escrito em **Rust**, o que o torna significativamente mais rápido que a V1 (que era 100% Python) - a documentação oficial reporta a validação como uma das mais rápidas entre bibliotecas do gênero em Python.
- Para validar dados **sem** precisar definir uma classe `BaseModel` completa (por exemplo, validar apenas um `list[int]` ou um tipo primitivo com regras), use `TypeAdapter`, que evita o overhead de criar um modelo só para isso:

```python
from pydantic import TypeAdapter

ValidadorLista = TypeAdapter(list[int])
print(ValidadorLista.validate_python(["1", "2", "3"]))
#> [1, 2, 3]
```

- O schema de validação de cada modelo é **construído uma vez** (na definição da classe) e reaproveitado em todas as instâncias - por isso, evite recriar classes de modelo dinamicamente dentro de loops ou funções chamadas com frequência.

## Projeto prático (cenário real): serviço de processamento de pedidos via fila

Vamos juntar praticamente tudo que vimos nos dois posts em um cenário realista: um **worker** (por exemplo, um consumer Celery/RabbitMQ) que recebe mensagens de pedidos de um sistema externo (em camelCase), valida, calcula totais e decide o fluxo de acordo com a forma de pagamento.

```python
from datetime import datetime
from typing import Annotated, Literal, Union

from pydantic import (
    AfterValidator,
    BaseModel,
    ConfigDict,
    EmailStr,
    Field,
    computed_field,
    field_validator,
    model_validator,
)
from pydantic_settings import BaseSettings, SettingsConfigDict


# --- Configuração da aplicação -------------------------------------------------

class ConfiguracaoWorker(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="WORKER_")

    rabbitmq_url: str
    fila_pedidos: str = "pedidos.processar"
    desconto_maximo_percentual: float = Field(default=20.0, ge=0, le=100)


# --- Tipos reutilizáveis --------------------------------------------------------

def validar_cep(valor: str) -> str:
    digitos = "".join(filter(str.isdigit, valor))
    if len(digitos) != 8:
        raise ValueError("CEP deve conter 8 dígitos")
    return f"{digitos[:5]}-{digitos[5:]}"


CEP = Annotated[str, AfterValidator(validar_cep)]


# --- Modelos aninhados -----------------------------------------------------------

class Endereco(BaseModel):
    model_config = ConfigDict(populate_by_name=True)

    rua: str = Field(alias="street")
    cidade: str = Field(alias="city")
    cep: CEP = Field(alias="zipCode")


class ItemPedido(BaseModel):
    model_config = ConfigDict(populate_by_name=True)

    sku: str
    quantidade: int = Field(gt=0, alias="quantity")
    preco_unitario: float = Field(gt=0, alias="unitPrice")

    @computed_field
    @property
    def subtotal(self) -> float:
        return round(self.quantidade * self.preco_unitario, 2)


# --- Discriminated union para forma de pagamento ----------------------------------

class PagamentoCartao(BaseModel):
    tipo: Literal["cartao"]
    numero_final: str
    parcelas: int = Field(ge=1, le=12)


class PagamentoPix(BaseModel):
    tipo: Literal["pix"]
    chave: str


Pagamento = Annotated[Union[PagamentoCartao, PagamentoPix], Field(discriminator="tipo")]


# --- Modelo principal da mensagem da fila -----------------------------------------

class MensagemPedido(BaseModel):
    model_config = ConfigDict(populate_by_name=True, extra="forbid")

    pedido_id: str = Field(alias="orderId")
    cliente_email: EmailStr = Field(alias="customerEmail")
    endereco_entrega: Endereco = Field(alias="shippingAddress")
    itens: list[ItemPedido] = Field(alias="items")
    pagamento: Pagamento
    desconto_percentual: float = Field(default=0, ge=0, alias="discountPercent")
    criado_em: datetime = Field(alias="createdAt")

    @field_validator("itens")
    @classmethod
    def validar_pelo_menos_um_item(cls, itens: list[ItemPedido]) -> list[ItemPedido]:
        if not itens:
            raise ValueError("o pedido precisa ter ao menos um item")
        return itens

    @computed_field
    @property
    def valor_total(self) -> float:
        subtotal = sum(item.subtotal for item in self.itens)
        return round(subtotal * (1 - self.desconto_percentual / 100), 2)

    @model_validator(mode="after")
    def validar_parcelamento_pix(self) -> "MensagemPedido":
        if isinstance(self.pagamento, PagamentoPix) and self.desconto_percentual > 0:
            # regra de negócio fictícia: pagamentos via Pix não acumulam desconto de cupom
            raise ValueError("desconto de cupom não é aplicável para pagamentos via Pix")
        return self


# --- "Worker" que processa a mensagem ---------------------------------------------

def processar_mensagem(payload_bruto: dict, config: ConfiguracaoWorker) -> None:
    try:
        pedido = MensagemPedido.model_validate(payload_bruto)
    except Exception as erro:
        # Em um cenário real: logar estruturado e enviar para uma dead-letter queue
        print(f"[REJEITADO] payload inválido: {erro}")
        return

    if pedido.desconto_percentual > config.desconto_maximo_percentual:
        print(f"[REJEITADO] desconto acima do permitido ({config.desconto_maximo_percentual}%)")
        return

    print(f"[OK] pedido {pedido.pedido_id} - total: R$ {pedido.valor_total} "
          f"- pagamento: {pedido.pagamento.tipo} - entrega: {pedido.endereco_entrega.cidade}")


# --- Simulando o fluxo -------------------------------------------------------------

if __name__ == "__main__":
    config = ConfiguracaoWorker(rabbitmq_url="amqp://guest:guest@localhost:5672/")

    payload_externo = {
        "orderId": "ORD-777",
        "customerEmail": "cliente@example.com",
        "shippingAddress": {
            "street": "Av. Boa Viagem, 500",
            "city": "Recife",
            "zipCode": "51.021-000",
        },
        "items": [
            {"sku": "TEC-01", "quantity": 1, "unitPrice": 350.0},
            {"sku": "MOU-02", "quantity": 2, "unitPrice": 89.9},
        ],
        "pagamento": {"tipo": "pix", "chave": "cliente@example.com"},
        "discountPercent": 0,
        "createdAt": "2026-08-11T14:30:00",
    }

    processar_mensagem(payload_externo, config)
    #> [OK] pedido ORD-777 - total: R$ 529.8 - pagamento: pix - entrega: Recife
```

### O que esse projeto demonstra, ponta a ponta

1. **Configuração tipada** (`ConfiguracaoWorker`) carregada de variáveis de ambiente, com limites de negócio (`desconto_maximo_percentual`) já validados na inicialização.
2. **Aliases** (`populate_by_name` + `Field(alias=...)`) traduzindo automaticamente o payload externo em camelCase para os nomes de campo em português usados internamente.
3. **Tipo reutilizável via `Annotated`** (`CEP`) normalizando e validando o CEP em qualquer modelo que o utilize.
4. **Modelos aninhados** (`Endereco`, `ItemPedido`) compondo a mensagem principal, com validação recursiva automática.
5. **Discriminated union** decidindo em tempo de validação se o pagamento é `PagamentoCartao` ou `PagamentoPix`, sem `if`/`isinstance` manual.
6. **`computed_field`** calculando `subtotal` por item e `valor_total` do pedido, já refletido em qualquer serialização futura.
7. **`field_validator`** garantindo uma regra simples de coleção (pelo menos um item).
8. **`model_validator`** aplicando uma regra de negócio que depende de dois campos ao mesmo tempo (pagamento + desconto).
9. Um **fluxo de erro realista**: se a validação falhar, a mensagem é rejeitada de forma controlada (em produção, isso normalmente vai para uma dead-letter queue), sem derrubar o worker.

## Conclusão

Com os dois posts desta série, você tem hoje uma base sólida - dos fundamentos até um projeto de validação de dados com múltiplas camadas (configuração, composição de modelos, regras de negócio e integração com formatos externos), da forma como ele apareceria em um serviço real de backend em produção.

## Links

- [Documentação oficial do Pydantic - Get Started](https://pydantic.dev/docs/validation/latest/get-started/)
- [Introdução ao Pydantic - Python Academy](https://pythonacademy.com.br/blog/introducao-ao-pydantic-validacao-e-modelagem-de-dados-simplificada-com-python)
- [Documentação oficial - Validators](https://docs.pydantic.dev/latest/concepts/validators/)
- [Documentação oficial - Pydantic Settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)
- [Repositório oficial no GitHub](https://github.com/pydantic/pydantic)
- [Pydantic Parte 1 - Fundamentos]({{< relref "posts/pydantic-parte-1-fundamentos/index.md" >}})
