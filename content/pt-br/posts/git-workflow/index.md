---
title: "Fluxo de Trabalho no Git: do commit ao rebase, sem trauma"
date: 2026-07-29
draft: false
tags: ["git", "github", "versionamento", "feature-branch", "rebase", "workflow"]
categories: ["ferramentas"]
description: "Um guia completo sobre o fluxo de trabalho no Git: os três estados do repositório, feature branches, merge vs rebase, resolução de conflitos e como corrigir os erros mais comuns do dia a dia."
cover: git-workflow.png
---

Se você já trabalhou em qualquer projeto de software em equipe, sabe que o Git deixou de ser "só um detalhe técnico" para se tornar parte do seu raciocínio diário. Entender **como** e **quando** usar cada comando é o que separa quem só "sobrevive" ao Git de quem realmente flui com ele.

Neste artigo eu quero ir além do básico (`init`, `add`, `commit`, `push`) e mostrar o fluxo de trabalho que uso no dia a dia, incluindo **feature branches**, **rebase** e as correções de erro mais comuns que aparecem no trabalho real.

## O que você vai ver

- Configuração inicial e os três estados do Git
- Como estruturar o trabalho com *feature branches*
- Merge vs Rebase: quando usar cada um
- Como resolver conflitos sem entrar em pânico
- Como desfazer e corrigir erros: `amend`, `reset`, `revert`, `stash`
- Boas práticas para não sofrer em equipe

## Configuração inicial do Git

Antes de qualquer coisa, configure seu nome e e-mail - eles ficam gravados em cada commit que você fizer:

```bash
git config --global user.name "Seu Nome"
git config --global user.email "[email protected]"
```

Alguns ajustes que eu recomendo desde o primeiro dia:

```bash
# define "main" como nome padrão para novas branches
git config --global init.defaultBranch main

# editor padrão para mensagens de commit e rebase interativo
git config --global core.editor "nvim"

# evita fast-forward silencioso em pulls (evita histórico confuso)
git config --global pull.rebase true
```

Para iniciar um repositório novo:

```bash
git init
```

Ou, para trabalhar em um projeto existente:

```bash
git clone [email protected]:usuario/repositorio.git
```

## Os três estados do Git

O Git organiza o seu trabalho local em três áreas:

1. **Working Directory** - os arquivos como estão no seu disco agora.
2. **Staging Area (Index)** - uma área intermediária com o que *vai* entrar no próximo commit.
3. **Repository (HEAD)** - o histórico de commits já confirmados.

Visualmente, o fluxo entre essas áreas funciona assim:

```text
Working Directory        Staging Area (Index)         Repository (HEAD)
------------------        ---------------------         ------------------
 arquivo.py (editado)  --git add-->  arquivo.py   --git commit-->   o---o---o
                                     (pronto p/                              ^
                                      commit)                              HEAD

        <---------------------- git reset ----------------------------------
        <----------------------------- git restore --------------------------
```

- `git add` move do *Working Directory* para o *Staging*.
- `git commit` move do *Staging* para o *Repository*.
- `git reset` e `git restore` fazem o caminho inverso, cada um com um alcance diferente (você vai ver isso na seção de correção de erros).

```bash
# ver o estado atual dos arquivos
git status

# mover um arquivo específico para o staging
git add caminho/do/arquivo.py

# mover tudo que foi modificado
git add .

# criar um commit com o que está no staging
git commit -m "feat: adiciona validação de CPF no formulário"
```

Uma boa prática aqui é escrever mensagens de commit no padrão [Conventional Commits](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `refactor:`, `docs:`, `chore:`...). Isso facilita gerar changelog automático e entender o histórico rapidamente.

## Feature Branch: isolando o trabalho

A branch `main` (ou `master`, em projetos mais antigos) deve representar sempre um código estável, pronto para ir para produção. Por isso, **não trabalhamos direto nela**.

O fluxo de *feature branch* consiste em criar uma branch isolada para cada funcionalidade, correção ou experimento:

```bash
# cria e já muda para a nova branch
git checkout -b feature/login-com-google

# forma mais moderna, equivalente ao checkout -b
git switch -c feature/login-com-google
```

Algumas convenções de nomenclatura comuns:

- `feature/nome-da-funcionalidade`
- `fix/nome-do-bug`
- `hotfix/correcao-urgente-em-producao`
- `chore/atualiza-dependencias`

Trabalhando assim, você ganha algumas vantagens importantes:

- Pode errar, testar e refazer commits **sem afetar a main**.
- Fica fácil abrir um Pull Request específico para revisão.
- Vários desenvolvedores podem trabalhar em paralelo sem pisar no trabalho um do outro.
- Se algo der muito errado, basta descartar a branch.

