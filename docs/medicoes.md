# Medições: original vs melhorado

Regras de trabalho: medir o original antes de mexer em qualquer coisa. Uma alteração de cada vez. Medir depois de cada alteração. Anotar também o que não melhorou.

Condições da medição: imagem `backend:latest` construída a partir do commit `9715fef`, Docker Desktop no Windows, stack arrancada só com `compose.yml` (sem o override) e com segredos próprios da sessão. As medições de execução são uma amostra em repouso, feita depois de o healthcheck passar. A imagem original está guardada com a tag `backend:baseline`.

## Hipótese inicial

Previsão inicial: o tamanho vinha das dependências Python.

Resultado medido: a maior parte do tamanho da imagem original vem da imagem base `python:3.14` (cerca de 1,2 GB, 76% do total das camadas) porque inclui compiladores, bibliotecas de desenvolvimento, git e outras ferramentas que a app não precisa para correr. As dependências instaladas pelo uv são a camada de 319 MB (cerca de 20%).

## Métricas

Só entram aqui as que o Dockerfile e o compose alteram diretamente. As de execução servem para confirmar que nada piorou.

| Grupo | Métrica | Ferramenta | Original | Depois |
| --- | --- | --- | --- | --- |
| Imagem | Tamanho da imagem | `docker image ls backend` | 2,11 GB em disco, 541 MB de conteúdo comprimido | |
| Imagem | Camadas e tamanho de cada grupo | `docker history --no-trunc` | 25 linhas, 16 com tamanho acima de 0. Base python:3.14 cerca de 1,2 GB (134 + 65 + 202 + 695 + 19,9 + 83,3 MB). Binário uv 53,2 MB. Dependências Python 319 MB. App e frontend cerca de 1,4 MB | |
| Imagem | Tempo de build limpo | `docker compose build` | 159,3 s (inclui o download das imagens base) | |
| Imagem | Tempo de build com cache | `docker compose build`, sem alterações | 2,3 s (todas as etapas `CACHED`) | |
| Imagem | Tempo de pull | n/a | n/a até haver um registry | |
| Conteúdo | Utilizador com que o processo corre | `docker run --rm backend:latest id` | root (uid=0) | |
| Conteúdo | Dependências de desenvolvimento na imagem final | `uv pip list` e `du -sk` dentro da imagem | pytest 9.1.1, mypy 2.3.1, ruff 0.16.9, ty 0.0.84, coverage 7.16.1, prek 0.5.3, zizmor 1.30.1. Pelo menos cerca de 70 MB em `site-packages` (mypy, mypyc, pytest, coverage) | |
| Execução | CPU e memória em repouso | `docker stats --no-stream` | Backend 1,03% CPU, 485,4 MiB (4 workers, 10 PIDs). Postgres 0,00%, 49,96 MiB | |
| Execução | Healthcheck, reinícios, exit codes | `docker inspect` | healthy aos 6 min, restarts=0 (sem política de restart). Com `changethis` por defeito o arranque em modo produção falha (exit 1) | |

## Registo de alterações

| # | Alteração | Hipótese | Resultado medido |
| --- | --- | --- | --- |
| 1 | Excluir o grupo de desenvolvimento do `uv sync` (nas duas chamadas) | A camada das dependências desce de 319 MB para no máximo cerca de 249 MB (poupança mínima estimada de 70 MB, só em `site-packages`) | |

## Como ler cada valor

**Tamanho em disco e conteúdo comprimido.** O "em disco" é o que a imagem ocupa descomprimida no teu Docker. O "conteúdo comprimido" é aproximadamente o que viaja num push ou pull. Compara sempre disco com disco, com a mesma ferramenta. Não sei explicar porque a soma das camadas (cerca de 1,57 GB) fica abaixo dos 2,11 GB. As percentagens deste documento são sobre a soma das camadas.

**Camadas.** Cada instrução do Dockerfile cria uma camada, e o tamanho é o que essa instrução escreveu em disco. O `docker history` lê-se de baixo para cima. As camadas de baixo (Debian, ferramentas de rede, git, compiladores, o próprio Python) vêm da imagem base e não são do template. Os 695 MB são compiladores e bibliotecas de desenvolvimento, os 202 MB são git, mercurial e subversion, e os 65 MB são utilitários como o `curl` do healthcheck. Os 83,3 MB são o Python compilado. O binário do uv serve para instalar, não para correr. Os 319 MB são as dependências Python.

**Tempo de build limpo e com cache.** O limpo mistura o download das imagens base com o trabalho de build, por isso depende da rede. O de 2,3 s mostra que o Docker reutilizou todas as camadas, e é o piso de tempo. Falta medir o caso do dia a dia: mudar um ficheiro da app e reconstruir.

**Tempo de pull.** Só se mede com um registry real, depois de um push.

**Utilizador.** `uid=0` é root. Se a app for comprometida, o atacante começa com privilégios de root dentro do container. Um utilizador sem privilégios limita esse estrago.

**Dependências de desenvolvimento.** São ferramentas de teste e de verificação que não fazem falta em produção. Aumentam o tamanho e a superfície de ataque. O `sqlalchemy`, apesar de ser grande, é de execução, porque o `sqlmodel` assenta nele.

**CPU e memória em repouso.** É uma única amostra sem carga. A memória do backend é a soma dos 4 workers e muda pouco com o tamanho da imagem, por isso é uma hipótese a confirmar. Compara usando os mesmos 4 workers e o mesmo tempo de espera.

**Healthcheck, reinícios, exit codes.** O healthcheck corre um `curl` ao endpoint de saúde de 10 em 10 segundos. Os reinícios dão sempre 0 porque o `compose.yml` não define política de restart. O exit code só tem valor quando o processo termina, e o que importa aqui é o estado `healthy`.

## Fase 2 (só depois da fase 1 medida)

Prometheus e cAdvisor para observar os containers. Não entra nos pacotes do gig até estar feito e funcionar neste repo.