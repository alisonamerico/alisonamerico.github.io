---
title: "Números Mágicos em Python: Guia do Iniciante para Código Mais Limpo"
date: 2026-02-17
draft: false
tags: ["python", "iniciantes", "boas-praticas", "codigo-limpo", "pep8"]
categories: ["python", "dicas-de-codigo"]
description: "Aprenda o que são números mágicos em Python, por que eles prejudicam seu código e como substituí-los por constantes significativas. Um guia prático com exemplos."
cover: magic-numbers-python.png
---


## O que são Números Mágicos?

Você já viu um código assim?

```python
def calcular_desconto(preco):
    return preco * 0.15
```

O que significa `0.15`? É 15%? Um desconto? Um imposto?

**Números mágicos** são valores numéricos que aparecem no código sem nenhuma explicação. Eles tornam o código difícil de ler, manter e depurar.

---

## Por que Números Mágicos são Problema?

### 1. Difícil de Ler
Outros desenvolvedores (ou você mesmo, no futuro) não conseguem entender o significado dos números.

### 2. Difícil de Manter
Se você precisar mudar um valor, terá que procurar em todo o código.

### 3. Fácil de Cometer Erros
O mesmo número pode ter significados diferentes em lugares distintos.

---

## A Solução: Constantes Nomeadas

O [PEP 8](https://peps.python.org/pep-0008/) recomenda usar `UPPER_SNAKE_CASE` para constantes. Além disso, boas práticas de código limpo recomendam substituir números mágicos por constantes nomeadas para melhor legibilidade.

### Exemplo Prático

**❌ Evite assim:**
```python
def calcular_desconto(preco):
    return preco * 0.15
```

**✅ Faça assim:**
```python
TAXA_DESCONTO = 0.15

def calcular_desconto(preco):
    return preco * TAXA_DESCONTO
```

Outro exemplo:

**❌ Evite assim:**
```python
# O que é 86400? Quanto tempo é isso?
tempo_em_segundos = 86400
```

**✅ Faça assim:**
```python
SEGUNDOS_POR_DIA = 86400

tempo_em_segundos = SEGUNDOS_POR_DIA
```

---

## O Zen do Python

O [PEP 20](https://peps.python.org/pep-0020/) (O Zen do Python) nos ensina:

> **"Explícito é melhor que implícito."**

> **"Legibilidade conta."**

Constantes nomeadas tornam o código explícito e legível!

---

## Quando NÃO Precisa de Constantes

Nem todo número é "mágico". Esses são aceitáveis:

- **`0`, `1`, `-1`** - frequentemente usados como valores iniciais, limites ou retornos
- **Contadores de loop** - como `for index in range(10)`
- **Índices de arrays** - como `itens[0]` ou `itens[-1]`
- **Matemática simples** - como `x * 2` ou `y + 1` onde a operação é clara
- **Comparações booleanas** - como `if quantidade > 0`

Exemplo onde constantes NÃO são necessárias:
```python
# Isso está certo - o significado é óbvio
for index in range(10):
    print(index)

if len(itens) > 0:
    return itens[0]

resultado = valor * 2  # Dobrar é claro
```

---

## Conclusão

Substituir números mágicos por constantes nomeadas é um hábito simples que melhora muito a qualidade do seu código. Sua equipe (e seu eu do futuro) agradecerão!

**Lembre-se:** Seu código é lido muito mais vezes do que é escrito. Vale a pena torná-lo claro.

---

## Referências

- [PEP 8 - Guia de Estilo para Código Python](https://peps.python.org/pep-0008/)
- [PEP 20 - O Zen do Python](https://peps.python.org/pep-0020/)
- [Real Python - Constantes em Python](https://realpython.com/python-constants/)
- [Clean Code de Robert C. Martin - G25: Replace Magic Numbers With Named Constants](https://gist.github.com/eujoy/5d62a0d398571cb51bf6217cc3dfda2e)
- [Martin Fowler - Replace Magic Number with Symbolic Constant](https://martinfowler.com/refactoring/catalog/replaceMagicNumberWithSymbolicConstant.html)
