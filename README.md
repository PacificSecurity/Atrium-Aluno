# Atrium — laboratório da atividade individual

Este é o laboratório da atividade individual de segurança web, disponibilizado
para estudo e prática após a avaliação. O pacote traz as oito variantes, de A a H,
e o [writeup das 23 flags](WRITEUP.md).

Se você não participou da atividade, comece pela [descrição da atividade](ATIVIDADE.md)
para conhecer o objetivo, o escopo, as regras e os tipos de desafio.

A aplicação mantém os mecanismos da atividade. As imagens usam dados de cenário
e flags próprios para prática; os valores não correspondem aos utilizados nas
instâncias da avaliação original. Pagamentos e envio de e-mails são simulados.

## Começar

Com o Docker em execução, abra um terminal na pasta deste repositório:

```bash
docker load -i atrium-lab.tar.gz
docker run -d --name atrium --cap-drop=ALL --security-opt=no-new-privileges -p 127.0.0.1:8000:8000 atrium-lab:A
```

Abra [http://127.0.0.1:8000](http://127.0.0.1:8000), clique em **Criar conta** e
use o código **`ATRIUM-3FA145`**. Cadastre dados fictícios e uma senha descartável.
Depois, faça login com a conta criada.

O primeiro comando carrega as oito imagens; o segundo inicia somente a variante A.
Não é preciso instalar Python, usar Compose ou executar scripts de preparação.

**Compatibilidade:** este tar contém imagens `linux/amd64`, para computadores
Intel/AMD com Docker configurado para contêineres Linux. Em Macs Apple Silicon
e outras máquinas ARM64, é necessária emulação AMD64. Consulte
[Como subir](COMO-SUBIR.md) para os requisitos e os comandos de operação.

## Arquivos

| Arquivo | Conteúdo |
|---|---|
| [ATIVIDADE.md](ATIVIDADE.md) | Contexto, objetivo, escopo, regras e distribuição das flags |
| [atrium-lab.tar.gz](atrium-lab.tar.gz) | As oito imagens Docker prontas para carregar |
| [SHA256SUMS](SHA256SUMS) | Checksum para conferir a integridade do tar |
| [COMO-SUBIR.md](COMO-SUBIR.md) | Cadastro, troca de variante, reinício e solução de problemas |
| [WRITEUP.md](WRITEUP.md) | Como encontrar e explorar cada falha, com exemplos e explicações |

## Como praticar

Escolha uma flag que queira rever e tente resolvê-la pelo navegador ou pelo seu
proxy. Consulte a seção correspondente do writeup quando precisar de ajuda.
Guarde as respostas que sustentam a conclusão: o objetivo é entender a falha e
conseguir explicar por que o teste funcionou.

A numeração não indica a ordem de resolução. Por exemplo, a flag 8 permite virar
organizador; depois vem a flag 7, que permite refazer o login nesse papel.
O writeup informa os pré-requisitos de cada etapa.

Cada imagem tem cenário, código de cadastro e flags fixos. Dois colegas usando
a mesma imagem terão os mesmos valores. Para recomeçar, remova o contêiner e
crie outro; para guardar seu progresso, use `docker stop` e `docker start`.

O laboratório é vulnerável de propósito. Mantenha o endereço `127.0.0.1` nos
comandos e use os exercícios apenas nesse ambiente local.
