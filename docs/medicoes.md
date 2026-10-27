# Medições: original vs melhorado

Regras de trabalho: medir o original antes de mexer em qualquer coisa. Uma alteração de cada vez. Medir depois de cada alteração. Anotar também o que não melhorou.

Condições da medição: imagem original construída a partir do commit `9715fef`, Docker Desktop no Windows, stack arrancada só com `compose.yml` (sem o override) e com segredos próprios da sessão. A tag `baseline` (`c34e874`) tem o mesmo Dockerfile, e a diferença é só a pasta `.github` removida. Imagens guardadas: `backend:baseline` (original), `backend:nodev` (alteração 1), `backend:slim` (alteração 2, que inclui a 1), `backend:nonroot` (alteração 4, que inclui as anteriores), `backend:nonroot-newbase` (a mesma imagem de uma fase, reconstruída na base nova) e `backend:multistage` (alteração 5). A alteração 3 é só no `compose.yml`, por isso não tem imagem própria.

A etiqueta `python:3.14-slim` avançou entre a alteração 4 e a 5, da 3.14.9 (digest `35caba83...`) para a 3.14.10 (digest `82abc9f5...`). Por isso a alteração 5 compara-se com a `nonroot-newbase`, e não com a `nonroot`.

Protocolo de CPU e memória: stack nova, 6 minutos depois de o backend arrancar (confirmado com `docker compose ps`) e três amostras de `docker stats --no-stream` com 30 s de intervalo. As medições anteriores ao protocolo estão marcadas como tal e não são comparáveis entre si.

## Hipótese inicial

Previsão inicial: o tamanho vinha das dependências Python.

Resultado medido: a maior parte do tamanho da imagem original vem da imagem base `python:3.14` (cerca de 1,2 GB, 76% do total das camadas) porque inclui compiladores, bibliotecas de desenvolvimento, git e outras ferramentas que a app não precisa para correr. As dependências instaladas pelo uv são a camada de 319 MB (cerca de 20%).

## Métricas

Só entram aqui as que o Dockerfile e o compose alteram diretamente. As de execução servem para confirmar que nada piorou. As colunas são acumuladas, e a da alteração 5 é o estado atual.

