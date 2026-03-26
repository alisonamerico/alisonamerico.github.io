---
title: Zoxide - O Comando cd Inteligente que Seu Terminal Merece
date: 2026-03-26
draft: false
tags: ["zoxide", "terminal", "cli", "produtividade", "shell", "fzf"]
categories: ["terminal"]
description: "Descubra o zoxide, um jump de diretório extremamente rápido escrito em Rust que aprende seus diretórios mais visitados e permite navegar com apenas alguns toques."
cover: zoxide-command.png
---

> **Nível:** iniciante  
> **Pré-requisitos:** conhecimento básico de linha de comando e terminal

---

## O Problema com o cd Tradicional

Se você passa um tempo significativo no terminal, conhece a dor de navegar por estruturas de diretórios profundas. Mesmo com o uso de **IA** hoje em dia, em algum momento **VOCÊ** precisará acessar seus diretórios de forma rápida e inteligente.

```bash
cd ~/projects/web-app/frontend/src/components
cd ../../../../  # Voltar para cima é igualmente "divertido"
cd /home/user/workspace/my-super-long-project-name
```

É repetitivo, lento e propenso a erros de digitação. Você provavelmente usa completion com tab constantemente, ou pior, simplesmente decorando caminhos que digita demais.

**E se existisse uma forma melhor?**

______________________________________________________________________

## O que é Zoxide?

