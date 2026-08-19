---
title: "Dataclasses em Python Explicado - Conceitos, Código e Boas Práticas"
date: 2026-08-19
draft: false
tags: ["python", "dataclasses", "poo", "backend", "boilerplate"]
categories: ["Python"]
description: "Um guia completo sobre Dataclasses em Python, explicando a PEP 557, os métodos especiais gerados, field(), valores padrão, __post_init__, imutabilidade, comparação, slots e casos práticos."
cover: dataclasses.png
---

## Introdução

Quando uma classe Python existe principalmente para representar dados, é comum escrevermos muito código repetitivo. Precisamos declarar `__init__()`, atribuir atributos, implementar `__repr__()` para debugging e, muitas vezes, `__eq__()` para comparar objetos.

As **Dataclasses** reduzem esse *boilerplate* e tornam a intenção da classe mais explícita.

---

## O que são Dataclasses?

Uma **Dataclass** é uma classe normal que foi processada pelo decorador `@dataclass`. O decorador examina as anotações de tipo e adiciona métodos gerados, como `__init__()`, `__repr__()` e `__eq__()`.

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int
```

Podemos então fazer:

```python
user = User("Alison", 38)

print(user)
# User(name='Alison', age=38)
```

E:

```python
user1 = User("Alison", 38)
user2 = User("Alison", 38)

print(user1 == user2)
# True
```

A [documentação oficial](https://docs.python.org/3/library/dataclasses.html) explica que o decorador **não cria uma nova classe**: ele processa a classe existente, adiciona os métodos necessários e retorna a mesma classe.

---

## Quando surgiram?

Dataclasses foram propostas na [PEP 557](https://peps.python.org/pep-0557/) e adicionadas ao **Python 3.7**. A proposta buscava oferecer uma solução da biblioteca padrão para classes que representam principalmente dados, reduzindo código repetitivo.

Antes delas, era comum escrever classes manualmente, usar `namedtuple`, `typing.NamedTuple` ou bibliotecas como `attrs`.

---

## O que acontece quando a Dataclass é criada?

Esta é uma das partes mais importantes para entender o recurso.

Considere:

```python
@dataclass
class User:
    name: str
    age: int
```

Quando o decorador é executado, ele examina `User.__annotations__` e identifica `name` e `age` como campos.

Com os parâmetros padrão (`init=True`, `repr=True`, `eq=True`), ele gera métodos equivalentes a:

```python
def __init__(self, name: str, age: int):
    self.name = name
    self.age = age

def __repr__(self):
    return f"User(name={self.name!r}, age={self.age!r})"

def __eq__(self, other):
    if other.__class__ is self.__class__:
        return (
            self.name == other.name
            and self.age == other.age
        )
    return NotImplemented
```

Isso é uma representação conceitual do resultado; não significa que o código-fonte desses métodos seja inserido literalmente na classe dessa forma.

---

## Getters e Setters: O que são e como funcionam em Python?

### O que são?

Em linguagens como Java ou C#, é comum ver código assim:

```java
// Java - getters e setters tradicionais
public class User {
    private String name;
    private int age;

    public String getName() { return this.name; }
    public void setName(String name) { this.name = name; }

    public int getAge() { return this.age; }
    public void setAge(int age) { this.age = age; }
}
```

Nessas linguagens, o acesso a atributos passa sempre por métodos:

```java
User user = new User();
user.setName("Alison");
System.out.println(user.getName());
```

### Como funciona em Python?

Em Python, o acesso a atributos é **direto** - não precisamos de métodos `get_` e `set_`:

```python
user = User("Alison", 38)

# Acesso direto (não precisa de get_name())
print(user.name)   # Alison

# Atribuição direta (não precisa de set_name())
user.age = 39
```

### A Dataclass cria getters e setters?

**Não.** É comum encontrar a afirmação de que uma Dataclass "cria getters e setters para cada atributo". Isso **não é verdade** para Python.

Dataclasses **não geram** automaticamente métodos como:

```python
get_name()
set_name()
get_age()
set_age()
```

O que a Dataclass gera são os **métodos especiais** (`__init__`, `__repr__`, `__eq__`), que rolam por "debaixo dos panos" e não são chamados diretamente pelo programador.

### E se eu precisar de lógica no acesso ou na alteração?

Se você precisa de validação ou lógica ao ler ou alterar um atributo, use `@property`:

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    _age: int

    @property
    def age(self) -> int:
        """Getter: lógica ao acessar o valor."""
        return self._age

    @age.setter
    def age(self, value: int) -> None:
        """Setter: lógica ao alterar o valor."""
        if value < 0:
            raise ValueError("Age cannot be negative")
        self._age = value
```

