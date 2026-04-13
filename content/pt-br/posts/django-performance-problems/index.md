---
title: 5 Problemas de Performance no Django ORM e Como Resolver
date: 2026-04-10
draft: false
tags: [django, python, performance, orm, database, best-practices]
categories: [django, performance, coding-tips]
description: Aprenda os 5 problemas mais comuns de performance em aplicações Django ORM e suas soluções comprovadas com exemplos práticos e explicações detalhadas.
cover: django-performance-orm.png
---

Como desenvolvedores Django, frequentemente focamos em fazer nossas aplicações funcionarem corretamente. Escrevemos código limpo, seguimos boas práticas e testamos cuidadosamente. Mas há um **assassino silencioso** que pode derrubar até o código mais elegante: **a performance do banco de dados**.

Neste guia completo, vou guiar você pelos **5 problemas mais comuns de performance** no Django ORM. Para cada problema, explicarei:

- **Por que** acontece (a causa raiz)
- **Como** identificar no seu código
- **O que** acontece no banco de dados
- **A solução exata** com exemplos de código

Vamos lá!

______________________________________________________________________

## Os 5 Problemas de Performance de Um Olhar

| # | Problema | Solução | Queries (Lento) | Queries (Rápido) | Impacto |
|---|----------|-----------|----------------|----------------|--------|
| 1 | N+1 Query (ForeignKey) | `select_related()` | N+1 | 1 | 🔴 Crítico |
| 2 | N+1 Query (ManyToMany) | `prefetch_related()` | N+1 | 2 | 🔴 Crítico |
| 3 | SELECT * desnecessário | `only()` / `defer()` | 1 | 1 | 🟡 Médio |
| 4 | len() vs count() | `.count()` | 1 | 2 | 🟡 Médio |
| 5 | Paginação ausente | `Paginator` | Todos | Tamanho da página | 🔴 Crítico |

______________________________________________________________________

## Problema 1: N+1 Query com ForeignKey

### 🔍 O Que É?

O problema N+1 é o **mais comum e mais danoso** problema de performance em aplicações Django. Ele ocorre quando você busca uma lista de objetos e depois acessa um relacionamento ForeignKey para cada um deles.

### 🧠 Por Que Acontece?

Quando você chama `Book.objects.all()`, o Django executa:

```sql
SELECT * FROM book;
```

Isso retorna 50 livros. Mas aqui está o ponto crucial: **o Django é preguiçoso (lazy)**. Ele não carrega os dados relacionados `author` até que você realmente acesse-os.

Quando seu template ou código faz:

```html
{% for livro in livros %}
    {{ livro.autor.nome }}  <!-- Acessando ForeignKey aqui -->
{% endfor %}
```

O Django pensa: "Ah, você precisa do autor agora? Deixa eu buscar." Então ele executa:

```sql
SELECT * FROM autor WHERE id = 1;
SELECT * FROM autor WHERE id = 2;
SELECT * FROM autor WHERE id = 3;
-- ... e assim por diante para cada livro!
```

### 📊 A Matemática

| Livros | Queries SQL (Lento) | Queries SQL (Rápido) |
|--------|--------------------|--------------------|
| 10 | 11 | 1 |
| 50 | 51 | 1 |
| 100 | 101 | 1 |
| 1.000 | 1.001 | 1 |
| 10.000 | 10.001 | 1 |

Com apenas 1.000 livros, você está fazendo **1.001 viagens ao banco de dados**! Cada viagem tem latência (tipicamente 1-10ms), então você está desperdiçando 1-10 segundos só em comunicação com o banco de dados.

### 💻 O Código

**❌ Versão Lenta (O Problema):**

```python
# views.py
def livros_lento(request):
    """Isso dispara 51 queries!"""
    livros = Book.objects.all()[:50]  # Query 1: SELECT * FROM book
    for livro in livros:
        print(livro.autor.nome)  # Queries 2-51: Uma por livro!
    return render(request, 'livros.html', {'livros': livros})
```

