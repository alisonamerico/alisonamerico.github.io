---
layout: post
title: "Instalando Pyenv para isolar o ambiente do sistema"
date: 2019-03-14 22:47:40
image: '/assets/img/instalando-pyenv-para-isolar-o-ambinte-do-sistema/o-que-eh-pyenv.png'
category: 'tutorial'
tags:
- python
- pyenv
---


Como está no título do próprio repositório [pyenv](https://github.com/pyenv/pyenv)
"Simple Python version management", usando a tradução livre, é um gerenciador de versões do Python instaladas no S.O. Onde podemos indicar qual a versão do python desejamos utilizar para não correr o risco de danificar o Python que já vem instalado no sistema.

Obs.: Este tutorial foi criado com base nas aulas do curso [Python Pro](https://www.python.pro.br/)


Primeiro precisamos instalar o [pyenv-installer](https://github.com/pyenv/pyenv-installer), nele contém as instruções e um shell script que irá facilitar a instalação.

```bash
$ curl -L https://github.com/pyenv/pyenv-installer/raw/master/bin/pyenv-installer | bash
```

Após o término do download, irá mostrar as linhas que devem ser inseridas no .bashrc

Obs.: no repositório do pyenv-installer recomenda que coloque no **.bashrc** a linha abaixo:

```bash
$ eval "$(pyenv virtualenv-init -)"
```

Mas por estar utilizando a IDE Pycharm, não preciso inserir essa linha, pois a IDE já ativa o virtualenv env. Logo não precisamos do pyenv gerenciando isso. Caso você use outro editor insira a linha mencionada.
 
Vamos editar o .bashrc com o comando abaixo (estou usando o nano por conveniência, mas você pode utilizar o editor que preferir):

```bash
$ sudo nano ~/.bashrc 
```

Insira as seguintes linhas para ativar o pyenv automaticamante:

```bash
export PATH="~/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
```
Depois é só salvar e fechar.

Para verificar se o pyenv está instalado basta digitar:

```bash
$ pyenv
```

Para conferir a lista de todos os interpretadores disponíveis rodando no pyenv:

```bash
$ pyenv install -l (“ l ” de listar)
```

Irá mostrar várias versões, porém o que nós queremos nesse momento são as duas versões do python 2 e 3 mais atuais (que não seja as versões de desenvolvimento “dev”). No momento deste tutorial encontram-se nas seguintes versões:

Python 2
```bash
> 2.7.15
```

Python 3
```bash
> 3.7.2
```

Para instalar as versões:

```bash
$ pyenv install 3.7.2
```
e depois 
```bash
$ pyenv install 2.7.15
```

Você pode conferir as versoes disponiveis no pyenv:

```bash
$ pyenv versions
  system
    * 2.7.15 (set by /home/alison/.pyenv/version)
    * 3.7.2 (set by /home/alison/.pyenv/version)
```
Obs.: Como estas duas versões já estão instaladas no meu sistema, elas aparecem com o “ * ” asterisco, mas no caso de não está instalado, o que irá aparece com o  “ * ” asterisco, será apenas o “system”.

Para setar o python global:

```bash
$ pyenv global <versao>
```

Nesse caso ficará:

```bash
$ pyenv global 3.7.2
```

Verificar a versão com comando **which python** irá mostrar justamente o python do pyenv

```bash
$ which python

/home/alison/.pyenv/shims/python
```

Verificar versão instalada:

```bash
$ python -V
```

Saída:

```bash
Python 3.7.2
```

Inclusive podemos setar várias versões de python nesse global:

```bash
$ pyenv global 3.7.2 2.7.15
```
Obs.: Nesse caso a ordem em que é colocada influência na instalação padrão que irá ficar no global. A versão 3.7.2 do python será identificada como padrão no global porque ela está na frente da versão 2.7.15.

Conferir as versões instaladas:

```bash
$ python2 -V
Python 2.7.15
```
```bash
$ python3 -V
Python 3.7.2
```
ou 
```bash
$ pyenv global
3.7.2
2.7.15
```
