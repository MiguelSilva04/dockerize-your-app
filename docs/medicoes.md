# Medições: original vs melhorado

Regras de trabalho: medir o original antes de mexer em qualquer coisa. Uma alteração de cada vez. Medir depois de cada alteração. Anotar também o que não melhorou.

Condições da medição: imagem construída a partir do commit `9715fef`, Docker Desktop no Windows, stack arrancada só com `compose.yml` (sem o override) e com segredos próprios da sessão. As medições de execução são uma amostra em repouso, feita depois de o healthcheck passar. Imagens guardadas: `backend:baseline` (original), `backend:nodev` (alteração 1) e `backend:slim` (alteração 2, que inclui a 1).

## Hipótese inicial

Previsão inicial: o tamanho vinha das dependências Python.

Resultado medido: a maior parte do tamanho da imagem original vem da imagem base `python:3.14` (cerca de 1,2 GB, 76% do total das camadas) porque inclui compiladores, bibliotecas de desenvolvimento, git e outras ferramentas que a app não precisa para correr. As dependências instaladas pelo uv são a camada de 319 MB (cerca de 20%).

## Métricas

Só entram aqui as que o Dockerfile e o compose alteram diretamente. As de execução servem para confirmar que nada piorou. A coluna da alteração 2 é acumulada.

| Grupo | Métrica | Ferramenta | Original | Alt. 1: `--no-dev` | Alt. 2: base slim |
| --- | --- | --- | --- | --- | --- |
| Imagem | Tamanho da imagem | `docker image ls backend` | 2,11 GB em disco, 541 MB comprimidos | 1,88 GB, 478 MB (menos 0,23 GB e 63 MB) | 439 MB, 106 MB (face ao original, menos 1,67 GB e 435 MB, cerca de 79% e 80%) |
| Imagem | Camadas | `docker history` | Base python:3.14 cerca de 1,2 GB, binário uv 53,2 MB, dependências 319 MB, app e frontend cerca de 1,4 MB | Dependências 319 para 142 MB. Segundo `uv sync` de 135 kB para 1,79 MB. Resto igual | Base Debian slim cerca de 135 MB (87,7 + 4,92 + 42,1) em vez de cerca de 1,2 GB. Resto igual à alt. 1 |
| Imagem | Pacotes Python instalados | `uv pip list` | 77 | 63 (menos 14) | 63 |
| Imagem | Versão do Python | `python --version` | 3.14.9 | 3.14.9 | 3.14.9 (confirmado) |
| Imagem | Tempo de build limpo | `docker build` | 159,3 s (inclui download das bases) | 13,4 s (bases e frontend em cache; não comparável) | 20,5 s (inclui download da base slim; não comparável) |
| Imagem | Tempo de build com cache | `docker build`, sem alterações | 2,3 s | por medir | por medir |
| Imagem | Tempo de pull | n/a | n/a até haver um registry | n/a | n/a |
| Conteúdo | Utilizador do processo | `docker run --rm <imagem> id` | root (uid=0) | root | root |
| Conteúdo | Dependências de desenvolvimento | `uv pip list` | pytest, mypy, ruff, ty, coverage, prek, zizmor | nenhuma destas | nenhuma destas |
| Conteúdo | `curl` presente | `command -v curl` | sim (camada de 65 MB) | sim | não |
| Segurança | Vulnerabilidades da imagem base | aviso no `FROM` (ferramenta e data por confirmar) | 2 críticas e 30 altas | n/a (base igual) | 6 altas |
| Execução | CPU e memória em repouso | `docker stats --no-stream` | 1,03%, 485,4 MiB (4 workers) | 4,60%, 477,5 MiB aos 51 s (amostra cedo demais; repetir) | por medir |
| Execução | Healthcheck | `docker ps`, `docker inspect` | healthy aos 6 min. Com `changethis` o arranque falha (exit 1) | healthy aos 51 s, `prestart` sem erros | `unhealthy`: sem `curl`, 5 falhas com `executable file not found in $PATH`. `prestart` por validar |

## Registo de alterações

