# Ambiente de desenvolvimento remoto com Docker

Imagens de desenvolvimento via SSH, preparadas para VS Code Remote - SSH, Claude Code, .NET e Node.js. Há duas alternativas independentes:

| Ambiente | Base | SSH local | Memória padrão |
| --- | --- | --- | --- |
| `Ubuntu/` | Ubuntu 24.04 | `2222` | 3 GB |
| `Debian/` | Debian 12 Slim | `2223` | 8 GB |

Para uma imagem menor, prefira Debian. Ubuntu é útil se sua aplicação ou ferramentas já esperam essa distribuição.

## Incluído nas imagens

- OpenSSH configurado para autenticação apenas com chave pública;
- usuário não-root `dev`, com `sudo` sem senha para desenvolvimento;
- .NET SDK 10;
- Node.js 22 LTS;
- Claude Code instalado para o usuário `dev`;
- Git, `curl`, `wget`, `tar`, `zip`, certificados e pré-requisitos do VS Code Remote - SSH.

O VS Code **não é instalado dentro do container**. Instale-o no computador host com a extensão **Remote - SSH**; o VS Code Server será instalado automaticamente ao conectar.

## Pré-requisitos

- Docker Desktop em execução, usando containers Linux;
- Docker Compose v2;
- uma chave SSH pública. No PowerShell, crie uma se ainda não possuir:

```powershell
ssh-keygen -t ed25519 -C "devbox"
```

Por padrão, o Compose usa `${USERPROFILE}\.ssh\id_ed25519.pub`. Para usar outra chave pública:

```powershell
$env:SSH_PUBLIC_KEY_PATH = "$env:USERPROFILE\.ssh\minha-chave.pub"
```

## Estrutura

```text
.
├── Ubuntu/
│   ├── Dockerfile
│   └── compose.yaml
├── Debian/
│   ├── Dockerfile
│   └── compose.yaml
└── README.md
```

Cada ambiente monta a sua própria pasta `workspace/`, ao lado do respectivo `compose.yaml`. O login do Claude Code e as extensões do VS Code Server são preservados em volumes Docker.

## Executar com Ubuntu

```powershell
cd Ubuntu
docker compose up -d --build
```

Conecte pelo terminal:

```powershell
ssh -p 2222 dev@localhost
```

No VS Code Remote - SSH, conecte em `dev@localhost`, porta `2222`.

## Executar com Debian

```powershell
cd Debian
docker compose up -d --build
```

Conecte pelo terminal:

```powershell
ssh -p 2223 dev@localhost
```

No VS Code Remote - SSH, conecte em `dev@localhost`, porta `2223`. Os dois ambientes podem ficar ativos ao mesmo tempo, pois usam portas e nomes de container distintos.

## Comandos úteis

Execute estes comandos dentro da pasta do ambiente escolhido (`Ubuntu` ou `Debian`):

```powershell
# Acompanhar os logs
docker compose logs -f

# Abrir um shell sem SSH
docker compose exec devbox bash

# Parar e remover o container, preservando dados do Claude e extensões
docker compose down

# Remover também os dados persistentes
docker compose down -v
```

Para refazer a imagem sem cache:

```powershell
docker compose build --no-cache
docker compose up -d
```

## Claude Code

Após entrar no container, execute:

```bash
claude
```

Conclua a autenticação pelo link exibido. O estado fica salvo no volume `claude-config`, portanto normalmente não será necessário autenticar novamente depois de recriar o container.

## Publicar no Docker Hub com Buildx

Autentique-se no Docker Hub:

```powershell
docker login
```

Na raiz do repositório, publique a imagem Ubuntu para `linux/amd64` e `linux/arm64`:

```powershell
docker buildx build --platform linux/amd64,linux/arm64 --file Ubuntu/Dockerfile --tag harunokenobi/ubuntu-dev-ssh:latest --push Ubuntu
```

Para publicar a variante Debian:

```powershell
docker buildx build --platform linux/amd64,linux/arm64 --file Debian/Dockerfile --tag harunokenobi/debian-dev-ssh:latest --push Debian
```

`--push` é necessário para publicar uma imagem multi-arquitetura diretamente com Buildx. Para testar localmente no Docker Desktop antes de publicar, use a arquitetura local e `--load`:

```powershell
docker buildx build --platform linux/amd64 --file Debian/Dockerfile --tag harunokenobi/debian-dev-ssh:local --load Debian
```

## Segurança

- As imagens não aceitam senha nem login SSH como `root`.
- A chave pública é copiada para o container na inicialização com permissões restritas; reinicie o container após trocar a chave no host.
- Nunca adicione tokens, arquivos privados ou a chave privada SSH à imagem ou ao repositório.
- Não exponha as portas `2222` e `2223` em rede pública sem firewall, controle de acesso e uma revisão de segurança.