Visualmente, o histórico fica assim enquanto a feature está em desenvolvimento:

```text
main       o---o---o---------------------o---o---o
                    \
feature/login        o---o---o---o
                      (commits isolados,
                       não afetam a main)
```

Enquanto você trabalha na sua feature, a `main` continua recebendo outros merges. Por isso é importante manter sua branch atualizada - e é aqui que entram merge e rebase.

## Merge vs Rebase

Os dois comandos resolvem o mesmo problema - trazer mudanças de uma branch para outra - mas de formas diferentes, e isso importa bastante no histórico do projeto.

### `git merge`

O merge cria um **novo commit de mesclagem** (merge commit) que junta os dois históricos:

```bash
git checkout main
git pull origin main

git checkout feature/login-com-google
git merge main
```

Vantagens: preserva o histórico exatamente como ele aconteceu, é mais seguro para branches compartilhadas e não reescreve commits já enviados.

Desvantagem: o histórico pode ficar "poluído" com vários commits de merge, especialmente em projetos com muitas branches de curta duração.

```text
Antes do merge:

main       o---o---o
                    \
feature              o---o---o

Depois do merge (novo commit de mesclagem "M"):

main       o---o---o-------------M
                    \            /
feature              o---o---o--/
```

### `git rebase`

O rebase **reaplica** os commits da sua branch em cima do último estado da branch base, como se você tivesse começado seu trabalho a partir daquele ponto mais recente:

```bash
git checkout feature/login-com-google
git fetch origin
git rebase origin/main
```

O resultado é um histórico linear, sem commits de merge extras - como se toda a feature tivesse sido desenvolvida do zero em cima da `main` mais atual.

```text
Antes do rebase:

main       o---o---o---o        <- main avançou com novos commits
                    \
feature              o---o---o  <- seus commits antigos

Depois do "git rebase origin/main":

main       o---o---o---o
                        \
feature                  o'---o'---o'   <- seus commits reaplicados
                                          (hashes novos!)
```

> ⚠️ **Regra de ouro do rebase:** nunca faça rebase de uma branch que já foi compartilhada com outras pessoas (ex: `main`, `develop`) e que já foi *pushada*. Rebase reescreve o histórico (novos hashes de commit), e isso quebra o trabalho de quem já baixou os commits antigos. Use rebase livremente na **sua** branch de feature, antes de abrir o PR ou enquanto só você trabalha nela.

### Rebase interativo: limpando o histórico

Antes de abrir um Pull Request, é comum "limpar" a sequência de commits com o rebase interativo:

```bash
git rebase -i HEAD~5
```

Isso abre um editor onde você pode, para cada um dos últimos 5 commits:

- `pick` - manter o commit como está
- `reword` - manter as mudanças, mas editar a mensagem
- `squash` ou `fixup` - juntar o commit com o anterior
- `drop` - descartar o commit

É muito útil para transformar vários commits do tipo "wip", "ajuste", "corrige typo" em um histórico limpo e compreensível antes de enviar para revisão.

### Quando usar cada um?

| Situação | Recomendação |
|---|---|
| Atualizar sua branch de feature local com a `main` | `rebase` |
| Trazer uma feature finalizada para a `main` via Pull Request | `merge` (geralmente feito pela interface do GitHub/GitLab) |
| Branch já compartilhada com o time | `merge` (nunca rebase) |
| Limpar commits antes de abrir o PR | `rebase -i` |

## Resolvendo conflitos

Conflitos acontecem quando o Git não consegue decidir sozinho como combinar duas mudanças na mesma linha de código. Tanto `merge` quanto `rebase` podem gerar conflitos - a diferença é como você resolve.

### Exemplo prático

Imagine que, na `main`, alguém alterou a função de desconto:

```python
# main
def calcular_desconto(valor):
    return valor * 0.9  # 10% de desconto
```

E, na sua branch `feature/desconto-especial`, você alterou a mesma linha:

```python
# feature/desconto-especial
def calcular_desconto(valor):
    return valor * 0.85  # 15% de desconto
```

Ao rodar `git merge main` (ou `git rebase origin/main`), o Git não sabe qual das duas versões manter e marca o arquivo como conflitante:

```bash
$ git merge main
Auto-merging app.py
CONFLICT (content): Merge conflict in app.py
Automatic merge failed; fix conflicts and then commit the result.
```

Abrindo `app.py`, você verá os marcadores de conflito:

```python
def calcular_desconto(valor):
<<<<<<< HEAD
    return valor * 0.9  # 10% de desconto
=======
    return valor * 0.85  # 15% de desconto
>>>>>>> feature/desconto-especial
```