| Grupo | Métrica | Ferramenta | Original | Alt. 1: `--no-dev` | Alt. 2: base slim | Alt. 3: healthcheck em Python | Alt. 4: utilizador sem privilégios | Alt. 5: multi-stage |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Imagem | Tamanho da imagem | `docker image ls backend` | 2,11 GB em disco, 541 MB comprimidos | 1,88 GB, 478 MB (menos 0,23 GB e 63 MB) | 439 MB, 106 MB (face ao original, menos 1,67 GB e 435 MB, cerca de 79% e 80%) | igual (a imagem não muda) | 439 MB, 106 MB (igual à alt. 2 neste arredondamento) | 362 MB, 82,8 MB. Referência na mesma base (`nonroot-newbase`, 3.14.10, uma fase): 442 MB, 106 MB. Menos 80 MB (18%) e 23 MB (22%). Face ao original, menos cerca de 83% em disco e 85% comprimidos |
| Imagem | Camadas | `docker history` | Base python:3.14 cerca de 1,2 GB, binário uv 53,2 MB, dependências 319 MB, app e frontend cerca de 1,4 MB | Dependências 319 para 142 MB. Segundo `uv sync` de 135 kB para 1,79 MB. Resto igual | Base Debian slim cerca de 135 MB (87,7 + 4,92 + 42,1) em vez de cerca de 1,2 GB. Resto igual à alt. 1 | igual | Uma camada nova, `groupadd` e `useradd`, com 41 kB. `USER` com 0 B. Resto igual | Sem a camada do `uv` (53,2 MB). O ambiente passa numa camada de 141 MB (antes 142 MB e 1,79 MB). Base cerca de 137 MB (87,7 + 7,31 + 42,1; a base nova pesa mais 2,4 MB). Código, frontend e utilizador como antes |
| Imagem | Pacotes Python instalados | `uv pip list` | 77 | 63 (menos 14) | 63 | igual | 63 (camada das dependências em cache) | não verificado (o `uv` já não está na imagem para os listar) |
| Imagem | Versão do Python | `python --version` | 3.14.9 | 3.14.9 | 3.14.9 (confirmado) | igual | 3.14.9 (mesma base) | 3.14.10 (a etiqueta avançou) |
| Imagem | `uv` presente na imagem final | `command -v uv` | sim (53,2 MB) | sim | sim | sim | sim | não (`sem uv`) |
| Imagem | Tempo de build limpo | `docker build` | 159,3 s (inclui download das bases) | 13,4 s (bases e frontend em cache; não comparável) | 20,5 s (inclui download da base slim; não comparável) | n/a | n/a | 16,7 s (cache parcial; não comparável) |
| Imagem | Tempo de build com cache | `docker build`, sem alterações | 2,3 s | por medir | por medir | n/a | 2,9 s (as 9 primeiras etapas em cache; só o `useradd` e o `WORKDIR` foram refeitos) | por medir |
| Imagem | Tempo de pull | n/a | n/a até haver um registry | n/a | n/a | n/a | n/a | n/a |
| Conteúdo | Utilizador do processo | `docker run --rm <imagem> id` | root (uid=0) | root | root | root | `uid=10001(appuser)`. O campo `User` da imagem é `10001:10001` | `uid=10001(appuser)` |
| Conteúdo | Dependências de desenvolvimento | `uv pip list` | pytest, mypy, ruff, ty, coverage, prek, zizmor | nenhuma destas | nenhuma destas | nenhuma destas | nenhuma destas (camada em cache) | nenhuma destas (o `builder` usa `--no-dev`; não reverificado na imagem) |
| Conteúdo | `curl` presente | `command -v curl` | sim (camada de 65 MB) | sim | não | não | não (base em cache) | não (base slim) |
| Conteúdo | Escrita no código da app | `touch` dentro do container | n/a | n/a | n/a | n/a | `Permission denied` em `/app`, em `/app/backend/app/api` (pasta existente) e ao criar `/app/backend/app/teste` | `Permission denied` ao criar `/app/backend/app/teste` |
| Execução | CPU e memória em repouso | `docker stats --no-stream` | Antes do protocolo: 1,03% e 485,4 MiB (6 min, uma amostra, 10 PIDs); repetição com espera não registada: CPU 0,94%, 0,66% e 13,78%, memória 493,1, 493,0 e 493,2 MiB, 10 PIDs. Protocolo: CPU 0,91%, 1,05% e 0,91%, memória 489,6, 489,5 e 476,4 MiB, 10 PIDs | Antes do protocolo: 4,60% e 477,5 MiB aos 51 s (não comparável) | não medido (stack `unhealthy`) | Antes do protocolo. Amostra 1: 1,14% e 481,8 MiB (tempo desde o arranque por registar; 11 PIDs). Amostra 2: 0,80% e 494,1 MiB (cerca de 1 min após o arranque; 11 PIDs). | Protocolo: CPU 1,29%, 0,77% e 0,66%, memória 499,0, 499,0 e 499,5 MiB, 10 PIDs. Aos 33 min: CPU 0,69%, 0,99% e 0,83%, memória 504,5, 504,2 e 504,8 MiB, 11 PIDs | por medir |
| Execução | Healthcheck e login | `docker inspect`, login via API | healthy aos 6 min. Com `changethis` o arranque falha (exit 1) | healthy aos 51 s, `prestart` sem erros | `unhealthy`: sem `curl`, 5 falhas com `executable file not found in $PATH` | `healthy`, 5 verificações com ExitCode 0. Apontado a uma porta sem nada, o comando sai com código 1. Login via API devolve 200 | `healthy` aos 53 s. `prestart` de raiz como `appuser`, com as 5 migrações. Login via API devolve 200 | `healthy` (verificado 2 min depois do arranque). `prestart` de raiz com as 5 migrações, login via API 200, e `import app` a partir de `/` a imprimir `/app/backend/app/__init__.py` |

## Registo de alterações