**SQL Gerado:**

```sql
-- Query 1
SELECT * FROM book LIMIT 50;

-- Queries 2-51 (uma para cada livro!)
SELECT * FROM autor WHERE id = 1;
SELECT * FROM autor WHERE id = 2;
...
SELECT * FROM autor WHERE id = 50;
```

**✅ Versão Rápida (A Solução):**

```python
# views.py
def livros_rapido(request):
    """Isso dispara apenas 1 query!"""
    livros = Book.objects.select_related('autor')[:50]
    for livro in livros:
        print(livro.autor.nome)  # Já está na memória!
    return render(request, 'livros.html', {'livros': livros})
```

**SQL Gerado:**

```sql
-- Uma única query com JOIN!
SELECT livro.*, autor.*
FROM book AS livro
INNER JOIN autor ON livro.autor_id = autor.id
LIMIT 50;
```

### 🎯 A Solução: `select_related()`

O método `select_related()` executa um **JOIN SQL** no nível do banco de dados. Os dados relacionados voltam em uma única query.

```python
# Sintaxe
Book.objects.select_related('autor')              # 1 nível
Book.objects.select_related('autor', 'editora')  # Múltiplas relações
Book.objects.select_related('autor__pais')        # Aninhado (2 níveis)
```

### 🧪 Exemplo Real em Template

**❌ Lento:**

```html
<!-- templates/livros_lento.html -->
<h1>Livros</h1>
<table>
    <tr>
        <th>Título</th>
        <th>Autor</th>
        <th>Preço</th>
    </tr>
    {% for livro in livros %}
    <tr>
        <td>{{ livro.titulo }}</td>
        <td>{{ livro.autor.nome }}</td>  <!-- N+1 acontece aqui! -->
        <td>{{ livro.preco }}</td>
    </tr>
    {% endfor %}
</table>
```

**✅ Rápido:**

```html
<!-- templates/livros_rapido.html -->
<!-- Basta adicionar select_related('autor') na view! -->
```

### 📝 Quando Usar

- `select_related()` para relacionamentos **ForeignKey**
- `select_related()` para relacionamentos **OneToOneField**
- Pode encadear aninhado: `select_related('autor__pais')`

______________________________________________________________________

## Problema 2: N+1 Query com ManyToMany

### 🔍 O Que É?

Similar ao problema de ForeignKey, mas ocorre com relacionamentos **ManyToManyField** e **ForeignKey Reverso**. Cada acesso a `.all()` em um ManyToMany dispara uma nova query.

### 🧠 Por Que Acontece?

Relacionamentos ManyToMany requerem uma **tabela de junção** no banco de dados. O Django não pode simplesmente buscar os itens relacionados em um JOIN simples como ForeignKey.

**Código lento:**

```python
livros = Book.objects.all()[:50]  # Query 1: Buscar livros
for livro in livros:
    for tag in livro.tags.all():   # Queries 2-51: Buscar tags para cada livro!
        print(tag.nome)
```

### 💻 O Código

**❌ Versão Lenta:**

```python
def tags_lento(request):
    """Isso dispara 51 queries!"""
    livros = Book.objects.all()[:50]  # 1 query para livros
    for livro in livros:
        tags = livro.tags.all()  # 1 query por livro = 50 queries
        list(tags)
    return render(request, 'tags.html', {'livros': livros})
```

**SQL Gerado:**

```sql
-- Query 1
SELECT * FROM book LIMIT 50;

-- Queries 2-51
SELECT tag.*
FROM tag
INNER JOIN livro_tags ON tag.id = livro_tags.tag_id
WHERE livro_tags.book_id = 1;

-- (repetido para cada livro)
```

**✅ Versão Rápida:**

```python
def tags_rapido(request):
    """Isso dispara apenas 2 queries!"""
    livros = Book.objects.prefetch_related('tags')[:50]
    for livro in livros:
        tags = livro.tags.all()  # Usa o cache pré-buscado
        list(tags)
    return render(request, 'tags.html', {'livros': livros})
```