- Tudo entre `<<<<<<< HEAD` e `=======` é a versão da branch em que você está.
- Tudo entre `=======` e `>>>>>>> feature/...` é a versão que está sendo trazida.

Você resolve editando manualmente para o resultado desejado (aqui, decidindo manter os 15%) e removendo os marcadores:

```python
def calcular_desconto(valor):
    return valor * 0.85  # 15% de desconto (decisão do time de vendas)
```

E então finaliza o processo:

```bash
# durante um merge com conflito
git status                    # mostra os arquivos em conflito
git add app.py                # marca o conflito como resolvido
git commit                    # finaliza o merge

# durante um rebase com conflito
git status
git add app.py
git rebase --continue          # continua o rebase, NÃO use "git commit"

# para abortar e voltar ao estado anterior ao conflito
git merge --abort
git rebase --abort
```

### Resolvendo rápido, mantendo um dos lados

Quando você sabe de antemão que quer manter inteiramente uma das versões (sem misturar), pode pular a edição manual:

```bash
# mantém a versão da sua branch atual
git checkout --ours app.py

# mantém a versão que está sendo integrada
git checkout --theirs app.py

git add app.py
git commit   # ou git rebase --continue
```

> ⚠️ **Cuidado com o rebase:** durante um `rebase`, o significado de `--ours` e `--theirs` é **invertido** em relação ao merge, porque o Git está reaplicando seus commits *sobre* a outra branch. Na dúvida, sempre confira o conteúdo do arquivo antes de commitar - não confie apenas no nome da flag.

Dica: rode `git rebase --abort` sem medo se as coisas saírem do controle - você volta exatamente ao estado antes de começar o rebase.

## Git Worktree: complemento (não substituto) da feature branch

Um erro comum é achar que `git worktree` é "outra forma de criar feature branch". Na verdade, são coisas diferentes que resolvem problemas diferentes:

- **Feature branch** é um conceito do histórico do Git: uma referência que aponta para uma sequência isolada de commits.
- **Worktree** é um mecanismo que permite ter **mais de um diretório de trabalho** com branches diferentes fazendo checkout **ao mesmo tempo**, todos compartilhando o mesmo repositório `.git` (mesmos objetos, mesmo histórico, mesmas branches).

Ou seja: você continua usando feature branches normalmente - o worktree só muda *onde* e *como* você acessa cada uma delas no seu disco.

### O problema que o worktree resolve

Sem worktree, se você está no meio de uma feature (com mudanças não commitadas) e precisa atender um hotfix urgente, o fluxo típico é:

```bash
git stash                      # guarda o trabalho em andamento
git switch main
git switch -c hotfix/corrige-login
# ... resolve o hotfix, commita, envia ...
git switch feature/nova-funcionalidade
git stash pop                  # recupera o trabalho
```

Funciona, mas é um vai-e-vem arriscado - principalmente se você tem arquivos gerados, servidores rodando ou testes em andamento que dependem do estado atual dos arquivos.

Com `git worktree`, você cria uma **pasta adicional** apontando para outra branch, sem mexer no seu diretório de trabalho atual:

```bash
# cria um novo worktree em ../projeto-hotfix, na branch hotfix/corrige-login
git worktree add ../projeto-hotfix -b hotfix/corrige-login main

# lista os worktrees ativos
git worktree list

# remove o worktree quando não precisar mais dele
git worktree remove ../projeto-hotfix
```

Agora você tem duas pastas, cada uma com seu próprio *working directory*, checkouts independentes, mas compartilhando o mesmo histórico de commits:

```text
~/projetos/meu-app              (branch: feature/nova-funcionalidade)
~/projetos/meu-app-hotfix       (branch: hotfix/corrige-login)

                 mesmo repositório .git por trás dos dois
```

### Feature branch vs Worktree

| Aspecto | Feature Branch | Git Worktree |
|---|---|---|
| O que é | Uma linha isolada de commits dentro do histórico | Um diretório de trabalho extra, ligado ao mesmo repositório |
| Troca de contexto | `git switch`/`checkout` troca os arquivos na mesma pasta | Basta abrir outra pasta/terminal - nada muda na pasta atual |
| Precisa de `stash`? | Sim, se houver mudanças não commitadas | Não - cada worktree mantém seu próprio estado |
| Uso típico | Isolar o desenvolvimento de uma funcionalidade | Trabalhar em duas branches ao mesmo tempo (ex: hotfix urgente + feature em andamento), rodar testes em paralelo, ou revisar um PR sem sair da sua branch atual |
| Substitui o outro? | Não | Não - os dois se complementam |

