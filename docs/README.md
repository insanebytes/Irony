# Documentación de Irony — Índice General

**Irony** es un framework de código abierto para .NET que permite definir gramáticas de lenguajes de programación directamente en C# y obtener un parser LALR(1) completamente funcional sin necesidad de generadores de código externos.

Autor original: Roman Ivantsov. Licencia: MIT.

---

## Contenido

| Documento | Descripción |
|---|---|
| [01 — Guía de Gramática](01-gramatica.md) | Cómo definir una gramática: terminales, no-terminales, reglas BNF, flags y métodos de marcado |
| [02 — Referencia de Terminales](02-terminales.md) | Referencia exhaustiva de todos los terminales disponibles y sus opciones |
| [03 — Parser LALR](03-parser-lalr.md) | Arquitectura interna del parser LALR(1), algoritmo de construcción y limitaciones |
| [04 — Tutorial AST](04-ast-tutorial.md) | Cómo construir un Árbol de Sintaxis Abstracta (AST) desde el resultado del parseo |
| [05 — Ejemplos](05-ejemplos.md) | Ejemplos completos y comentados de gramáticas reales |

---

## Inicio Rápido

### Instalación

```bash
dotnet add package Irony
```

O agregar referencia al proyecto `Irony/010.Irony.csproj`.

### Flujo Básico

```
Texto fuente
    │
    ▼
Scanner (Lexer)
    │  Tokens
    ▼
Parser LALR(1)
    │  Parse Tree (CST)
    ▼
AstBuilder
    │  AST (nodos custom)
    ▼
Tu lógica (evaluador, compilador, etc.)
```

### Ejemplo Mínimo

```csharp
// 1. Definir la gramática
public class CalcGrammar : Grammar {
    public CalcGrammar() {
        var number = new NumberLiteral("number");
        var expr   = new NonTerminal("Expr");
        var binOp  = new NonTerminal("BinOp");

        binOp.Rule = ToTerm("+") | "-" | "*" | "/";
        expr.Rule  = number | expr + binOp + expr | "(" + expr + ")";

        RegisterOperators(10, "+", "-");
        RegisterOperators(20, "*", "/");
        MarkTransient(binOp);
        MarkPunctuation("(", ")");

        Root = expr;
    }
}

// 2. Parsear
var parser = new Parser(new CalcGrammar());
var tree   = parser.Parse("3 + 4 * 2");

if (!tree.HasErrors())
    Console.WriteLine(tree.Root);   // Nodo raíz del parse tree
```

---

## Arquitectura del Framework

```
Irony.dll
├── Parsing/
│   ├── Grammar/          ← Clases para definir la gramática (Grammar, NonTerminal, BnfTerm...)
│   ├── Terminals/        ← Terminales concretos (NumberLiteral, StringLiteral, IdentifierTerminal...)
│   ├── Scanner/          ← Lexer / tokenizador
│   ├── Parser/           ← Motor LALR, acciones, pila de parseo
│   │   ├── ParserActions/   ← Shift, Reduce, Accept, Error...
│   │   └── SpecialActionsHints/  ← Hints de gramática
│   ├── Data/             ← Estructuras de datos (GrammarData, ParserData, LanguageData)
│   │   └── Construction/ ← Builders que construyen el autómata LALR
│   └── TokenFilters/     ← Filtros de tokens (CodeOutlineFilter para Python-style)
└── Ast/                  ← Interfaces y builder del AST (AstBuilder, IAstNodeInit...)

Irony.Interpreter.dll
└── Ast/                  ← Nodos AST concretos para intérpretes (AstNode, BinaryOperationNode...)
```

---

## Versión y Compatibilidad

- **.NET Framework** 4.x y **.NET Standard** / **.NET Core** (según rama del proyecto).
- Todos los tipos principales están en el namespace `Irony.Parsing`.
- Los nodos AST base del intérprete están en `Irony.Interpreter.Ast`.
