---
layout: post
title: "O básico sobre Poetry: Gerenciamento de Dependências para Python"
date: 2020-07-04 21:30:10
image: "/assets/img/o-basico-poetry/poetry.jpeg"
category: "tutorial"
tags:
  - python
  - poetry
  - virtualenv
---

## Definição:

Poetry ajuda a declarar, gerenciar e instalar dependências de projetos Python, garantindo que você tenha a pilha certa em qualquer lugar.

## Versões do Python suportadas:

2.7 e 3.4+

## Referências:

Obs.: Esse passo a passo foi criado com base na [documentação](https://python-poetry.org/docs/) e do vídeo do Canal [CodeShow](https://www.youtube.com/watch?v=_XszPRFHQQ4&t=852s) (Bruno Rocha):

## O por que de usar o Poetry:

https://github.com/python-poetry/poetry#why

## Instalação:

❯ curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python

Obs.: Executar esse script no sistema(não precisa criar dentro do ambiente virtual).

Aparecerá ao parecido com isso:

```
Retrieving Poetry metadata

# Welcome to Poetry!

This will download and install the latest version of Poetry,
a dependency and package manager for Python.

It will add the `poetry` command to Poetry's bin directory, located at:

$HOME/.poetry/bin

This path will then be added to your `PATH` environment variable by
modifying the profile files located at:

$HOME/.profile
$HOME/.zprofile

You can uninstall at any time with `poetry self:uninstall`,
or by executing this script with the --uninstall option,
and these changes will be reverted.

Installing version: 0.12.2

- Downloading poetry-0.12.2-darwin.tar.gz (7.00MB)

Poetry (0.12.2) is installed now. Great!

To get started you need Poetry's bin directory (\$HOME/.poetry/bin) in your `PATH`
environment variable. Next time you log in this will be done
automatically.

To configure your current shell run `source $HOME/.poetry/env`

```

## Configuração:

Note que é informado que será adicionado à sua variável de ambiente `PATH`
modificando os arquivos de perfil localizados em:

```bash
$HOME/.profile
$HOME/.zprofile
```

Mas para zsh não tem /.zprofile, no geral os arquivos de configurações ficam no /.zshrc

Se estiver usando terminal zsh, será preciso configurar o .zshrc:

```bash
❯ nano ~/.zshrc
```

insira a seguinte comando:

```bash
export PATH="$HOME/.poetry/bin:$PATH"
```

com isso conseguirá executar os comandos do poetry.

Para testar execute:

```bash
❯ poetry --version
```

## Comandos Básicos

Criar novo projeto:

```bash
❯ poetry new nomedoprojeto
```

Criar ambiente virtual dentro da pasta do projeto:

```bash
❯ poetry env use versão-do-python
Ex.: poetry env use 3.8
```

Para ativar o ambiente virtual:

```bash
❯ poetry shell

```

Instalar uma pacote:

```bash
❯ poetry add nome-do-pacote
Ex.: poetry add django
```

Instalar uma pacote de desenvolvimento(ou seja, aquele pacote que não é para ir produção):

```bash
❯ poetry add --dev nome-do-pacote
Ex.: poetry add --dev pytest
```

Remover um pacote:

```bash
❯ poetry remove --dev nome-do-pacote
Ex.: poetry remove --dev pytest
```

Atualizar as versões dos pacotes:

```bash
❯ poetry update
```