| # | Alteração | Hipótese | Resultado medido |
| --- | --- | --- | --- |
| 1 | Excluir o grupo de desenvolvimento do `uv sync`, nas duas chamadas | A camada das dependências desce de 319 MB para no máximo cerca de 249 MB (poupança mínima estimada de 70 MB, só em `site-packages`) | Camada de 319 MB para 142 MB (menos 177 MB, 55%). Menos 14 pacotes. Imagem de 2,11 GB para 1,88 GB (11%). Stack `healthy` e `prestart` sem erros. A previsão aguentou como teto, mas a estimativa de poupança ficou curta porque não contei binários, dependências exclusivas dos pacotes de desenvolvimento nem `.pyc` |
| 2 | Trocar `FROM python:3.14` por `FROM python:3.14-slim` | Pode partir a app (previsão minha). A previsão de tamanho não foi registada antes de construir | O build correu. Imagem de 1,88 GB para 439 MB (menos 77%), mesma versão de Python. Sem `curl`, o healthcheck falha. Por validar: `prestart` e login com os segredos, e a memória |

## Por resolver

- Healthcheck sem `curl`: decidir entre instalar o `curl` ou usar o Python no `compose.yml`, e medir o efeito.
- Validar a imagem slim com os segredos da sessão: `prestart`, `healthy` e login na app.
- Utilizador sem privilégios (`uid` diferente de 0).
- Repetir o `docker stats` cinco minutos depois do arranque, nas três imagens.

## Como ler cada valor

**Tamanho em disco e conteúdo comprimido.** O "em disco" é o que a imagem ocupa descomprimida no teu Docker. O "conteúdo comprimido" é aproximadamente o que viaja num push ou pull. Compara sempre disco com disco, com a mesma ferramenta. Não sei explicar porque a soma das camadas do original (cerca de 1,57 GB) fica abaixo dos 2,11 GB do `docker image ls`. As percentagens das camadas são sobre a soma das camadas.

**Camadas.** Cada instrução do Dockerfile cria uma camada, e o tamanho é o que essa instrução escreveu em disco. O `docker history` lê-se de baixo para cima. As camadas de baixo vêm da imagem base e não são do template. Na imagem completa, os 695 MB são compiladores e bibliotecas de desenvolvimento, os 202 MB são git, mercurial e subversion, os 65 MB são utilitários como o `curl`, e os 83,3 MB são o Python compilado. O binário do `uv` serve para instalar, não para correr.

**Imagem completa e imagem slim.** A completa é uma caixa de ferramentas para construir software, e a slim traz o Python e pouco mais. Para correr a app não é preciso compilar nada, mas a slim não traz coisas que a completa trazia sem ninguém as pedir, como o `curl`. Quem troca de base tem de descobrir essas dependências implícitas.

**Pacotes Python.** Os 14 que saíram com o `--no-dev` foram: ast-serialize, coverage, iniconfig, librt, mypy, mypy-extensions, packaging, pathspec, pluggy, prek, pytest, ruff, ty e zizmor. Alguns são dependências que só os de desenvolvimento usavam.

**Vulnerabilidades.** É a contagem de falhas conhecidas (CVEs) nos pacotes da imagem base. Menos pacotes dão menos vulnerabilidades, mas o número muda quando a base é atualizada, e não diz se a falha é explorável neste contexto. Regista sempre a ferramenta e a data.

**Utilizador.** `uid=0` é root. Se a app for comprometida, o atacante começa com privilégios de root dentro do container.

**Tempos de build.** O limpo mistura download de imagens com trabalho de build e depende da rede. O de 2,3 s é o piso, com tudo em cache. Os builds das alterações têm cache parcial, por isso não são comparáveis entre si.

**CPU e memória em repouso.** É uma única amostra sem carga. A memória é a soma dos 4 workers e muda pouco com o tamanho da imagem. Compara sempre com os mesmos 4 workers e o mesmo tempo de espera depois de arrancar.

**Healthcheck.** O healthcheck corre um comando dentro do container de 10 em 10 segundos. Um erro como `executable file not found` quer dizer que o comando nem arrancou. Os reinícios dão sempre 0 porque o `compose.yml` não define política de restart. O exit code só tem valor quando o processo termina.

## Fase 2 (só depois da fase 1 medida)

Prometheus e cAdvisor para observar os containers. Não entra nos pacotes do gig até estar feito e funcionar neste repo.