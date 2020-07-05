---
layout: post
title: "Usando Pipenv para gerar ambiente virtual e instalação de dependências"
date: 2019-04-03 21:23:10
image: '/assets/img/usando-pipenv-para-gerar-ambiente-virtual-e-instalacao-de-dependencias/pipenv.png'
category: 'tutorial'
tags:
- python
- pipenv
- virtualenv
---

Para instalar:

```bash
$ pip install pipenv 
```
Obs.: Por padrão o pipenv cria o ambiente virtual dentro de uma pasta específica e interna dele. Contudo será criado o virtualenv dentro da pasta do projeto porque precisamos saber onde fica esse virtualenv para configurar no pycharm. 

Para isso é preciso editar o `.bashrc` 
```bash
$ sudo nano ~/.bashrc
```
e nós iremos exportar uma variável de ambiente:
```bash
export PIPENV_VENV_IN_PROJECT=1
```
Com isso irá alterar o comportamento do pipenv para que ele crie o ambiente virtual dentro do meu projeto com o nome .venv

Para ativar o ambiente virtual:
```bash
$ pipenv shell
```