Na prática: você continua criando uma feature branch para cada funcionalidade; o worktree é só uma ferramenta a mais para quando você precisa **estar em duas branches fisicamente ao mesmo tempo**, sem o custo de clonar o repositório inteiro de novo.

## Corrigindo erros comuns

Aqui estão os "socorros" que mais uso no dia a dia.

### Esqueci de incluir um arquivo no último commit

```bash
git add arquivo-esquecido.py
git commit --amend --no-edit
```

### Quero mudar a mensagem do último commit

```bash
git commit --amend -m "fix: corrige mensagem anterior"
```

> ⚠️ `--amend` reescreve o commit (novo hash). Só use em commits que **ainda não foram enviados** (`push`) ou em branches só suas.

### Fiz commit na branch errada

```bash
# desfaz o commit mas mantém as mudanças no working directory
git reset --soft HEAD~1

# muda para a branch correta
git switch nome-da-branch-certa

# refaz o commit lá
git commit -m "feat: minha mudança na branch certa"
```

### Preciso desfazer o último commit

```bash
# mantém as alterações no staging
git reset --soft HEAD~1

# mantém as alterações no working directory (fora do staging)
git reset --mixed HEAD~1

# descarta as alterações completamente (cuidado, é destrutivo!)
git reset --hard HEAD~1
```

### Já enviei o commit errado e outras pessoas já puxaram esse histórico

Nesse caso, **não** use `reset` (que reescreve histórico). Use `revert`, que cria um novo commit desfazendo as mudanças, sem apagar o histórico:

```bash
git revert <hash-do-commit>
```

### Quero descartar mudanças não commitadas em um arquivo

```bash
# forma moderna
git restore arquivo.py

# forma antiga, ainda muito usada
git checkout -- arquivo.py
```

### Preciso trocar de branch, mas tenho mudanças no meio do caminho

```bash
git stash                     # guarda as mudanças temporariamente
git switch outra-branch
# ... faz o que precisa ...
git switch branch-original
git stash pop                 # traz as mudanças de volta
```

### Enviei um push, mas preciso corrigir o histórico da minha própria feature branch

```bash
git push --force-with-lease origin feature/login-com-google
```

> `--force-with-lease` é mais seguro que `--force` puro: ele falha se alguém mais enviou commits para a branch remota depois da sua última atualização, evitando que você sobrescreva o trabalho de outra pessoa sem perceber.

## Atualizando o repositório remoto

```bash
# baixa as referências remotas sem alterar sua branch local
git fetch origin

# baixa e já integra (merge ou rebase, conforme configuração)
git pull origin main

# envia sua branch para o remoto
git push origin feature/login-com-google
```

## Um fluxo completo, do início ao fim

1. Atualize a `main` local: `git checkout main && git pull`
2. Crie a feature branch: `git switch -c feature/nova-funcionalidade`
3. Trabalhe em commits pequenos e com mensagens claras
4. Antes de abrir o PR, atualize com a main: `git fetch && git rebase origin/main`
5. Se necessário, limpe o histórico: `git rebase -i HEAD~N`
6. Envie: `git push -u origin feature/nova-funcionalidade`
7. Abra o Pull Request e peça revisão
8. Após aprovado, faça o merge (geralmente *squash and merge* ou *merge commit*, dependendo do padrão do time)
9. Delete a branch local e remota depois do merge

## Conclusão

Dominar o fluxo básico do Git (`init`, `add`, `commit`, `push`) é o primeiro passo, mas o que realmente muda sua produtividade em equipe é entender **feature branches**, saber escolher entre **merge e rebase**, e ter confiança para **corrigir seus próprios erros** sem entrar em pânico.

Com esses comandos na manga, você deixa de ter medo do Git e passa a usá-lo como o que ele é: uma ferramenta poderosa para trabalhar em equipe com segurança.

Ficou com alguma dúvida sobre algum desses comandos? Deixe nos comentários - vou adorar ajudar!

## Referências

- [Pro Git Book (gratuito, em português)](https://git-scm.com/book/pt-br/v2)
- [Documentação oficial do Git](https://git-scm.com/docs)
- [git-rebase - documentação oficial](https://git-scm.com/docs/git-rebase)
- [git-worktree - documentação oficial](https://git-scm.com/docs/git-worktree)
- [git-merge - documentação oficial](https://git-scm.com/docs/git-merge)
- [Atlassian Git Tutorials](https://www.atlassian.com/git/tutorials)
- [GitHub Docs - Resolvendo um conflito de merge](https://docs.github.com/pt/pull-requests/collaborating-with-pull-requests/addressing-merge-conflicts/resolving-a-merge-conflict-using-the-command-line)
- [Conventional Commits](https://www.conventionalcommits.org/pt-br/v1.0.0/)
