# Ejemplos Completos de Gramáticas con Irony

Esta sección presenta ejemplos completos, comentados y listos para compilar que cubren los patrones más comunes al definir gramáticas con Irony.

---

## Índice

1. [Ejemplo 1 — Calculadora de Expresiones](#1-ejemplo-1--calculadora-de-expresiones)
2. [Ejemplo 2 — Lenguaje de Configuración (clave = valor)](#2-ejemplo-2--lenguaje-de-configuración-clave--valor)
3. [Ejemplo 3 — Mini-lenguaje Imperativo con Bucles y Condicionales](#3-ejemplo-3--mini-lenguaje-imperativo-con-bucles-y-condicionales)
4. [Ejemplo 4 — Consultas SQL Simplificadas](#4-ejemplo-4--consultas-sql-simplificadas)
5. [Ejemplo 5 — Resolución de Conflictos con Hints](#5-ejemplo-5--resolución-de-conflictos-con-hints)
6. [Ejemplo 6 — Lenguaje con Indentación (Python-style)](#6-ejemplo-6--lenguaje-con-indentación-python-style)
7. [Ejemplo 7 — Gramática con Recuperación de Errores](#7-ejemplo-7--gramática-con-recuperación-de-errores)
8. [Ejemplo 8 — Strings con Expresiones Embebidas](#8-ejemplo-8--strings-con-expresiones-embebidas)

---

## 1. Ejemplo 1 — Calculadora de Expresiones

Gramática completa para una calculadora con operadores aritméticos, precedencia, paréntesis y soporte para números enteros y flotantes.

```csharp
using Irony.Parsing;

[Language("Calculator", "1.0", "Calculadora aritmética")]
public class CalculatorGrammar : Grammar {

    public CalculatorGrammar() : base(caseSensitive: false) {

        // ── Terminales ──────────────────────────────────────────────
        var number = new NumberLiteral("number",
            NumberOptions.AllowStartEndDot | NumberOptions.AllowUnderscore);
        // Permite: 3, 3.14, .5, 3., 1_000_000

        // ── No-terminales ───────────────────────────────────────────
        var expr    = new NonTerminal("Expr");
        var term    = new NonTerminal("Term");
        var binOp   = new NonTerminal("BinOp");
        var unOp    = new NonTerminal("UnOp");
        var unExpr  = new NonTerminal("UnExpr");
        var parExpr = new NonTerminal("ParExpr");

        // ── Reglas BNF ──────────────────────────────────────────────
        binOp.Rule   = ToTerm("+") | "-" | "*" | "/" | "%" | "**";
        unOp.Rule    = ToTerm("+") | "-";
        term.Rule    = number | parExpr | unExpr;
        parExpr.Rule = "(" + expr + ")";
        unExpr.Rule  = unOp + term + ReduceHere();
        expr.Rule    = term | expr + binOp + expr;

        // ── Precedencia y Asociatividad ─────────────────────────────
        RegisterOperators(10, "+", "-");          // Suma y resta
        RegisterOperators(20, "*", "/", "%");     // Multiplicación y módulo
        RegisterOperators(30, Associativity.Right, "**"); // Exponenciación (derecha)
        RegisterOperators(40, unOp);              // Unario — mayor precedencia

        // ── Términos especiales ─────────────────────────────────────
        MarkPunctuation("(", ")");
        RegisterBracePair("(", ")");
        MarkTransient(term, binOp, unOp, parExpr);

        // ── Raíz ────────────────────────────────────────────────────
        Root = expr;
    }
}

// ── Uso ──────────────────────────────────────────────────────────────────────
var language = new LanguageData(new CalculatorGrammar());
var parser   = new Parser(language);

var inputs = new[] { "3 + 4 * 2", "(3 + 4) * 2", "-5 ** 2", "1_000 * 1.5" };
foreach (var input in inputs) {
    var tree = parser.Parse(input);
    if (tree.HasErrors())
        Console.WriteLine($"Error: {tree.ParserMessages[0].Message}");
    else
        Console.WriteLine($"OK: {input}");
}
```

**Puntos clave:**
- `ReduceHere()` resuelve el conflicto entre `unExpr` (que comienza con `+`/`-`) y el operador binario.
- `MarkTransient` elimina los nodos de agrupación del parse tree.
- `RegisterBracePair` activa el control de paréntesis desbalanceados.

---

## 2. Ejemplo 2 — Lenguaje de Configuración (clave = valor)

```csharp
[Language("Config", "1.0", "Lenguaje de configuración simple")]
public class ConfigGrammar : Grammar {

    public ConfigGrammar() : base(caseSensitive: false) {

        // ── Terminales ──────────────────────────────────────────────
        var key     = new IdentifierTerminal("key");
        var strVal  = new StringLiteral("string", "\"", StringOptions.AllowsAllEscapes);
        strVal.AddStartEnd("'", StringOptions.AllowsAllEscapes);
        var numVal  = new NumberLiteral("number");
        var boolVal = new ConstantTerminal("bool");
        boolVal.Add("true",  true);
        boolVal.Add("false", false);
        boolVal.Add("yes",   true);
        boolVal.Add("no",    false);
        var comment = new CommentTerminal("comment", "#", "\n");
        NonGrammarTerminals.Add(comment);
        var comment2 = new CommentTerminal("comment2", "//", "\n");
        NonGrammarTerminals.Add(comment2);

        // ── No-terminales ───────────────────────────────────────────
        var config  = new NonTerminal("Config");
        var entry   = new NonTerminal("Entry");
        var section = new NonTerminal("Section");
        var sectionHeader = new NonTerminal("SectionHeader");
        var value   = new NonTerminal("Value");
        var list    = new NonTerminal("List");
        var listItems = new NonTerminal("ListItems");

        // ── Reglas BNF ──────────────────────────────────────────────
        // Archivo de configuración: mezcla de secciones y entradas sueltas
        config.Rule = MakeStarRule(config, NewLinePlus, section | entry);

        // Sección: [nombre] seguida de entradas
        sectionHeader.Rule = "[" + key + "]";
        section.Rule = sectionHeader + NewLinePlus + MakePlusRule(entry, NewLinePlus, entry);

        // Entrada: clave = valor
        entry.Rule = key + "=" + value
                   | key + ":" + value;

        // Valor simple o lista
        value.Rule = strVal | numVal | boolVal | key | list;

        // Lista entre corchetes: [a, b, c]
        listItems.Rule = MakeStarRule(listItems, ToTerm(","), value);
        list.Rule = "[" + listItems + "]";

        // ── Marcado ─────────────────────────────────────────────────
        MarkPunctuation("=", ":", "[", "]", ",");
        RegisterBracePair("[", "]");
        MarkTransient(value);

        // ── Raíz ────────────────────────────────────────────────────
        Root = config;
        LanguageFlags = LanguageFlags.NewLineBeforeEOF;
    }
}

// ── Uso ──────────────────────────────────────────────────────────────────────
var parser = new Parser(new ConfigGrammar());
var config = @"
# Configuración de la aplicación

[database]
host = ""localhost""
port = 5432
enabled = true

[features]
tags = [web, api, v2]
timeout = 30
";

var tree = parser.Parse(config.Trim());
Console.WriteLine(tree.HasErrors() ? "Errores de parseo" : "OK");
```

**Puntos clave:**
- `ConstantTerminal` mapea literales a valores booleanos directamente.
- `NewLinePlus` como delimitador entre entradas es más robusto que `NewLine`.
- `LanguageFlags.NewLineBeforeEOF` maneja archivos sin salto de línea final.

---

## 3. Ejemplo 3 — Mini-lenguaje Imperativo con Bucles y Condicionales

```csharp
[Language("MiniLang", "1.0", "Mini-lenguaje imperativo")]
public class MiniLangGrammar : Grammar {

    public MiniLangGrammar() : base(caseSensitive: true) {

        // ── Terminales ──────────────────────────────────────────────
        var number = new NumberLiteral("number");
        var str    = new StringLiteral("string", "\"", StringOptions.AllowsAllEscapes);
        var id     = new IdentifierTerminal("id");
        var comment = new CommentTerminal("comment", "//", "\n");
        NonGrammarTerminals.Add(comment);

        // ── No-terminales ───────────────────────────────────────────
        var program    = new NonTerminal("Program");
        var block      = new NonTerminal("Block");
        var stmtList   = new NonTerminal("StmtList");
        var stmt       = new NonTerminal("Stmt");
        var ifStmt     = new NonTerminal("IfStmt");
        var whileStmt  = new NonTerminal("WhileStmt");
        var forStmt    = new NonTerminal("ForStmt");
        var assignStmt = new NonTerminal("AssignStmt");
        var printStmt  = new NonTerminal("PrintStmt");
        var returnStmt = new NonTerminal("ReturnStmt");
        var funcDef    = new NonTerminal("FuncDef");
        var paramList  = new NonTerminal("ParamList");
        var argList    = new NonTerminal("ArgList");
        var expr       = new NonTerminal("Expr");
        var term       = new NonTerminal("Term");
        var binOp      = new NonTerminal("BinOp");
        var unOp       = new NonTerminal("UnOp");
        var funcCall   = new NonTerminal("FuncCall");
        var elseClause = new NonTerminal("ElseClause");

        // ── Reglas BNF ──────────────────────────────────────────────
        program.Rule  = MakeStarRule(program, stmt);
        block.Rule    = "{" + stmtList + "}";
        stmtList.Rule = MakeStarRule(stmtList, stmt);
        stmt.Rule     = assignStmt + ";"
                      | ifStmt
                      | whileStmt
                      | forStmt
                      | printStmt + ";"
                      | funcDef
                      | returnStmt + ";"
                      | funcCall + ";";

        // if (cond) block [else block]
        elseClause.Rule = ("else" + block).Q();
        ifStmt.Rule = "if" + "(" + expr + ")" + block
                    + elseClause + PreferShiftHere();
        // Nota: PreferShiftHere() en la regla del if-else resuelve el dangling-else

        // while (cond) block
        whileStmt.Rule = "while" + "(" + expr + ")" + block;

        // for (init; cond; update) block
        forStmt.Rule = "for" + "(" + assignStmt + ";" + expr + ";" + assignStmt + ")" + block;

        // var id = expr; o id = expr;
        assignStmt.Rule = "var" + id + "=" + expr
                        | id + "=" + expr;

        printStmt.Rule  = "print" + "(" + argList + ")";
        returnStmt.Rule = "return" + expr.Q();

        // function name(params) { body }
        paramList.Rule = MakeStarRule(paramList, ToTerm(","), id);
        funcDef.Rule   = "function" + id + "(" + paramList + ")" + block;

        funcCall.Rule  = id + "(" + argList + ")";
        argList.Rule   = MakeStarRule(argList, ToTerm(","), expr);

        // Expresiones
        binOp.Rule  = ToTerm("+") | "-" | "*" | "/" | "%" | "==" | "!=" | "<" | ">" | "<=" | ">=" | "&&" | "||";
        unOp.Rule   = ToTerm("-") | "!";
        term.Rule   = number | str | funcCall | id | "(" + expr + ")" | "true" | "false" | "null";
        expr.Rule   = term | expr + binOp + expr | unOp + expr;

        // ── Precedencia ─────────────────────────────────────────────
        RegisterOperators(10, "||");
        RegisterOperators(20, "&&");
        RegisterOperators(30, "==", "!=");
        RegisterOperators(40, "<", ">", "<=", ">=");
        RegisterOperators(50, "+", "-");
        RegisterOperators(60, "*", "/", "%");
        RegisterOperators(70, Associativity.Right, unOp); // unario

        // ── Marcado ─────────────────────────────────────────────────
        MarkPunctuation("(", ")", "{", "}", ";", ",");
        RegisterBracePair("(", ")");
        RegisterBracePair("{", "}");
        MarkTransient(term, binOp, unOp, stmt);
        MarkReservedWords("if", "else", "while", "for", "function", "return",
                          "print", "var", "true", "false", "null");

        // ── Reporte de errores ───────────────────────────────────────
        AddTermsReportGroup("operador", "+", "-", "*", "/", "%");
        AddTermsReportGroup("comparación", "==", "!=", "<", ">", "<=", ">=");
        AddToNoReportGroup(";");

        Root = program;
        LanguageFlags = LanguageFlags.CreateAst;
    }
}
```

**Puntos clave:**
- La expresión `elseClause.Rule = ("else" + block).Q()` hace el `else` opcional sin usar una regla aparte para el if-con-else.
- `PreferShiftHere()` en la regla del `if` resuelve el dangling-else.
- `MarkReservedWords` impide usar keywords como nombres de variable.

---

## 4. Ejemplo 4 — Consultas SQL Simplificadas

```csharp
[Language("MiniSQL", "1.0", "Subconjunto de SQL")]
public class MiniSqlGrammar : Grammar {

    public MiniSqlGrammar() : base(caseSensitive: false) {

        // ── Terminales ──────────────────────────────────────────────
        var number = new NumberLiteral("number");
        var str    = new StringLiteral("string", "'", StringOptions.AllowsDoubledQuote);
        var id     = new IdentifierTerminal("id");
        // SQL permite identificadores con comillas:
        var quotedId = new StringLiteral("quoted-id", "\"", StringOptions.None);
        quotedId.SetOutputTerminal(this, id);  // convierte el string a "id"
        var comment = new CommentTerminal("comment", "--", "\n");
        NonGrammarTerminals.Add(comment);
        var blockComment = new CommentTerminal("block-comment", "/*", "*/");
        NonGrammarTerminals.Add(blockComment);

        // ── No-terminales ───────────────────────────────────────────
        var query       = new NonTerminal("Query");
        var selectStmt  = new NonTerminal("SelectStmt");
        var insertStmt  = new NonTerminal("InsertStmt");
        var updateStmt  = new NonTerminal("UpdateStmt");
        var deleteStmt  = new NonTerminal("DeleteStmt");
        var columnList  = new NonTerminal("ColumnList");
        var colRef      = new NonTerminal("ColRef");
        var tableRef    = new NonTerminal("TableRef");
        var whereClause = new NonTerminal("WhereClause");
        var expr        = new NonTerminal("Expr");
        var binOp       = new NonTerminal("BinOp");
        var value       = new NonTerminal("Value");
        var assignList  = new NonTerminal("AssignList");
        var assignment  = new NonTerminal("Assignment");
        var valueList   = new NonTerminal("ValueList");
        var orderBy     = new NonTerminal("OrderBy");
        var orderItem   = new NonTerminal("OrderItem");
        var limitClause = new NonTerminal("LimitClause");

        // ── Reglas BNF ──────────────────────────────────────────────
        query.Rule = selectStmt | insertStmt | updateStmt | deleteStmt;

        // SELECT
        columnList.Rule = MakePlusRule(columnList, ToTerm(","), colRef);
        colRef.Rule     = id + "." + id
                        | id
                        | ToTerm("*");
        tableRef.Rule   = id + "." + id
                        | id;
        whereClause.Rule = "WHERE" + expr;
        orderItem.Rule   = colRef | colRef + "ASC" | colRef + "DESC";
        orderBy.Rule     = "ORDER" + "BY" + MakePlusRule(new NonTerminal("OrderItems"),
                               ToTerm(","), orderItem);
        limitClause.Rule = "LIMIT" + number;

        selectStmt.Rule = "SELECT" + columnList
                        + "FROM" + tableRef
                        + whereClause.Q()
                        + orderBy.Q()
                        + limitClause.Q();

        // INSERT
        insertStmt.Rule = "INSERT" + "INTO" + tableRef
                        + "(" + columnList + ")"
                        + "VALUES" + "(" + valueList + ")";

        // UPDATE
        assignment.Rule = colRef + "=" + expr;
        assignList.Rule = MakePlusRule(assignList, ToTerm(","), assignment);
        updateStmt.Rule = "UPDATE" + tableRef
                        + "SET" + assignList
                        + whereClause.Q();

        // DELETE
        deleteStmt.Rule = "DELETE" + "FROM" + tableRef
                        + whereClause.Q();

        // Expresiones
        value.Rule     = number | str | id | "NULL" | "TRUE" | "FALSE";
        valueList.Rule = MakePlusRule(valueList, ToTerm(","), value);
        binOp.Rule     = ToTerm("=") | "!=" | "<>" | "<" | ">" | "<=" | ">="
                        | "AND" | "OR" | "LIKE" | "NOT" + "LIKE" | "IN" | "NOT" + "IN"
                        | "+" | "-" | "*" | "/";
        expr.Rule      = value | colRef | expr + binOp + expr
                        | "NOT" + expr
                        | "(" + expr + ")"
                        | "IS" + "NULL"
                        | "IS" + "NOT" + "NULL";

        // ── Precedencia ─────────────────────────────────────────────
        RegisterOperators(10, "OR");
        RegisterOperators(20, "AND");
        RegisterOperators(30, "NOT");
        RegisterOperators(40, "=", "!=", "<>", "<", ">", "<=", ">=", "LIKE", "IN");
        RegisterOperators(50, "+", "-");
        RegisterOperators(60, "*", "/");

        // ── Marcado ─────────────────────────────────────────────────
        MarkPunctuation("(", ")", ",");
        RegisterBracePair("(", ")");
        MarkTransient(binOp, value);
        MarkReservedWords("SELECT", "FROM", "WHERE", "INSERT", "INTO", "VALUES",
                          "UPDATE", "SET", "DELETE", "ORDER", "BY", "ASC", "DESC",
                          "LIMIT", "AND", "OR", "NOT", "LIKE", "IN", "IS", "NULL",
                          "TRUE", "FALSE");

        // ── Reporte de errores ───────────────────────────────────────
        AddOperatorReportGroup("operador SQL");
        AddToNoReportGroup(",", ")");

        Root = query;
    }
}
```

---

## 5. Ejemplo 5 — Resolución de Conflictos con Hints

Demostración de los distintos mecanismos de resolución de conflictos.

```csharp
[Language("ConflictDemo", "1.0", "Demostración de resolución de conflictos")]
public class ConflictDemoGrammar : Grammar {

    public ConflictDemoGrammar() : base(caseSensitive: true) {
        var id     = new IdentifierTerminal("id");
        var number = new NumberLiteral("number");
        var expr   = new NonTerminal("Expr");
        var stmt   = new NonTerminal("Stmt");
        var stmtList = new NonTerminal("StmtList");
        var ifStmt = new NonTerminal("IfStmt");
        var forStmt = new NonTerminal("ForStmt");
        var postfixExpr = new NonTerminal("PostfixExpr");
        var prefixExpr  = new NonTerminal("PrefixExpr");
        var callExpr    = new NonTerminal("CallExpr");
        var argList     = new NonTerminal("ArgList");

        // ── Caso 1: Dangling-else con PreferShiftHere ────────────────
        // Sin hint: conflicto shift/reduce en el token "else"
        // Con hint: el "else" se asocia al "if" más interno
        ifStmt.Rule =
            "if" + "(" + expr + ")" + stmt
          | "if" + "(" + expr + ")" + stmt + PreferShiftHere() + "else" + stmt;

        // ── Caso 2: Postfix vs llamada — PreferShiftHere ─────────────
        // Sin hint: "id++" podría reducir a "id" antes de ver "++"
        // "id()" podría reducir a "id" antes de ver "("
        postfixExpr.Rule = expr + PreferShiftHere() + "++"
                         | expr + PreferShiftHere() + "--";
        callExpr.Rule    = expr + PreferShiftHere() + "(" + argList + ")";

        // ── Caso 3: Unario vs binario — ReduceHere ───────────────────
        // Después de reconocer "unOp term", reducir antes de ver el binOp
        prefixExpr.Rule = ToTerm("-") + expr + ReduceHere()
                        | "!" + expr + ReduceHere();

        // ── Caso 4: ShiftIf/ReduceIf — lookahead contextual ─────────
        // Ejemplo: "f(x)" puede ser llamada o declaración
        // Reducir como "expresión" si viene antes ")" o ";"
        // Shiftear (interpretar como declaración) si viene "{"
        //   (esto es solo ilustrativo, la implementación real requeriría
        //    más contexto de la gramática)
        var ambigNT = new NonTerminal("AmbigNT");
        var hint = ReduceIf(";", "{");   // Reducir si ';' aparece antes que '{'
        ambigNT.Rule = id + "(" + argList + ")" + hint;

        // ── Reglas auxiliares ────────────────────────────────────────
        argList.Rule = MakeStarRule(argList, ToTerm(","), expr);
        expr.Rule    = number | id | callExpr | postfixExpr | prefixExpr
                     | expr + "+" + expr | expr + "-" + expr
                     | expr + "*" + expr | expr + "/" + expr
                     | "(" + expr + ")";
        stmt.Rule    = ifStmt | forStmt | expr + ";" | "{" + stmtList + "}";
        stmtList.Rule = MakeStarRule(stmtList, stmt);
        forStmt.Rule = "for" + id + "in" + expr + stmt;

        RegisterOperators(10, "+", "-");
        RegisterOperators(20, "*", "/");
        MarkPunctuation("(", ")", "{", "}", ";", ",");
        RegisterBracePair("(", ")");
        RegisterBracePair("{", "}");
        MarkTransient(stmt);
        MarkReservedWords("if", "else", "for", "in");

        Root = stmtList;
    }
}
```

**Cuándo usar cada hint:**

| Conflicto | Herramienta | Cuándo |
|---|---|---|
| Dangling-else | `PreferShiftHere()` | El shift es la interpretación correcta (más interna) |
| Operador postfijo | `PreferShiftHere()` | El operador debe asociarse con el token anterior |
| Operador unario | `ReduceHere()` | Reducir la expresión antes del siguiente operador |
| Ambigüedad contextual | `ReduceIf()` / `ShiftIf()` | La decisión depende de tokens futuros |
| Ambigüedad compleja | `CustomActionHere()` | La lógica es demasiado compleja para los otros hints |

---

## 6. Ejemplo 6 — Lenguaje con Indentación (Python-style)

```csharp
[Language("IndentLang", "1.0", "Lenguaje sensible a la indentación")]
public class IndentLangGrammar : Grammar {

    public IndentLangGrammar() : base(caseSensitive: true) {

        // ── Terminales ──────────────────────────────────────────────
        var number  = new NumberLiteral("number");
        var str     = new StringLiteral("string", "\"", StringOptions.AllowsAllEscapes);
        var id      = new IdentifierTerminal("id");
        var comment = new CommentTerminal("comment", "#", "\n");
        NonGrammarTerminals.Add(comment);

        // ── No-terminales ───────────────────────────────────────────
        var program   = new NonTerminal("Program");
        var block     = new NonTerminal("Block");
        var stmt      = new NonTerminal("Stmt");
        var ifStmt    = new NonTerminal("IfStmt");
        var elifClause = new NonTerminal("ElifClause");
        var elseClause = new NonTerminal("ElseClause");
        var whileStmt = new NonTerminal("WhileStmt");
        var assignStmt = new NonTerminal("AssignStmt");
        var printStmt = new NonTerminal("PrintStmt");
        var expr      = new NonTerminal("Expr");
        var binOp     = new NonTerminal("BinOp");
        var term      = new NonTerminal("Term");

        // ── Reglas BNF ──────────────────────────────────────────────
        // El bloque usa los tokens virtuales INDENT/DEDENT:
        block.Rule    = Indent + MakePlusRule(new NonTerminal("Stmts"), stmt) + Dedent;

        // Las sentencias terminan con Eos (End-of-Statement), no con ";":
        assignStmt.Rule = id + "=" + expr;
        printStmt.Rule  = "print" + "(" + expr + ")";

        elifClause.Rule = MakeStarRule(elifClause, "elif" + "(" + expr + ")" + ":" + block);
        elseClause.Rule = ("else" + ":" + block).Q();

        ifStmt.Rule = "if" + expr + ":" + block + elifClause + elseClause;
        whileStmt.Rule = "while" + expr + ":" + block;

        stmt.Rule = (assignStmt | printStmt | ifStmt | whileStmt) + Eos;

        // Expresiones
        binOp.Rule = ToTerm("+") | "-" | "*" | "/" | "==" | "!=" | "<" | ">" | "<=" | ">=";
        term.Rule  = number | str | id | "(" + expr + ")";
        expr.Rule  = term | expr + binOp + expr;

        program.Rule = MakeStarRule(program, stmt);

        // ── Precedencia ─────────────────────────────────────────────
        RegisterOperators(10, "==", "!=", "<", ">", "<=", ">=");
        RegisterOperators(20, "+", "-");
        RegisterOperators(30, "*", "/");

        // ── Marcado ─────────────────────────────────────────────────
        MarkPunctuation("(", ")", ":");
        RegisterBracePair("(", ")");
        MarkTransient(term, binOp);
        MarkReservedWords("if", "elif", "else", "while", "print");

        Root = program;

        // ── Flags necesarios para lenguajes con indentación ─────────
        // DisableScannerParserLink es OBLIGATORIO con CodeOutlineFilter:
        LanguageFlags = LanguageFlags.NewLineBeforeEOF
                      | LanguageFlags.DisableScannerParserLink
                      | LanguageFlags.CreateAst;
    }

    // ── Configurar el filtro de indentación ──────────────────────────
    public override void CreateTokenFilters(LanguageData language, TokenFilterList filters) {
        var filter = new CodeOutlineFilter(
            language.GrammarData,
            OutlineOptions.ProduceIndents | OutlineOptions.CheckBraces,
            continuationTerminal: null  // sin continuación de línea
        );
        filters.Add(filter);
    }
}
```

**Código de entrada válido para esta gramática:**
```python
x = 10
if x > 5:
    print(x)
    y = x * 2
else:
    y = 0
while y > 0:
    print(y)
    y = y - 1
```

---

## 7. Ejemplo 7 — Gramática con Recuperación de Errores

```csharp
[Language("RobustLang", "1.0", "Lenguaje con recuperación de errores")]
public class RobustLangGrammar : Grammar {

    public RobustLangGrammar() : base(caseSensitive: true) {
        var number = new NumberLiteral("number");
        var id     = new IdentifierTerminal("id");
        var expr   = new NonTerminal("Expr");
        var stmt   = new NonTerminal("Stmt");
        var stmtList = new NonTerminal("StmtList");
        var block   = new NonTerminal("Block");
        var funcDef = new NonTerminal("FuncDef");
        var paramList = new NonTerminal("ParamList");
        var binOp   = new NonTerminal("BinOp");
        var term    = new NonTerminal("Term");

        // Reglas normales:
        binOp.Rule    = ToTerm("+") | "-" | "*" | "/";
        term.Rule     = number | id | "(" + expr + ")";
        expr.Rule     = term | expr + binOp + expr;
        paramList.Rule = MakeStarRule(paramList, ToTerm(","), id);
        block.Rule    = "{" + stmtList + "}";
        stmtList.Rule = MakePlusRule(stmtList, stmt);
        stmt.Rule     = expr + ";" | funcDef;
        funcDef.Rule  = "function" + id + "(" + paramList + ")" + block;

        // ── ErrorRules — recuperación sincronizada en ";" y "}" ──────
        // Si hay un error dentro de una sentencia, saltar hasta el ";"
        stmt.ErrorRule = SyntaxError + ";";

        // Si hay un error dentro de un bloque, saltar hasta el "}"
        block.ErrorRule = "{" + SyntaxError + "}";

        // Si hay un error en una expresión, saltar hasta ";" o ")"
        // expr.ErrorRule = SyntaxError; // comentado — provocaría conflictos

        RegisterOperators(10, "+", "-");
        RegisterOperators(20, "*", "/");

        MarkPunctuation("(", ")", "{", "}", ";", ",");
        RegisterBracePair("(", ")");
        RegisterBracePair("{", "}");
        MarkTransient(term, binOp);
        MarkReservedWords("function");

        Root = stmtList;
        LanguageFlags = LanguageFlags.CreateAst;
    }
}

// ── Uso con manejo de múltiples errores ─────────────────────────────────────
var parser = new Parser(new RobustLangGrammar());
var source = @"
a + b;
error sin punto y coma
c * d;
function foo(x) { return bad syntax; }
e + f;
";

var tree = parser.Parse(source.Trim());

// Con recuperación de errores, el parser continúa después de cada error:
Console.WriteLine($"Estado: {tree.Status}");
foreach (var msg in tree.ParserMessages) {
    var level = msg.Level == ErrorLevel.Error ? "ERROR" : "WARNING";
    Console.WriteLine($"[{level}] Línea {msg.Location.Line + 1}: {msg.Message}");
}

// Algunos nodos del árbol pueden ser nodos de error:
void PrintTree(ParseTreeNode node, int depth = 0) {
    var indent = new string(' ', depth * 2);
    var marker = node.IsError ? " ⚠️" : "";
    Console.WriteLine($"{indent}{node.Term?.Name ?? "?"}{marker}");
    foreach (var child in node.ChildNodes)
        PrintTree(child, depth + 1);
}
if (tree.Root != null)
    PrintTree(tree.Root);
```

---

## 8. Ejemplo 8 — Strings con Expresiones Embebidas

```csharp
[Language("TemplateLang", "1.0", "Lenguaje con strings plantilla")]
public class TemplateLangGrammar : Grammar {

    public TemplateLangGrammar() : base(caseSensitive: true) {
        var number = new NumberLiteral("number");
        var id     = new IdentifierTerminal("id");

        // ── Expresión (declarada primero para referenciarla en el template) ──
        var Expr = new NonTerminal("Expr");

        // ── String con expresiones embebidas ────────────────────────
        // Estilo Ruby/JavaScript: "Hola, #{nombre}!"
        var str = new StringLiteral("string", "\"",
            StringOptions.AllowsAllEscapes | StringOptions.IsTemplate);

        // Configurar el motor de plantillas:
        var templateSettings = new StringTemplateSettings {
            StartTag = "#{",   // Inicio de expresión embebida
            EndTag   = "}",    // Fin de expresión embebida
            ExpressionRoot = Expr  // Cómo parsear las expresiones dentro
        };
        str.AstConfig.Data = templateSettings;

        // El parser de snippets necesita saber cómo parsear "Expr" independientemente:
        this.SnippetRoots.Add(Expr);

        // ── Reglas ──────────────────────────────────────────────────
        var term  = new NonTerminal("Term");
        var binOp = new NonTerminal("BinOp");
        var stmt  = new NonTerminal("Stmt");
        var prog  = new NonTerminal("Program");

        binOp.Rule = ToTerm("+") | "-" | "*" | "/";
        term.Rule  = number | str | id | "(" + Expr + ")";
        Expr.Rule  = term | Expr + binOp + Expr;
        stmt.Rule  = id + "=" + Expr + ";"
                   | "print" + "(" + Expr + ")" + ";";
        prog.Rule  = MakePlusRule(prog, stmt);

        RegisterOperators(10, "+", "-");
        RegisterOperators(20, "*", "/");
        MarkPunctuation("(", ")", ";");
        RegisterBracePair("(", ")");
        MarkTransient(term, binOp);
        MarkReservedWords("print");

        Root = prog;
        LanguageFlags = LanguageFlags.CreateAst;
    }
}

/*
Código de entrada válido:
    nombre = "World";
    edad = 25;
    msg = "Hola, #{nombre}! Tienes #{edad * 2} años en binario.";
    print(msg);
*/
```

**Cómo funcionan los templates:**
1. El scanner reconoce el string hasta encontrar `#{`.
2. Al encontrar `#{`, crea un token de apertura de plantilla.
3. El scanner de snippet parsea la expresión interior usando `ExpressionRoot`.
4. Al encontrar `}`, retoma el parseo del string externo.
5. El nodo `StringTemplateNode` en el AST contiene los segmentos literales y las sub-expresiones intercaladas.
