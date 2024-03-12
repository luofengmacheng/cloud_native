## ANTLR4规则解析生成器(四)：优化计算器程序

### 1 问题背景

基于antlr4的基础框架可以实现一个简单的支持四则运算的计算器程序，例如，Listener的代码如下：

``` python
class Listener(exprListener):
    def __init__(self):
        self.result = {}

    def enterProg(self, ctx:exprParser.ProgContext):
        pass

    def exitProg(self, ctx:exprParser.ProgContext):
        pass

    def enterExpr(self, ctx:exprParser.ExprContext):
        pass

    def exitExpr(self, ctx:exprParser.ExprContext):
        if ctx.getChildCount() == 3:
            if ctx.getChild(0).getText() == "(":
                self.result[ctx.getText()] = self.result[ctx.getChild(1).getText()]
            else:
                opc = ctx.getChild(1).getText()
                v1 = self.result[ctx.getChild(0).getText()]
                v2 = self.result[ctx.getChild(2).getText()]
                if opc == "+":
                    self.result[ctx.getText()] = v1 + v2
                elif opc == "-":
                    self.result[ctx.getText()] = v1 - v2
                elif opc == "*":
                    self.result[ctx.getText()] = v1 * v2
                elif opc == "/":
                    self.result[ctx.getText()] = v1 / v2
                else:
                    ctx.result[ctx.getText()] = 0
        elif ctx.getChildCount() == 2:
            opc = ctx.getChild(0).getText()
            if opc == "+":
                v = self.result[ctx.getChild(1).getText()]
                self.result[ctx.getText()] = v
            elif opc == "-":
                v = self.result[ctx.getChild(1).getText()]
                self.result[ctx.getText()] = - v
        elif ctx.getChildCount() == 1:
            self.result[ctx.getText()] = int(ctx.getChild(0).getText())
```

g4文件内容如下：

```
grammar expr;
prog:   (expr NEWLINE)* ;
expr:   expr ('*'|'/') expr
    | expr ('+'|'-') expr
    |   INT
    |   '(' expr ')'
    ;
NEWLINE : [\r\n]+ ;
INT     : [0-9]+ ;
```

上面的代码存在以下三个问题：

* 由于antlr4默认是为每条规则生成`{RuleName}Context`、`enter{RuleName}`和`exit{RuleName}`，如果规则的分支比较多，就需要在访问节点时对孩子节点进行一些判断，才能知道匹配的是哪个分支，如上面的计算器中的通过`ctx.getChildCount()`获取孩子个数以及对运算符的判断
* 在遍历生成的语法分析树时，某些业务场景需要进行值的传递，如上面的计算器中，在进行表达式的运算时，获取第一个孩子和第三个孩子的值，这里使用了一个字典进行传递，是否有其他更好的方式
* 运算符优先级和结合性是如何处理的

### 2 单独处理备选分支

为了能够对规则的备选分支进行单独处理，而不是在规则的处理逻辑中进行判断，antlr4可以对分支添加标签，在生成Listener和Visitor时可以生成标签的类型：`{Label}Context`，Listener和Visitor在处理节点时就不是规则名，而是标签名了。

对上述的计算器的g4文件加上标签名：

```
grammar expr;
prog:   (expr NEWLINE)* ;
expr:   expr ('*'|'/') expr #DIVMULTI
    | expr ('+'|'-') expr #ADDSUB 
    |   INT #INT
    |   '(' expr ')' #BRACKET
    ;
NEWLINE : [\r\n]+ ;
INT     : [0-9]+ ;
```

使用`antlr4`工具再次生成的Parser中就会创建继承自`{RuleName}Context`的类`{Label}Context`，如下例中，`BRACKETContext`、`DIVMULTIContext`、`ADDSUBContext`、`INTContext`全部都继承自`exprContext`。

