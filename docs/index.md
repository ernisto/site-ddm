# Fazendo seu próprio parser

Parsers são programas responsáveis por transformar um texto (que seguir uma gramatica) em uma estrutura de dado mais fácil de trabalhar. São comuns em ferramentas de linguagem de programação, como compiladores, interpretadores, e até mais comumente para transformar em um hash map, ou dicionário manipulável o seu JSON, YAML, XML, etc.
Aqui será mostrado como fazer um parser tipicamente de uma linguagem de programação, usando a linguagem [Luau](https://github.com/luau-lang/luau), uma linguagem de programação de scripting simples, rápida e tipada, derivada da linguagem [Lua](https://www.lua.org/).

## Abstrações (divisões do programa)
Parsers de linguagens de programação, diferente de arquivos de configuração simples, são normalmente divididas em algumas etapas, tanto por desempenho, quanto para fornecer mais recursos para os compiladores e interpretadores.

1. O **Lexer**, que será responsável por ler todo o texto, e transformar vários pedacinhos desses textos em **Tokens** (lexing), que geralmente vai ser um pequeno objeto que guarda seu tipo, e opcionalmente vai guardar algumas outras informações úteis, como o texto cru do **Token**, e posição onde o **Token** foi encontrado.
	- **Tokens** são pequenos pedaços do texto, eles são usados pelo **Parser** para discernir pequenos pedaços de texto mais eficientemente, invés de sempre fazer lexing em contextos possivelmente errados, o que resultara no lexing sendo feito várias vezes para o mesmo **Token**.
	- Alguns tipos de **Token**s bem comuns em uma linguagem de programação seriam
		- **palavras** (como **if**, ou nome das variáveis).
		- **Números** (como **10**, **1.5**, ou **3e+2**).
		- **Strings** (como **"Olá Mundo"**).
2. O **Token Stream** (corrente/fluxo de **Tokens**), que será a parte do programa responsável por dar o **Token** atual, avançar para o próximo, e ocasionalmente voltar para um anterior. É importante ter um fluxo sequencial dos **Tokens**, porque vamos descartando os tokens já usados, facilitando o trabalho com os tokens restantes.
3. O próprio **Parser** em si, que será quem vai ler toda a cadeia de **Tokens** para a própria estrutura de dados mais conveniente para você, se for um parser de JSON você vai querer transformar em uma Map, no caso de linguagens de programação, vamos transformar em uma **AST** (**A**bstract **S**yntax **T**ree).
	- **AST** como uma árvore que tem vários galhos emergidos de outros galhos, árvores na programação também tem esses "galhos", porém são chamados de **Nodes** (nós), onde cada **Node** pode se referir a alguns outros **Nodes**, cada tipo de node vai representar um conjunto de tokens, com esse conjunto você vai aos poucos juntando, e obtendo informações mais detalhadas do código.
	- Alguns tipos de **Nodes** típicos em uma linguagem de programação são
		- **Node** de blocos de código, onde guarda vários outros **Nodes** de **Statements**.
		- **Nodes** de **Statements** (instruções), como um.
			- **Node** para definição de variável, onde guardara o nome da variável, e um **Node** de **Expression** para representar o valor inicial dela.
			- **Node** para um if, que guardaria vários nós possibilidades, onde cada possibilidade teria um **Node** de **Expression** para representar a condição, e outro de bloco, para as instruções executadas caso a condição seja verdadeira.
		- **Nodes** de **Expressions** (expressões), que representam valores primitivos, ou operações de outras expressões.
# Mão na massa
Primeiro vamos começar definindo os tipos de tokens
```lua
type pos = {
    x: number, -- coluna dentro da linha atual
    y: number,  -- linha do código fonte
    z: number  -- coluna absoluta, sendo somada com a coluna das outras linhas
}

type Token = NumberToken | StringToken | WordToken | CharToken
type baseToken = { -- tipo base para os outros tipos de tokens, com dados que todos terao
    start: pos,
    final: pos,
    trivia: string  -- espaços em branco após o token
}

type StringToken = baseToken & { kind: 'string', content: string }
type NumberToken = baseToken & { kind: 'number', value: number }
type WordToken = baseToken & { kind: 'word', text: string }
type CharToken = baseToken & { kind: 'char', char: string }
```
E então vamos começar com o lexer, como um tutorial introdutório, vai ser uma implementação simples, porém lerda (ainda mais por estarmos usando uma linguagem de alto nível) porém vai ser suficiente para entender os fundamentos.
```lua
-- context
local x = 1
local y = 1
local z = 1    -- posicao inicial da string que estamos lendo
local pos = { x = x, y = y, z = z }
local tokens = {}

-- redefine estados globais do modulo
-- pode ocassionar bugs se fizer isso, antes de terminar de usar os estados anteriores
local function lex(sourceCode: string) -> {Token}

	x = 1
	y = 1
	z = 1    -- posicao inicial da string que estamos lendo
	pos = { x = x, y = y, z = z }
	tokens = {}
end

-- futuramente podemos incluir gramatica de comentários aqui
local function scanTrivia(): string

	local trivias = {}

	local _,final, trivia = string.find(sourceCode, "^([ \t]*)", z)
	if final then table.insert(trivias, trivia); z = final+1 end

	while string.sub(sourceCode, z, z) == '\n' do
		x = 1
		y += 1
		z += 1
		_,final, trivia = string.find(sourceCode, "^([ \t]*)", z)
		if final then table.insert(trivias, trivia); z = final+1 end
	end

	pos = { x = x, y = y, z = z }
	return table.concat(trivias)
end

-- normalmente
local function scanNumber(): NumberToken?

	local start = pos

	local _,final, decimal = string.find(sourceCode, "^(%d+)", z)
	if decimal then z = final+1 end

	local _,final, fractional = string.find(sourceCode, "^%.(%d+)", z)
	if fractional then z = final+1 end

	if not decimal and not fractional then return end  -- aqui ja desconsideramos esse token como um número

	local final = { x = x + (z - start.z - 1), y = y, z = final }
	return { kind = 'number',
		start = start,
		final = final,
		trivia = scanTrivia(),

		decimal = decimal,
		fractional = fractional,
		exponent = exponent,
		expSign = sign,
	}
end
local function scanString(): StringToken?

	local start = pos

	local _,final, content = string.find(sourceCode, "^(%b\"\")")
	if not final then return end
	z = final - 1

	local final = { x = x + (z - start.z - 1), y = y, z = final }
	return { kind = 'string'
		start = start,
		final = final,
		trivia = scanTrivia(),
		content = content
	}
end
local function scanWord(): WordToken?

	local start = pos
	local _,final, word = string.find(sourceCode, "^(%w+)")
	if not final then return end

	z = final - 1

	local final = { x = x + (z - start.z - 1), y = y, z = final }
	return { kind = 'word',
		start = start,
		final = final,
		trivia = scanTrivia(),
		word = word,
	}
end
local function scanChar(): Token

	local start = pos
	local _,final, char = string.find(sourceCode, "^(.)")
	z += 1

	local final = { x = x + 1, y = y, z = final }
	return { kind = 'char',
		start = start,
		final = final,
		trivia = scanTrivia(),
		char = char,
	}
end
local function scanToken()
	return scanNumber()
		or scanString()
		or scanWord()
		or scanChar()
end
```
Após ter feito o **Lexer**, vamos tokenizar tudo de uma vez, e expor alguns métodos para consumir os tokens desde o início, descartando os anteriores
```lua
local tokens: {Token}
local cursor: number
local token: Token

local function tokenizeAll(_sourceCode: string)

    lex(_sourceCode)
    tokens = {}
    repeat
        local token = scanToken()
        if token then table.insert(tokens, token) end

    until not token

    cursor = 1
    token = tokens[cursor]
end

local function consume(): Token

    local last = token
    cursor += 1
    token = tokens[cursor]
    return last
end

-- uma das razões para tokenizar tudo primeiro, é para não ter que re-tokenizar o mesmo
-- token Word por não atender a diferentes exigencias do parametro 'specific' que vão aparecer conforme o parsing de um arquivo inteiro
local function consumeWord(specific: string?): WordToken?
    return if token and token.kind == 'word' and (not specific or token.text == specific)
        then consume() :: any else nil
end
local function consumeChar(specific: string?): CharToken?
    return if token and token.kind == 'char' and (not specific or token.char == specific)
        then consume() :: any else nil
end
local function consumeNum(): NumberToken?
    return if token and token.kind == 'number' then consume() :: any else nil
end
local function consumeStr(): StringToken?
    return if token and token.kind == 'string' then consume() :: any else nil
end
```
Após ter feito o **Lexer**, vamos escrever o **Token Stream**, que usará o **Lexer** para gerar uma **AST** do código recebido, usando os **Tokens** definidos previamente.
vamos definir os tipos de **Nodes**(nós) que teremos na nossa gramática
```lua
type block = { kind: 'block', stats: {stat} }
type stat = assignment | callment | if_stat
type assignment = { kind: 'assignment', targetName: string, value: expr }
type callment = { kind: 'callment', func: expr, args: {expr} }
type if_stat = { kind: 'if', cond: expr, then_body: block, else_body: block? }

type expr = str_expr | num_expr | read_expr | biop_expr
type str_expr = { kind: 'str', str: StringToken }
type num_expr = { kind: 'num', num: NumberToken }
type read_expr = { kind: 'read', name: WordToken }
type biop_expr = { kind: 'biop', op: string, right: expr, left: expr }
```

E então, vamos implementar o parser de fato. Vale notar que existe várias estratégias de implementação, mas o parser implementado aqui é um parser **LL** (**L**eft-to-right **L**eft-must), descendente e recursivo ser combinar pequenas funções em funções maiores. Esse tipo de estratégia é a forma mais simples, e flexível de se implementar.
```lua
local parse_body: () -> block?

local function token_to_str(token: Token)
    return if token.kind == 'word' then token.text
        elseif token.kind == 'char' then token.char
        elseif token.kind == 'string' then token.content
        else tostring(token.value)
end
local function parsing_error(message: string): never
    error(`erro em {token.start.y}:{token.start.x}:\
        {message} (recebido: '{token_to_str(token)}')`)
end

-- expr
-- aqui é importante a configuração de precedencia e associatividade de cada operador
local LEVEL = {     -- todos os operadores: left-associative: ex: ((1 + 2) + 3) + 4
    ['<'] = 1,
    ['<='] = 1,
    ['>'] = 1,
    ['>='] = 1,
    ['=='] = 1,
    ['!='] = 1,

    ['+'] = 2,
    ['-'] = 2,

    ['*'] = 3,
    ['/'] = 3,
    ['%'] = 3,

    ['^'] = 4, -- exceto esse: right-associative, ex: 2^(3^(4^5))
}

local function parse_atom(): expr?

    local str = consumeStr()
    if str then return { kind = 'str', str = str } end

    local num = consumeNum()
    if num then return { kind = 'num', num = num } end

    local name = consumeWord()
    if name then return { kind = 'read', name = name } end

    return
end

local function parse_expr(_left: expr?, _currentLevel: number?): expr?

    local left = _left or parse_atom()
    if not left then return end

    local currentLevel = _currentLevel or 0
    repeat
        local op = if token and token.kind == 'char' then token else nil
        if not op then break end
        local op = op.char

        local level = LEVEL[op]
        if not level then break end -- volta o cursor, como se estivesse devolvendo o token consumido

        if level <= currentLevel then break end
        consume()

        local subParseLevel = if op == '^' then level-1 else level  -- por causa que esse operador em especifico é right-associative
        local right = parse_expr(nil, subParseLevel)
            or parsing_error(`expressao esperada`)

        left = { kind = 'biop', op = op, left = left, right = right }

    until false
    return left
end

-- stat
local function parse_assignment()

    if not consumeWord('set') then return end
    local varName = consumeWord()
    local _1 = consumeWord('to') or parsing_error(`token 'to' esperado`)
    local value = parse_expr()

    return {
        kind = 'assignment',
        targetName = varName,
        value = value
    }
end
local function parse_callment()

    local func = parse_expr()
    if not func then return end

    local _1 = consumeChar('(') or parsing_error(`token '(' esperado`)
    local args = { parse_expr() }

    while consumeChar(',') do

        local arg = parse_expr() or parsing_error(`argumento depois da ',' esperado`)
        table.insert(args, arg)
    end
    local _2 = consumeChar(')') or parsing_error(`token ')' esperado`)

    return { kind = 'callment', func = func, args = args }
end
local function parse_if(): if_stat?

    if not consumeWord('if') then return end
    local cond = parse_expr() or parsing_error(`expressao esperada`)
    local then_body = parse_body() or parsing_error(`instrucao esperada`)
    local else_body = if consumeWord('else') then parse_body() or parsing_error(`instrucao esperada`) else nil

    return { kind = 'if', cond = cond, then_body = then_body, else_body = else_body }
end

local function parse_stat(): stat?
    return parse_if()
        or parse_assignment() -- precedencia importante aqui
        or parse_callment()
end

-- block
local function parse_block(): block?

    if not consumeChar('{') then return end
    local stats = {}
    repeat
        local stat = parse_stat()
        if not stat then break end

        table.insert(stats, stat)
        consumeChar(';')
    until false

    local _1 = consumeChar('}') or parsing_error(`token '}' esperado`)
    return { kind = 'block', stats = stats }
end

local function parse_simple_block(): block?

    local stat = parse_stat()
    if not stat then return end

    return { kind = 'block', stats = { stat } }
end
function parse_body(): block?
    return parse_block()
        or parse_simple_block()
end
```
Por fim, você pode testar rodando
```lua
tokenizeAll([[{
	set a to 1*2*3 + 4 + 10 + 3^5^7;
	if a < 4 {
		print("a..4")
	} else if 6 < a {
		print("6..a")
	} else {
		print("4..a..6")
	}
}]])

print(tokens)
print('ast:', parse_block())
```
E por hoje é tudo, vale notar vários pontos a serem melhorados nesse tipo de implementação.

- Primeiro de tudo, parsers são consideravelmente caros, e por isso, boa parte são implementados em linguagens de baixo nível, para um desempenho melhor em arquivos maiores.
	- Boa parte da implementação é a forma mais eficiente, embora ainda haja algumas estratégias que podem ser adotadas para melhorar muito mais o desempenho
	- O TokenStream pode ser transformado em uma espécie de lazy stream (corrente preguiçosa), onde invés de tentar todas as combinações possíveis para cada token do texto, você usa a própria expectativa do parser para começar tentando "lexar" os Tokens corretos primeiro (que será o caso mais comum onde o parser vai agir). Misturando isso com também, ordenar os sub-parsers na ordem mais estatisticamente comum, você pode cortar mais que o dobro de tentativas de lexing inúteis. Não só isso, como tentar lexar na hora de consumir te dá resposta imediata se o token é do tipo esperado, oque elimina para nós uma operação de leitura da propriedade "kind" (micro otimização de grande peso).
	- Otimizar parsers recursivos com recursão de calda, transformando recursões em meros loops (desde que não exija criar qualquer tipo de stack (oque na prática, teria o mesmo custo de recursão, porém mais cara por estar sendo implementada em luau, invés de internamente, em C)).
	- Lembrar que micro otimizações vão importar nessas pequenas funções, pois alguma delas poderão ser chamadas milhares de vezes em um único documento
- Esse parser simplesmente trava no primeiro erro encontrado, ele ainda não é "ignorante a erros", então se quiser diagnósticos mais detalhados, você deve procurar implementar "recuperação de erro"
- Ele exporta muito poucas informações do parsing, como localização de cada token

Portanto, é uma implementação apenas para demonstração de conceito, serve apenas para ter uma ideia de onde começar para fazer o seu próprio