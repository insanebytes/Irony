# Guía Completa de Definición de Gramáticas en Irony

Una gramática en Irony es una clase C# que hereda de `Grammar` y declara su estructura dentro del constructor. No hay archivos `.g`, `.yacc` ni generadores de código — todo es C# puro.

---

## Índice

1. [Estructura general](#1-estructura-general)
2. [Propiedades globales de la gramática](#2-propiedades-globales-de-la-gramática)
3. [Terminales estándar incorporados](#3-terminales-estándar-incorporados)
4. [KeyTerms y `ToTerm()`](#4-keyterms-y-toterm)
5. [NonTerminal](#5-nonterminal)
6. [Operadores BNF: `+`, `|` y `Q()`](#6-operadores-bnf--y-q)
7. [MakePlusRule y MakeStarRule](#7-makeplusrule-y-makestarrule)
8. [Registro de Operadores y Precedencia](#8-registro-de-operadores-y-precedencia)
9. [RegisterBracePair](#9-registerbracepair)
10. [Métodos de Marcado](#10-métodos-de-marcado)
11. [Grupos de Reporte de Errores](#11-grupos-de-reporte-de-errores)
12. [LanguageFlags](#12-languageflags)
13. [Hints de Gramática (resolución de conflictos)](#13-hints-de-gramática-resolución-de-conflictos)
14. [SnippetRoots](#14-snippetroots)
15. [NonGrammarTerminals](#15-nongrammarterminals)
16. [Lenguajes con Indentación](#16-lenguajes-con-indentación)
17. [Métodos virtuales sobreescribibles](#17-métodos-virtuales-sobreescribibles)
18. [TermFlags — Flags de términos](#18-termflags--flags-de-términos)

---

## 1. Estructura General

```csharp
using Irony.Parsing;

[Language("MiLenguaje", "1.0", "Descripción opcional")]
public class MiGramatica : Grammar {

    public MiGramatica() : base(caseSensitive: true) {

        // ── Paso 1: Definir terminales ────────────────────────────
        var number     = new NumberLiteral("number");
        var identifier = new IdentifierTerminal("identifier");
        var comment    = new CommentTerminal("comment", "//", "\n");
        NonGrammarTerminals.Add(comment);

        // ── Paso 2: Declarar no-terminales ───────────────────────
        var program    = new NonTerminal("Program");
        var statement  = new NonTerminal("Statement");
        var expr       = new NonTerminal("Expr");

        // ── Paso 3: Reglas BNF ───────────────────────────────────
        program.Rule   = MakePlusRule(program, statement);
        statement.Rule = expr + ";" | "return" + expr + ";";
        expr.Rule      = number | identifier | expr + "+" + expr;

        // ── Paso 4: Precedencia ──────────────────────────────────
        RegisterOperators(10, "+", "-");
        RegisterOperators(20, "*", "/");

        // ── Paso 5: Marcado de términos especiales ───────────────
        MarkPunctuation(";");
        MarkTransient(expr);  // si expr es solo un "wrapper"

        // ── Paso 6: Configuración general ───────────────────────
        Root = program;
        LanguageFlags = LanguageFlags.CreateAst;
    }
}
```

El atributo `[Language]` es opcional pero útil: lo usa `GrammarExplorer` para mostrar el nombre, versión y descripción de la gramática.

---

## 2. Propiedades Globales de la Gramática

### `CaseSensitive`

```csharp
public readonly bool CaseSensitive;
// Se establece solo en el constructor:
public Grammar() : this(true) { }
public Grammar(bool caseSensitive)
```

Cuando es `false`, todas las comparaciones de `KeyTerm` se hacen en minúsculas. Los identificadores y literales tienen sus propias reglas.

### `Root`

```csharp
public NonTerminal Root;
```

El símbolo de inicio de la gramática. **Obligatorio**. El parser crea internamente la producción aumentada `Root' → Root`.

### `GrammarComments`

```csharp
public string GrammarComments;
```

Texto libre que se muestra en la pestaña "Grammar Info" del `GrammarExplorer`.

### `DefaultCulture`

```csharp
public CultureInfo DefaultCulture = CultureInfo.InvariantCulture;
```

Cultura usada en conversiones numéricas y de cadena.

### Propiedades de Consola (para REPL)

```csharp
public string ConsoleTitle;
public string ConsoleGreeting;
public string ConsolePrompt;           // Prompt normal, por defecto ">"
public string ConsolePromptMoreInput;  // Prompt cuando se espera más entrada, por defecto "."
```

---

## 3. Terminales Estándar Incorporados

Estos terminales son propiedades de `Grammar` y están disponibles directamente en el constructor:

| Terminal | Tipo | Descripción |
|---|---|---|
| `Empty` | `Terminal` | Elemento vacío para hacer partes opcionales: `rule \| Empty` |
| `NewLine` | `NewLineTerminal` | Fin de línea físico. Activa `UsesNewLine = true` automáticamente. |
| `Indent` | `Terminal` | Token de sangría (Python-style). Lo produce `CodeOutlineFilter`, no el scanner. |
| `Dedent` | `Terminal` | Token de desangría (Python-style). Lo produce `CodeOutlineFilter`. |
| `Eos` | `Terminal` | End-of-Statement virtual (fin de sentencia en lenguajes sensibles a la sangría). |
| `Eof` | `Terminal` | Fin de archivo. Se agrega automáticamente como lookahead del `Root`. |
| `Skip` | `Terminal` | Token artificial que el parser ignora completamente. |
| `SyntaxError` | `Terminal` | Token de error (no lo produce el scanner). |
| `NewLinePlus` | `NonTerminal` | Uno o más saltos de línea (lazy, se crea al primer acceso). |
| `NewLineStar` | `NonTerminal` | Cero o más saltos de línea (lazy, se crea al primer acceso). |

**Ejemplo de uso:**

```csharp
// Punto y coma opcional al final de una sentencia:
statement.Rule = expr + ";".Q();

// O equivalente con Empty:
optSemi.Rule = ToTerm(";") | Empty;
statement.Rule = expr + optSemi;

// Separar sentencias con saltos de línea:
program.Rule = MakePlusRule(program, NewLine, statement);
```

---

## 4. KeyTerms y `ToTerm()`

Un `KeyTerm` representa una palabra clave o símbolo especial de la gramática (p.ej. `if`, `while`, `+`, `==`).

```csharp
// Forma básica — el nombre para mensajes de error es igual al texto:
KeyTerm kwIf = ToTerm("if");
KeyTerm plus = ToTerm("+");

// Con nombre custom para mensajes de error:
KeyTerm kwIf = ToTerm("if", "palabra-clave-if");
```

> **Nota importante:** `ToTerm` reutiliza el mismo objeto `KeyTerm` si se llama con el mismo texto. Es seguro llamarlo múltiples veces.

### Palabras Reservadas

```csharp
// Marcar palabras que no pueden usarse como identificadores:
MarkReservedWords("if", "else", "while", "for", "return", "true", "false");
```

Con esto, si el scanner reconoce `while` como identificador, el parser lo reclasifica automáticamente como keyword al consultar la tabla de acciones.

---

## 5. NonTerminal

Los no-terminales son los símbolos de la gramática que tienen reglas de producción (lados izquierdos de las reglas BNF).

### Constructores

```csharp
var nt = new NonTerminal("NombreNT");
var nt = new NonTerminal("NombreNT", "alias-en-errores");

// Con tipo AST asociado (requiere LanguageFlags.CreateAst):
var nt = new NonTerminal("NombreNT", typeof(MiNodoAST));

// Con delegado creador de nodo AST:
var nt = new NonTerminal("NombreNT", (ctx, node) => { node.AstNode = new MiNodo(); });

// Con regla BNF directa en el constructor:
var nt = new NonTerminal("Expr", expr1 | expr2);
```

### Propiedades Clave

| Propiedad | Descripción |
|---|---|
| `Rule` | La expresión BNF de producción. Asignación obligatoria. |
| `ErrorRule` | Regla alternativa de recuperación de errores (como `error` en yacc). |
| `NodeCaptionTemplate` | Plantilla de texto para mostrar el nodo en exploradores: `"if #{0} then #{2}"` donde `#{i}` es el índice del hijo. |
| `AstConfig` | Configuración del nodo AST (tipo, creador, mapa de hijos). |

### `NodeCaptionTemplate`

```csharp
ifStmt.NodeCaptionTemplate = "if #{0}";           // muestra el primer hijo
funcCall.NodeCaptionTemplate = "call #{0}(...)";   // llama a la función #{0}
```

Los `#{i}` se reemplazan con el texto del hijo `i` al mostrar el nodo en el árbol.

---

## 6. Operadores BNF: `+`, `|` y `Q()`

### `+` — Secuencia (concatenación)

```csharp
// A seguido de B seguido de C:
rule.Rule = termA + termB + termC;

// Mezcla con strings (se convierten automáticamente a KeyTerm):
ifStmt.Rule = "if" + "(" + condition + ")" + statement;
```

### `|` — Alternativa (OR)

```csharp
// A o B o C:
type.Rule = ToTerm("int") | "float" | "bool" | "string";

// Combinación:
expr.Rule = number | identifier | "(" + expr + ")" | expr + binOp + expr;
```

### `Q()` — Elemento Opcional (Cero o Uno)

```csharp
// Elemento opcional (azúcar sintáctica para "term | Empty"):
var optElse = new NonTerminal("OptElse");
optElse.Rule = ("else" + statement).Q();

// O directamente en la regla:
returnStmt.Rule = "return" + expr.Q() + ";";
```

> **Nota:** `Q()` crea internamente un nuevo `NonTerminal` con nombre `"termName?"` y regla `this | Empty`. Solo funciona si `Grammar.CurrentGrammar` está activo (dentro del constructor).

---

## 7. MakePlusRule y MakeStarRule

Estos métodos generan gramáticas de listas de forma eficiente usando recursividad izquierda, que se aplana automáticamente en el parse tree.

### `MakePlusRule` — Una o más repeticiones

```csharp
// Lista de uno o más elementos (sin delimitador):
stmtList.Rule = MakePlusRule(stmtList, statement);

// Con delimitador:
argList.Rule = MakePlusRule(argList, ToTerm(","), expr);
```

### `MakeStarRule` — Cero o más repeticiones

```csharp
// Lista de cero o más elementos:
optList.Rule = MakeStarRule(optList, item);

// Con delimitador:
paramList.Rule = MakeStarRule(paramList, ToTerm(","), param);
```

### `TermListOptions` (parámetro avanzado de `MakeListRule`)

```csharp
// Las opciones se combinan con | (flags):
protected BnfExpression MakeListRule(
    NonTerminal list,
    BnfTerm delimiter,
    BnfTerm listMember,
    TermListOptions options = TermListOptions.PlusList);
```

| Opción | Valor | Descripción |
|---|---|---|
| `None` | `0x00` | Sin opciones |
| `AllowEmpty` | `0x01` | Permite la lista vacía (como en StarList) |
| `AllowTrailingDelimiter` | `0x02` | Permite delimitador final: `a, b, c,` |
| `AddPreferShiftHint` | `0x04` | Agrega hint para resolver conflictos en listas adyacentes |
| `PlusList` | `AddPreferShiftHint` | Combinación predeterminada para `MakePlusRule` |
| `StarList` | `AllowEmpty \| AddPreferShiftHint` | Combinación predeterminada para `MakeStarRule` |

**Ejemplo con trailing delimiter:**

```csharp
var imports = new NonTerminal("Imports");
imports.Rule = MakeListRule(imports, ToTerm(","), identifier,
    TermListOptions.StarList | TermListOptions.AllowTrailingDelimiter);
// Permite: import a, b, c,   ← coma final OK
```

---

## 8. Registro de Operadores y Precedencia

### `RegisterOperators`

```csharp
// Asociatividad izquierda por defecto:
RegisterOperators(int precedence, params string[] opSymbols);
RegisterOperators(int precedence, params BnfTerm[] opTerms);

// Con asociatividad explícita:
RegisterOperators(int precedence, Associativity associativity, params string[] opSymbols);
RegisterOperators(int precedence, Associativity associativity, params BnfTerm[] opTerms);
```

**Regla general:** A mayor número, mayor precedencia. Los números solo son relevantes en relación entre sí.

```csharp
RegisterOperators(10, "?");               // ternario — menor precedencia
RegisterOperators(15, "&", "&&", "|", "||");
RegisterOperators(20, "==", "<", "<=", ">", ">=", "!=");
RegisterOperators(30, "+", "-");
RegisterOperators(40, "*", "/");
RegisterOperators(50, Associativity.Right, "**");  // exponenciación — derecha
RegisterOperators(60, "!");               // negación unaria — mayor
```

### `Associativity`

| Valor | Descripción |
|---|---|
| `Left` | Asociativo por la izquierda: `a - b - c` = `(a - b) - c` (defecto) |
| `Right` | Asociativo por la derecha: `a ** b ** c` = `a ** (b ** c)` |
| `Neutral` | No se puede encadenar sin paréntesis |

### Herencia de Precedencia en Operadores Compuestos

Cuando un no-terminal `BinOp` es una combinación `"+" | "-" | ...`, necesita heredar la precedencia del operador que lo produce:

```csharp
// Opción 1: MarkTransient (el nodo desaparece, se reemplaza por el operador real)
MarkTransient(BinOp);

// Opción 2: Flag de herencia (el nodo permanece pero hereda precedencia)
BinOp.SetFlag(TermFlags.InheritPrecedence);
```

---

## 9. RegisterBracePair

```csharp
RegisterBracePair("(", ")");
RegisterBracePair("[", "]");
RegisterBracePair("{", "}");
RegisterBracePair("begin", "end");  // palabras clave también funcionan
```

Vincula pares de delimitadores. Efectos:
- El scanner enlaza los tokens correspondientes mediante `Token.OtherBrace`.
- El parser detecta y reporta errores de desbalanceo automáticamente.
- El `CodeOutlineFilter` usa estos pares para suprimir `INDENT`/`DEDENT` dentro de paréntesis.

---

## 10. Métodos de Marcado

Estos métodos configuran el comportamiento del parser y del builder de AST para términos específicos.

### `MarkPunctuation`

```csharp
MarkPunctuation(params string[] symbols);
MarkPunctuation(params BnfTerm[] terms);
```

Marca términos como puntuación pura (flags `IsPunctuation | NoAstNode`). Los nodos de puntuación:
- Se **omiten** de la lista `ChildNodes` al construir el parse tree.
- No generan nodos en el AST.

```csharp
MarkPunctuation(";", ",", ":", "(", ")", "{", "}", "[", "]");
MarkPunctuation(comma, semicolon);
```

### `MarkTransient`

```csharp
MarkTransient(params NonTerminal[] nonTerminals);
```

Un no-terminal transient se **reemplaza por su único hijo no-puntuación** en el parse tree. Útil para nodos de agrupación sin significado semántico propio.

```csharp
// "ParExpr" solo existe para los paréntesis — es transparente:
MarkTransient(ParExpr, Term, Statement, BinOp, UnOp);
```

> **Regla:** Un transient con cero hijos válidos genera un nodo vacío (que también se omite en listas).

### `MarkReservedWords`

```csharp
MarkReservedWords(params string[] reservedWords);
```

Impide que esas palabras sean reconocidas como identificadores.

### `MarkMemberSelect`

```csharp
MarkMemberSelect(params string[] symbols);
```

Marca símbolos como "selectores de miembro" (típicamente `.` y `::`). Esto activa el dropdown de autocompletado en editores compatibles con VS SDK.

### `MarkNotReported`

```csharp
MarkNotReported(params BnfTerm[] terms);
MarkNotReported(params string[] symbols);
```

Excluye esos terminales de la lista de "terminales esperados" en los mensajes de error de sintaxis. Útil para operadores de baja prioridad o tokens internos.

---

## 11. Grupos de Reporte de Errores

Permiten personalizar cómo se muestran los terminales en los mensajes de error:
`"Syntax error, expected: <lista>"`.

```csharp
// Agrupar varios operadores bajo un único alias:
AddTermsReportGroup("operador aritmético", "+", "-", "*", "/");
AddTermsReportGroup("operador comparación", "==", "!=", "<", ">", "<=", ">=");

// Alias para todos los operadores registrados:
AddOperatorReportGroup("operador");

// Excluir completamente de los mensajes:
AddToNoReportGroup("(", "++", "--");
AddToNoReportGroup(NewLine);
```

---

## 12. LanguageFlags

```csharp
this.LanguageFlags = LanguageFlags.CreateAst | LanguageFlags.NewLineBeforeEOF;
```

| Flag | Descripción |
|---|---|
| `NewLineBeforeEOF` | Inserta un token `NewLine` justo antes del EOF. **Solo activar si se usa `NewLine` terminal explícitamente en las reglas** (lenguajes como Basic). |
| `EmitLineStartToken` | Emite tokens `LINE_START` al inicio de cada línea (activado automáticamente por `CodeOutlineFilter`). |
| `DisableScannerParserLink` | Desactiva la comunicación entre scanner y parser. **Necesario cuando se usan `TokenFilter`** (como `CodeOutlineFilter` en Python). |
| `CreateAst` | Activa la construcción automática del AST tras el parseo exitoso. |
| `SupportsCommandLine` | Activa el modo REPL/línea de comandos (`ScriptApp`). |
| `TailRecursive` | Marca el lenguaje como tail-recursive (optimización para Scheme/Lisp). |
| `SupportsBigInt` | Activa soporte de `BigInteger` en el runtime del intérprete. |
| `SupportsComplex` | Activa soporte de números complejos en el runtime. |
| `SupportsRational` | Activa soporte de números racionales en el runtime. |

---

## 13. Hints de Gramática (resolución de conflictos)

Los hints son instrucciones especiales insertadas dentro de expresiones BNF que guían al constructor del autómata LALR cuando encuentra estados con conflictos.

### `PreferShiftHere()` — el caso dangling-else

El problema más clásico: el `else` debe asociarse con el `if` más cercano.

```csharp
// Sin hint → conflicto shift/reduce:
stmt.Rule = "if" + expr + stmt
          | "if" + expr + stmt + "else" + stmt;

// Con hint → se prefiere el shift (else va al if más cercano):
stmt.Rule = "if" + expr + stmt
          | "if" + expr + stmt + PreferShiftHere() + "else" + stmt;
```

### `ReduceHere()`

Fuerza la reducción en esa posición cuando hay conflicto.

```csharp
unaryExpr.Rule = unOp + term + ReduceHere();
```

### `ReduceIf(thisSymbol, comesBefore...)` / `ShiftIf(...)`

Inspecciona los próximos tokens del scanner para decidir. **No son parte de la tabla LALR** — son comprobaciones en tiempo de ejecución.

```csharp
// Reducir si aparece "then" antes de "else" o ";"
var hint = ReduceIf("then", "else", ";");
statement.AddHintToAll(hint);

// También funciona con terminales:
var hint2 = ReduceIf(myTerminal, otherTerminal1, otherTerminal2);
```

> **Advertencia:** `MaxPreviewTokens` (defecto: 1000) limita cuántos tokens se inspeccionan antes de descartar la condición.

### `ImplyPrecedenceHere(precedence, associativity)`

Impone precedencia implícita en un punto de la producción sin declarar operador formal.

```csharp
// Para casos como "NOT LIKE" en SQL (dos tokens que actúan como un operador):
protected GrammarHint ImplyPrecedenceHere(int precedence, Associativity assoc);
```

### `CustomActionHere(executeMethod, previewMethod)`

Máxima flexibilidad: inyecta lógica personalizada como acción de parser en un punto específico.

```csharp
stmt.Rule = term + CustomActionHere(
    executeMethod: (context, node) => {
        // lógica custom de ejecución
    },
    previewMethod: (context) => {
        // lógica de preview (opcional)
        return PreviewActionResult.Shift;
    }
) + restOfRule;
```

### `NonTerminal.AddHintToAll(hint)`

Agrega un hint al final de **todas** las producciones de un no-terminal.

```csharp
// Primero hay que asignar Rule, luego agregar el hint:
listNT.Rule = MakePlusRule(listNT, item);
listNT.AddHintToAll(PreferShiftHere());
```

---

## 14. SnippetRoots

Permite parsear fragmentos de código que no comienzan desde la raíz principal. Útil para strings con expresiones embebidas o para parseo incremental.

```csharp
var exprRoot = new NonTerminal("Expr");
this.SnippetRoots.Add(exprRoot);

// Parsear un snippet con root específico:
var snippetParser = new Parser(language, exprRoot);
var tree = snippetParser.Parse("x + 1");
```

También se usa con `StringTemplateSettings`:

```csharp
var templateSettings = new StringTemplateSettings();
templateSettings.ExpressionRoot = Expr;
this.SnippetRoots.Add(Expr);
stringLit.AstConfig.Data = templateSettings;
```

---

## 15. NonGrammarTerminals

Terminales que el scanner reconoce pero el parser ignora (se descartan del flujo de tokens antes de llegar al parser).

```csharp
// Los comentarios siempre van aquí:
var lineComment  = new CommentTerminal("comment", "//", "\n");
var blockComment = new CommentTerminal("block-comment", "/*", "*/");
NonGrammarTerminals.Add(lineComment);
NonGrammarTerminals.Add(blockComment);
```

Los tokens producidos por estos terminales tienen el flag `IsNonGrammar` y son ignorados durante el parseo. **No** deben aparecer en las reglas BNF.

---

## 16. Lenguajes con Indentación (Python-style)

Para lenguajes donde la sangría define bloques (Python, Haskell, CoffeeScript):

### Paso 1: Crear el filtro en `CreateTokenFilters`

```csharp
public override void CreateTokenFilters(LanguageData language, TokenFilterList filters) {
    var filter = new CodeOutlineFilter(
        language.GrammarData,
        OutlineOptions.ProduceIndents | OutlineOptions.CheckBraces,
        continuationTerminal: ToTerm("\\")  // null si no hay continuación de línea
    );
    filters.Add(filter);
}
```

`OutlineOptions`:

| Opción | Descripción |
|---|---|
| `ProduceIndents` | Genera tokens `INDENT` y `DEDENT` |
| `CheckBraces` | Dentro de `()`, `[]`, `{}` no se generan `INDENT`/`DEDENT` |
| `CheckOperator` | (En desarrollo) Suprime saltos de línea después de operadores |

### Paso 2: Agregar `LanguageFlags.DisableScannerParserLink`

```csharp
LanguageFlags = LanguageFlags.DisableScannerParserLink | LanguageFlags.CreateAst;
```

### Paso 3: Usar los terminales virtuales en las reglas

```csharp
block.Rule = Indent + stmtList + Dedent;
stmt.Rule  = simpleStmt + Eos
           | compoundStmt;
```

---

## 17. Métodos Virtuales Sobreescribibles

Estos métodos de `Grammar` permiten personalizar comportamientos específicos:

| Método | Cuándo sobreescribir |
|---|---|
| `CreateTokenFilters(language, filters)` | Para agregar `CodeOutlineFilter` u otros filtros de tokens. |
| `TryMatch(context, source)` | Cuando el scanner no encuentra terminal: último recurso para custom-matching. |
| `GetParseNodeCaption(node)` | Personalizar el texto de un nodo en el árbol de parseo (usado por `GrammarExplorer`). |
| `OnScannerSelectTerminal(context)` | Cuando hay ambigüedad entre terminales. Modificar `context.CurrentTerminals` para elegir. |
| `SkipWhitespace(source)` | Para lenguajes con whitespace no estándar. |
| `IsWhitespaceOrDelimiter(ch)` | Define qué caracteres son delimitadores para el parseo rápido. |
| `OnGrammarDataConstructed(language)` | Hook post-construcción de `GrammarData`. |
| `OnLanguageDataConstructed(language)` | Hook post-construcción completa de `LanguageData`. |
| `ConstructParserErrorMessage(context, expected)` | Personaliza el mensaje de error de sintaxis. |
| `ReportParseError(context)` | Override completo del reporte de errores. |
| `BuildAst(language, parseTree)` | Override del proceso de construcción del AST. |

### Ejemplo — Mensaje de Error Personalizado

```csharp
public override string ConstructParserErrorMessage(
    ParsingContext context,
    StringSet expectedTerms) {

    if (expectedTerms.Count == 0)
        return "Error de sintaxis inesperado.";
    return $"Se esperaba: {string.Join(", ", expectedTerms)}";
}
```

---

## 18. TermFlags — Flags de Términos

Los flags se pueden leer/escribir directamente en cualquier `BnfTerm`:

```csharp
term.SetFlag(TermFlags.IsOperator);
term.SetFlag(TermFlags.IsPunctuation, false);  // quitar flag
bool esOp = term.Flags.IsSet(TermFlags.IsOperator);
```

| Flag | Descripción |
|---|---|
| `IsOperator` | Marcado como operador (por `RegisterOperators`) |
| `IsOpenBrace` / `IsCloseBrace` / `IsBrace` | Delimitadores (por `RegisterBracePair`) |
| `IsLiteral` | Literal (número, string, etc.) |
| `IsConstant` | Constante |
| `IsPunctuation` | Puntuación — se omite de `ChildNodes` |
| `IsDelimiter` | Delimitador |
| `IsReservedWord` | Palabra reservada |
| `IsMemberSelect` | Selector de miembro (p.ej. `.`) |
| `InheritPrecedence` | El no-terminal hereda precedencia de sus hijos |
| `IsNonScanner` | Token no producido por el scanner (p.ej. `INDENT`) |
| `IsNonGrammar` | Ignorado por el parser (p.ej. comentarios) |
| `IsTransient` | Se reemplaza por su hijo en el árbol |
| `IsNotReported` | No aparece en mensajes de error esperados |
| `IsNullable` | Puede producir la cadena vacía (calculado) |
| `IsKeyword` | Es una keyword (calculado) |
| `IsMultiline` | El token puede abarcar múltiples líneas |
| `IsList` | Es un no-terminal de lista (generado por `MakePlusRule`) |
| `IsListContainer` | Contenedor de lista (generado por `MakeStarRule`) |
| `NoAstNode` | No crear nodo AST para este término |
| `AstDelayChildren` | Retrasar la creación de nodos AST de los hijos (compilación lazy) |
