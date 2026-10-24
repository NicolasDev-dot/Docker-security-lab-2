# Docker Security Lab 2

Reforço de segurança em um container Python pequeno: criar a imagem do zero, escanear com o Trivy, investigar os achados, corrigir o que dá pra corrigir, reconstruir, escanear de novo e comparar.

Este é um projeto de aprendizado. O objetivo **não** era deixar o scanner com "zero" achados, e sim entender de onde vem cada vulnerabilidade e qual é uma resposta segura pra ela.

## Objetivo

Praticar um fluxo real de segurança de containers:

```
Criar a imagem -> Escanear (Trivy) -> Investigar os pacotes -> Corrigir no Dockerfile
-> Reconstruir -> Escanear de novo -> Comparar -> Documentar
```

## O que eu construí do zero

Uma das partes que mais me ensinou foi criar tudo desde o início, a começar pelo Dockerfile:

- escrevi o `Dockerfile` (imagem base, pasta de trabalho, cópia do app e comando de inicialização)
- construí a imagem com `docker build` e rodei o container com `docker run`
- usei `docker history` pra entender de quais camadas a imagem é feita (Debian -> Python -> aplicação)
- só depois comecei a analisar a segurança, o que me ajudou a entender de onde cada pacote vulnerável vinha

## Ambiente

- Windows 11, PowerShell, Docker Desktop
- Trivy 0.71.0
- Imagem base: `python:slim` (Debian 13 "Trixie")
- Aplicação: um `app.py` pequeno que imprime uma mensagem

Dockerfile original:

```dockerfile
FROM python:slim
WORKDIR /app2
COPY app/app.py /app2
CMD [ "python","app.py" ]
```

Dockerfile final (v4):

```dockerfile
FROM python:3.15-slim
WORKDIR /app2
RUN pip uninstall -y pip
COPY app/app.py /app2
CMD [ "python","app.py" ]
```

## Como reproduzir

```powershell
docker build --pull -t docker3:v4 .
trivy image --severity HIGH,CRITICAL --output scan-v4.txt docker3:v4
Select-String -Path scan-v4.txt -Pattern "Total:"
docker run --rm docker3:v4
```

## Resultados (somente HIGH e CRITICAL)

| Imagem | O que mudou | Debian | Python | Relatório |
|---|---|---|---|---|
| `docker3:latest` | ponto de partida (Debian 13.6) | 64 (61 HIGH, 3 CRITICAL) | 4 HIGH | `scan-antes.txt` |
| `docker3:v2` | imagem base atualizada com `--pull` (Debian 13.7) | 44 (44 HIGH, 0 CRITICAL) | 4 HIGH | `scan-depois.txt` |
| `docker3:v3` | versão fixada em `python:3.15-slim` + `pip install --upgrade pip` | 44 | 4 HIGH | `scan-v3.txt` |
| `docker3:v4` | pip removido | 44 (44 HIGH, 0 CRITICAL) | nenhum reportado | `scan-v4.txt` |

Total: **68 achados (3 CRITICAL) -> 44 achados (0 CRITICAL)**.

## O que eu tentei

### Tentativa 1: atualizar a imagem base

- Comando: `docker build --pull -t docker3:v2 .`
- Resultado: Debian 13.6 -> 13.7, CRITICAL de 3 para 0 e achados do Debian de 64 para 44.
- Por que funcionou: o `perl-base` passou de `5.40.1-6` para `5.40.1-6+deb13u1`, que é a versão com o patch de segurança do Debian. Outros pacotes (como `openssl`, `libssl3t64`, `gzip` e `libsqlite3-0`) também deixaram de aparecer no relatório.
- Observação: a tag `python:slim` não fixa versão, então o Python mudou de 3.14.7 para 3.15.0 sozinho. O app continuou rodando normalmente.

### Tentativa 2: fixar a versão e atualizar o pip (não funcionou)

- Mudança: `FROM python:3.15-slim` e `RUN pip install --no-cache-dir --upgrade pip`.
- Resultado: números exatamente iguais aos da v2 (Debian 44, Python 4).
- O que eu aprendi: os 4 achados Python eram `msgpack` 1.1.2, `setuptools` 70.3.0 e `urllib3` 2.7.0 (2 CVEs). Existiam versões corrigidas pra todos, mas eu não conseguia aplicar as correções: o `pip list` mostrava só o `pip`, e o comando `find /usr/local -iname "*msgpack*"` mostrou que o `msgpack` está dentro de `pip/_vendor`. Ou seja, são cópias embutidas no próprio pip, então não dá pra atualizar a biblioteca separadamente, e um pip mais novo não trouxe cópias corrigidas.
- Esse teste também funcionou como controle: fixar a versão não mudou o conteúdo da imagem.