**SQL Gerado:**

```sql
-- Query 1: Buscar livros
SELECT * FROM book LIMIT 50;

-- Query 2: Buscar TODAS as tags relacionadas de uma vez
SELECT tag.*, livro_tags.book_id
FROM tag
INNER JOIN livro_tags ON tag.id = livro_tags.tag_id
WHERE livro_tags.book_id IN (1, 2, 3, ..., 50);
```

### 🎯 A Solução: `prefetch_related()`

Diferente de `select_related()` que usa um JOIN, `prefetch_related()` executa **duas queries separadas** e junta elas em Python. Isso é mais flexível e funciona com ManyToMany.

```python
# Sintaxe
Book.objects.prefetch_related('tags')              # 1 nível
Book.objects.prefetch_related('tags', 'avaliacoes')  # Múltiplas
Book.objects.prefetch_related('tags__categoria')    # Aninhado
```

### ⚖️ select_related vs prefetch_related

| Recurso | `select_related()` | `prefetch_related()` |
|---------|-------------------|---------------------|
| Operação SQL | JOIN | Duas queries + junção Python |
| Queries | **1** | **2** |
| Uso para | ForeignKey, OneToOne | ManyToMany, FK Reverso |
| Pode filtrar relacionado | ❌ Limitado | ✅ Sim (com `Prefetch`) |
| Suporte aninhado | ✅ Sim | ✅ Sim |

### 📝 Quando Usar Cada Um

```python
# ForeignKey → select_related
Book.objects.select_related('autor')

# ManyToMany → prefetch_related
Book.objects.prefetch_related('tags')

# Ambos na mesma query
Book.objects.select_related('autor').prefetch_related('tags')

# ForeignKey Reverso → prefetch_related
Autor.objects.prefetch_related('livros')
```

______________________________________________________________________

## Problema 3: SELECT * Desnecessário

### 🔍 O Que É?

O método `all()` do Django gera `SELECT *`, que recupera **todas as colunas** da tabela do banco de dados, incluindo campos de texto grandes e dados binários que você não precisa.

### 🧠 Por Que Importa?

Considere este cenário:

```python
class Livro(models.Model):
    titulo = models.CharField(max_length=200)       # ~200 bytes
    sinopse = models.TextField()                      # Pode ter 10KB+!
    imagem_capa = models.ImageField(upload_to='capas')  # Pode ter 1MB+!
    arquivo_pdf = models.FileField(upload_to='pdfs')  # Pode ter 10MB+!
    ano_publicacao = models.IntegerField()
    preco = models.DecimalField(max_digits=6, decimal_places=2)
```

Se você só precisa de `titulo` e `preco`, mas faz query `Livro.objects.all()`, você está buscando **10MB+ de dados por livro** desnecessariamente!

### 💻 O Código

**❌ Versão Lenta:**

```python
def campos_lento(request):
    """Carrega TODOS os campos incluindo os grandes!"""
    livros = Livro.objects.all()[:50]
    dados = [{'titulo': l.titulo, 'preco': l.preco} for l in livros]
    return render(request, 'campos.html', {'dados': dados})
```

**SQL Gerado:**

```sql
SELECT id, titulo, sinopse, imagem_capa, arquivo_pdf, ano_publicacao, preco
FROM livro
LIMIT 50;
```

Isso busca todas as colunas, incluindo campos potencialmente massivos como `sinopse`, `imagem_capa` e `arquivo_pdf`.

**✅ Versão Rápida:**

```python
def campos_rapido(request):
    """Carrega SOMENTE os campos que precisamos!"""
    livros = Livro.objects.only('titulo', 'preco')[:50]
    dados = [{'titulo': l.titulo, 'preco': l.preco} for l in livros]
    return render(request, 'campos.html', {'dados': dados})
```

