# Como subir o Atrium

O arquivo `atrium-lab.tar.gz` já contém as oito variantes. Você carrega as imagens
uma vez e escolhe qual delas quer executar.

## Requisitos

- Docker Desktop atualizado, ou Docker Engine 28 ou superior, em execução.
- Espaço para o tar (cerca de 48 MB) e para as imagens carregadas.
- Computador Intel/AMD com o Docker configurado para contêineres Linux,
  ou uma máquina ARM64 com emulação AMD64 disponível no Docker.
- A porta `8000` livre. Se estiver ocupada, use a alternativa abaixo.

O pacote é **`linux/amd64`**. Em Macs Apple Silicon e outras máquinas ARM64,
depende de emulação. A opção `--platform linux/amd64` seleciona a plataforma,
mas não instala a emulação. Se aparecer `exec format error`, confira a
configuração do Docker.

Os comandos de operação abaixo funcionam em Bash, PowerShell e Prompt de Comando.
Não é preciso extrair o tar nem instalar Python ou Compose para subir o lab.

## 1. Conferir o arquivo

Na pasta que contém `atrium-lab.tar.gz` e `SHA256SUMS`:

**macOS:**

```bash
shasum -a 256 -c SHA256SUMS
```

**Linux:**

```bash
sha256sum -c SHA256SUMS
```

O resultado esperado é `atrium-lab.tar.gz: OK`.

**PowerShell:**

```powershell
Get-FileHash .\atrium-lab.tar.gz -Algorithm SHA256
Get-Content .\SHA256SUMS
```

Compare os dois hashes, ignorando a diferença entre maiúsculas e minúsculas.
Se não coincidirem, obtenha outra cópia do arquivo antes de continuar.

## 2. Carregar e iniciar

```bash
docker load -i atrium-lab.tar.gz
docker run -d --name atrium --cap-drop=ALL --security-opt=no-new-privileges -p 127.0.0.1:8000:8000 atrium-lab:A
```

O `load` carrega as tags `atrium-lab:A` até `atrium-lab:H`. O `run` inicia somente
A, em segundo plano. Espere alguns segundos e abra
[http://127.0.0.1:8000](http://127.0.0.1:8000).

Para conferir se o contêiner está em execução:

```bash
docker ps --filter name=atrium
```

O mapeamento de porta deve começar com `127.0.0.1:8000`.

O endereço `127.0.0.1` mantém a porta publicada no computador. As opções
`--cap-drop=ALL` e `--security-opt=no-new-privileges` retiram privilégios que o
laboratório não precisa. Não use `--privileged`, não monte pastas pessoais e não
publique o lab em um servidor ou túnel. A recomendação de Docker 28 ou superior
considera a [correção do acesso a portas locais em versões antigas](https://docs.docker.com/engine/network/port-publishing/#publishing-ports).

Essas opções limitam o acesso ao lab e os privilégios do processo, mas não
bloqueiam todas as conexões de saída do contêiner.

## Cadastro

Na página inicial, clique em **Criar conta**. Use nome e e-mail fictícios, uma
senha descartável com pelo menos oito caracteres e o código da variante:

| Variante | Código | Evento |
|:--:|---|---|
| A | `ATRIUM-3FA145` | XXIV Congresso de Cardiologia Intervencionista |
| B | `ATRIUM-C47E12` | Festival Vertente 2026 |
| C | `ATRIUM-C49D1E` | ExpoLogística Sul 2026 |
| D | `ATRIUM-D20D6A` | Hack Atlântico 2026 |
| E | `ATRIUM-44EDDA` | Convenção Anual Nexo 2026 |
| F | `ATRIUM-26C393` | Fórum de Direito Digital 2026 |
| G | `ATRIUM-8FB208` | EducaTech Nordeste 2026 |
| H | `ATRIUM-F7BC13` | DevSummit Brasil 2026 |

Depois do cadastro, faça login. A instância aceita até 20 contas; alguns
exercícios precisam de duas contas diferentes.

## Parar e retomar

```bash
docker stop atrium
docker start atrium
```

Esses comandos preservam contas, ingressos e progresso. Os dados ficam na área
gravável do contêiner, sem volumes ou pastas compartilhadas com o computador.
As sessões expiram normalmente; ao voltar, pode ser necessário fazer login.

## Zerar ou trocar de variante

Para apagar os dados do teste e começar de novo na variante A:

```bash
docker rm -f atrium
docker run -d --name atrium --cap-drop=ALL --security-opt=no-new-privileges -p 127.0.0.1:8000:8000 atrium-lab:A
```

Para testar outra variante, troque a letra final, por exemplo `atrium-lab:D`, e
use o código de cadastro correspondente. Não é necessário carregar o tar de novo.

Remover o contêiner apaga as contas e o progresso. A imagem continua no Docker;
um novo contêiner começa com os mesmos dados de cenário e valores de flags.

## Porta ocupada

Use `8081` no computador, mantendo `8000` dentro do contêiner:

```bash
docker run -d --name atrium --cap-drop=ALL --security-opt=no-new-privileges -p 127.0.0.1:8081:8000 atrium-lab:A
```

Abra [http://127.0.0.1:8081](http://127.0.0.1:8081). Se uma tentativa anterior
criou um contêiner chamado `atrium`, remova-o antes de repetir o `run`.

## Se não funcionar

### Aviso de plataforma num Mac recente

Em Mac com chip Apple (M1 em diante), o Docker avisa algo como:

```
WARNING: The requested image's platform (linux/amd64) does not match
the detected host platform (linux/arm64/v8)
```

Se a emulação AMD64 estiver disponível, o laboratório pode rodar mesmo com
esse aviso. O desempenho pode ser menor. Para declarar a plataforma ao criar
o contêiner, use:

```bash
docker run -d --platform linux/amd64 --name atrium --cap-drop=ALL --security-opt=no-new-privileges -p 127.0.0.1:8000:8000 atrium-lab:A
```

### Outros problemas

| Situação | O que conferir |
|---|---|
| `Cannot connect to the Docker daemon` | Inicie o Docker e espere ele ficar disponível. |
| O nome `atrium` já está em uso | Use `docker start atrium` para retomar. Para recomeçar, use `docker rm -f atrium`. |
| `port is already allocated` | Use a alternativa de porta acima. |
| `exec format error` | A imagem AMD64 precisa de um computador Intel/AMD ou de emulação compatível no Docker. |
| A página não abre | Veja `docker ps -a` e `docker logs atrium`. |
| `codigo_invalido` | Confira a letra da imagem e o código na tabela de cadastro. |
| `limite_de_contas` | Zere o contêiner para criar novas contas. |

Para acompanhar os logs, use `docker logs -f atrium`. `Ctrl+C` encerra apenas
esse acompanhamento; o laboratório continua em execução.

## Encerrar e remover as imagens

Quando não precisar mais do laboratório:

```bash
docker rm -f atrium
docker image rm atrium-lab:A atrium-lab:B atrium-lab:C atrium-lab:D atrium-lab:E atrium-lab:F atrium-lab:G atrium-lab:H
```

Se o Docker disser que uma imagem está em uso, confira em `docker ps -a` qual
contêiner ainda depende dela antes de removê-lo.
