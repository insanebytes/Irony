# Referencia de Terminales en Irony

Los terminales son los elementos hoja del árbol de parseo — representan los tokens que produce el scanner directamente desde el texto fuente. Esta referencia cubre todos los terminales disponibles en Irony con sus opciones y ejemplos.

---

## Índice

1. [Terminal (base)](#1-terminal-base)
2. [NumberLiteral](#2-numberliteral)
3. [StringLiteral](#3-stringliteral)
4. [IdentifierTerminal](#4-identifierterminal)
5. [CommentTerminal](#5-commentterminal)
6. [KeyTerm](#6-keyterm)
7. [RegExBasedTerminal](#7-regexbasedterminal)
8. [RegExLiteral](#8-regexliteral)
9. [FreeTextLiteral](#9-freetextliteral)
10. [ConstantTerminal](#10-constantterminal)
11. [CustomTerminal](#11-customterminal)
12. [DsvLiteral](#12-dsvliteral)
13. [FixedLengthLiteral](#13-fixedlengthliteral)
14. [WikiTerminales](#14-wikiterminales)
15. [Terminales de Filtro (NewLineTerminal, LineContinuationTerminal)](#15-terminales-de-filtro)
16. [ImpliedSymbolTerminal](#16-impliedsymbolterminal)
17. [TerminalFactory — Métodos de Fábrica](#17-terminalfactory--métodos-de-fábrica)
18. [Prioridad de Terminales](#18-prioridad-de-terminales)
19. [Eventos de Terminal](#19-eventos-de-terminal)

---

## 1. Terminal (base)

`Terminal` es la clase base abstracta para todos los terminales de Irony. Hereda de `BnfTerm`.

```csharp
// Constructores principales:
public Terminal(string name)
public Terminal(string name, TokenCategory category)
public Terminal(string name, TokenCategory category, TermFlags flags)
public Terminal(string name, string errorAlias, TokenCategory category, TermFlags flags)
```

### Propiedades

| Propiedad | Tipo | Descripción |
|---|---|---|
| `Category` | `TokenCategory` | Categoría del token generado |
| `Priority` | `int` | Prioridad en la selección cuando múltiples terminales coinciden |
| `OutputTerminal` | `Terminal` | Terminal que se adjunta al token de salida (por defecto es `this`) |
| `EditorInfo` | `TokenEditorInfo` | Información para coloreado de sintaxis en editores |
| `IsPairFor` | `Terminal` | Terminal de cierre correspondiente (brace pair) |

### `TokenCategory`

| Categoría | Descripción |
|---|---|
| `Content` | Token con contenido semántico (identificadores, literales) |
| `Outline` | Token estructural (INDENT, DEDENT, EOS, NewLine) |
| `Comment` | Comentario |
| `Error` | Token de error |

---

## 2. NumberLiteral

Terminal para reconocer literales numéricos con gran flexibilidad de formatos.

### Constructores

```csharp
// Básico:
var num = new NumberLiteral("number");

// Con opciones:
var num = new NumberLiteral("number", NumberOptions.IntOnly);
var num = new NumberLiteral("number", NumberOptions.Hex | NumberOptions.Binary);

// Con tipo de nodo AST:
var num = new NumberLiteral("number", NumberOptions.Default, typeof(NumberNode));

// Con delegado creador:
var num = new NumberLiteral("number", NumberOptions.Default, (ctx, node) => { ... });
```

### `NumberOptions` — Opciones disponibles

| Opción | Valor | Descripción |
|---|---|---|
| `None` / `Default` | `0x00` | Sin opciones adicionales |
| `AllowStartEndDot` | `0x01` | Permite `.5` y `5.` (Python) |
| `IntOnly` | `0x02` | Solo reconoce enteros (sin punto decimal ni exponente) |
| `NoDotAfterInt` | `0x04` | Con `IntOnly`: no captura el punto si lo sigue otro terminal. Útil cuando otro terminal maneja floats. |
| `AllowSign` | `0x08` | Permite `+` o `-` como parte del literal numérico |
| `DisableQuickParse` | `0x10` | Desactiva la ruta de parseo rápido (para debugging) |
| `AllowLetterAfter` | `0x20` | El número puede ir seguido de letra sin separador: `0xFF` (hex literal) |
| `AllowUnderscore` | `0x40` | Permite guiones bajos: `1_000_000` (Ruby, Python 3.6+) |
| `Binary` | `0x0100` | Activa soporte para números binarios (p.ej. `0b1010`) |
| `Octal` | `0x0200` | Activa soporte para números octales (p.ej. `0o777` o `0777`) |
| `Hex` | `0x0400` | Activa soporte para números hexadecimales (p.ej. `0xFF`) |

### Propiedades Adicionales

```csharp
// Tipos .NET a intentar para enteros (en orden de preferencia):
num.DefaultIntTypes = new TypeCode[] { TypeCode.Int32, TypeCode.Int64, NumberLiteral.TypeCodeBigInt };

// Tipo por defecto para flotantes:
num.DefaultFloatType = TypeCode.Double;  // o TypeCode.Single

// Caracteres de exponente y sus tipos resultantes:
num.ExponentSymbols.Add('e', TypeCode.Double);
num.ExponentSymbols.Add('E', TypeCode.Double);
num.ExponentSymbols.Add('d', TypeCode.Double);
num.ExponentSymbols.Add('f', TypeCode.Single);

// Agregar prefijos para distintas bases:
num.AddPrefix("0x", NumberOptions.Hex);
num.AddPrefix("0b", NumberOptions.Binary);
num.AddPrefix("0o", NumberOptions.Octal);
```

### Constantes de TypeCode

| Constante | Valor | Descripción |
|---|---|---|
| `TypeCodeBigInt` | 30 | `System.Numerics.BigInteger` |
| `TypeCodeImaginary` | 31 | Número imaginario (complejo) |

### Ejemplos de Uso

```csharp
// Enteros y flotantes estándar:
var number = new NumberLiteral("number");

// Python-style (permite .5 y 5., soporta big integers):
var number = new NumberLiteral("number", NumberOptions.AllowStartEndDot);
number.DefaultIntTypes = new TypeCode[] { TypeCode.Int32, TypeCode.Int64, NumberLiteral.TypeCodeBigInt };

// C-style con hex y octal:
var number = new NumberLiteral("number", NumberOptions.IntOnly);
number.AddPrefix("0x", NumberOptions.Hex | NumberOptions.AllowLetterAfter);
number.AddPrefix("0",  NumberOptions.Octal);

// Solo enteros sin floats:
var intLit = new NumberLiteral("intLit", NumberOptions.IntOnly | NumberOptions.NoDotAfterInt);
```

---

## 3. StringLiteral

Terminal para reconocer cadenas de texto con soporte para múltiples delimitadores, escapes y strings plantilla.

### Constructores

```csharp
// Básico:
var str = new StringLiteral("string");

// Con delimitador y opciones:
var str = new StringLiteral("string", "\"", StringOptions.AllowsAllEscapes);

// Con tipo de nodo AST:
var str = new StringLiteral("string", "\"", StringOptions.AllowsAllEscapes, typeof(StringNode));
```

### Métodos de Configuración

```csharp
// Agregar más delimitadores de inicio/fin:
str.AddStartEnd("\"", StringOptions.AllowsAllEscapes);
str.AddStartEnd("'",  StringOptions.AllowsAllEscapes);
// Heredoc o raw strings:
str.AddStartEnd("\"\"\"", "\"\"\"", StringOptions.AllowsLineBreak | StringOptions.NoEscapes);
str.AddStartEnd("r\"", "\"", StringOptions.NoEscapes);  // Python raw string
```

### `StringOptions` — Opciones disponibles

| Opción | Valor | Descripción |
|---|---|---|
| `None` | `0x00` | Sin opciones |
| `IsChar` | `0x01` | Trata el literal como carácter único |
| `AllowsDoubledQuote` | `0x02` | Convierte `''` en `'` (estilo SQL) |
| `AllowsLineBreak` | `0x04` | Permite saltos de línea dentro del string (multi-línea) |
| `IsTemplate` | `0x08` | String con expresiones embebidas: `"hola #{nombre}"` |
| `NoEscapes` | `0x10` | Desactiva procesamiento de secuencias de escape |
| `AllowsUEscapes` | `0x20` | Permite `\uXXXX` |
| `AllowsXEscapes` | `0x40` | Permite `\xXX` |
| `AllowsOctalEscapes` | `0x80` | Permite `\ooo` |
| `AllowsAllEscapes` | `0xE0` | Combinación de `AllowsUEscapes \| AllowsXEscapes \| AllowsOctalEscapes` |

### Strings Plantilla (Templates)

Para strings que contienen expresiones evaluables como `"Hola, #{nombre}"`:

```csharp
var str = new StringLiteral("string", "\"",
    StringOptions.AllowsAllEscapes | StringOptions.IsTemplate);
str.AddStartEnd("'", StringOptions.AllowsAllEscapes | StringOptions.IsTemplate);
str.AstConfig.NodeType = typeof(StringTemplateNode);

var templateSettings = new StringTemplateSettings {
    StartTag       = "#{",   // inicio de expresión embebida (Ruby-style)
    EndTag         = "}",    // fin de expresión embebida
    ExpressionRoot = Expr    // no-terminal que parsea las expresiones embebidas
};
this.SnippetRoots.Add(Expr);
str.AstConfig.Data = templateSettings;
```

### Ejemplos

```csharp
// Strings de C#:
var str = new StringLiteral("string", "\"", StringOptions.AllowsAllEscapes);
str.AddStartEnd("@\"", "\"", StringOptions.AllowsLineBreak | StringOptions.NoEscapes);

// Strings de SQL (comillas simples, ' duplicado para escape):
var str = new StringLiteral("string", "'", StringOptions.AllowsDoubledQuote);

// Strings multilínea de Python:
var str = new StringLiteral("string");
str.AddStartEnd("\"\"\"", StringOptions.AllowsLineBreak);
str.AddStartEnd("'''",    StringOptions.AllowsLineBreak);
str.AddStartEnd("\"",     StringOptions.AllowsAllEscapes);
str.AddStartEnd("'",      StringOptions.AllowsAllEscapes);
```

---

## 4. IdentifierTerminal

Terminal para reconocer identificadores (nombres de variables, funciones, tipos, etc.).

### Constructores

```csharp
// Básico — acepta letras, dígitos y guión bajo:
var id = new IdentifierTerminal("identifier");

// Con opciones:
var id = new IdentifierTerminal("identifier", IdOptions.AllowsEscapes);

// Con caracteres extra en el cuerpo y al inicio:
var id = new IdentifierTerminal("identifier", extraChars: "_$", extraFirstChars: "@");
```

Por defecto: `AllFirstChars = A-Za-z`, `AllChars = A-Za-z0-9_`.

### `IdOptions` — Opciones disponibles

| Opción | Valor | Descripción |
|---|---|---|
| `None` | `0x00` | Sin opciones |
| `AllowsEscapes` | `0x01` | Permite secuencias `\uXXXX` en el identificador (C#) |
| `CanStartWithEscape` | `0x03` | Puede comenzar con una secuencia de escape |
| `IsNotKeyword` | `0x10` | Los identificadores con este prefijo no son tratados como keywords |
| `NameIncludesPrefix` | `0x20` | El prefijo se incluye en el nombre del token |

### Propiedades y Configuración

```csharp
// Restricción de capitalización:
id.CaseRestriction = CaseRestriction.None;        // sin restricción (defecto)
id.CaseRestriction = CaseRestriction.FirstUpper;  // primer carácter mayúscula
id.CaseRestriction = CaseRestriction.FirstLower;  // primer carácter minúscula
id.CaseRestriction = CaseRestriction.AllUpper;    // todo mayúsculas
id.CaseRestriction = CaseRestriction.AllLower;    // todo minúsculas

// Categorías Unicode para el primer carácter:
id.StartCharCategories.Add(UnicodeCategory.UppercaseLetter);
id.StartCharCategories.Add(UnicodeCategory.LowercaseLetter);

// Categorías Unicode para los demás caracteres:
id.CharCategories.Add(UnicodeCategory.DecimalDigitNumber);

// Agregar prefijos (como @ en C# para keywords-as-identifiers):
id.AddPrefix("@", IdOptions.IsNotKeyword | IdOptions.NameIncludesPrefix);

// Información de coloreado para editores:
id.KeywordEditorInfo = new TokenEditorInfo(TokenType.Keyword, TokenColor.Keyword, TokenTriggers.None);
```

### Ejemplos

```csharp
// C# — identificadores con prefijo @ y escapes Unicode:
var id = new IdentifierTerminal("identifier");
id.AddPrefix("@", IdOptions.IsNotKeyword | IdOptions.NameIncludesPrefix);
id.Options = IdOptions.AllowsEscapes | IdOptions.CanStartWithEscape;

// Python — identificadores estándar:
var id = new IdentifierTerminal("name", "_", "_");

// SQL — case insensitive se maneja a nivel de Grammar(caseSensitive: false):
var id = new IdentifierTerminal("id");
// Opcionalmente con soporte de "[nombre]" o "`nombre`":
id.AddPrefix("[", IdOptions.NameIncludesPrefix);  // etc.
```

---

## 5. CommentTerminal

Terminal para reconocer comentarios. Siempre se agrega a `NonGrammarTerminals`.

### Constructor

```csharp
public CommentTerminal(string name, string startSymbol, params string[] endSymbols)
```

### Ejemplos

```csharp
// Comentario de línea (termina en newline o EOF):
var lineComment = new CommentTerminal("line-comment", "//", "\r", "\n", "\u2085", "\u2028", "\u2029");
NonGrammarTerminals.Add(lineComment);

// Comentario de bloque:
var blockComment = new CommentTerminal("block-comment", "/*", "*/");
NonGrammarTerminals.Add(blockComment);

// Comentario de línea estilo Python/Shell:
var hashComment = new CommentTerminal("comment", "#", "\n", "\r");
NonGrammarTerminals.Add(hashComment);

// Comentarios anidados (solo delimita por el inicio del siguiente, no por anidación):
var nested = new CommentTerminal("nested", "(*", "*)");
NonGrammarTerminals.Add(nested);
```

> **Nota:** Si el símbolo de fin incluye `\n`, el terminal se marca como comentario de línea y también acepta EOF como terminador válido (para comentarios sin salto de línea al final del archivo).

### Propiedades

```csharp
public string StartSymbol;   // Símbolo de apertura
public StringList EndSymbols; // Lista de símbolos de cierre (el primero encontrado gana)
```

---

## 6. KeyTerm

Representa palabras clave y símbolos especiales fijos del lenguaje. Normalmente se crean via `ToTerm()`.

```csharp
// Via Grammar.ToTerm (recomendado — reutiliza instancias):
var kwIf   = ToTerm("if");
var kwElse = ToTerm("else", "else-keyword");  // nombre custom para errores
var plus   = ToTerm("+");
var arrow  = ToTerm("=>");

// Acceso al diccionario completo:
KeyTermTable KeyTerms;
```

### Propiedades de KeyTerm

```csharp
public string IsPairFor;  // Terminal de cierre si este es un brace de apertura
```

> **No crear `KeyTerm` directamente** — siempre usar `ToTerm()` para garantizar que la misma cadena no produce dos instancias distintas (lo que causaría errores de gramática).

---

## 7. RegExBasedTerminal

Terminal que utiliza una expresión regular para el reconocimiento.

```csharp
var hex = new RegExBasedTerminal("hex", @"0x[0-9a-fA-F]+");
var sciNotation = new RegExBasedTerminal("sci", @"\d+\.\d+([eE][+-]?\d+)?");
```

> **Nota de rendimiento:** Las expresiones regulares se anclan al inicio (`^`). El terminal tiene prioridad `TerminalPriority.Normal` por defecto — puede ajustarse si hay ambigüedad con otros terminales.

```csharp
hex.Priority = TerminalPriority.High;
```

---

## 8. RegExLiteral

Terminal para reconocer **literales de expresión regular** en el código fuente (como en JavaScript: `/patrón/flags`).

```csharp
var regexLit = new RegExLiteral("regex");
// Por defecto reconoce: /patrón/flags
// El valor del token es la cadena de la regex sin los delimitadores
```

---

## 9. FreeTextLiteral

Terminal que captura texto libre hasta que encuentra uno de los terminadores especificados.

```csharp
// Texto hasta ";" o fin de línea:
var freeText = new FreeTextLiteral("text", FreeTextOptions.None, ";", "\n");

// Con escape:
var freeText = new FreeTextLiteral("text", @"\", ";", "\n");
```

### `FreeTextOptions`

```csharp
[Flags]
public enum FreeTextOptions {
    None          = 0x0,
    ConsumeTerminator   = 0x01,  // Incluir el terminador en el token
    IncludeTerminator   = 0x02,  // Retener pero no consumir el terminador
    AllowEof      = 0x04,  // EOF es un terminador válido
    AllowEmpty    = 0x08,  // Permitir texto vacío
}
```

---

## 10. ConstantTerminal

Mapea cadenas específicas a valores constantes en tiempo de parseo.

```csharp
var boolLit = new ConstantTerminal("bool");
boolLit.Add("true",  true);
boolLit.Add("false", false);
boolLit.Add("null",  null);

// Con case-insensitive (hereda de la gramática):
var boolLit = new ConstantTerminal("bool");
boolLit.Add("True",  true);
boolLit.Add("False", false);
boolLit.Add("None",  null);  // Python
```

El valor asociado queda disponible en `token.Value` tras el parseo.

---

## 11. CustomTerminal

Terminal con lógica de reconocimiento completamente personalizada vía delegado.

```csharp
public delegate Token MatchHandler(Terminal terminal, ParsingContext context, ISourceStream source);

var custom = new CustomTerminal("custom", MyMatchMethod);

private static Token MyMatchMethod(Terminal terminal, ParsingContext context, ISourceStream source) {
    // Si no hay match, retornar null:
    if (source.PreviewChar != '@') return null;

    // Leer hasta el fin del token:
    source.PreviewPosition++;
    while (!source.EOF() && char.IsLetterOrDigit(source.PreviewChar))
        source.PreviewPosition++;

    return source.CreateToken(terminal.OutputTerminal);
}
```

---

## 12. DsvLiteral

Terminal para reconocer datos delimitados (CSV, TSV y similares).

```csharp
// CSV:
var csv = new DsvLiteral("csv", ',');

// TSV:
var tsv = new DsvLiteral("tsv", '\t');
```

El valor del token es un array de strings con los valores separados.

---

## 13. FixedLengthLiteral

Terminal que reconoce tokens de longitud fija (útil para formatos de datos binarios o de ancho fijo).

```csharp
// Token de exactamente 8 caracteres:
var fixed8 = new FixedLengthLiteral("code", 8);
```

---

## 14. WikiTerminales

Terminales específicos para parsear formatos wiki (WikiCreole, Codeplex, etc.).

### `WikiTextTerminal`

Captura texto libre entre marcadores wiki.

```csharp
var wikiText = new WikiTextTerminal("text");
```

### `WikiTagTerminal`

Reconoce etiquetas wiki con delimitadores configurables.

```csharp
var bold   = new WikiTagTerminal("bold",   WikiTermType.Format, "**", "**");
var italic = new WikiTagTerminal("italic", WikiTermType.Format, "//", "//");
var h1     = new WikiTagTerminal("h1",     WikiTermType.Heading, "=",  "=\n");
```

### `WikiBlockTerminal`

Reconoce bloques estructurados.

```csharp
var codeBlock = new WikiBlockTerminal("code", WikiBlockType.IndentedBlock, "    ", "\n");
```

---

## 15. Terminales de Filtro

### `NewLineTerminal`

```csharp
// Siempre disponible como Grammar.NewLine:
program.Rule = MakePlusRule(program, NewLine, statement);
```

Activar `UsesNewLine` automáticamente en la gramática evita que los saltos de línea se traten como whitespace.

### `LineContinuationTerminal`

Permite que una línea lógica se extienda en múltiples líneas físicas mediante un símbolo de continuación (como `\` en Python).

```csharp
// Se agrega a NonGrammarTerminals:
var continuation = new LineContinuationTerminal("continuation", "\\");
NonGrammarTerminals.Add(continuation);
```

---

## 16. ImpliedSymbolTerminal

Terminal que no corresponde a ningún símbolo en el texto fuente — representa una inserción implícita del parser. Rara vez se usa directamente.

```csharp
var implied = new ImpliedSymbolTerminal("implied-semicolon");
```

---

## 17. TerminalFactory — Métodos de Fábrica

La clase estática `TerminalFactory` provee métodos de conveniencia para crear terminales comunes con configuraciones predefinidas para lenguajes populares.

```csharp
// Identificador estilo C# (con @ y escapes Unicode):
var csharpId = TerminalFactory.CreateCSharpIdentifier("identifier");

// String estilo C#:
var csharpStr = TerminalFactory.CreateCSharpString("string");

// Número estilo C#:
var csharpNum = TerminalFactory.CreateCSharpNumber("number");

// Identificador estilo SQL (entre corchetes, comillas o backticks):
var sqlId = TerminalFactory.CreateSqlIdentifier("identifier");
```

---

## 18. Prioridad de Terminales

Cuando múltiples terminales pueden coincidir con el mismo carácter de entrada, la **prioridad** decide cuál se intenta primero.

```csharp
public static class TerminalPriority {
    public static int Low          = -1000;
    public static int Normal       = 0;      // Defecto
    public static int High         = 1000;
    public static int ReservedWords = 900;
}
```

```csharp
// Los comentarios tienen prioridad High por defecto (CommentTerminal):
// priority = TerminalPriority.High

// Subir prioridad de un terminal:
myTerminal.Priority = TerminalPriority.High;

// Bajar prioridad:
myTerminal.Priority = TerminalPriority.Low;
```

> **Regla práctica:** Los terminales más específicos deben tener mayor prioridad que los más generales. Por ejemplo, si tienes `!=` e `!`, el operador `!=` debe tener mayor prioridad (o ser más largo — el scanner prefiere el match más largo por defecto).

---

## 19. Eventos de Terminal

Todo terminal puede suscribirse a eventos para customizar su comportamiento en tiempo de parseo:

### `ValidateToken`

Se dispara después de que el scanner crea el token, antes de que el parser lo reciba.

```csharp
myTerminal.ValidateToken += (sender, args) => {
    var token = args.Context.CurrentToken;
    if (token.ValueString.Length > 64) {
        args.Context.CurrentToken = args.Context.CreateErrorToken("Identificador demasiado largo");
    }
};
```

### `ParserInputPreview`

Se dispara cuando el parser recibe el token (justo antes de buscar la acción correspondiente).

```csharp
myTerminal.ParserInputPreview += (sender, args) => {
    // args.Context tiene acceso al estado actual del parser
};
```

### `Shifting` (en `BnfTerm`)

Se dispara cuando el parser hace shift sobre este término.

```csharp
myTerm.Shifting += (sender, args) => {
    Console.WriteLine($"Shifting: {args.Context.CurrentParserInput}");
};
```

### `AstNodeCreated` (en `BnfTerm`)

Se dispara inmediatamente después de que el builder crea el nodo AST.

```csharp
funcCall.AstNodeCreated += (sender, args) => {
    var node = (FunctionCallNode)args.AstNode;
    node.Resolve(symbolTable);
};
```