``` python
class Listener(exprListener):
    def __init__(self):
        self.result = {}

    def exitProg(self, ctx:exprParser.ProgContext):
        self.result[ctx.getText()] = self.result[ctx.expr(0).getText()]

    # 在处理括号时，将括号中间的内容放到字典:节点字符串->括号内表达式的值
    def exitBRACKET(self, ctx:exprParser.BRACKETContext):
        self.result[ctx.getText()] = self.result[ctx.getChild(1).getText()]
    
    # 在处理乘除时，根据操作符将结果放到字典:节点字符串->乘除计算的结果
    def exitDIVMULTI(self, ctx:exprParser.DIVMULTIContext):
        opc = ctx.getChild(1).getText()
        v1 = self.result[ctx.expr(0).getText()]
        v2 = self.result[ctx.expr(1).getText()]
        if opc == "*":
            self.result[ctx.getText()] = v1 * v2
        elif opc == "/":
            self.result[ctx.getText()] = v1 / v2
    
    # 在处理加减时，根据操作符将结果放到字典:节点字符串->加减计算的结果
    def exitADDSUB(self, ctx:exprParser.ADDSUBContext):
        opc = ctx.getChild(1).getText()
        v1 = self.result[ctx.expr(0).getText()]
        v2 = self.result[ctx.expr(1).getText()]
        if opc == "+":
            self.result[ctx.getText()] = v1 + v2
        elif opc == "-":
            self.result[ctx.getText()] = v1 - v2
    
    # 直接处理终结符的词法时，直接将当前节点转换为整数保存到字典:节点字符串->整数值
    def exitINT(self, ctx:exprParser.INTContext):
        self.result[ctx.getText()] = int(ctx.getText())
```

用标签的方式就将之前的判断直接去掉，将代码分别放在多个处理函数中。

### 3 共享数据

从计算器的语法规则和实现看，计算本身也是个递归的过程，例如，要计算加法时，就需要得到左子树和右子树的结果，然后进行相加，但是Listener中的`enter{RuleName}`和`exit{RuleName}`都是没有返回值的，没有办法获取到左右子树的值，有三种方式解决这个问题：

* 使用Listener遍历语法分析树时，可以使用上面的字典或者栈结构存储中间结果
* 使用Visitor遍历语法分析树时，`visit{RuleName}`默认返回最后一个孩子的结果，可以在该函数的处理逻辑中调用visit遍历子树从而获取子树的结果
* 为语法分析树打标注
  
第一种和第二种方法已经介绍过，这里介绍第三种方法，为语法分析树打标注。

打标注的主要的想法是，为树的每个节点关联一个记录，当要获取子树的值时，直接获取子树的根节点关联的记录值。

修改g4语法规则文件：

```
grammar expr;
prog:   (expr NEWLINE)* ;
expr returns [int value]:   expr ('*'|'/') expr #DIVMULTI
    | expr ('+'|'-') expr #ADDSUB 
    |   INT #INT
    |   '(' expr ')' #BRACKET
    ;
NEWLINE : [\r\n]+ ;
INT     : [0-9]+ ;
```

为expr规则加上返回值，重新生成Listener和Parser后，生成的ExprContext类中多了个成员变量value：

``` python
class ExprContext(ParserRuleContext):
    __slots__ = 'parser'

    def __init__(self, parser, parent:ParserRuleContext=None, invokingState:int=-1):
        super().__init__(parent, invokingState)
        self.parser = parser
        self.value = None

    def getRuleIndex(self):
        return exprParser.RULE_expr

    def copyFrom(self, ctx:ParserRuleContext):
        super().copyFrom(ctx)
        self.value = ctx.value
```

然后在Listener和Visitor中使用该值进行传递：

``` python
class Listener(exprListener):
    def __init__(self):
        self.result = {}

    def exitProg(self, ctx:exprParser.ProgContext):
        ctx.value = ctx.getChild(0).value

    def exitBRACKET(self, ctx:exprParser.BRACKETContext):
        ctx.value = ctx.getChild(1).value

    def exitDIVMULTI(self, ctx:exprParser.DIVMULTIContext):
        opc = ctx.getChild(1).getText()
        v1 = ctx.expr(0).value
        v2 = ctx.expr(1).value
        if opc == "*":
            ctx.value = v1 * v2
        elif opc == "/":
            ctx.value = v1 / v2

    def exitADDSUB(self, ctx:exprParser.ADDSUBContext):
        opc = ctx.getChild(1).getText()
        v1 = ctx.expr(0).value
        v2 = ctx.expr(1).value
        if opc == "+":
            ctx.value = v1 + v2
        elif opc == "-":
            ctx.value = v1 - v2

    def exitINT(self, ctx:exprParser.INTContext):
        ctx.value = int(ctx.getText())
```

