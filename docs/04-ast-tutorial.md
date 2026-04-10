# Tutorial: Generación de un AST desde el Resultado del Parseo

El **Árbol de Sintaxis Abstracta** (AST) es una representación simplificada del código fuente que retiene solo la estructura semánticamente relevante, eliminando detalles sintácticos como puntuación, palabras clave estructurales y nodos de agrupación intermedios.

---

## Índice

1. [Parse Tree vs AST](#1-parse-tree-vs-ast)
2. [Interfaces y Clases Base del AST](#2-interfaces-y-clases-base-del-ast)
3. [Método 1 — Construcción Automática](#3-método-1--construcción-automática)
4. [Método 2 — Construcción Manual Post-Parseo](#4-método-2--construcción-manual-post-parseo)
5. [Mapeo de Hijos con `AstConfig.PartsMap`](#5-mapeo-de-hijos-con-astconfigpartsmap)
6. [AstContext — Configuración de Tipos por Defecto](#6-astcontext--configuración-de-tipos-por-defecto)
7. [Personalizar el AstBuilder](#7-personalizar-el-astbuilder)
8. [Usar los Nodos del Irony.Interpreter](#8-usar-los-nodos-del-ironyinterpreter)
9. [Ejemplo Completo — Calculadora con AST](#9-ejemplo-completo--calculadora-con-ast)
10. [Ejemplo Completo — Lenguaje de Expresiones con Variables](#10-ejemplo-completo--lenguaje-de-expresiones-con-variables)
11. [Navegar y Recorrer el AST](#11-navegar-y-recorrer-el-ast)
12. [Patrones Comunes y Buenas Prácticas](#12-patrones-comunes-y-buenas-prácticas)

---

## 1. Parse Tree vs AST

### Parse Tree (Concrete Syntax Tree)

El **parse tree** es lo que produce directamente el parser. Contiene **todos** los nodos de la gramática incluyendo:
- Puntuación (`;`, `,`, `(`, `)`, etc.)
- Palabras clave estructurales (`if`, `while`, etc.)
- Nodos de agrupación intermedios (expresiones entre paréntesis, wrappers)
- Nodos transitorios

```
Código: 3 + 4 * 2

Parse Tree:
Expr
├── Expr
│   └── number: "3"
├── BinOp: "+"
└── Expr
    ├── Expr
    │   └── number: "4"
    ├── BinOp: "*"
    └── Expr
        └── number: "2"
```

### AST

El **AST** conserva solo lo estructuralmente relevante:

```
BinExpr("+")
├── NumberNode(3)
└── BinExpr("*")
    ├── NumberNode(4)
    └── NumberNode(2)
```

---

## 2. Interfaces y Clases Base del AST

### `IAstNodeInit` — Interface mínima requerida

```csharp
namespace Irony.Ast {
    public interface IAstNodeInit {
        void Init(AstContext context, ParseTreeNode parseNode);
    }
}
```

Es la **interface mínima** que un nodo AST debe implementar para ser creado automáticamente por `AstBuilder`. El método `Init` es llamado inmediatamente después de crear la instancia del nodo.

### `IBrowsableAstNode` — Para GrammarExplorer

```csharp
public interface IBrowsableAstNode {
    int Position { get; }       // Posición en el texto fuente
    IEnumerable GetChildNodes(); // Hijos para mostrar en el árbol
}
```

Implementar esta interface hace que el nodo sea visible en el árbol AST del `GrammarExplorer`.

### `AstContext`

```csharp
public class AstContext {
    public readonly LanguageData Language;
    public Type DefaultNodeType;             // Tipo por defecto para NonTerminals
    public Type DefaultLiteralNodeType;      // Tipo por defecto para NumberLiteral, StringLiteral
    public Type DefaultIdentifierNodeType;   // Tipo por defecto para IdentifierTerminal
    public LogMessageList Messages;          // Para reportar errores durante construcción del AST
    public Dictionary<object, object> Values; // Datos compartidos durante construcción
}
```

### `AstNodeConfig`

```csharp
public class AstNodeConfig {
    public Type NodeType;                    // Tipo .NET del nodo AST
    public AstNodeCreator NodeCreator;       // Delegado custom para crear el nodo
    public Func<object> DefaultNodeCreator;  // (Interno) Creador compilado
    public int[] PartsMap;                   // Mapa de hijos relevantes
    public object Data;                      // Datos de configuración adicionales
}
```

---

## 3. Método 1 — Construcción Automática

Este es el método recomendado. El `AstBuilder` crea automáticamente los nodos AST tras el parseo exitoso.

### Paso 1: Activar el flag en la gramática

```csharp
this.LanguageFlags = LanguageFlags.CreateAst;
```

### Paso 2: Asociar tipos de nodo AST a los no-terminales

```csharp
// Opción A — En el constructor del NonTerminal:
var binExpr = new NonTerminal("BinExpr", typeof(BinaryExprNode));
var funcDef = new NonTerminal("FuncDef", typeof(FunctionDefNode));

// Opción B — Via AstConfig después de la creación:
var stmt = new NonTerminal("Statement");
stmt.AstConfig.NodeType = typeof(StatementNode);

// Opción C — Con delegado (máximo control):
var complexNT = new NonTerminal("Complex");
complexNT.AstConfig.NodeCreator = (context, parseNode) => {
    // Crear e inicializar el nodo manualmente:
    var myNode = new ComplexNode();
    myNode.Init(context, parseNode);
    parseNode.AstNode = myNode;
};
```

> **Nota:** Los terminales de puntuación y los nodos marcados con `NoAstNode` o `MarkTransient` no necesitan tipo AST — se omiten automáticamente.

### Paso 3: Implementar los nodos AST

```csharp
public class BinaryExprNode : IAstNodeInit, IBrowsableAstNode {

    public string Operator { get; private set; }
    public IAstNodeInit Left  { get; private set; }
    public IAstNodeInit Right { get; private set; }
    public SourceSpan Span    { get; private set; }

    // Requerido por IAstNodeInit:
    public void Init(AstContext context, ParseTreeNode parseNode) {
        Span = parseNode.Span;
        // parseNode.ChildNodes ya tiene la puntuación filtrada.
        // Con "Expr + BinOp + Expr" donde BinOp es Transient:
        Left     = (IAstNodeInit)parseNode.ChildNodes[0].AstNode;
        Operator = parseNode.ChildNodes[1].Token.Text;  // el operador es un token
        Right    = (IAstNodeInit)parseNode.ChildNodes[2].AstNode;
    }

    // Requerido por IBrowsableAstNode (opcional pero recomendado):
    public int Position => Span.Location.Position;
    public IEnumerable GetChildNodes() => new[] { Left, Right };

    public override string ToString() => $"BinExpr({Operator})";
}
```

### Paso 4: Parsear y acceder al AST

```csharp
var grammar  = new MiGramatica();
var language = new LanguageData(grammar);
var parser   = new Parser(language);

var parseTree = parser.Parse("3 + 4 * 2");

if (!parseTree.HasErrors()) {
    // El nodo raíz del AST:
    var rootAstNode = parseTree.Root.AstNode;
    Console.WriteLine(rootAstNode);

    // Cast al tipo esperado:
    var expr = (BinaryExprNode)parseTree.Root.AstNode;
    Console.WriteLine(expr.Operator);  // "+"
}
```

---

## 4. Método 2 — Construcción Manual Post-Parseo

Útil cuando no se puede modificar las clases de nodo AST (p.ej. son de una biblioteca externa) o cuando se prefiere una construcción más explícita.

```csharp
// Sin LanguageFlags.CreateAst en la gramática.
var parseTree = parser.Parse(sourceCode);

if (!parseTree.HasErrors()) {
    var ast = BuildNode(parseTree.Root);
}

private MiNodo BuildNode(ParseTreeNode node) {
    // Filtrar puntuación manualmente si no se usó MarkPunctuation:
    var children = node.ChildNodes
        .Where(c => !c.Term.Flags.IsSet(TermFlags.IsPunctuation))
        .ToList();

    switch (node.Term.Name) {

        case "BinExpr":
            return new BinExprNode(
                BuildNode(children[0]),
                children[1].Token.Text,   // operador
                BuildNode(children[2])
            );

        case "number":
            return new NumberNode(Convert.ToDouble(node.Token.Value));

        case "identifier":
            return new IdentifierNode(node.Token.Text);

        case "FuncCall":
            var args = children.Skip(1).Select(BuildNode).ToList();
            return new FuncCallNode(children[0].Token.Text, args);

        default:
            // Para nodos de un solo hijo (wrappers/transitorios no marcados):
            if (children.Count == 1)
                return BuildNode(children[0]);

            throw new Exception($"Nodo desconocido: {node.Term.Name} " +
                                $"en {node.Span.Location}");
    }
}
```

---

## 5. Mapeo de Hijos con `AstConfig.PartsMap`

Cuando una producción tiene hijos que no son relevantes para el AST (palabras clave, delimitadores que no se marcan con `MarkPunctuation`), se puede especificar exactamente qué hijos mapear:

```csharp
// Producción: "if" condition "then" trueBranch "else" falseBranch "end"
// Índices:       0      1       2       3          4       5         6
// Solo interesan índices 1, 3 y 5:
ifStmt.AstConfig.PartsMap = new int[] { 1, 3, 5 };
```

En el método `Init` del nodo AST usar `GetMappedChildNodes()`:

```csharp
public void Init(AstContext context, ParseTreeNode parseNode) {
    // GetMappedChildNodes() respeta el PartsMap si está definido:
    var mapped = parseNode.GetMappedChildNodes();
    // mapped[0] = condition (índice original 1)
    // mapped[1] = trueBranch (índice original 3)
    // mapped[2] = falseBranch (índice original 5)
    Condition   = (ExprNode)mapped[0].AstNode;
    TrueBranch  = (StmtNode)mapped[1].AstNode;
    FalseBranch = (StmtNode)mapped[2].AstNode;
}
```

> **Ventaja:** Permite que los nodos AST sean independientes de cambios en la gramática (posición de keywords, etc.) ya que el mapa se configura en la gramática, no en el nodo.

---

## 6. AstContext — Configuración de Tipos por Defecto

En vez de especificar el tipo AST en cada non-terminal, se pueden configurar tipos por defecto en el `AstContext`:

```csharp
// Sobreescribir BuildAst en la gramática:
public override void BuildAst(LanguageData language, ParseTree parseTree) {
    if (!LanguageFlags.IsSet(LanguageFlags.CreateAst)) return;

    var context = new AstContext(language) {
        DefaultLiteralNodeType    = typeof(LiteralValueNode),
        DefaultIdentifierNodeType = typeof(IdentifierNode),
        DefaultNodeType           = typeof(GenericAstNode)
    };
    var builder = new AstBuilder(context);
    builder.BuildAst(parseTree);
}
```

El `AstBuilder.GetDefaultNodeType()` usa estos tipos cuando un término no tiene tipo explícito:

```
Si es NumberLiteral o StringLiteral → DefaultLiteralNodeType
Si es IdentifierTerminal            → DefaultIdentifierNodeType
Cualquier otro término              → DefaultNodeType
```

---

## 7. Personalizar el AstBuilder

Para control total sobre el proceso de construcción, heredar de `AstBuilder`:

```csharp
public class MiAstBuilder : AstBuilder {

    public MiAstBuilder(AstContext context) : base(context) { }

    // Sobreescribir para cambiar el tipo por defecto según el término:
    protected override Type GetDefaultNodeType(BnfTerm term) {
        if (term.Name.EndsWith("Stmt"))
            return typeof(StatementNode);
        if (term.Name.EndsWith("Expr"))
            return typeof(ExpressionNode);
        return base.GetDefaultNodeType(term);
    }

    // Sobreescribir para interceptar la construcción de cada nodo:
    public override void BuildAst(ParseTreeNode parseNode) {
        // Pre-proceso:
        Console.WriteLine($"Building AST for: {parseNode.Term.Name}");

        base.BuildAst(parseNode);

        // Post-proceso:
        if (parseNode.AstNode is IResolvable resolvable)
            resolvable.Resolve(symbolTable);
    }
}
```

Y registrar en la gramática:

```csharp
public override void BuildAst(LanguageData language, ParseTree parseTree) {
    if (!LanguageFlags.IsSet(LanguageFlags.CreateAst)) return;
    var context = new AstContext(language);
    var builder = new MiAstBuilder(context);
    builder.BuildAst(parseTree);
}
```

---

## 8. Usar los Nodos del Irony.Interpreter

`Irony.Interpreter.dll` provee una jerarquía completa de nodos AST ejecutables en `Irony.Interpreter.Ast`. Estos nodos implementan `IAstNodeInit` y son adecuados para construir intérpretes:

### `AstNode` — Clase base

```csharp
public class AstNode : IAstNodeInit, IBrowsableAstNode, IVisitableNode {
    public AstNode Parent;
    public BnfTerm Term;
    public SourceSpan Span;
    public string Role;              // Rol del nodo en su padre (p.ej. "condition")
    public virtual string AsString { get; }  // Para mostrar en GrammarExplorer
    public AstNodeList ChildNodes;   // Hijos del AST

    // Delegados de evaluación:
    public EvaluateMethod Evaluate;  // Evalúa el nodo y retorna un valor
    public ValueSetterMethod SetValue; // Asigna un valor al nodo (para l-values)

    public virtual void Init(AstContext context, ParseTreeNode treeNode) {
        this.Term = treeNode.Term;
        this.Span = treeNode.Span;
        treeNode.AstNode = this;
    }
}
```

### Nodos disponibles en `Irony.Interpreter.Ast`

| Clase | Descripción |
|---|---|
| `LiteralValueNode` | Literales numéricos y de string |
| `IdentifierNode` | Identificadores/variables |
| `BinaryOperationNode` | Operaciones binarias (`+`, `-`, `*`, `/`, comparaciones, etc.) |
| `UnaryOperationNode` | Operaciones unarias (`-`, `!`, etc.) |
| `IfNode` | Expresión/sentencia `if` (incluyendo ternario `?:`) |
| `IncDecNode` | Operadores `++` y `--` (prefijo y postfijo) |
| `FunctionCallNode` | Llamada a función |
| `FunctionDefNode` | Definición de función |
| `LambdaNode` | Expresión lambda |
| `ParamListNode` | Lista de parámetros |
| `AssignmentNode` | Asignación y asignaciones compuestas (`+=`, etc.) |
| `StatementListNode` | Lista de sentencias (programa, cuerpo de función) |
| `MemberAccessNode` | Acceso a miembro (`obj.field`) |
| `IndexedAccessNode` | Acceso indexado (`obj[index]`) |
| `ExpressionListNode` | Lista de expresiones (argumentos) |
| `StringTemplateNode` | String con expresiones embebidas |
| `EmptyStatementNode` | Sentencia vacía |
| `NullNode` | Nodo nulo explícito |

---

## 9. Ejemplo Completo — Calculadora con AST

```csharp
// ── Gramática ──────────────────────────────────────────────────────────────────
[Language("Calc", "1.0", "Calculadora de expresiones")]
public class CalcGrammar : Grammar {
    public CalcGrammar() : base(caseSensitive: false) {
        var number  = new NumberLiteral("number");
        var binOp   = new NonTerminal("BinOp");
        var expr    = new NonTerminal("Expr",    typeof(ExprNode));
        var parExpr = new NonTerminal("ParExpr");

        binOp.Rule   = ToTerm("+") | "-" | "*" | "/" | "**";
        parExpr.Rule = "(" + expr + ")";
        expr.Rule    = number | parExpr | expr + binOp + expr;

        RegisterOperators(10, "+", "-");
        RegisterOperators(20, "*", "/");
        RegisterOperators(30, Associativity.Right, "**");

        MarkPunctuation("(", ")");
        MarkTransient(parExpr, binOp);

        Root = expr;
        LanguageFlags = LanguageFlags.CreateAst;
    }
}

// ── Nodo AST ───────────────────────────────────────────────────────────────────
public class ExprNode : IAstNodeInit, IBrowsableAstNode {

    public double? Literal;     // Null si es operación, valor si es literal
    public string Operator;     // Null si es literal
    public ExprNode Left, Right;
    public SourceLocation Location;

    public void Init(AstContext context, ParseTreeNode parseNode) {
        parseNode.AstNode = this;
        Location = parseNode.Span.Location;

        if (parseNode.Token != null) {
            // Es un token hoja (literal numérico):
            Literal = Convert.ToDouble(parseNode.Token.Value);
            return;
        }

        // Es una operación binaria después de la transparencia:
        // ChildNodes contiene: [exprIzq, operadorToken, exprDer]
        Left     = (ExprNode)parseNode.ChildNodes[0].AstNode;
        Operator = parseNode.ChildNodes[1].Token.Text;
        Right    = (ExprNode)parseNode.ChildNodes[2].AstNode;
    }

    public double Evaluate() {
        if (Literal.HasValue) return Literal.Value;
        double l = Left.Evaluate(), r = Right.Evaluate();
        return Operator switch {
            "+"  => l + r,
            "-"  => l - r,
            "*"  => l * r,
            "/"  => r == 0 ? throw new DivideByZeroException() : l / r,
            "**" => Math.Pow(l, r),
            _    => throw new InvalidOperationException($"Operador desconocido: {Operator}")
        };
    }

    // IBrowsableAstNode:
    public int Position => Location.Position;
    public IEnumerable GetChildNodes() {
        if (Left != null)  yield return Left;
        if (Right != null) yield return Right;
    }

    public override string ToString() =>
        Literal.HasValue ? Literal.Value.ToString() : $"({Operator})";
}

// ── Uso ────────────────────────────────────────────────────────────────────────
public static void Main() {
    var language = new LanguageData(new CalcGrammar());

    if (language.ErrorLevel == GrammarErrorLevel.Error) {
        foreach (var e in language.Errors)
            Console.Error.WriteLine(e);
        return;
    }

    var parser = new Parser(language);

    string[] tests = {
        "3 + 4 * 2",          // 11
        "(3 + 4) * 2",        // 14
        "2 ** 10",             // 1024
        "10 / (5 - 3)",        // 5
    };

    foreach (var code in tests) {
        var tree = parser.Parse(code);
        if (tree.HasErrors()) {
            Console.WriteLine($"Error en '{code}':");
            foreach (var msg in tree.ParserMessages)
                Console.WriteLine($"  {msg}");
        } else {
            var root = (ExprNode)tree.Root.AstNode;
            Console.WriteLine($"'{code}' = {root.Evaluate()}");
        }
    }
}
```

---

## 10. Ejemplo Completo — Lenguaje de Expresiones con Variables

```csharp
// ── Gramática ──────────────────────────────────────────────────────────────────
[Language("ExprLang", "1.0", "Lenguaje de expresiones con variables")]
public class ExprLangGrammar : Grammar {
    public ExprLangGrammar() : base(caseSensitive: true) {
        var number   = new NumberLiteral("number");
        var str      = new StringLiteral("string", "\"", StringOptions.AllowsAllEscapes);
        var id       = new IdentifierTerminal("id");
        var comment  = new CommentTerminal("comment", "#", "\n");
        NonGrammarTerminals.Add(comment);

        var expr     = new NonTerminal("Expr");
        var term     = new NonTerminal("Term");
        var binOp    = new NonTerminal("BinOp");
        var assign   = new NonTerminal("Assign",   typeof(AssignNode));
        var program  = new NonTerminal("Program",  typeof(ProgramNode));
        var stmt     = new NonTerminal("Stmt");

        term.Rule  = number | str | id | "(" + expr + ")";
        binOp.Rule = ToTerm("+") | "-" | "*" | "/" | "==" | "!=" | "<" | ">" | "<=" | ">=" | "&&" | "||";
        expr.Rule  = term | expr + binOp + expr;
        assign.Rule = id + "=" + expr;
        stmt.Rule  = assign | expr | Empty;
        program.Rule = MakePlusRule(program, NewLine, stmt);

        RegisterOperators(10, "||");
        RegisterOperators(20, "&&");
        RegisterOperators(30, "==", "!=", "<", ">", "<=", ">=");
        RegisterOperators(40, "+", "-");
        RegisterOperators(50, "*", "/");

        MarkPunctuation("(", ")");
        RegisterBracePair("(", ")");
        MarkTransient(term, binOp, stmt);

        Root = program;
        LanguageFlags = LanguageFlags.NewLineBeforeEOF | LanguageFlags.CreateAst;
    }
}

// ── Nodos AST ──────────────────────────────────────────────────────────────────

public abstract class ExprBase : IAstNodeInit {
    public abstract object Evaluate(Dictionary<string, object> env);
    public virtual void Init(AstContext context, ParseTreeNode node) {
        node.AstNode = this;
    }
}

public class NumberNode : ExprBase {
    private readonly double _value;
    public NumberNode(double value) { _value = value; }
    public override void Init(AstContext ctx, ParseTreeNode node) {
        base.Init(ctx, node);
    }
    public override object Evaluate(Dictionary<string, object> env) => _value;
}

public class StringNode : ExprBase {
    private readonly string _value;
    public StringNode(string value) { _value = value; }
    public override object Evaluate(Dictionary<string, object> env) => _value;
}

public class VarNode : ExprBase {
    public string Name { get; private set; }
    public override void Init(AstContext ctx, ParseTreeNode node) {
        base.Init(ctx, node);
        Name = node.Token.Text;
    }
    public override object Evaluate(Dictionary<string, object> env) {
        if (env.TryGetValue(Name, out var val)) return val;
        throw new Exception($"Variable no definida: {Name}");
    }
}

public class BinOpNode : ExprBase {
    private ExprBase _left, _right;
    private string _op;
    public override void Init(AstContext ctx, ParseTreeNode node) {
        base.Init(ctx, node);
        _left  = (ExprBase)node.ChildNodes[0].AstNode;
        _op    = node.ChildNodes[1].Token.Text;
        _right = (ExprBase)node.ChildNodes[2].AstNode;
    }
    public override object Evaluate(Dictionary<string, object> env) {
        var l = _left.Evaluate(env);
        var r = _right.Evaluate(env);
        // Simplificado para números:
        double ld = Convert.ToDouble(l), rd = Convert.ToDouble(r);
        return _op switch {
            "+"  => ld + rd,
            "-"  => ld - rd,
            "*"  => ld * rd,
            "/"  => ld / rd,
            "==" => ld == rd,
            "!=" => ld != rd,
            "<"  => ld < rd,
            ">"  => ld > rd,
            "<=" => ld <= rd,
            ">=" => ld >= rd,
            "&&" => Convert.ToBoolean(l) && Convert.ToBoolean(r),
            "||" => Convert.ToBoolean(l) || Convert.ToBoolean(r),
            _    => throw new Exception($"Operador desconocido: {_op}")
        };
    }
}

public class AssignNode : IAstNodeInit {
    public string VarName   { get; private set; }
    public ExprBase Value   { get; private set; }
    public void Init(AstContext ctx, ParseTreeNode node) {
        node.AstNode = this;
        VarName = node.ChildNodes[0].Token.Text;
        Value   = (ExprBase)node.ChildNodes[1].AstNode;
    }
    public void Execute(Dictionary<string, object> env) {
        env[VarName] = Value.Evaluate(env);
    }
}

public class ProgramNode : IAstNodeInit {
    private List<IAstNodeInit> _stmts = new List<IAstNodeInit>();
    public void Init(AstContext ctx, ParseTreeNode node) {
        node.AstNode = this;
        foreach (var child in node.ChildNodes)
            if (child.AstNode != null)
                _stmts.Add((IAstNodeInit)child.AstNode);
    }
    public object Execute(Dictionary<string, object> env) {
        object last = null;
        foreach (var stmt in _stmts) {
            if (stmt is AssignNode a) a.Execute(env);
            else if (stmt is ExprBase e) last = e.Evaluate(env);
        }
        return last;
    }
}

// ── Uso ────────────────────────────────────────────────────────────────────────
var parser = new Parser(new ExprLangGrammar());
var source = @"
x = 10
y = x * 2 + 5
z = x > 5
z
";
var tree = parser.Parse(source.Trim());
if (!tree.HasErrors()) {
    var env  = new Dictionary<string, object>();
    var prog = (ProgramNode)tree.Root.AstNode;
    var result = prog.Execute(env);
    Console.WriteLine($"Resultado: {result}");
    Console.WriteLine($"Variables: x={env["x"]}, y={env["y"]}, z={env["z"]}");
}
```

---

## 11. Navegar y Recorrer el AST

### Acceso directo a nodos del Parse Tree

```csharp
var root = parseTree.Root;

// Verificar si es un nodo de terminal:
if (root.Token != null)
    Console.WriteLine($"Token: {root.Token.Text}");

// Verificar si es un nodo de no-terminal:
if (root.Token == null)
    Console.WriteLine($"NT: {root.Term.Name}, hijos: {root.ChildNodes.Count}");

// Encontrar el primer token descendente:
var firstToken = root.FindToken();
var tokenText  = root.FindTokenAndGetText();

// Ubicación:
Console.WriteLine($"Línea: {root.Span.Location.Line}");
Console.WriteLine($"Columna: {root.Span.Location.Column}");
Console.WriteLine($"Posición: {root.Span.Location.Position}");
Console.WriteLine($"Longitud: {root.Span.Length}");
```

### Recorrido del AST con el patrón Visitor

```csharp
public interface IVisitor {
    void Visit(NumberNode node);
    void Visit(BinOpNode node);
    void Visit(VarNode node);
    // ... etc
}

public interface IVisitable {
    void Accept(IVisitor visitor);
}

// En cada nodo:
public class BinOpNode : ExprBase, IVisitable {
    public void Accept(IVisitor visitor) => visitor.Visit(this);
}

// Visitor concreto — por ejemplo, imprimir el árbol:
public class PrintVisitor : IVisitor {
    private int _indent = 0;
    public void Visit(NumberNode node) =>
        Console.WriteLine(new string(' ', _indent * 2) + node.Value);
    public void Visit(BinOpNode node) {
        Console.WriteLine(new string(' ', _indent * 2) + $"({node.Operator}");
        _indent++;
        node.Left.Accept(this);
        node.Right.Accept(this);
        _indent--;
        Console.WriteLine(new string(' ', _indent * 2) + ")");
    }
    // ...
}
```

### Recorrido del Parse Tree (sin AST)

```csharp
private void WalkParseTree(ParseTreeNode node, int depth = 0) {
    var indent = new string(' ', depth * 2);
    if (node.Token != null) {
        Console.WriteLine($"{indent}[Token] {node.Term.Name}: '{node.Token.Text}'");
    } else {
        Console.WriteLine($"{indent}[NT] {node.Term.Name} ({node.ChildNodes.Count} hijos)");
        foreach (var child in node.ChildNodes)
            WalkParseTree(child, depth + 1);
    }
}
WalkParseTree(parseTree.Root);
```

---

## 12. Patrones Comunes y Buenas Prácticas

### Patrón: Tabla de tipos AST centralizada

```csharp
// En la gramática, asignar tipos en un paso separado para mayor claridad:
private void ConfigureAstNodes() {
    Root.AstConfig.NodeType                 = typeof(ProgramNode);
    _stmtNT.AstConfig.NodeType              = typeof(StatementNode);
    _binExprNT.AstConfig.NodeType           = typeof(BinaryExprNode);
    _funcCallNT.AstConfig.NodeType          = typeof(FunctionCallNode);
    _funcCallNT.NodeCaptionTemplate         = "call #{0}(...)";
    _funcCallNT.AstConfig.PartsMap          = new[] { 0, 2 }; // nombre y arglist
}
```

### Patrón: Ignorar nodos irrelevantes

```csharp
// En Init(), omitir hijos que sean null (punctuation/transient):
public void Init(AstContext ctx, ParseTreeNode node) {
    node.AstNode = this;
    Children = node.ChildNodes
        .Where(c => c.AstNode != null)
        .Select(c => (IMyNode)c.AstNode)
        .ToList();
}
```

### Patrón: Nodo de lista

```csharp
// Para no-terminales generados por MakePlusRule/MakeStarRule:
public class StatementListNode : IAstNodeInit {
    public List<StmtNode> Statements { get; } = new List<StmtNode>();

    public void Init(AstContext ctx, ParseTreeNode node) {
        node.AstNode = this;
        // MakePlusRule aplana la lista — todos los hijos son sentencias directas:
        foreach (var child in node.ChildNodes) {
            if (child.AstNode is StmtNode stmt)
                Statements.Add(stmt);
        }
    }
}
```

### Patrón: Verificar errores antes de usar el AST

```csharp
var tree = parser.Parse(source);

// Siempre verificar errores antes de acceder al AST:
if (tree.HasErrors()) {
    foreach (var error in tree.ParserMessages.Where(m => m.Level == ErrorLevel.Error))
        Console.Error.WriteLine($"Error en línea {error.Location.Line + 1}: {error.Message}");
    return;
}

// Solo aquí es seguro usar tree.Root.AstNode:
var program = (ProgramNode)tree.Root.AstNode;
```

### Resumen de Flags y sus Efectos en el AST

| Configuración | Efecto en el Parse Tree / AST |
|---|---|
| `MarkPunctuation(...)` | Los nodos se omiten de `ChildNodes` |
| `MarkTransient(...)` | El nodo se reemplaza por su único hijo no-puntuación |
| `term.AstConfig.NodeType = typeof(...)` | Se crea instancia de ese tipo y se llama `Init` |
| `term.AstConfig.NodeCreator = ...` | Se llama el delegado; la lógica de init queda en el delegado |
| `term.SetFlag(TermFlags.NoAstNode)` | No se crea ningún nodo AST para este término |
| `term.SetFlag(TermFlags.AstDelayChildren)` | Los hijos no se construyen inmediatamente (compilación lazy) |
| `term.AstConfig.PartsMap = ...` | `GetMappedChildNodes()` retorna solo los hijos indicados |