**SQL Gerado:**

```sql
SELECT id, titulo, preco
FROM livro
LIMIT 50;
```

### 🎯 A Solução: `only()` e `defer()`

```python
# only(): Carrega SOMENTE estes campos (mais id)
Livro.objects.only('titulo', 'preco')

# defer(): Carrega tudo EXCETO estes campos
Livro.objects.defer('sinopse', 'imagem_capa', 'arquivo_pdf')

# Relacionamentos aninhados
Livro.objects.select_related('autor').only('titulo', 'autor__nome')
```

### ⚠️ Aviso Importante

Após usar `only()` ou `defer()`, o objeto está **incompleto**. Acessar um campo adiado dispara uma nova query!

```python
livros = Livro.objects.only('titulo', 'preco')

livro = livros[0]
print(livro.titulo)     # ✅ OK - já carregado
print(livro.preco)      # ✅ OK - já carregado
print(livro.sinopse)    # ❌ NOVA QUERY! - campo adiado
```

### 📊 Quando Usar

| Situação | Método |
|----------|--------|
| Sabe exatamente quais campos precisa | `only('campo1', 'campo2')` |
| Sabe quais campos NÃO precisa | `defer('campo_grande', 'campo_arquivo')` |
| Use com `select_related()` | `only('titulo', 'autor__nome')` |

______________________________________________________________________

## Problema 4: len() vs count()

### 🔍 O Que É?

Usar `len()` em um QuerySet **avalia todo o queryset** e carrega todos os registros na memória só para contá-los. Usar `.count()` executa um `SELECT COUNT(*)` no nível do banco de dados.

### 🧠 Por Que Importa?

```python
# Esses parecem similares mas são MUITO diferentes:
len(Livro.objects.all())      # Carrega TODOS 1 milhão de livros na memória!
Livro.objects.count()         # Apenas pergunta: "quantas linhas?"
```

### 💻 O Código

**❌ Versão Lenta:**

```python
def contagem_lenta(request):
    """Carrega TODOS os livros na memória só para contá-los!"""
    livros = Livro.objects.all()      # Carrega 1 milhão de registros
    total = len(livros)                # Python conta na memória
    return render(request, 'contagem.html', {'total': total})
```

**SQL Gerado:**

```sql
SELECT * FROM livro;  # Retorna 1 milhão de linhas!
```

Depois o Python conta na memória. Isso é **extremamente lento e intensivo em memória**.

**✅ Versão Rápida:**

```python
def contagem_rapida(request):
    """Conta no nível do banco de dados - super rápido!"""
    total = Livro.objects.count()    # Banco conta as linhas
    return render(request, 'contagem.html', {'total': total})
```

**SQL Gerado:**

```sql
SELECT COUNT(*) FROM livro;  # Retorna apenas o número: 1000000
```

### 📊 Comparação de Performance

| Registros | `len()` | `.count()` | Aceleração |
|-----------|---------|------------|------------|
| 100 | 5ms | 1ms | 5x |
| 10.000 | 500ms | 1ms | 500x |
| 1.000.000 | 50s | 2ms | 25.000x |

### 📝 Quando Usar Cada Um

| Situação | Método | Por quê |
|----------|--------|---------|
| Precisa SOMENTE da contagem | `.count()` | ✅ Eficiente |
| Precisa da contagem E da lista | `len(queryset)` ou `.count()` | Qualquer um funciona |
| Já tem dados em cache | `len(lista_de_objetos)` | ✅ Fine |
| Verificando se queryset está vazio | `.exists()` | ✅ Mais eficiente |

### 💡 Dica Pro: O Método `.exists()`

```python
# ❌ Errado - carrega todos os registros
if len(Livro.objects.filter(ano_publicacao=2024)):
    pass

# ✅ Correto - apenas verifica existência
if Livro.objects.filter(ano_publicacao=2024).exists():
    pass
```

______________________________________________________________________

## Problema 5: Paginação Ausente

