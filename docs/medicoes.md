# Medições: original vs melhorado

Regras de trabalho: medir o original antes de mexer em qualquer coisa. Uma alteração de cada vez. Medir depois de cada alteração. Anotar também o que não melhorou.

Condições da medição: imagem original construída a partir do commit `9715fef`, Docker Desktop no Windows, stack arrancada só com `compose.yml` (sem o override) e com segredos próprios da sessão. A tag `baseline` (`c34e874`) tem o mesmo Dockerfile, e a diferença é só a pasta `.github` removida. As medições de execução são uma amostra em repouso, feita depois de o healthcheck passar. Imagens guardadas: `backend:baseline` (original), `backend:nodev` (alteração 1), `backend:slim` (alteração 2, que inclui a 1) e `backend:nonroot` (alteração 4, que inclui as anteriores). A alteração 3 é só no `compose.yml`, por isso não tem imagem própria.

## Hipótese inicial

Previsão inicial: o tamanho vinha das dependências Python.

Resultado medido: a maior parte do tamanho da imagem original vem da imagem base `python:3.14` (cerca de 1,2 GB, 76% do total das camadas) porque inclui compiladores, bibliotecas de desenvolvimento, git e outras ferramentas que a app não precisa para correr. As dependências instaladas pelo uv são a camada de 319 MB (cerca de 20%).

## Métricas

Só entram aqui as que o Dockerfile e o compose alteram diretamente. As de execução servem para confirmar que nada piorou. As colunas são acumuladas, e a da alteração 4 é o estado atual.

| Grupo | Métrica | Ferramenta | Original | Alt. 1: `--no-dev` | Alt. 2: base slim | Alt. 3: healthcheck em Python | Alt. 4: utilizador sem privilégios |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Imagem | Tamanho da imagem | `docker image ls backend` | 2,11 GB em disco, 541 MB comprimidos | 1,88 GB, 478 MB (menos 0,23 GB e 63 MB) | 439 MB, 106 MB (face ao original, menos 1,67 GB e 435 MB, cerca de 79% e 80%) | igual (a imagem não muda) | 439 MB, 106 MB (igual à alt. 2 neste arredondamento) |
| Imagem | Camadas | `docker history` | Base python:3.14 cerca de 1,2 GB, binário uv 53,2 MB, dependências 319 MB, app e frontend cerca de 1,4 MB | Dependências 319 para 142 MB. Segundo `uv sync` de 135 kB para 1,79 MB. Resto igual | Base Debian slim cerca de 135 MB (87,7 + 4,92 + 42,1) em vez de cerca de 1,2 GB. Resto igual à alt. 1 | igual | Uma camada nova, `groupadd` e `useradd`, com 41 kB. `USER` com 0 B. Resto igual |
| Imagem | Pacotes Python instalados | `uv pip list` | 77 | 63 (menos 14) | 63 | igual | 63 (camada das dependências em cache) |
| Imagem | Versão do Python | `python --version` | 3.14.9 | 3.14.9 | 3.14.9 (confirmado) | igual | 3.14.9 (mesma base) |
| Imagem | Tempo de build limpo | `docker build` | 159,3 s (inclui download das bases) | 13,4 s (bases e frontend em cache; não comparável) | 20,5 s (inclui download da base slim; não comparável) | n/a | n/a |
| Imagem | Tempo de build com cache | `docker build`, sem alterações | 2,3 s | por medir | por medir | n/a | 2,9 s (as 9 primeiras etapas em cache; só o `useradd` e o `WORKDIR` foram refeitos) |
| Imagem | Tempo de pull | n/a | n/a até haver um registry | n/a | n/a | n/a | n/a |
| Conteúdo | Utilizador do processo | `docker run --rm <imagem> id` | root (uid=0) | root | root | root | `uid=10001(appuser)`. O campo `User` da imagem é `10001:10001` |
| Conteúdo | Dependências de desenvolvimento | `uv pip list` | pytest, mypy, ruff, ty, coverage, prek, zizmor | nenhuma destas | nenhuma destas | nenhuma destas | nenhuma destas (camada em cache) |
| Conteúdo | `curl` presente | `command -v curl` | sim (camada de 65 MB) | sim | não | não | não (base em cache) |
| Conteúdo | Escrita no código da app | `touch` dentro do container | n/a | n/a | n/a | n/a | `Permission denied` em `/app` e em `/app/backend/app/api` |
| Execução | CPU e memória em repouso | `docker stats --no-stream` | 1,03%, 485,4 MiB (6 min após o arranque; 4 workers, 10 PIDs) | 4,60%, 477,5 MiB aos 51 s (amostra cedo demais; não comparável) | não medido (stack `unhealthy`) | Amostra 1: 1,14%, 481,8 MiB (tempo desde o arranque por registar; 11 PIDs). Amostra 2: 0,80%, 494,1 MiB (cerca de 1 min após o arranque; 11 PIDs). 4 workers | por medir |
| Execução | Healthcheck e login | `docker inspect`, login via API | healthy aos 6 min. Com `changethis` o arranque falha (exit 1) | healthy aos 51 s, `prestart` sem erros | `unhealthy`: sem `curl`, 5 falhas com `executable file not found in $PATH` | `healthy`, 5 verificações com ExitCode 0. Apontado a uma porta sem nada, o comando sai com código 1. Login via API devolve 200 | `healthy` aos 53 s. `prestart` de raiz como `appuser`, com as 5 migrações. Login via API devolve 200 |

