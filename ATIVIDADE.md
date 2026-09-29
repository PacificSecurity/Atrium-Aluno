# A atividade: objetivo, escopo e desafios

Esta é uma versão pública e adaptada do enunciado da atividade final do módulo
Web. Ela apresenta o contexto do exercício para quem não participou da turma e
quer praticar com o laboratório deste repositório.

Para executar a aplicação, siga o [COMO-SUBIR.md](COMO-SUBIR.md). As soluções
estão no [WRITEUP.md](WRITEUP.md); tente investigar os desafios antes de consultá-las.

## Objetivo

A proposta era atuar como responsável por um teste de invasão em uma aplicação
web fictícia, reunindo os conteúdos estudados no módulo. O trabalho consistia em
entender a aplicação, formular hipóteses, explorar falhas e explicar suas causas
e possíveis correções.

Encontrar uma flag demonstrava a exploração de um caminho. Entender por que
aquele caminho funcionava também fazia parte da atividade.

## A aplicação

A Atrium é uma plataforma SaaS fictícia de gestão de eventos corporativos. Ela
inclui criação de eventos, ingressos por lote, cupons, transferências, reembolsos,
credenciamento e consultas de vendas e repasses. Pagamentos e envio de e-mails
são simulados.

Existem quatro níveis de acesso:

1. Participante.
2. Credenciamento.
3. Organizador.
4. Operador da plataforma.

A conta criada pelo aluno começa como participante. Há desafios acessíveis
antes do cadastro e outros que dependem de avançar pelos papéis da aplicação.

O sistema é grande de propósito: várias telas e funcionalidades legítimas não
contêm flags. A investigação exige observar os fluxos e testar hipóteses;
encontrar uma tela ou um campo diferente não basta para demonstrar uma falha.

## Escopo

Na avaliação original, cada participante recebia uma instância individual. O
escopo autorizado era exclusivamente o subdomínio daquela instância.

Ficavam fora do escopo:

- Outras instâncias e outros subdomínios.
- O domínio principal.
- A plataforma de submissão das flags.
- O servidor e a infraestrutura que hospedavam os ambientes.

Neste repositório, a prática acontece na instância local do Atrium iniciada com
Docker e nos componentes do laboratório que fazem parte dela. O computador que
executa o Docker, os demais serviços da máquina, a rede e outros sites não são
alvos do exercício. Siga o guia de execução, que publica a aplicação somente
em `127.0.0.1`.

## Regras de investigação

Estas eram as regras da atividade e são a referência para reproduzir a experiência:

- **Nenhuma flag exige adivinhação.** Os caminhos começam em informações da
  aplicação ou em convenções públicas e documentadas.
- **Sem fuzzing, força bruta, enumeração de diretórios ou wordlists.** Essas
  técnicas eram proibidas e não são necessárias para resolver os desafios.
- **Sem scanners automatizados.** A proposta é investigar os fluxos e testar
  hipóteses com base no comportamento observado.
- **Sem vítima simulada.** Nenhuma flag depende de um bot, navegador headless
  ou de outra pessoa clicar em um link. A resolução não exige interação de terceiros.
- **Use as contas com cuidado.** Cada instância aceita no máximo 20 contas;
  alguns desafios precisam de duas contas diferentes.

## Flags e pontuação

São **23 flags**, que somavam **100 pontos** na avaliação original: as flags
`01` a `22` e uma flag final, identificada por `FF`.

O formato é `OCEAN{NN-<12 caracteres hexadecimais minúsculos>}`. O campo `NN`
identifica a flag; na final, ele é substituído por `FF`.

A numeração não determina a ordem de resolução. Algumas flags dependem de
informações ou acessos obtidos em outras etapas.

### Tipos de desafio

| Categoria | Quantidade de flags | Pontos |
|---|---:|---:|
| Information Disclosure / Misconfiguration | 4 | 7 |
| Parameter Tampering | 1 | 3 |
| Authentication e Access Control | 3 | 10 |
| Business Logic | 3 | 17 |
| CSRF e CORS | 4 | 14 |
| Análise de cliente | 1 | 3 |
| Cross-Site Scripting (XSS) | 1 | 7 |
| GraphQL | 3 | 9 |
| SQL Injection | 1 | 5 |
| Server-Side Request Forgery (SSRF) | 1 | 9 |
| Encadeamento: flag final | 1 | 16 |
| **Total** | **23** | **100** |

No enunciado original, as flags 20 e 21 estavam agrupadas sob Cross-Site Scripting,
totalizando duas flags e 10 pontos. A tabela acima segue a distinção do writeup
atual: análise de cliente na flag 20 e XSS na flag 21. A quantidade total de
flags e a pontuação permanecem as mesmas.

### Etapas indicadas no enunciado original

Esta tabela mostra o percurso sugerido aos participantes. As etapas não são uma
garantia de exigência de sessão em cada endpoint: o writeup detalha os acessos
efetivos e as dependências de cada desafio.

| Flag | Etapa do percurso | Pontos |
|---|---|---:|
| 01 | Sem conta | 1 |
| 02 | Durante o cadastro | 3 |
| 03 | Sem conta | 2 |
| 04 | Sem conta | 1 |
| 05 | Organizador | 3 |
| 06 | Conta de participante | 3 |
| 07 | Escalada de privilégio | 4 |
| 08 | Escalada de privilégio | 3 |
| 09 | Conta de participante | 2 |
| 10 | Conta de participante | 3 |
| 11 | Conta de participante | 4 |
| 12 | Organizador | 5 |
| 13 | Conta de participante | 5 |
| 14 | Conta de participante | 6 |
| 15 | Conta de participante | 6 |
| 16 | Conta de participante | 2 |
| 17 | Conta de participante | 5 |
| 18 | Sem conta | 2 |
| 19 | Sem conta | 5 |
| 20 | Organizador | 3 |
| 21 | Organizador | 7 |
| 22 | Operador da plataforma | 9 |
| FF | Operador da plataforma | 16 |
| **Total** | **23 flags** | **100** |

## Como a atividade foi avaliada

A avaliação original tinha duas partes independentes, de 100 pontos cada:

- **Flags:** pontuação acumulada pelos desafios resolvidos.
- **Análise técnica:** um relatório em PDF, com até 10 páginas, explicando três
  fluxos completos. As flags previstas eram 12, 21 e 22; quando uma delas não
  tivesse sido resolvida, poderia ser substituída por outra explorada pelo aluno.

A análise incluía sumário, introdução, desenvolvimento e conclusão. Para cada
fluxo, eram esperados a pista inicial, o passo a passo, evidências relevantes e
uma recomendação concreta de correção. A conclusão relacionava as falhas e suas
prioridades de tratamento.

Esses critérios explicam a proposta pedagógica da atividade. A versão pública
é destinada a estudo e prática local, sem obrigação de entrega ou submissão.

## Como usar este material

1. Siga o [guia de execução](COMO-SUBIR.md) e escolha uma das oito variantes,
   de A a H.
2. Explore as funcionalidades públicas e crie sua conta com o código da variante.
3. Observe requisições e respostas, formule hipóteses e teste-as dentro do escopo.
4. Consulte o [writeup](WRITEUP.md) para comparar o seu raciocínio com os percursos
   documentados e entender as dependências entre os desafios.

As imagens distribuídas usam dados de cenário e flags próprios para prática.
Os valores são fixos por imagem e não correspondem aos utilizados nas instâncias
da avaliação original.