### 🔍 O Que É?

Carregar todos os registros de uma vez sobrecarrega o banco de dados, a memória do servidor e o navegador. Com grandes volumes de dados, isso causa timeouts, crashes e péssima experiência do usuário.

### 🧠 Por Que Importa?

```python
# Carregando 100.000 livros de uma vez:
livros = Livro.objects.all()  # 100.000 registros

# Uso de memória:
# - Python: ~500MB
# - Banco de dados: 100.000 linhas enviadas
# - Rede: 50MB+ transferidos
# - Navegador: Congela enquanto renderiza 100.000 linhas
```

### 💻 O Código

**❌ Versão Lenta:**

```python
def paginacao_lenta(request):
    """Carrega TODOS os livros - pode crashar com grandes volumes!"""
    livros = Livro.objects.select_related('autor').all()[:200]
    return render(request, 'livros.html', {'livros': livros})
```

**✅ Versão Rápida:**

```python
from django.core.paginator import Paginator

def paginacao_rapida(request):
    """Carrega apenas 20 livros por vez - eficiente!"""
    qs = Livro.objects.select_related('autor').only(
        'titulo', 'ano_publicacao', 'autor__nome'
    )
    paginator = Paginator(qs, per_page=20)
    page = paginator.get_page(request.GET.get('page', 1))
    return render(request, 'livros.html', {'page': page})
```

**SQL Gerado (Página 1):**

```sql
-- Query 1: Contagem total
SELECT COUNT(*) FROM livro;

-- Query 2: Página atual
SELECT livro.id, livro.titulo, livro.ano_publicacao, autor.nome
FROM livro
INNER JOIN autor ON livro.autor_id = autor.id
LIMIT 20 OFFSET 0;
```

### 📝 Template com Paginação

```html
<h1>Livros - Página {{ page.number }} de {{ page.paginator.num_pages }}</h1>

<ul>
{% for livro in page.object_list %}
    <li>{{ livro.titulo }} por {{ livro.autor.nome }}</li>
{% endfor %}
</ul>

<nav>
    {% if page.has_previous %}
        <a href="?page={{ page.previous_page_number }}">Anterior</a>
    {% endif %}
    
    <span>Página {{ page.number }}</span>
    
    {% if page.has_next %}
        <a href="?page={{ page.next_page_number }}">Próxima</a>
    {% endif %}
</nav>
```

### 📊 Impacto na Performance

| Livros | Sem Paginação | Com Paginação (20/página) |
|--------|--------------|--------------------------|
| 100 | 100 carregados | 20 carregados |
| 1.000 | 1.000 carregados | 20 carregados |
| 100.000 | 💥 Crash | 20 carregados |

### 🎯 Boas Práticas de Paginação

```python
# Sempre combine com select_related e only
qs = Livro.objects.select_related('autor').only(
    'titulo', 'autor__nome'
)
paginator = Paginator(qs, per_page=20)

# Lide com números de página inválidos
page = paginator.get_page(request.GET.get('page', 1))
# Automaticamente lida com page=0, page=9999, etc.
```

______________________________________________________________________

## Como Identificar Esses Problemas

### 1. Django Debug Toolbar

Instale o Django Debug Toolbar para ver a contagem de queries por request:

```bash
pip install django-debug-toolbar
```

Adicione ao `settings.py`:

```python
INSTALLED_APPS = [
    'debug_toolbar',
    # ...
]

MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
    # ...
]
```

Visite `/__debug__/` em desenvolvimento para ver as queries.

### 2. Django Silk

Para profiling em produção:

```bash
pip install django-silk
```

Visite `/silk/` para ver o profiling de requests.

### 3. Middleware de Contagem de Queries

Crie um middleware personalizado para contar queries:

```python
# middleware.py
class QueryCountMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        from django.db import connection
        initial = len(connection.queries)
        response = self.get_response(request)
        query_count = len(connection.queries) - initial
        response['X-Query-Count'] = query_count
        return response
```