| # | Alteração | Hipótese | Resultado medido |
| --- | --- | --- | --- |
| 1 | Excluir o grupo de desenvolvimento do `uv sync`, nas duas chamadas | A camada das dependências desce de 319 MB para no máximo cerca de 249 MB (poupança mínima estimada de 70 MB, só em `site-packages`) | Camada de 319 MB para 142 MB (menos 177 MB, 55%). Menos 14 pacotes. Imagem de 2,11 GB para 1,88 GB (11%). Stack `healthy` e `prestart` sem erros. A previsão aguentou como teto, mas a estimativa ficou curta. Causa provável, por confirmar: binários em `/app/.venv/bin` (ruff, ty, prek, zizmor), dependências exclusivas dos pacotes de desenvolvimento e ficheiros `.pyc` |
| 2 | Trocar `FROM python:3.14` por `FROM python:3.14-slim` | Pode partir a app (previsão minha). A previsão de tamanho não foi registada antes de construir | Imagem de 1,88 GB para 439 MB (menos 77%), mesma versão de Python. Sem `curl`, o healthcheck original falhou nas 5 tentativas. O `prestart` ligou-se à base de dados e o login devolveu 200 |
| 3 | Healthcheck do `compose.yml` passa a usar o `urllib` do Python em vez do `curl` | O `curl` não existe na imagem slim, por isso o healthcheck original falha | 5 verificações com ExitCode 0 e `healthy`. O mesmo comando, apontado a uma porta sem nada, sai com código 1 |
| 4 | Utilizador sem privilégios (`uid` 10001) no Dockerfile, com `groupadd`, `useradd` e `USER 10001:10001`, depois do último `uv sync` | Tamanho igual (uns kB no máximo), porque criar um utilizador não remove nada. A previsão inicial, "mais pequeno", foi corrigida antes de medir | Tamanho igual: 439 MB e 106 MB, com a camada nova a pesar 41 kB. `USER` da imagem `10001:10001`. Migrações a partir de base vazia como `appuser`, `healthy` em 53 s, login 200 e escrita no código da app negada. Build de 2,9 s, só com as duas últimas etapas refeitas. Memória em repouso: aos 6 minutos, medianas de cerca de 487 MiB (original) e 499 MiB, uma diferença de cerca de 12 MiB (2%), menor do que a variação entre amostras do original (476,4 a 493,2 MiB). Não se atribui à imagem |
| 5 | Multi-stage no backend: fase `builder` com o `uv` e as duas chamadas `uv sync --no-dev`, e fase final `runtime` só com o ambiente (`/app/.venv`), o código, o frontend compilado, o utilizador e o `CMD` | Cerca de 400 MB (previsão minha). Valor derivado: cerca de 386 MB (439 MB menos os 53,2 MB do `uv`). Risco: a app não arrancar por faltar algo na fase final | 362 MB e 82,8 MB, contra 442 MB e 106 MB da mesma imagem de uma fase na base nova (menos 80 MB e 23 MB). As camadas explicam cerca de 56 MB (o `uv` e uns 2,8 MB do `.venv`), e os restantes 24 MB da coluna "em disco" estão por explicar. As duas previsões ficaram curtas. Duas builds falharam pelo caminho: sem o código no `builder` (`Distribution not found at: file:///app/backend`) e sem o `pyproject.toml` (`/app/backend does not appear to be a Python project`). O `builder` precisa do código em `app` e do `pyproject.toml`, e dispensa os `scripts` e o `alembic.ini`. A base avançou da 3.14.9 para a 3.14.10 entre a alteração 4 e a 5, por isso a referência é a imagem de uma fase na mesma base. Stack de raiz: 5 migrações, `healthy`, `sem uv`, `appuser`, escrita no código negada e login 200 |

As alterações 1 e 2 estão no mesmo commit, mas foram medidas em builds separados. A alteração 5 foi medida contra uma imagem de uma fase construída na mesma base.

## Por resolver

- Explicar a diferença entre a soma das camadas e o tamanho em disco do `docker image ls`, que aparece em todas as imagens.
- Confirmar a causa da poupança da alteração 1, medindo o tamanho dos binários em `/app/.venv/bin` na imagem original.
- Identificar o 11.º processo que aparece nas amostras aos 33 minutos (aos 6 minutos há 10).
- Medir CPU e memória da imagem multi-stage com o protocolo, e o tempo de build com cache em cada alteração e no caso do dia a dia (mudar um ficheiro da app e reconstruir).
- Listar os pacotes Python da imagem multi-stage por outro meio, já que o `uv` não está lá.
- Contar as vulnerabilidades da imagem base com uma ferramenta, registando o nome e a data.
- Demonstrar o multi-stage com um pacote sem wheel, que precisa de compilador no `builder` e só da biblioteca de execução na fase final.
- Melhorias ao compose, revisão do `.dockerignore` (o contexto de um build sem cache incremental foi de 862,44 kB) e push para um registry.

## Como ler cada valor

**Tamanho em disco e conteúdo comprimido.** O "em disco" é o que a imagem ocupa descomprimida no teu Docker. O "conteúdo comprimido" é aproximadamente o que viaja num push ou pull. Compara sempre disco com disco, com a mesma ferramenta. Quando a etiqueta da base avança (como a `3.14-slim`, de 3.14.9 para 3.14.10), a imagem de referência tem de ser reconstruída na base nova, senão a comparação mistura duas coisas. As percentagens das camadas são sobre a soma das camadas.

