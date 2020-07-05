---
layout: post
title: "O básico sobre Poetry: Gerenciamento de Dependências para Python"
date: 2020-07-05 10:36:10
image: "/assets/img/o-basico-poetry/poetry.jpeg"
category: "tutorial"
tags:
  - python
  - poetry
  - virtualenv
---

## Definição:

> Poetry ajuda a declarar, gerenciar e instalar dependências de projetos Python, garantindo que você tenha a pilha certa em qualquer lugar.

## Versões do Python suportadas:

```
2.7 e 3.4+
```

## Referências:

Obs.: Esse passo a passo foi criado com base na [documentação](https://python-poetry.org/docs/) e do vídeo do Canal [CodeShow](https://www.youtube.com/watch?v=_XszPRFHQQ4&t=852s) (Bruno Rocha):

## O por que de usar o Poetry:

```
https://github.com/python-poetry/poetry#why
```

Obs.: Minha intenção maior aqui é apresentar mais uma opção de ferramenta para gerenciar pacotes Python e não dizer que ferramenta A é melhor que B.

## Instalação:

```
❯ curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python
```

Obs.: Executar esse script no sistema (não precisa criar dentro do ambiente virtual).

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

Se estiver usando o bash:

```bash
❯ nano ~/.bashrc
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
❯ poetry new meuprojeto
```

![new-project](/assets/img/o-basico-poetry/new-project.png)

Obs.: Note que são criadas duas pastas com o mesmo nome "meuprojeto".
O primeiro é respectivo ao nome do projeto propriamente dito, já o segungo faz referência ao nome do pacote. É importante ter esta pasta interna (pasta com o `__init__.py`) pois é ela que formará o pacote a ser distribuído.

Esse é o comportamento padrão, mas se assim como eu, você não gostar dessa estrutura com os nomes das pastas duplicadas, existem outras opções para configurar a estrutura do projeto.

Com a opção `--name` é possível informar qual o nome do pacote que se deseja criar:

```bash
❯ poetry new meuprojeto --name meupacote
```

Ex.:

![new-project-package](/assets/img/o-basico-poetry/new-project-package.png)

A outra opção é utilizar o comando `init`.

```bash
❯ poetry init
```

Este comando o ajudará a criar um arquivo `pyproject.toml` interativamente, solicitando que você forneça informações básicas sobre o seu pacote.

A saída final se parecerá com isso (vai depender de como você vai querer configurar):

```
~/Workspace/poetry-exemplo
❯ poetry init

This command will guide you through creating your pyproject.toml config.

Package name [poetry-exemplo]:  meuprojeto
Version [0.1.0]:
Description []:  Exemplo criação de projetos com poetry.
Author [alisonamerico <alison.americo@gmail.com>, n to skip]:
License []:
Compatible Python versions [^3.8]:

Would you like to define your main dependencies interactively? (yes/no) [yes] yes
You can specify a package in the following forms:
  - A single name (requests)
  - A name and a constraint (requests ^2.23.0)
  - A git url (git+https://github.com/python-poetry/poetry.git)
  - A git url with a revision (git+https://github.com/python-poetry/poetry.git#develop)
  - A file path (../my-package/my-package.whl)
  - A directory (../my-package/)
  - A url (https://example.com/packages/my-package-0.1.0.tar.gz)

Search for package to add (or leave blank to continue):

Would you like to define your development dependencies interactively? (yes/no) [yes] yes
Search for package to add (or leave blank to continue):

Generated file

[tool.poetry]
name = "meuprojeto"
version = "0.1.0"
description = "Exemplo criação de projetos com poetry."
authors = ["alisonamerico <alison.americo@gmail.com>"]

[tool.poetry.dependencies]
python = "^3.8"

[tool.poetry.dev-dependencies]

[build-system]
requires = ["poetry>=0.12"]
build-backend = "poetry.masonry.api"


Do you confirm generation? (yes/no) [yes] yes

~/Workspace/poetry-exemplo is 📦 v0.1.0 via 🐍 v3.8.3 took 1m29s
❯ code .

```

![new-project-init](/assets/img/o-basico-poetry/new-project-init.png)

O arquivo `pyproject.toml` é o mais importante. Ele irá gerenciar seu projeto e suas dependências.

Depois você pode criar as pastas como bem desejar.

Criar ambiente virtual dentro da pasta do projeto:

```bash
❯ poetry env use versão-do-python
Ex.: poetry env use 3.8
```

Note que ao executar o comando, a pasta `.venv` será criada dentro do diretório:

`/.cache/pypoetry/virtualenvs/`

Ex.:

```bash
❯ poetry env use 3.8
Creating virtualenv mypoetry-obe-pgZu-py3.8 in /home/alison/.cache/pypoetry/virtualenvs
Using virtualenv: /home/alison/.cache/pypoetry/virtualenvs/mypoetry-obe-pgZu-py3.8
```

Mas, se ao criar um virtualenv, você quiser que a pasta fique dentro do projeto que está trabalhando, basta executar o comando:

```bash
❯ poetry config virtualenvs.in-project true
```

e será criado a pasta do ambiente virtual `.venv` dentro do projeto.

Para ativar o ambiente virtual:

```bash
❯ poetry shell

```

Instalar uma pacote:

```bash
❯ poetry add nome-do-pacote
Ex.: poetry add django
```

Instalar uma pacote de desenvolvimento (ou seja, aquele pacote que não é para ir produção):

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

Comenta aí em baixo o que você achou!

Espero que tenha ajudado! :)