______________________________________________________________________

## Resumo: Referência Rápida

```python
# ═══════════════════════════════════════════════════════════
# FOLHA DE DICAS DE OTIMIZAÇÃO
# ═══════════════════════════════════════════════════════════

# 1. ForeignKey → select_related (JOIN)
Livro.objects.select_related('autor')

# 2. ManyToMany → prefetch_related (2 queries)
Livro.objects.prefetch_related('tags')

# 3. Carregar apenas campos necessários
Livro.objects.only('titulo', 'preco')

# 4. Contar eficientemente (não len()!)
Livro.objects.count()

# 5. Sempre paginar
from django.core.paginator import Paginator
Paginator(qs, per_page=20)

# ═══════════════════════════════════════════════════════════
# COMBINANDO OTIMIZAÇÕES
# ═══════════════════════════════════════════════════════════

# Melhor: select_related + only + paginate
livros = Livro.objects.select_related('autor').only(
    'titulo', 'ano_publicacao', 'autor__nome'
)
paginator = Paginator(livros, per_page=20)
page = paginator.get_page(request.GET.get('page', 1))
```

______________________________________________________________________

## Experimente Você Mesmo: Aplicação Demo

🚀 **Explore a aplicação demo ao vivo e teste todas as otimizações na prática!**

A aplicação demo inclui:

- **5 demonstrações interativas** - Uma para cada problema de performance
- **Comparação lado a lado** - Implementações Lentas vs Rápidas
- **Contagem de queries em tempo real** - Veja exatamente quantas queries cada abordagem gera
- **Modo Playground** - Teste cenários personalizados

### Links da Demo ao Vivo

| Problema | Versão Lenta | Versão Rápida |
|----------|--------------|--------------|
| N+1 ForeignKey | [`/books/slow/`](/books/slow/) | [`/books/fast/`](/books/fast/) |
| N+1 ManyToMany | [`/tags/slow/`](/tags/slow/) | [`/tags/fast/`](/tags/fast/) |
| SELECT * | [`/fields/slow/`](/fields/slow/) | [`/fields/fast/`](/fields/fast/) |
| len() vs count() | [`/count/slow/`](/count/slow/) | [`/count/fast/`](/count/fast/) |
| Paginação | [`/paginate/slow/`](/paginate/slow/) | [`/paginate/fast/`](/paginate/fast/) |

### Playground Interativo

Teste todos os problemas interativamente em: [**`/playground/`**](/playground/)

______________________________________________________________________

## Conclusão

Esses 5 problemas de performance são os **maiores culpados** atrás de aplicações Django lentas. A boa notícia? Eles são todos **fáceis de resolver** uma vez que você sabe o que procurar:

1. **N+1 ForeignKey** → Use `select_related()`
1. **N+1 ManyToMany** → Use `prefetch_related()`
1. \*\*SELECT \*\*\* → Use `only()` ou `defer()`
1. **len()** → Use `.count()`
1. **Todos os registros** → Use paginação

Comece a aplicar essas otimizações hoje e veja a performance da sua aplicação disparar! 🚀

______________________________________________________________________

## Referências

- [Documentação do Django ORM](https://docs.djangoproject.com/pt-br/topics/db/queries/)
- [select_related()](https://docs.djangoproject.com/pt-br/ref/models/querysets/#select-related)
- [prefetch_related()](https://docs.djangoproject.com/pt-br/ref/models/querysets/#prefetch-related)
- [only() e defer()](https://docs.djangoproject.com/pt-br/ref/models/querysets/#only-and-defer)
- [Paginator do Django](https://docs.djangoproject.com/pt-br/topics/pagination/)
- [Django Debug Toolbar](https://django-debug-toolbar.readthedocs.io/)
- [Django Silk](https://github.com/jazzband/django-silk)
- [Aplicação Demo no GitHub](https://github.com/alisonamerico/django-perf-demo)