**Camadas.** Cada instrução do Dockerfile cria uma camada, e o tamanho é o que essa instrução escreveu em disco. O `docker history` lê-se de baixo para cima. As camadas de baixo vêm da imagem base e não são do template. Na imagem completa, os 695 MB são compiladores e bibliotecas de desenvolvimento, os 202 MB são git, mercurial e subversion, os 65 MB são utilitários como o `curl`, e os 83,3 MB são o Python compilado. A soma das camadas fica abaixo do tamanho em disco do `docker image ls` em todas as imagens (cerca de 1,57 GB contra 2,11 GB no original, 333 MB contra 439 MB na de utilizador sem privilégios, 279 MB contra 362 MB na multi-stage), e não sei explicar porquê.

**Multi-stage.** É um Dockerfile com vários `FROM`. A imagem final é só a última fase, e leva apenas o que se copia das anteriores com `COPY --from`. Aqui, a fase `builder` tem o `uv` e instala as dependências, e a fase `runtime` recebe só o ambiente (`/app/.venv`), o código e o frontend compilado. O binário do `uv` fica na fase descartada, e é daí que vêm os 53,2 MB a menos. Há três cuidados. O `.venv` aponta para o Python que o criou, por isso as duas fases partem da mesma base. O pacote `app` está instalado em modo editável, a apontar para `/app/backend`, por isso o código tem de estar nesse caminho na fase final. E as instruções `ENV`, `WORKDIR`, `USER` e `CMD` não passam de uma fase para a outra. O `builder` também precisa do código em `app` e do `pyproject.toml`, para o `uv sync` instalar o pacote.

**Pacotes Python.** Os 14 que saíram com o `--no-dev` foram: ast-serialize, coverage, iniconfig, librt, mypy, mypy-extensions, packaging, pathspec, pluggy, prek, pytest, ruff, ty e zizmor. Alguns são dependências que só os de desenvolvimento usavam.

**Utilizador.** `uid=0` é root. Se a app for comprometida, o atacante começa com privilégios de root dentro do container. Com um utilizador sem privilégios, o processo não consegue alterar o código da app, e o teste do `touch` mostra a recusa. Em Kubernetes, a opção `runAsNonRoot` precisa de poder verificar que o utilizador da imagem não é root, e para isso o `USER` da imagem deve ser um número (a confirmar na documentação do Kubernetes). Por isso o `USER` é `10001:10001`, e o `useradd` só serve para o uid ter um nome (`appuser`) para quem depurar a imagem.

**Tempos de build.** O limpo mistura download de imagens com trabalho de build e depende da rede. O de 2,3 s é o piso, com tudo em cache. O de 2,9 s da alteração 4 mostra o que acontece quando só as últimas camadas mudam, e é por isso que o `useradd` está perto do fim do Dockerfile. Os outros builds têm cache parcial, por isso não são comparáveis entre si.

**CPU e memória em repouso.** São amostras sem carga, e o `CMD` arranca 4 workers, por isso a memória inclui os 4 processos. A memória varia mesmo dentro de uma execução: uma das amostras do original caiu 13 MiB em 30 segundos. No original a memória variou entre 476,4 e 493,2 MiB, e na imagem sem privilégios entre 499,0 e 504,8 MiB. Aos 6 minutos, as medianas são cerca de 487 e 499 MiB, uma diferença de cerca de 12 MiB (2%) que é menor do que a variação entre amostras do original. Com tão poucas execuções não se atribui à imagem. O CPU salta entre amostras (houve um pico de 13,78%), por isso regista-se o intervalo. Os processos (PIDs) são 10 aos 6 minutos nas duas imagens, e 11 nas amostras aos 33 minutos e em algumas anteriores. Não sei ainda o que é o processo extra.

**Healthcheck.** O healthcheck corre um comando dentro do container de 10 em 10 segundos. Um erro como `executable file not found` quer dizer que o comando nem arrancou. O novo usa o `urllib` do Python, que já está na imagem, e falha com código 1 quando a ligação é recusada. Os reinícios dão sempre 0 porque o `compose.yml` não define política de restart. O exit code só tem valor quando o processo termina. O endpoint de saúde só devolve `true` e não toca na base de dados, por isso o login via API é que prova que a base de dados e a verificação da password funcionam.

**`import app` a partir de `/`.** O diretório de trabalho da imagem é `/app/backend`, e o Python põe o diretório atual no caminho de procura, por isso um `import app` feito lá dentro funcionaria mesmo que o modo editável estivesse partido. Com `-w /`, o diretório atual deixa de ajudar, e o import só funciona se o apontador `.pth` do `.venv` para `/app/backend` estiver a funcionar.

## Fase 2 (só depois da fase 1 medida)

Prometheus e cAdvisor para observar os containers. Não entra nos pacotes do gig até estar feito e funcionar neste repo.