# Medições: original vs melhorado

Regras de trabalho: medir o original antes de mexer em qualquer coisa. Uma alteração de cada vez. Medir depois de cada alteração. Anotar também o que não melhorou.

## Hipótese inicial (escreve antes de medir)

A maior parte do tamanho da imagem original vem de: ______ porque ______.

## Métricas

Só entram aqui as que o Dockerfile e o compose alteram diretamente. As de execução servem para confirmar que nada piorou.

| Grupo | Métrica | Ferramenta a descobrir | Original | Depois |
| --- | --- | --- | --- | --- |
| Imagem | Tamanho da imagem | `docker image ls` | | |
| Imagem | Número de camadas e tamanho de cada uma | `docker history` | | |
| Imagem | Tempo de build (limpo e com cache) | `time` à volta do build | | |
| Imagem | Tempo de pull | a descobrir | | |
| Conteúdo | Utilizador com que o processo corre | a descobrir | | |
| Conteúdo | Dependências de desenvolvimento presentes na imagem final (pytest, mypy, ruff) | a descobrir | | |
| Execução | CPU e memória em repouso | `docker stats` | | |
| Execução | Healthcheck a passar, reinícios e exit codes | a descobrir | | |

## Registo de alterações

| # | Alteração | Hipótese | Resultado medido |
| --- | --- | --- | --- |
| 1 | | | |

## Fase 2 (só depois da fase 1 medida)

Prometheus e cAdvisor para observar os containers. Não entra nos pacotes do gig até estar feito e funcionar neste repo.