## Registo de alterações

| # | Alteração | Hipótese | Resultado medido |
| --- | --- | --- | --- |
| 1 | Excluir o grupo de desenvolvimento do `uv sync`, nas duas chamadas | A camada das dependências desce de 319 MB para no máximo cerca de 249 MB (poupança mínima estimada de 70 MB, só em `site-packages`) | Camada de 319 MB para 142 MB (menos 177 MB, 55%). Menos 14 pacotes. Imagem de 2,11 GB para 1,88 GB (11%). Stack `healthy` e `prestart` sem erros. A previsão aguentou como teto, mas a estimativa ficou curta. Causa provável, por confirmar: binários em `/app/.venv/bin` (ruff, ty, prek, zizmor), dependências exclusivas dos pacotes de desenvolvimento e ficheiros `.pyc` |
| 2 | Trocar `FROM python:3.14` por `FROM python:3.14-slim` | Pode partir a app (previsão minha). A previsão de tamanho não foi registada antes de construir | Imagem de 1,88 GB para 439 MB (menos 77%), mesma versão de Python. Sem `curl`, o healthcheck original falhou nas 5 tentativas. O `prestart` ligou-se à base de dados e o login devolveu 200 |
| 3 | Healthcheck do `compose.yml` passa a usar o `urllib` do Python em vez do `curl` | O `curl` não existe na imagem slim, por isso o healthcheck original falha | 5 verificações com ExitCode 0 e `healthy`. O mesmo comando, apontado a uma porta sem nada, sai com código 1 |
| 4 | Utilizador sem privilégios (`uid` 10001) no Dockerfile, com `groupadd`, `useradd` e `USER 10001:10001`, depois do último `uv sync` | Tamanho igual (uns kB no máximo), porque criar um utilizador não remove nada. A previsão inicial, "mais pequeno", foi corrigida antes de medir | Tamanho igual: 439 MB e 106 MB, com a camada nova a pesar 41 kB. `USER` da imagem `10001:10001`. Migrações a partir de base vazia como `appuser`, `healthy` em 53 s, login 200 e escrita no código da app negada. Build de 2,9 s, só com as duas últimas etapas refeitas |

As alterações 1 e 2 estão no mesmo commit, mas foram medidas em builds separados.

## Por resolver