**Zoxide** é um comando `cd` inteligente, inspirado em ferramentas como [z](https://github.com/rupa/z) e [autojump](https://github.com/joelmc/autojump). Ele aprende quais diretórios você usa com mais frequência e permite "saltar" para eles com apenas alguns toques.

### Principais Características

- **Extremamente rápido**: Escrito em Rust, o zoxide é incrivelmente rápido , mesmo com milhares de entradas em seu banco de dados.
- **Multiplataforma**: Funciona em Linux, macOS, Windows e BSD.
- **Independente de shell**: Suporta Bash, Zsh, Fish, PowerShell, Nushell e mais.
- **Correspondência aproximada (fuzzy)**: Encontra diretórios mesmo com correspondências parciais.
- **Modo interativo**: Integração nativa com fzf para seleção visual.
- **Baseado em frequência**: Combina frequência e recência para classificar diretórios de forma inteligente.

______________________________________________________________________

## Instalação

### Linux (Ubuntu/Debian)

```bash
# Script oficial (recomendado)
curl -sSfL https://raw.githubusercontent.com/ajeetdsouza/zoxide/main/install.sh | sh

# Ou via apt (Ubuntu 21.04+)
sudo apt install zoxide
```

### macOS

```bash
# Via Homebrew (recomendado)
brew install zoxide
```

### Windows

```bash
# Via winget
winget install ajeetdsouza.zoxide

# Ou via scoop
scoop install zoxide
```

______________________________________________________________________

## Configuração do Shell

Após instalar, você precisa inicializar o zoxide no seu shell. Adicione a linha apropriada ao arquivo de configuração do seu shell:

### Zsh (mais comum)

Adicione ao `~/.zshrc`:

```bash
eval "$(zoxide init zsh)"
```

### Bash

Adicione ao `~/.bashrc`:

```bash
eval "$(zoxide init bash)"
```

### Fish

Adicione ao `~/.config/fish/config.fish`:

```bash
zoxide init fish | source
```

> **Importante**: Certifique-se de reiniciar o terminal ou source no seu arquivo de configuração após adicionar esta linha.

______________________________________________________________________

## Uso Básico

Uma vez inicializado, o zoxide substitui seu `cd` com superpoderes. Aqui estão os comandos essenciais:

### Saltar para um Diretório

```bash
z blog            # Saltar para o diretório mais rankeado que corresponde a "blog"
z trabalho projeto    # Saltar para diretório que corresponde a "trabalho" e "projeto"
z projetos/blog   # Saltar para subdiretório que corresponde a "projetos" e "blog"
```

### Navegar Como um cd Normal

```bash
z ~/documentos    # Funciona como cd normal
z projetos/       # Entra em caminho relativo
z ..              # Sobe um nível
z -               # Entra no diretório anterior (como cd -)
```

### Modo Interativo (com fzf)

```bash
zi                 # Abre fzf para selecionar entre todos os diretórios conhecidos
zi blog            # Abre fzf filtrado para diretórios contendo "blog"
```

Se você não tem o fzf instalado, o zoxide ainda funciona , mas você deveria instalar o fzf para a experiência interativa completa:

```bash
# macOS
brew install fzf

# Linux
sudo apt install fzf   # ou: sudo dnf install fzf

# Windows
scoop install fzf
```

______________________________________________________________________

## Importante: Como o Zoxide Aprende Diretórios

Um mal-entendido comum é que o zoxide descobre automaticamente todos os diretórios do seu sistema. **Não funciona assim.** O zoxide aprende **apenas** de diretórios que você visita usando o próprio zoxide.

### Por que "no match found" acontece

Imagine que você cria um novo projeto:

```bash
mkdir ~/projects/novo-projeto
cd ~/projects/novo-projeto
z novo-projeto
```

**Resultado:**

```
zoxide: no match found
```

O diretório não existe no banco de dados do zoxide porque você usou `cd` para entrar, não `z`. Criar uma pasta com `mkdir` também não a adiciona ao zoxide.

### Como adicionar diretórios

**Opção 1: Usar `z` diretamente (recomendado)**

```bash
z ~/projects/novo-projeto   # Entra E adiciona ao banco de dados
```

**Opção 2: Usar `zoxide add`**

```bash
zoxide add ~/projects/novo-projeto   # Adiciona sem entrar
```

**Opção 3: Usar modo interativo**

```bash
zi   # Abre fzf - você pode digitar um novo caminho e ele será adicionado
```

### Uma vez adicionado, fica

Após a primeira visita com `z`, você pode saltar para lá a qualquer momento com apenas:

```bash
z novo-projeto   # Agora funciona!
```

O zoxide rastreia com que frequência e há quanto tempo você visita cada diretório, então seus diretórios mais usados sobem para o topo.

______________________________________________________________________

## Comandos de Consulta

Quer ver o que o zoxide sabe sem saltar? Use comandos de consulta:

```bash
zoxide query foo          # Mostra o caminho que 'z foo' saltaria
zoxide query -l           # Lista todos os diretórios rastreados com pontuações
zoxide query --score      # Mostra diretórios com suas pontuações de frequância
zoxide query -i foo       # Busca interativa
```

______________________________________________________________________

## Importando de Outras Ferramentas

Se você está migrando de autojump, z.lua ou zsh-z, você pode importar seus dados existentes:

```bash
# De autojump
zoxide import --from=autojump "/path/to/autojump/db"

# De z (bash/zsh)
zoxide import --from=z "path/to/z/db"
```

______________________________________________________________________

## Configuração Avançada

### Personalizando o Prefixo do Comando

Por padrão, o zoxide usa `z` e `zi`. Você pode mudar isso:

```bash
# Usar 'j' ao invés de 'z'
eval "$(zoxide init zsh --cmd j)"
```

### Variáveis de Ambiente

```bash
# Mostrar o diretório antes de saltar (como cd)
export _ZO_ECHO=1

# Excluir certos diretórios
export _ZO_EXCLUDE_DIRS="$HOME:$HOME/private/*"

# Personalizar opções do fzf
export _ZO_FZF_OPTS="--height 40% --reverse"
```

______________________________________________________________________

## Solução de Problemas

### "zoxide: command not found"

1. Verifique se o zoxide está instalado: `which zoxide`
1. Certifique-se que ~/.local/bin está no seu PATH:
   ```bash
   echo $PATH | grep -q ~/.local/bin || export PATH="$HOME/.local/bin:$PATH"
   ```
1. Verifique se sua config do shell tem a linha de init

### "no match found"

Este é o problema mais comum e geralmente acontece porque:

1. O diretório nunca foi visitado com zoxide (`cd` não conta!)
1. O diretório foi criado com `mkdir` mas nunca entrou com `z`
1. O nome do diretório é muito genérico e não corresponde a nenhuma entrada

**Soluções:**

```bash
# Visite com z (entra E adiciona ao banco de dados)
z /caminho/completo/para/diretorio

# Ou adicione manualmente sem entrar
zoxide add /caminho/completo/para/diretorio

# Ou use o modo interativo - você pode digitar novos caminhos lá
zi
```

______________________________________________________________________

## Por que Zoxide ao invés de Alternativas?

| Ferramenta | Tipo | Prós | Contras |
|------------|------|------|---------|
| **Zoxide** | Binário Rust | Rápido, moderno, bem mantido | Requer instalação |
| **[z](https://github.com/rupa/z)** (original) | Script shell | Simples, sem dependências | Mais lento, menos funcionalidades |
| **[autojump](https://github.com/joelmc/autojump)** | Python | Multiplataforma | Mais lento, menos intuitivo |
| **[fasd](https://github.com/clvv/fasd)** | Script shell | Acesso rápido a arquivos também | Complexo, curva de aprendizado maior |

O zoxide é geralmente considerado o padrão moderno para salto de diretórios devido à sua base em Rust, velocidade e desenvolvimento ativo.

______________________________________________________________________

## Minha Experiência

Depois de usar zoxide por um tempo, posso navegar para projetos que uso diariamente com apenas 2-3 teclas. Em vez de digitar `cd ~/projects/blog/alisonamerico.github.io` toda vez, eu simplesmente digito `z blog`. Ele aprende meu comportamento e rankea diretórios por frequência e recência.

A combinação de `z` para saltos rápidos e `zi` para seleção visual mudou completamente como navego no terminal. É uma dessas ferramentas que parece que sempre deveria ter existido.

______________________________________________________________________

## Conclusão

Aprender a usar zoxide é um dos upgrades com maior ROI (Retorno sobre Investimento) que você pode fazer no seu fluxo de trabalho do terminal. Leva minutos para instalar, requer configuração mínima e imediatamente começa a facilitar sua vida.

Experimente, seus dedos vão agradecer.

**Recursos:**

- [Zoxide GitHub](https://github.com/ajeetdsouza/zoxide)
- [Documentação Oficial](https://zoxide.dev)