### Tentativa 3: remover o pip

- Mudança: `RUN pip uninstall -y pip`.
- Por quê: o app só imprime uma mensagem e não instala nada, então o pip não é necessário pra rodar. O que não está na imagem não pode ser explorado.
- Resultado: a seção Python sumiu do relatório (4 achados a menos). O app continua rodando, e `docker run --rm docker3:v4 pip --version` falha com `executable file not found in $PATH`, o que confirma que o pip saiu.
- Importante: eu **não corrigi** essas vulnerabilidades com patch. Eu **eliminei o componente** que continha elas (redução da superfície de ataque).

## O que eu não fiz, e por quê

- Tentei `apt-get remove --purge perl-base`. O Debian bloqueou, porque o `perl-base` é um pacote essencial do sistema. Remover pacotes essenciais só pra o scanner parecer melhor quebraria a imagem, então deixei como está.
- Não instalei ferramentas extras só pra investigar (como o `procps`).

## Risco residual

Os 44 achados restantes do Debian (todos HIGH) **não têm versão corrigida disponível** no relatório do Trivy:

| Grupo de pacotes | CVEs | Quantidade |
|---|---|---|
| Família util-linux (`bsdutils`, `libblkid1`, `liblastlog2-2`, `libmount1`, `libsmartcols1`, `libuuid1`, `login`, `mount`, `util-linux`) | CVE-2026-76642, CVE-2026-78408, CVE-2026-78409, CVE-2026-78410 | 36 (9 pacotes x 4) |
| ncurses (`libncursesw6`, `libtinfo6`, `ncurses-base`, `ncurses-bin`) | CVE-2025-69720 | 4 |
| systemd (`libsystemd0`, `libudev1`) | CVE-2026-16742 | 2 |
| `libacl1` | CVE-2026-54369 | 1 |
| `perl-base` | CVE-2026-9538 (status: fix_deferred) | 1 |

Avaliação preliminar de risco (baseada nos títulos das CVEs, sem testes): várias envolvem montagem de sistemas de arquivos com privilégio, o `systemd-homed` ou entradas malformadas em programas que este app não usa, e um container simples normalmente não executa isso. Aceito esse risco para uma imagem de laboratório e vou reescanear periodicamente, porque o Debian pode publicar patches a qualquer momento.

## Limitações

- O `pip uninstall` roda em uma camada acima da imagem base. O Trivy analisa o sistema de arquivos final, então não enxerga mais o pip, mas os arquivos ainda existem na camada de baixo. Um build em vários estágios (multi-stage) removeria de verdade.
- O container ainda roda como root.
- O Trivy atualiza o banco de vulnerabilidades com o tempo, então os números podem mudar mesmo sem a imagem mudar.
- Se o app um dia precisar de dependências, elas têm que ser instaladas durante o build, antes de remover o pip.

## Lições aprendidas

1. Um número maior em um novo scan nem sempre significa uma imagem pior: o banco de dados pode ter mudado.
2. Sempre verificar se um pacote é necessário antes de remover (o `perl-base` é essencial, o pip não era).
3. Algumas correções não são possíveis na imagem que você constrói: o código vulnerável pode vir embutido dentro de outra ferramenta.
4. Um experimento que falha também é um resultado.
5. "Corrigido por patch", "mitigado" e "risco aceito" são coisas diferentes, e o relatório precisa dizer qual é.
6. Fixar a versão da imagem base, ou o build muda sem você perceber.
7. Construir a imagem do zero, começando pelo Dockerfile, ajuda a entender de onde vem cada pacote que o scanner reporta.

## Próximos passos

- Build em vários estágios (multi-stage) pra o pip nunca chegar na imagem final
- Rodar como usuário sem privilégios (instrução `USER`)
- Fixar a imagem base pelo digest
- Reescanear periodicamente e automatizar (por exemplo, com GitHub Actions)

## Arquivos

- `Dockerfile`, `app/app.py`
- `scan-antes.txt`, `scan-depois.txt`, `scan-v3.txt`, `scan-v4.txt`
