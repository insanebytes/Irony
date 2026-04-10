# El Parser LALR(1) de Irony — Arquitectura, Algoritmo y Limitaciones

Esta sección documenta la arquitectura interna del parser LALR(1) de Irony, el algoritmo de construcción del autómata, el ciclo de parseo y las limitaciones inherentes de este tipo de parser.

---

## Índice

1. [Visión General del Pipeline](#1-visión-general-del-pipeline)
2. [Scanner — El Lexer](#2-scanner--el-lexer)
3. [Fundamentos del Parser LALR(1)](#3-fundamentos-del-parser-lalr1)
4. [Algoritmo de Construcción del Autómata](#4-algoritmo-de-construcción-del-autómata)
5. [Ciclo de Parseo](#5-ciclo-de-parseo)
6. [Tipos de Acciones del Parser](#6-tipos-de-acciones-del-parser)
7. [Vínculo Scanner–Parser](#7-vínculo-scannerparser)
8. [Manejo de Conflictos](#8-manejo-de-conflictos)
9. [Recuperación de Errores](#9-recuperación-de-errores)
10. [Estructuras de Datos Internas](#10-estructuras-de-datos-internas)
11. [LanguageData — Reutilización de Tablas](#11-languagedata--reutilización-de-tablas)
12. [Limitaciones del Parser LALR(1) en Irony](#12-limitaciones-del-parser-lalr1-en-irony)
13. [Diagnóstico y Tracing](#13-diagnóstico-y-tracing)

---

## 1. Visión General del Pipeline

```
                 ┌────────────────────────────────────────────────┐
                 │                 Parser (facade)                 │
                 │                                                 │
  Texto fuente ──▶  Scanner ──tokens──▶  Motor LALR ──ParseTree ──▶  AstBuilder ──▶ AST
                 │     ▲          │                                 │
                 │     └──────────┘                                 │
                 │   Scanner-Parser Link                            │
                 └────────────────────────────────────────────────┘
```

Clases principales:

| Clase | Namespace | Descripción |
|---|---|---|
| `Parser` | `Irony.Parsing` | Fachada principal. Orquesta Scanner + Motor LALR + AstBuilder |
| `Scanner` | `Irony.Parsing` | Lexer/Tokenizador |
| `ParsingContext` | `Irony.Parsing` | Estado completo del parseo en curso |
| `LanguageData` | `Irony.Parsing` | Datos compilados de la gramática (tablas LALR, datos del scanner) |
| `GrammarData` | `Irony.Parsing` | Datos en bruto de la gramática (producciones, terminales) |
| `ParserData` | `Irony.Parsing` | Tabla de acciones LALR y estados |
| `ScannerData` | `Irony.Parsing` | Datos del scanner (hash de terminales por primer carácter) |

---

## 2. Scanner — El Lexer

El scanner de Irony es un scanner dirigido por el parser — usa la información del estado actual para filtrar qué terminales son candidatos en cada posición.

### Flujo del Scanner

```
Carácter actual
      │
      ▼
Buscar en hash de primeros chars
      │
      ▼
Obtener lista de terminales candidatos
      │ (filtrado por ExpectedTerminals del estado actual)
      ▼
Intentar TryMatch en cada terminal (en orden de prioridad)
      │
      ▼
Retornar el token del match más largo
```

### `ISourceStream`

Interface que abstrae el texto fuente para el scanner:

```csharp
public interface ISourceStream {
    char PreviewChar { get; }       // Carácter en posición de preview
    int  PreviewPosition { get; set; } // Posición actual de preview
    bool EOF();
    Token CreateToken(Terminal terminal);
    Token CreateToken(Terminal terminal, object value);
    bool MatchSymbol(string symbol);
}
```

### `Token`

```csharp
public class Token {
    public Terminal Terminal;     // Terminal que lo reconoció
    public string   Text;         // Texto original en el fuente
    public object   Value;        // Valor .NET parseado (int, double, string, etc.)
    public string   ValueString;  // Representación string del valor
    public SourceLocation Location; // Línea/columna/posición en el fuente
    public int      Length;
    public KeyTerm  KeyTerm;      // Si es keyword, referencia al KeyTerm
    public Token    OtherBrace;   // Token de cierre correspondiente (si es brace)
    public TokenFlags Flags;
}
```

---

## 3. Fundamentos del Parser LALR(1)

### Items LR(0)

Un **item LR(0)** es una producción con un "punto" que indica el progreso del parseo:

```
S → E + • E      (se ha reconocido "E +", esperando "E")
S → • E + E      (al inicio, esperando "E")
S → E + E •      (item final/reduce)
```

### Estados del Parser

Un **estado del parser** es un conjunto de items LR(0) (la "clausura" de ese conjunto). El autómata construye un grafo de estados conectados por transiciones shift.

### Tabla de Acciones

Para cada par `(estado, lookahead)`, la tabla contiene:

| Acción | Descripción |
|---|---|
| **Shift** `s_N` | Consumir el token y pasar al estado N |
| **Reduce** `p_K` | Reducir usando la producción K |
| **Goto** `N` | Después de una reducción, ir al estado N (para no-terminales) |
| **Accept** | Parseo completado |
| **Error** | No hay acción — iniciar recuperación de errores |

### La diferencia LALR vs LR(0), SLR y LR(1)

| Método | Lookahead | Complejidad |
|---|---|---|
| LR(0) | Ninguno | Tablas pequeñas, pocas gramáticas válidas |
| SLR(1) | FOLLOW sets | Tablas pequeñas, más gramáticas |
| **LALR(1)** | Lookahead local exacto | Tablas compactas, casi todas las gramáticas prácticas |
| LR(1) canónico | Lookahead exacto completo | Tablas enormes, todas las gramáticas LR(1) |

LALR(1) fusiona los estados LR(1) que tienen el mismo núcleo de items LR(0), reduciendo el número de estados enormemente. La desventaja: puede introducir conflictos que LR(1) canónico no tendría.

---

## 4. Algoritmo de Construcción del Autómata

Irony implementa el algoritmo **DeRemer-Penello** para LALR(1), descrito en "Parsing Techniques" de Grune & Jacobs (2ª edición, §9.7). El proceso completo ocurre en `ParserDataBuilder.Build()`:

### Paso 1: `CreateParserStates()`

```
1. Crear el símbolo aumentado: Root' → Root
2. Estado inicial: clausura de {Root' → • Root}
3. Para cada estado:
   a. Para cada símbolo X tal que existe item "A → α • X β" en el estado:
      - Calcular el conjunto de items "shifted": {A → α X • β}
      - Calcular clausura de ese conjunto
      - Si no existe estado con ese núcleo → crear nuevo estado
      - Agregar transición Shift(X) → nuevo estado
4. Repetir hasta que no se agreguen más estados
```

### Paso 2: `GetReduceItemsInInadequateState()`

Un estado es **inadecuado** si contiene:
- Conflicto Shift/Reduce: hay un item final `A → α •` Y un item de shift para el mismo lookahead.
- Conflicto Reduce/Reduce: hay dos o más items finales con el mismo lookahead posible.

Solo los items de reducción en estados inadecuados necesitan cálculo de lookaheads.

### Paso 3: `ComputeTransitions(items)`

Para cada item de reducción que necesita lookaheads:

1. Se calcula el conjunto de **lookback transitions** (cadena de estados que llevó a ese item).
2. Se construye la relación **Include** entre transiciones: la transición `(p, A)` incluye a `(q, B)` si existe una producción `A → αBβ` donde `β ⇒* ε` (β es anulable).
3. La propagación de `Include` se hace **inmediatamente** al agregar la relación (optimización clave de la implementación de Irony — evita la clausura transitiva explícita).

### Paso 4: `ComputeLookaheads(items)`

Los lookaheads de un item de reducción son la unión de:
1. Los terminales en el `DirectRead` set de cada transición lookback.
2. Los terminales propagados a través de la cadena de `Include`.

### Paso 5: `ComputeStatesExpectedTerminals()`

Para cada estado se calcula `ExpectedTerminals`:

| Tipo de estado | Cómo se calcula |
|---|---|
| Solo items de shift | Unión de los símbolos actuales de todos los items de shift |
| Items de shift Y reduce | Unión de shift symbols + lookaheads de todos los reduce items |
| Solo reduce items (2+) | Unión de lookaheads de todos los reduce items |
| Un solo reduce item | **Caso especial**: no se calcula — el parser evita activar el scanner-parser link |

### Paso 6: `ComputeConflicts()`

Detecta conflictos en la tabla de acciones. Un conflicto existe cuando para el mismo `(estado, lookahead)` hay más de una acción posible. Los conflictos se registran en `state.BuilderData.Conflicts`.

### Paso 7: `ApplyHints()`

Itera sobre los `GrammarHint` declarados en la gramática y llama a `hint.Apply(language, lrItem)` para cada item relevante. Los hints pueden:
- Agregar acciones condicionales (`ConditionalParserAction`).
- Eliminar el conflicto del conjunto `Conflicts` (marcándolo como resuelto).

### Paso 8: `HandleUnresolvedConflicts()`

Los conflictos que quedan sin resolver:
- **Shift/Reduce:** Se resuelve por defecto eligiendo **Shift**. Se reporta como warning en `LanguageData.Errors`.
- **Reduce/Reduce:** Se reporta como **error** de gramática. El parser aún intenta usar la primera reducción disponible.

### Paso 9: `CreateRemainingReduceActions()`

Para los estados con un único item de reducción (sin conflictos), crea `ReduceParserAction` como `DefaultAction` del estado (no requiere lookahead).

---

## 5. Ciclo de Parseo

El ciclo principal está en `Parser.ParseAll()` → `ExecuteNextAction()`:

```
┌──────────────────────────────────────────────────────┐
│ while Status == Parsing:                             │
│   if CurrentInput == null AND state.DefaultAction == null: │
│       ReadInput() from Scanner                       │
│   if CurrentInput.IsError:                           │
│       RecoverFromError()                             │
│       continue                                       │
│   action = GetNextAction()                           │
│   if action == null:                                 │
│       if CheckPartialInputCompleted(): break          │
│       RecoverFromError()                             │
│   else:                                              │
│       action.Execute(context)                        │
└──────────────────────────────────────────────────────┘
```

### `GetNextAction()`

```
1. Si el estado tiene DefaultAction → retornar DefaultAction (reduce sin lookahead)
2. Si el token de entrada tiene un KeyTerm asociado:
   a. Buscar acción por KeyTerm en state.Actions
   b. Si hay acción → hacer "backpatch" del token (reclasificar como keyword)
3. Buscar acción por el terminal principal del token
4. Caso especial: Si es EOF y el lenguaje tiene NewLineBeforeEOF:
   - Insertar token NewLine y buscar acción por NewLine
5. Si no hay acción → retornar null (error)
```

---

## 6. Tipos de Acciones del Parser

### `ShiftParserAction`

```
Execute:
  1. Empujar el token actual como ParseTreeNode en la pila
  2. Ir al estado destino de la transición shift
  3. Limpiar CurrentInput (se consumió el token)
```

### `ReduceParserAction`

```
Execute:
  1. Determinar la producción a reducir (Production con N símbolos)
  2. GetResultNode(): crear nodo padre con los N hijos del tope de la pila
     - Omitir nodos de puntuación y transitorios vacíos
  3. Desapilar N estados
  4. Leer el estado ahora en el tope de la pila
  5. Hacer Goto: buscar acción Shift del no-terminal en el estado actual
  6. Ejecutar esa acción Shift (push del nuevo nodo)
  7. Disparar evento NonTerminal.Reduced
```

### `ReduceTransientParserAction`

Igual que Reduce, pero el resultado es el primer hijo no-puntuación en vez de un nuevo nodo padre. Implementa `MarkTransient`.

### `ReduceListBuilderParserAction`

Para listas (`MakePlusRule`):

```
Execute:
  1. El primer hijo es el nodo de lista existente (ya creado antes)
  2. El último hijo es el nuevo miembro de la lista
  3. Agregar el nuevo miembro directamente a listNode.ChildNodes
  4. Retornar el nodo de lista (no crear nuevo nodo)
```

Esto garantiza que la lista queda "aplanada" con todos sus elementos como hijos directos.

### `ReduceListContainerParserAction`

Para listas star (`MakeStarRule`) donde hay un no-terminal contenedor que envuelve la lista plus:

```
Execute:
  1. El hijo es la lista plus interna
  2. Copiar todos los ChildNodes de la lista plus al nodo contenedor
  3. El resultado parece como si el contenedor tuviera directamente todos los elementos
```

### `AcceptParserAction`

Marca el parseo como exitoso (`ParserStatus.Parsed`).

### `ErrorRecoveryParserAction`

Estrategia de recuperación:
1. Registrar el error en `ParseTree.ParserMessages`.
2. Desapilar estados hasta encontrar uno que tenga una `ErrorRule`.
3. Continuar el parseo con el símbolo `syntaxError`.

### `ConditionalParserAction`

Contiene una lista de entradas condicionales y una acción por defecto:

```csharp
foreach (var entry in ConditionalEntries) {
    if (entry.Condition(context))
        return entry.Action.Execute(context);
}
DefaultAction.Execute(context);
```

Creada por hints como `ReduceIf()` / `ShiftIf()`.

---

## 7. Vínculo Scanner–Parser

Una característica única de Irony es el **vínculo bidireccional** entre scanner y parser:

```
Estado actual del parser
         │
         ▼
  ExpectedTerminals
  (calculado al construir las tablas)
         │
         ▼
  Scanner filtra candidatos
  usando ExpectedTerminals
         │
         ▼
  Solo se intenta TryMatch para terminales esperados
```

### Beneficios

1. **Eliminación de ambigüedades léxicas:** Si en un contexto dado no se espera un número, el scanner no intentará reconocer números aunque el carácter de entrada sea un dígito.
2. **Mejores mensajes de error:** La lista de esperados es exacta.
3. **Soporte para keywords no reservadas:** Una palabra puede ser keyword en algunos contextos e identificador en otros.

### Desactivación

Para lenguajes con `TokenFilter` (como Python), el vínculo se desactiva:

```csharp
LanguageFlags |= LanguageFlags.DisableScannerParserLink;
```

### Caso Especial: Estado con DefaultAction

Cuando el estado actual tiene `DefaultAction` (un único reduce item), el parser **NO lee el siguiente token** del scanner. Como resultado:

- El scanner no se activa.
- El vínculo scanner-parser no se invoca.
- El reduce ocurre sin consumir ningún token de entrada.

---

## 8. Manejo de Conflictos

### Shift/Reduce

**Ejemplo:** El clásico dangling-else:

```
Estado: {
    if_stmt → "if" expr stmt •          (reduce)
    if_stmt → "if" expr stmt • "else" stmt  (shift con "else")
}
Lookahead: "else"
Conflicto: ¿reduce o shift?
```

**Resolución por defecto:** Shift. Este es el comportamiento correcto para el dangling-else (el else va al if más cercano). Irony lo implementa automáticamente y emite un **warning** (no error) que se puede suprimir con `PreferShiftHere()`.

**Resolución por precedencia:**

Cuando el lookahead es un operador con precedencia declarada y el item de reducción involucra un operador:

```
Precedencia(lookahead) > Precedencia(producción) → Shift
Precedencia(lookahead) < Precedencia(producción) → Reduce
Precedencia(lookahead) = Precedencia(producción):
  Associativity.Left  → Reduce
  Associativity.Right → Shift
  Associativity.Neutral → Error
```

### Reduce/Reduce

Dos producciones distintas pueden reducirse con el mismo lookahead. Generalmente indica ambigüedad verdadera en la gramática:

```
Estado: {
    typeA → "int" •    (lookahead: ";")
    typeB → "int" •    (lookahead: ";")
}
```

**No hay resolución automática correcta** — es necesario:
1. Rediseñar la gramática para eliminar la ambigüedad.
2. Usar `TokenPreviewHint` para inspeccionar el contexto futuro.
3. Usar `CustomActionHint` para lógica arbitraria.

---

## 9. Recuperación de Errores

Irony implementa una estrategia de recuperación similar a YACC:

### `ErrorRule` en NonTerminal

```csharp
// El parser puede recuperarse sincronizando en ";" después de un error:
statement.ErrorRule = Grammar.SyntaxError + ";";

// O en el cierre de bloque:
block.ErrorRule = "{" + Grammar.SyntaxError + "}";
```

### Proceso de Recuperación

```
1. Se detecta error (no hay acción para el token actual)
2. Registrar error en ParserMessages
3. Ejecutar ErrorRecoveryParserAction:
   a. Desapilar la pila hasta encontrar un estado con acción para "SYNTAX_ERROR"
   b. Simular shift de un token de error
   c. Continuar el parseo normal
4. Si no se puede recuperar → marcar ParseTree como Error
```

---

## 10. Estructuras de Datos Internas

### `ParserState`

```csharp
public class ParserState {
    public string Name;
    public ParserActionTable Actions;   // (BnfTerm → ParserAction)
    public ParserAction DefaultAction;  // Acción única (sin lookahead requerido)
    public TerminalSet ExpectedTerminals; // Para el Scanner-Parser link
    internal ParserStateData BuilderData; // Datos de construcción (solo durante build)
}
```

### `Production`

```csharp
public class Production {
    public NonTerminal LValue;      // Lado izquierdo
    public BnfTermList RValues;     // Lado derecho
    public LR0ItemList LR0Items;    // Items LR(0) para esta producción
    public GrammarHintList Hints;   // Hints asociados
}
```

### `ParseTreeNode`

```csharp
public class ParseTreeNode {
    public BnfTerm Term;               // Terminal o NonTerminal que lo produjo
    public Token Token;                // Solo si es un terminal (token leaf)
    public ParseTreeNodeList ChildNodes; // Hijos en el árbol
    public SourceSpan Span;            // Ubicación en el texto fuente
    public object AstNode;             // Nodo AST creado (si CreateAst está activo)
    public bool IsError;               // ¿Es un nodo de error?
    public TokenList Comments;         // Comentarios que preceden a este nodo
    public object Tag;                 // Datos custom de usuario
}
```

### `ParseTree`

```csharp
public class ParseTree {
    public ParseTreeNode Root;                // Nodo raíz
    public ParseTreeStatus Status;            // Parsing, Partial, Parsed, Error
    public string SourceText;
    public string FileName;
    public TokenList Tokens;                  // Todos los tokens producidos
    public LogMessageList ParserMessages;     // Errores y warnings
    public long ParseTimeMilliseconds;        // Tiempo de parseo
    public object Tag;                        // Datos custom de usuario
    public bool HasErrors();
}
```

---

## 11. LanguageData — Reutilización de Tablas

La construcción del autómata LALR **es costosa**. Para evitar reconstruirlo en cada parseo, se debe reutilizar el objeto `LanguageData`:

```csharp
// Construir una sola vez:
var grammar  = new MiGramatica();
var language = new LanguageData(grammar);  // ← construcción del autómata

// Crear parser(s) reutilizando language:
var parser1 = new Parser(language);
var parser2 = new Parser(language);  // ← no reconstruye el autómata

// Parsear múltiples veces:
var tree1 = parser1.Parse("código fuente 1");
var tree2 = parser1.Parse("código fuente 2");  // reutiliza el mismo parser
```

> **Advertencia:** Un objeto `Parser` **no es thread-safe** — cada hilo debe tener su propio `Parser`. Sin embargo, `LanguageData` **sí es thread-safe** y puede compartirse entre hilos.

### Verificar Errores de Gramática

```csharp
var language = new LanguageData(new MiGramatica());
if (language.ErrorLevel == GrammarErrorLevel.Error) {
    foreach (var error in language.Errors)
        Console.WriteLine(error);
}
```

---

## 12. Limitaciones del Parser LALR(1) en Irony

### Limitación 1: Solo 1 Token de Lookahead

El parser LALR(1) solo mira **el siguiente token** para decidir qué acción tomar. Esto es insuficiente para gramáticas que requieren más contexto.

**Ejemplo problemático:** Distinguir entre declaración de función y llamada de función en algunos lenguajes:

```
función(x)   ← ¿declaración o llamada?
```

**Solución en Irony:** Usar `TokenPreviewHint` para inspeccionar más tokens del scanner en tiempo de ejecución. Esto no es parte de la tabla LALR — es una comprobación ad-hoc.

### Limitación 2: Conflictos en Gramáticas Ambiguas

Las gramáticas naturalmente ambiguas (operadores sin precedencia, el dangling-else) producen conflictos que deben resolverse explícitamente.

**Soluciones disponibles:**
- `RegisterOperators` con precedencia y asociatividad.
- `PreferShiftHere()` / `ReduceHere()`.
- Reestructurar la gramática para eliminar la ambigüedad.

### Limitación 3: No Soporta GLR ni Earley

A diferencia de:
- **ANTLR** → LL(*), puede hacer lookahead arbitrario hacia adelante.
- **Bison con GLR** → Gramática General LR, permite ambigüedades con resolución múltiple.
- **Parsers Earley** → Soportan cualquier gramática libre de contexto.

Irony está fijo en LALR(1). Lenguajes como C++ con análisis dependiente del contexto son extremadamente difíciles de expresar en LALR(1).

### Limitación 4: Recursividad Izquierda Obligatoria para Listas Eficientes

Las listas deben expresarse con recursividad **por la izquierda** para evitar que la pila del parser crezca linealmente con el número de elementos:

```csharp
// CORRECTO: recursividad izquierda — pila de tamaño constante:
list.Rule = list + "," + item | item;

// INCORRECTO: recursividad derecha — pila crece con cada elemento:
list.Rule = item + "," + list | item;  // ← no usar esto
```

`MakePlusRule` y `MakeStarRule` generan automáticamente recursividad izquierda.

### Limitación 5: Precedencia Global

La precedencia se registra **por símbolo de forma global**. No es posible que el mismo operador `+` tenga distinta precedencia en diferentes contextos de la gramática.

### Limitación 6: Tiempo de Construcción de Tablas

Para gramáticas grandes (más de cientos de producciones), la construcción del autómata puede ser lenta (segundos). Siempre reutilizar `LanguageData`:

```csharp
// Patrón recomendado para aplicaciones de producción:
private static readonly LanguageData _language = new LanguageData(new MiGramatica());
private static Parser CreateParser() => new Parser(_language);
```

### Limitación 7: Mensajes de Error

Los mensajes por defecto listan todos los terminales esperados, lo que puede ser confuso para el usuario final. Siempre personalizar:

```csharp
public override string ConstructParserErrorMessage(ParsingContext ctx, StringSet expected) {
    // Usar grupos de reporte para simplificar la lista:
    return $"Error de sintaxis. Se esperaba: {string.Join(", ", expected)}";
}
```

Usar `AddTermsReportGroup`, `AddToNoReportGroup` y `MarkNotReported` para reducir el ruido en los mensajes de error.

---

## 13. Diagnóstico y Tracing

### Activar Tracing

```csharp
var parser = new Parser(language);

// Activar tracing (registra cada acción en el contexto):
parser.Context.TracingEnabled = true;

var tree = parser.Parse("source code");

// Ver el trace:
foreach (var message in tree.ParserMessages)
    Console.WriteLine(message);
```

### `ParserDataPrinter`

Imprime la tabla de acciones LALR para diagnóstico:

```csharp
var printer = new ParserDataPrinter();
printer.PrintStates(language, Console.Out);
```

### Verificar Errores de Gramática al Construir

```csharp
var language = new LanguageData(grammar);

Console.WriteLine($"Error level: {language.ErrorLevel}");
foreach (var error in language.Errors) {
    Console.WriteLine($"[{error.Level}] {error.Message}");
}
```

`GrammarErrorLevel` puede ser:
- `NoError` — sin errores
- `Warning` — warnings (conflictos resueltos, etc.)
- `Conflict` — conflictos no resueltos (parser puede funcionar con heurísticas)
- `Error` — errores graves que impiden el parseo

### `ScanOnly` — Solo Tokenizar

```csharp
// Útil para pruebas y diagnóstico del scanner:
var parseTree = parser.ScanOnly("source code", "filename");
foreach (var token in parseTree.Tokens) {
    Console.WriteLine($"{token.Terminal.Name}: {token.Text}");
}
```