### 3 运算符优先级和结合性

计算器可以支持四则运算，也可以支持括号，它们的优先级是不一样的。

在antlr4中，运算符的优先级是通过`规则的顺序`隐式指定的，如计算器的规则：

```
grammar expr;
prog:   (expr NEWLINE)* ;
expr returns [int value]:   expr ('*'|'/') expr #DIVMULTI
    | expr ('+'|'-') expr #ADDSUB 
    |   INT #INT
    |   '(' expr ')' #BRACKET
    ;
NEWLINE : [\r\n]+ ;
INT     : [0-9]+ ;
```

乘法和除法是同一优先级，加法和减法是同一优先级，并且乘除法的优先级比加减法高，因此，当解析`1+2*3`时，会解析为`1+(2*3)`，但是如果交换两条规则的顺序，就会解析为`(1+2)*3`。

处理完运算符优先级，还需要处理结合性：默认情况下，运算符是左结合的，也就是相同运算符优先级的情况下，先计算左边的，但是有些运算符是右结合的。

例如，`1+2+3`，由于是左结合的，因此，会解析为`(1+2)+3`，但是，指数运算符是右结合的，因此，`1^2^3`需要解析为`1^(2^3)`。

此时，需要加上`assoc`指定结合性：

```
grammar expr;
prog:   (expr NEWLINE)* ;
expr returns [int value]:   <assoc=right> expr '^' expr #POW
    |   expr ('*'|'/') expr #DIVMULTI
    |   expr ('+'|'-') expr #ADDSUB
    |   INT #INT
    |   '(' expr ')' #BRACKET
    ;
NEWLINE : [\r\n]+ ;
INT     : [0-9]+ ;
```

至此，我们的规则文件已经比较完善了，至少对于简易计算器来说是的，但是有个不足之处：规则中有很多字符串字面量。可以将其中的字符串字面量也使用词法规则进行表示：

```
grammar expr;
prog:   (expr NEWLINE)* ;
expr returns [int value]:   <assoc=right> expr POWER expr #POW
    |   expr (MULTI|DIV) expr #DIVMULTI
    |   expr (ADD|SUB) expr #ADDSUB
    |   INT #INT
    |   LBRA expr RBRA #BRACKET
    ;

POWER: '^';
MULTI: '*';
DIV: '/';
ADD: '+';
SUB: '-';
LBRA: '(';
RBRA: ')';

NEWLINE : [\r\n]+ ;
INT     : [0-9]+ ;
```

### 5 总结

本文从规则以及树的遍历上优化了我们的计算器程序：

* antlr4默认只会为一条规则生成一个节点处理函数，如果规则的备选分支比较多，在处理该节点时就需要在里面写很多的分支语句，通过为每个备选分支设置标签，可以为每个备选分支生成一个处理函数，在单个处理函数中就可以直接处理
* Listener适合只需要遍历语法分析树，不需要进行数据交互的场景，例如，语法检查，但是也可以通过创建一个全局的数据结构保存中间结果；Visitor适合可以自定义节点访问的场景，在访问节点时可以返回数据，例如，计算器程序。在需要共享数据的场景下，还可以给节点添加标注，也就是给节点增加一个关联值，在处理节点时，可以处理子树根节点对应的关联值。
* antlr4中是通过规则的顺序来指定分支匹配的优先级，默认是左结合，在需要右结合的场景，需要使用`assoc`指定结合性
* 对于规则中的字符串常量，可以使用有意义的词法规则代替，提高规则的可读性