Agora:

```python
user = User("Alison", 38)

print(user.age)      # 38 - por baixo chama o getter
user.age = 39        # por baixo chama o setter
user.age = -5        # ValueError: Age cannot be negative
```

O `property` fornece a interface `user.age`, mas por baixo existe lógica no getter e no setter. **Essa lógica não é criada automaticamente pela Dataclass.**

### Resumo visual

```
┌─────────────────────────────────────────────────────────────┐
│  OUTRAS LINGUAGENS (Java, C#)                               │
│  user.getName()  →  getter explícito                        │
│  user.setName(x) →  setter explícito                        │
├─────────────────────────────────────────────────────────────┤
│  PYTHON (padrão)                                            │
│  user.name       →  acesso direto ao atributo               │
│  user.name = x   →  atribuição direta                       │
├─────────────────────────────────────────────────────────────┤
│  PYTHON (com @property)                                     │
│  user.name       →  chama getter por baixo                  │
│  user.name = x   →  chama setter por baixo                  │
│  (interface igual, mas com lógica embutida)                 │
├─────────────────────────────────────────────────────────────┤
│  DATACLASS                                                  │
│  NÃO cria get_name() / set_name()                           │
│  Cria __init__, __repr__, __eq__ (métodos especiais)        │
└─────────────────────────────────────────────────────────────┘
```

---

## `__init__()`, `__repr__()` e `__eq__()`

### `__init__()`

Inicializa a instância depois que ela é criada.

```python
@dataclass
class User:
    name: str
    age: int
```

A Dataclass gera conceitualmente:

```python
def __init__(self, name: str, age: int):
    self.name = name
    self.age = age
```

Assim:

```python
user = User("Alison", 38)
```

funciona sem precisarmos escrever o método.

