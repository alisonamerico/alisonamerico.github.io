---
title: Entendo Mock e MagicMock em Python
date: 2026-01-13
draft: false
tags: ["python", "tests", "unittest", "mock"]
categories: ["testing"]
description: "Guia didático sobre Mock e MagicMock em Python, com explicações simples, exemplos práticos e resolução de problemas comuns."
---
![](images/perfil.jpg)
Este artigo foi escrito para **qualquer pessoa entender**, mesmo quem ainda está começando com testes em Python.
Vamos explicar **o que é cada conceito, por que ele existe e quando usar**, sempre com exemplos práticos e explicações claras.

---

## 1. O que é Mock? (explicação simples)

Um **Mock** é um objeto **falso**, criado apenas para testes, que **finge ser outro objeto real**.

Ele permite:
- Simular funções, métodos, classes e objetos inteiros
- Definir retornos e exceções
- Registrar chamadas (quantidade, argumentos, ordem)
- Isolar a unidade de código testada

Em vez de usar:
- um banco de dados real,
- uma API externa,
- um serviço de e-mail,
- ou um sistema complexo,

usamos um mock para **simular esse comportamento**.

### Por que isso é importante?

Porque em testes nós queremos:
- velocidade,
- previsibilidade,
- isolamento.

### Exemplo mental

Imagine uma função que envia e-mails:

```python
def send_email(to):
    print("Enviando email")
```

Em um teste, você **não quer enviar e-mails de verdade**.
Então você substitui essa função por um mock.

O mock:
- não envia nada,
- mas registra se foi chamado,
- com quais argumentos,
- e quantas vezes.

👉 Por isso dizemos que o mock é **flexível**: você define como ele deve se comportar.

👉 Qualquer atributo acessado em um `Mock` **vira automaticamente outro Mock**.

---

## 2. O que é MagicMock?

`MagicMock` é um tipo especial e subclasse de `Mock`.

Ele existe para simular objetos que usam **métodos mágicos do Python**.

### O que são métodos mágicos?

São métodos que:
- começam e terminam com `__`
- o Python chama automaticamente

Exemplos comuns:
- `__len__` → chamado quando usamos `len(obj)`
- `__iter__` → usado em `for`
- `__getitem__` → usado com `obj[x]`
- `__contains__` → usado com `in`
- `__str__` → usado com `print(obj)`

Esses métodos **não são chamados diretamente**, mas pelo próprio Python.

### Por que MagicMock existe?

O `Mock` normal **não lida bem** com esses métodos.
O `MagicMock` já vem preparado para isso.

---

## 3. Mock ≠ Stub ≠ Fake (diferenças importantes)

### Stub
Um **stub** apenas retorna valores fixos.

```python
def get_user_stub():
    return {"id": 1}
```

### Fake
Um **fake** tem uma implementação simples, mas funcional.

```python
class FakeEmailService:
    def __init__(self):
        self.sent = []

    def send(self, to):
        self.sent.append(to)
```

### Mock
Um **mock** registra chamadas e valida interações.

```python
from unittest.mock import Mock

email = Mock()
email.send("a@test.com")
email.send.assert_called_once_with("a@test.com")
```

---

## 4. Principais atributos e métodos

### return_value
Define o valor retornado pelo mock.
```python
mock.func.return_value = 10
```

### side_effect
Permite exceções, funções ou múltiplos retornos.
```python
mock.func.side_effect = Exception("Erro")
```

### called / call_count
Indicam se e quantas vezes foi chamado.
```python
assert mock.func.called
assert mock.func.call_count == 1
```

### call_args / call_args_list
Mostram os argumentos usados.
```python
mock.func(1)
mock.func(2)

assert mock.func.call_args_list == [((1,),), ((2,),)]
```

---

## 5. patch — substituindo dependências

Use `patch` para trocar dependências reais por mocks temporariamente.

A regra de ouro:

> **Você deve patchar onde o objeto é USADO, não onde é DEFINIDO.**

```python
# app/services.py
from app.email import send_email

def notify(user):
    send_email(user.email)
```

```python
with patch("app.services.send_email") as mock_send:
    notify(user)
```

❌ ERRADO:
```python
patch("app.email.send_email")
```

---

## 6. spec e autospec

Use para evitar erros silenciosos e garantir assinatura correta.


### spec
Restringe atributos válidos.

```python
mock = Mock(spec=MyClass)
```

### autospec

Restringe atributos **e assinatura**.

```python
@patch("module.func", autospec=True)
def test(mock_func):
    ...
```

✅ Evita erros silenciosos.

---
## 7. Mock com lógica → use Fake

> Se o mock contém lógica complexa, transforme-o em um fake.


## 8. Problemas comuns e como resolver

### ❌ Mock não funciona
➡️ Patch no lugar errado

### ❌ Teste passa mas quebra em produção
➡️ Falta de autospec

### ❌ Teste difícil de entender
➡️ Mocks demais

### ❌ Mock com lógica
➡️ Extraia para fake


## 9. Quando usar (e quando NÃO usar)

### Use mocks quando:
- Há chamadas HTTP
- Há acesso a banco de dados
- Há leitura/escrita de arquivos
- Há serviços externos
- Há dependências difíceis de reproduzir

### NÃO use mocks quando:
- Você está testando lógica pura
- O teste vira uma cópia da implementação
- O mock começa a ter lógica própria

> Regra prática: **mock dependências, não regras de negócio**.


## 10. Boas práticas reais

- Use `autospec=True`
- Mock menos, teste mais comportamento
- Nomeie bem seus mocks
- Um teste, um cenário

## 11. Anti‑padrões

🚫 Mockar método privado  
🚫 Mockar lógica de domínio  
🚫 Testar apenas chamadas  
🚫 Mocks aninhados em excesso  

---

## 12. Checklist mental

Antes de criar um mock:
- Isso é uma dependência externa?
- Esse teste precisa mesmo disso?
- Um fake seria mais simples?
- Estou testando comportamento ou implementação?

---

## Referências

- https://docs.python.org/3/library/unittest.mock.html