- Confirmar a causa da poupança da alteração 1, medindo o tamanho dos binários em `/app/.venv/bin` na imagem original.
- Medir CPU e memória da imagem sem privilégios, cinco minutos depois do arranque.
- Medir o tempo de build com cache em cada alteração e o caso do dia a dia (mudar um ficheiro da app e reconstruir).
- Contar as vulnerabilidades da imagem base com uma ferramenta, registando o nome e a data.
- Multi-stage no backend, melhorias ao compose, revisão do `.dockerignore` e push para um registry.

## Como ler cada valor

**Tamanho em disco e conteúdo comprimido.** O "em disco" é o que a imagem ocupa descomprimida no teu Docker. O "conteúdo comprimido" é aproximadamente o que viaja num push ou pull. Compara sempre disco com disco, com a mesma ferramenta. Não sei explicar porque a soma das camadas do original (cerca de 1,57 GB) fica abaixo dos 2,11 GB do `docker image ls`. As percentagens das camadas são sobre a soma das camadas.

**Camadas.** Cada instrução do Dockerfile cria uma camada, e o tamanho é o que essa instrução escreveu em disco. O `docker history` lê-se de baixo para cima. As camadas de baixo vêm da imagem base e não são do template. Na imagem completa, os 695 MB são compiladores e bibliotecas de desenvolvimento, os 202 MB são git, mercurial e subversion, os 65 MB são utilitários como o `curl`, e os 83,3 MB são o Python compilado. O binário do `uv` serve para instalar e não é usado pelo `CMD`.

**Imagem completa e imagem slim.** A completa é uma caixa de ferramentas para construir software, e a slim traz o Python e pouco mais. Para correr a app não é preciso compilar nada, mas a slim não traz coisas que a completa trazia sem ninguém as pedir, como o `curl`. Quem troca de base tem de descobrir essas dependências implícitas.

**Pacotes Python.** Os 14 que saíram com o `--no-dev` foram: ast-serialize, coverage, iniconfig, librt, mypy, mypy-extensions, packaging, pathspec, pluggy, prek, pytest, ruff, ty e zizmor. Alguns são dependências que só os de desenvolvimento usavam.

**Utilizador.** `uid=0` é root. Se a app for comprometida, o atacante começa com privilégios de root dentro do container. Com um utilizador sem privilégios, o processo não consegue alterar o código da app, e o teste do `touch` mostra a recusa. Em Kubernetes, a opção `runAsNonRoot` precisa de poder verificar que o utilizador da imagem não é root, e para isso o `USER` da imagem deve ser um número (a confirmar na documentação do Kubernetes). Por isso o `USER` é `10001:10001`, e o `useradd` só serve para o uid ter um nome (`appuser`) para quem depurar a imagem.

**Tempos de build.** O limpo mistura download de imagens com trabalho de build e depende da rede. O de 2,3 s é o piso, com tudo em cache. O de 2,9 s da alteração 4 mostra o que acontece quando só as últimas camadas mudam, e é por isso que o `useradd` está perto do fim do Dockerfile. Os outros builds têm cache parcial, por isso não são comparáveis entre si.

**CPU e memória em repouso.** São amostras únicas, sem carga. O `CMD` arranca 4 workers, por isso a memória inclui os 4 processos. Nas amostras (485,4, 481,8 e 494,1 MiB) não há diferença para lá do ruído entre medições. Compara sempre com os mesmos 4 workers e o mesmo tempo de espera depois de arrancar.

**Healthcheck.** O healthcheck corre um comando dentro do container de 10 em 10 segundos. Um erro como `executable file not found` quer dizer que o comando nem arrancou. O novo usa o `urllib` do Python, que já está na imagem, e falha com código 1 quando a ligação é recusada. Os reinícios dão sempre 0 porque o `compose.yml` não define política de restart. O exit code só tem valor quando o processo termina. O endpoint de saúde só devolve `true` e não toca na base de dados, por isso o login via API é que prova que a base de dados e a verificação da password funcionam.

## Fase 2 (só depois da fase 1 medida)

Prometheus e cAdvisor para observar os containers. Não entra nos pacotes do gig até estar feito e funcionar neste repo.