> 📖 [Documentação - object.__init__](https://docs.python.org/3/reference/datamodel.html#object.__init__)

### `__repr__()`

Fornece uma representação útil do objeto:

```python
print(User("Alison", 38))
# User(name='Alison', age=38)
```

Isso é especialmente útil em debugging, logs e coleções.

Podemos impedir que um campo apareça:

```python
from dataclasses import dataclass, field

@dataclass
class User:
    name: str
    password: str = field(repr=False)
```

> 📖 [Documentação - object.__repr__](https://docs.python.org/3/reference/datamodel.html#object.__repr__)

### `__eq__()`

Permite comparação por valor:

```python
user1 = User("Alison", 38)
user2 = User("Alison", 38)

user1 == user2
# True
```

O `eq=True` padrão faz a Dataclass gerar esse método.

> **Observação de versão:** no Python 3.13, a implementação gerada de `__eq__()` passou a comparar os campos individualmente, em vez de construir tuplas para a comparação.

> 📖 [Documentação - object.__eq__](https://docs.python.org/3/reference/datamodel.html#object.__eq__)

---

## `@dataclass` - Parâmetros do decorador

A assinatura atual é:

```python
@dataclass(
    *,
    init=True,
    repr=True,
    eq=True,
    order=False,
    unsafe_hash=False,
    frozen=False,
    match_args=True,
    kw_only=False,
    slots=False,
    weakref_slot=False,
)
```

Os parâmetros mais importantes são:

| Parâmetro | O que faz | Documentação |
|-----------|-----------|--------------|
| `init` | Gera `__init__()` | [dataclasses](https://docs.python.org/3/library/dataclasses.html) |
| `repr` | Gera `__repr__()` | [dataclasses](https://docs.python.org/3/library/dataclasses.html) |
| `eq` | Gera `__eq__()` | [dataclasses](https://docs.python.org/3/library/dataclasses.html) |
| `order` | Gera `__lt__`, `__le__`, `__gt__`, `__ge__` | [dataclasses](https://docs.python.org/3/library/dataclasses.html) |
| `unsafe_hash` | Controla a geração forçada de `__hash__()` | [dataclasses](https://docs.python.org/3/library/dataclasses.html) |
| `frozen` | Impede atribuições normais aos campos | [Frozen instances](https://docs.python.org/3/library/dataclasses.html#frozen-instances) |
| `match_args` | Controla `__match_args__` para pattern matching | [dataclasses](https://docs.python.org/3/library/dataclasses.html) |
| `kw_only` | Torna os campos keyword-only | [dataclasses](https://docs.python.org/3/library/dataclasses.html) |
| `slots` | Gera `__slots__` | [dataclasses](https://docs.python.org/3/library/dataclasses.html) |
| `weakref_slot` | Adiciona `__weakref__`, exige `slots=True` | [dataclasses](https://docs.python.org/3/library/dataclasses.html) |

---

## Valor padrão vs. `field(default=...)` - Qual a diferença?

Estas duas declarações:

```python
age: int = 18
```

e:

```python
age: int = field(default=18)
```

produzem, para esse caso simples, **o mesmo valor padrão** e **o mesmo parâmetro no `__init__()`**:

```python
def __init__(self, age: int = 18):
    self.age = age
```

A diferença é de **finalidade e flexibilidade**.

### Quando usar `age: int = 18`?

Use quando você simplesmente quer um valor padrão, sem configurações extras:

```python
@dataclass
class User:
    name: str
    age: int = 18
```

✅ **Vantagem:** mais curto, mais legível, suficiente para 90% dos casos.

### Quando usar `field(default=18)`?

Use quando também precisa **configurar o comportamento** daquele campo:

```python
from dataclasses import field

@dataclass
class User:
    name: str
    age: int = field(
        default=18,
        repr=False,      # não aparece no print()
        compare=False,   # não entra na comparação ==
    )
```

Nesse exemplo, `age` continua com valor padrão 18, mas:
- **não aparece** no `repr`
- **não participa** da igualdade/comparações geradas

### O que mais `field()` permite?

```python
field(
    default=18,                    # valor padrão simples
    default_factory=list,          # valor padrão mutável (função)
    init=False,                   # não aparece no __init__
    repr=False,                   # não aparece no __repr__
    compare=False,                # não entra em ==, <, >
    hash=False,                   # não entra no __hash__
    metadata={"description": "User age"},  # metadados extras
)
```

### Fluxograma de decisão

```
                    Você quer um valor padrão?
                           │
           ┌───────────────┴───────────────┐
           ▼                               ▼
    É só um valor simples?        Precisa de configuração extra?
           │                               │
           ▼                               ▼
    age: int = 18                  age: int = field(
    (mais legível)                      default=18,
                                        repr=False,
    ✅ Recomendado                      compare=False,
    para iniciantes                     ...)

                                        ✅ Recomendado
                                        quando precisa de controle
```

> 📖 [Documentação - field()](https://docs.python.org/3/library/dataclasses.html#dataclasses.field)

---

## Atributos com valores padrão

```python
@dataclass
class User:
    name: str
    age: int
    active: bool = True
```

Então:

```python
user = User("Alison", 38)
```

usa `True` para `active`.

> ⚠️ **Regra importante:** campos sem valor padrão devem aparecer **antes** dos campos com valor padrão.

---

## Valores mutáveis e `default_factory`

### ❌ O problema

Evite:

```python
@dataclass
class ShoppingCart:
    items: list[str] = []   # PERIGO! Todos os objetos compartilham a mesma lista
```

Isso cria **uma única lista** no momento da definição da classe. Todas as instâncias compartilham essa mesma lista.

### ✅ A solução

Para valores mutáveis, use:

```python
from dataclasses import field

@dataclass
class ShoppingCart:
    items: list[str] = field(default_factory=list)
```

`default_factory` recebe uma **função sem argumentos**. Ela é chamada quando uma nova instância precisa do valor padrão, criando uma lista **nova e independente** para cada objeto.

```python
cart1 = ShoppingCart()
cart2 = ShoppingCart()

cart1.items.append("Book")

print(cart1.items)  # ['Book']
print(cart2.items)  # []  ← lista separada!
```

> 📖 [Documentação - default_factory](https://docs.python.org/3/library/dataclasses.html#dataclasses.field)

---

## Por que usar `__post_init__()`?

`__post_init__()` é o **ponto de extensão** da inicialização automática da Dataclass. Ele é chamado automaticamente **depois** que o `__init__()` gerado atribui todos os campos.

### Quando usar?

| Situação | Exemplo |
|----------|---------|
| Calcular atributos derivados | `area = width * height` |
| Validar combinações de campos | `if price < 0: raise ValueError` |
| Normalizar dados depois da atribuição | converter string para maiúsculas |
| Inicialização que depende de vários atributos | `full_name = first + " " + last` |
| Trabalhar com `InitVar` | passar valores temporários para o init |
| Chamar `__init__()` de uma classe base | herança com dataclasses |

### Exemplo 1: Atributo derivado

```python
from dataclasses import dataclass, field

@dataclass
class Rectangle:
    width: float
    height: float
    area: float = field(init=False)  # não entra no __init__()

    def __post_init__(self):
        self.area = self.width * self.height
```

O que acontece por baixo quando criamos `Rectangle(10, 5)`:

```
1. __init__() gerado executa:
   self.width = 10
   self.height = 5

2. Automaticamente chama:
   self.__post_init__()

3. __post_init__() executa:
   self.area = 10 * 5  →  50.0
```

```python
rectangle = Rectangle(10, 5)
print(rectangle.area)  # 50.0
```

### Exemplo 2: Validação

```python
@dataclass
class Product:
    name: str
    price: float

    def __post_init__(self):
        if self.price < 0:
            raise ValueError("Price cannot be negative")
        if not self.name.strip():
            raise ValueError("Name cannot be empty")
```

```python
# Funciona normalmente
product = Product("Laptop", 1500.00)

# Levanta erro na criação
product = Product("Laptop", -100)
# ValueError: Price cannot be negative
```

### Exemplo 3: Normalização de dados

```python
@dataclass
class User:
    name: str
    email: str

    def __post_init__(self):
        self.name = self.name.strip().title()
        self.email = self.email.strip().lower()
```

```python
user = User("  alison  ", "  Alison@Email.COM  ")
print(user.name)   # Alison
print(user.email)  # alison@email.com
```

### ⚠️ Atenção

`__post_init__()` é chamado automaticamente **apenas quando a Dataclass gera o `__init__()`**. Se você usar `init=False` ou escrever seu próprio `__init__()`, a chamada automática **não acontece**.

> 📖 [Documentação - post-init processing](https://docs.python.org/3/library/dataclasses.html#post-init-processing)

---

## Métodos personalizados

Dataclasses continuam sendo classes Python normais:

```python
@dataclass
class Product:
    name: str
    price: float

    def apply_discount(self, percentage: float) -> None:
        self.price *= 1 - percentage / 100

    @property
    def price_with_tax(self) -> float:
        return self.price * 1.10
```

Podemos adicionar propriedades, métodos, validações e outras funcionalidades normalmente.

---

## `frozen=True` - Instâncias imutáveis

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Point:
    x: float
    y: float
```

Depois de criado:

```python
point = Point(10, 20)
point.x = 30
# FrozenInstanceError: cannot assign to field 'x'
```

Isso fornece uma forma de emular instâncias **somente-leitura**.

> ⚠️ **Atenção:** `frozen=True` não torna objetos internos profundamente imutáveis. Uma lista armazenada em um campo continua sendo uma lista mutável.

> 📖 [Documentação - Frozen instances](https://docs.python.org/3/library/dataclasses.html#frozen-instances)

---

## `order=True` - Ordenação de objetos

```python
@dataclass(order=True)
class Product:
    price: float
```

Gera:

```python
__lt__   # menor que (<)
__le__   # menor ou igual (<=)
__gt__   # maior que (>)
__ge__   # maior ou igual (>=)
```

Permite:

```python
Product(10) < Product(20)
# True
```

> ⚠️ **Só use `order=True` quando a ordenação realmente fizer sentido para o domínio.** Nem toda entidade possui um conceito útil de "menor" e "maior".

> 📖 [Documentação - dataclasses](https://docs.python.org/3/library/dataclasses.html)

---

## `slots=True` - Otimização de memória

```python
@dataclass(slots=True)
class User:
    name: str
    age: int
```

`slots=True` cria uma Dataclass baseada em `__slots__`, o que pode:
- Reduzir o overhead de memória de cada instância
- Acelerar o acesso a atributos
- **Restringir atributos arbitrários** (não é possível adicionar `user.novo_atributo = 1`)

> 📖 [Documentação - dataclasses](https://docs.python.org/3/library/dataclasses.html)

---

## `weakref_slot=True`

É um recurso mais avançado:

```python
@dataclass(slots=True, weakref_slot=True)
class User:
    name: str
```

Ele adiciona `__weakref__` e **exige** `slots=True`.

> 📖 [Documentação - dataclasses](https://docs.python.org/3/library/dataclasses.html)

---

## `kw_only=True` - Apenas argumentos nomeados

```python
@dataclass(kw_only=True)
class User:
    name: str
    age: int
```

Agora:

```python
User(name="Alison", age=38)   # ✅ válido
User("Alison", 38)             # ❌ TypeError
```

Útil para evitar confusão com a ordem dos argumentos.

> 📖 [Documentação - dataclasses](https://docs.python.org/3/library/dataclasses.html)

---

## Funções auxiliares do módulo `dataclasses`

O módulo oferece ferramentas auxiliares importantes:

```python
from dataclasses import (
    asdict,
    astuple,
    fields,
    replace,
    is_dataclass,
)
```

### `asdict()` - Converter para dicionário

```python
user = User(name="Alison", age=38)

asdict(user)
# {"name": "Alison", "age": 38}
```

> 📖 [Documentação - asdict](https://docs.python.org/3/library/dataclasses.html#dataclasses.asdict)

### `astuple()` - Converter para tupla

```python
astuple(user)
# ("Alison", 38)
```

> 📖 [Documentação - astuple](https://docs.python.org/3/library/dataclasses.html#dataclasses.astuple)

### `fields()` - Inspecionar campos

```python
for item in fields(User):
    print(item.name, item.type, item.default)

# name <class 'str'> <dataclasses._MISSING_TYPE object>
# age <class 'int'> 18
```

> 📖 [Documentação - fields](https://docs.python.org/3/library/dataclasses.html#dataclasses.fields)

### `replace()` - Criar cópia modificada

```python
updated = replace(user, age=39)
# User(name='Alison', age=39)  - user original não muda!
```

`replace()` cria uma nova instância e passa pelo `__init__()` e, consequentemente, pelo `__post_init__()`.

> 📖 [Documentação - replace](https://docs.python.org/3/library/dataclasses.html#dataclasses.replace)

### `is_dataclass()` - Verificar se é dataclass

```python
is_dataclass(User)     # True
is_dataclass(user)     # True (instância também funciona)
is_dataclass(str)      # False
```

> 📖 [Documentação - is_dataclass](https://docs.python.org/3/library/dataclasses.html#dataclasses.is_dataclass)

---

## Dataclass vs. classe normal vs. NamedTuple

| Característica | Classe normal | Dataclass | NamedTuple |
|---|---|---|---|
| Boilerplate | Maior | Menor | Menor |
| Mutabilidade | Normalmente sim | Sim por padrão | Não |
| Métodos personalizados | Excelente | Excelente | Possível |
| Igualdade por valor | Manual | Automática | Sim |
| Valores padrão | Manual | Sim | Sim |
| Semântica de tupla | Não | Não | Sim |
| Controle sobre campos | Máximo | Alto | Mais limitado |
| Performance (slots) | Sim | Sim (`slots=True`) | Sim (inherente) |

### Quando usar cada uma?

```
┌─────────────────────────────────────────────────────────────────┐
│  A classe representa PRINCIPALMENTE dados estruturados?         │
│                      │                                          │
│         ┌────────────┴────────────┐                           │
│         ▼                         ▼                           │
│        SIM                        NÃO                          │
│         │                         │                           │
│         ▼                         ▼                           │
│  ┌──────────────┐        ┌─────────────────┐                │
│  │  Precisa de  │        │  Classe normal  │                │
│  │  imutabilidade│        │  (controle      │                │
│  │  ou semântica │        │  total do       │                │
│  │  de tupla?    │        │  comportamento) │                │
│  │      │        │        └─────────────────┘                │
│  │  ┌───┴───┐    │                                           │
│  │  ▼       ▼    │                                           │
│  │ SIM     NÃO   │                                           │
│  │  │      │     │                                           │
│  │  ▼      ▼     │                                           │
│  │ NamedTuple  Dataclass                                      │
│  └───────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘
```

- Use uma **Dataclass** quando a classe representa principalmente dados estruturados.
- Use uma **classe normal** quando comportamento e regras de negócio exigem controle mais específico.
- Use uma **NamedTuple** quando a semântica de tupla, imutabilidade e compatibilidade com APIs que esperam tuplas forem importantes.

> 📖 [Documentação - NamedTuple](https://docs.python.org/3/library/typing.html#typing.NamedTuple)

---

## Casos práticos

### DTO (Data Transfer Object)

```python
@dataclass
class UserDTO:
    id: int
    name: str
    email: str
```

### Configuração

```python
@dataclass
class DatabaseConfig:
    host: str
    port: int = 5432
    database: str = "app"
    ssl: bool = True
```

### Resultado de processamento

```python
@dataclass
class ProcessingResult:
    success: bool
    processed: int
    errors: int
```

### Modelo interno com lógica

```python
@dataclass
class OrderItem:
    product_id: int
    quantity: int
    unit_price: float

    @property
    def total(self) -> float:
        return self.quantity * self.unit_price

    def __post_init__(self):
        if self.quantity <= 0:
            raise ValueError("Quantity must be positive")
```

---

## Erros e armadilhas comuns

### ❌ Não use listas, dicionários ou sets mutáveis diretamente como defaults

```python
# ERRADO
items: list[str] = []

# CERTO
items: list[str] = field(default_factory=list)
```

### ❌ Não confunda `frozen=True` com imutabilidade profunda

```python
@dataclass(frozen=True)
class Container:
    items: list[str]

container = Container(["a", "b"])
container.items.append("c")  # Funciona! A lista interna é mutável
```

### ❌ Não use `unsafe_hash=True` sem entender hashing e igualdade

Hash precisa ser consistente com a semântica de igualdade do objeto.

> 📖 [Documentação - unsafe_hash](https://docs.python.org/3/library/dataclasses.html#dataclasses.dataclass)

### ❌ Não use `order=True` sem uma ordenação natural

Nem toda entidade possui um conceito útil de "menor" e "maior".

### ❌ Não transforme toda classe em Dataclass

Dataclass é uma ferramenta, não uma regra arquitetural.

---

## Dataclass não é ORM

Uma Dataclass **não é automaticamente um modelo de banco de dados**.

Ela pode representar um DTO, um objeto de domínio, uma configuração ou um resultado de serviço, mas não substitui mecanismos de persistência como Django ORM ou SQLAlchemy.

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Banco de      │     │   SQLAlchemy/   │     │   Dataclass     │
│   Dados         │◄────│   Django ORM    │────►│   (DTO/Modelo)  │
│                 │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
       ↑                                               ↓
       └──────────── Persistência ─────────────────────┘

Dataclass = representação de dados
ORM       = persistência e consulta
```

---

## Conclusão

Dataclasses foram introduzidas no Python 3.7 pela [PEP 557](https://peps.python.org/pep-0557/) para reduzir *boilerplate* e tornar mais declarativas as classes orientadas a dados.

Os conceitos mais importantes são:

- Métodos especiais gerados (`__init__`, `__repr__`, `__eq__`)
- `field()` para configuração avançada de campos
- Valores padrão simples vs. `field(default=...)`
- `default_factory` para valores mutáveis
- `__post_init__()` como ponto de extensão da inicialização
- `frozen` para imutabilidade superficial
- `order` para ordenação
- `slots` para otimização de memória
- `kw_only` para argumentos nomeados
- Funções auxiliares: `asdict()`, `astuple()`, `fields()`, `replace()`, `is_dataclass()`

O ponto principal não é simplesmente escrever menos código.

É **declarar claramente a estrutura dos dados** e deixar o Python gerar comportamentos mecânicos quando eles fazem sentido.

---

## Referências

- [Python Documentation - dataclasses](https://docs.python.org/3/library/dataclasses.html)
- [Python Documentation - field()](https://docs.python.org/3/library/dataclasses.html#dataclasses.field)
- [Python Documentation - post-init processing](https://docs.python.org/3/library/dataclasses.html#post-init-processing)
- [Python Documentation - Class variables](https://docs.python.org/3/library/dataclasses.html#class-variables)
- [Python Documentation - Init-only variables](https://docs.python.org/3/library/dataclasses.html#init-only-variables)
- [Python Documentation - Frozen instances](https://docs.python.org/3/library/dataclasses.html#frozen-instances)
- [Python Documentation - Inheritance](https://docs.python.org/3/library/dataclasses.html#inheritance)
- [Python Documentation - Module contents](https://docs.python.org/3/library/dataclasses.html#module-contents)
- [Python Glossary - Special Method](https://docs.python.org/3/glossary.html#term-special-method)
- [Python Data Model - object.__init__](https://docs.python.org/3/reference/datamodel.html#object.__init__)
- [Python Data Model - object.__repr__](https://docs.python.org/3/reference/datamodel.html#object.__repr__)
- [Python Data Model - object.__eq__](https://docs.python.org/3/reference/datamodel.html#object.__eq__)
- [PEP 557 - Data Classes](https://peps.python.org/pep-0557/)
