## ANTLR4规则解析生成器(一)：入门

### 1 什么是ANTLR4

ANTLR是ANother Tool for Language Recognition的缩写，它是一个强大的用于读取、处理、执行和翻译结构化文本或二进制文件的语法分析器生成器，广泛用于构建语言、工具和框架，通过语法描述规则，它能够生成一个可以遍历解析树的解析器。ANTLR4是ANTLR的第4个版本。

### 2 为什么需要ANTLR4

以一个计算器的例子来说明，当我们需要开发一个计算器程序时，第一步就是要确认支持的边界，也就是要确认支持哪些运算，例如，假设只需要支持整数的四则运算，且不支持括号，也就是只支持`1+2*3`等简单的计算。然后就可以开始开发，开发的重点就变成对算式的解析，还需要处理运算符的优先级。在通常的书籍中，会基于栈和队列实现，并且需要自行处理运算符的优先级：[复杂计算器——四则运算表达式求值（中缀转后缀表达式](https://blog.csdn.net/DreamTrue520/article/details/128959621)。而使用ANTLR4就可以将算式的解析和实现分离，ANTLR4会将算式解析为语法树，然后提供遍历的机制去实现运算，因此，我们的代码就只需要实现运算即可。

简单来说，ANTLR4就是一个生成词法分析器和语法分析器的生成器，能够解析文本和二进制，解析后生成语法树，然后基于不同语言的SDK遍历该语法树，实现对应的逻辑。

使用ANTLR4通常分成三步：

* 编写语法规则文件（规则文件以g4为后缀），在规则文件中使用自顶向下的形式描述要解析的语法的格式
* 使用antlr4将规则文件转换成对应语言的语法解析代码
* 使用对应语言的SDK提供的函数，遍历语法树

### 3 环境搭建

* 安装java：建议安装比较高的版本，这里安装的是jdk17
* 安装虚拟环境：pip3 install virtualenv
* 创建虚拟环境并进入：virtualenv myenv && . myenv/bin/activate
* 安装antlr4：pip install antlr4-tools
* 安装对应语言的运行时库：对于python而言，只支持python3，安装antlr4-python3-runtime

这里面主要要注意的就是java的版本，不能用1.6或者1.8等比较低的版本。

如果使用vscode进行开发，可以安装ANTLR4 grammar syntax support插件；如果使用pycharm开发，可以安装ANTLR v4插件。

### 4 官方示例

#### 4.1 编写语法规则文件

```
grammar expr;
prog:   (expr NEWLINE)* ;
expr:   expr ('*'|'/') expr
    |   expr ('+'|'-') expr
    |   INT
    |   '(' expr ')'
    ;
NEWLINE : [\r\n]+ ;
INT     : [0-9]+ ;
```

语法规则文件是基于正则表达式并且从上到下的语法描述文件，很类似于编译原理里面的词法分析和语法分析。

* 除了grammer所在的行，每个分号结尾的部分都是描述一个规则
* grammer：声明一个语法的名称，名称为expr，该名称与文件名一致
* prog：整个规则的总体的描述，prog在这里也只是个名字，没有什么特殊含义，该规则的含义是，若干个(expr NEWLINE)
* expr：描述prog中的expr表达式，它是一种递归的形式，表达式有4种情况：表达式之间的乘除、表达式之间的加减、INT、表达式可以使用括号
* NEWLINE：若干换行符
* INT：若干数字组成

因此，上面就是一个计算器的语法描述文件，该计算器只支持整数的四则运算，并且可以通过括号调整优先级。

#### 4.2 生成语法解析器

将上述语法文件保存为expr.g4，然后使用antlr4工具生成语法解析器：

```
antlr4 -Dlanguage=Python3 expr.g4
```

就会在当前目录下生成一些python程序和文件：

* exprLexer.py：词法分析
* exprListener.py：继承自ParseTreeListener的空类exprListener
* exprParser.py：语法分析

#### 4.3 基于SDK实现逻辑

基于上面生成的类，然后结合antlr4提供的api，就可以得到antlr4为我们生成的AST(抽象语法树)，相当于我们只使用antlr4为我们解析表达式，但是具体的计算逻辑是需要编写代码去遍历AST。antlr4提供了两种方式遍历AST，一种是listener，另一种是visitor，默认是listener。

例如，当给定表达式为1+2*3时，会生成如下的一棵AST树：

![antlr4 AST](https://github.com/luofengmacheng/cloud_native/blob/master/antlr/pics/introduction1.jpg)

``` python
# Listener.py
from grammer.exprListener import exprListener
from grammer.exprParser import exprParser

class Listener(exprListener):
    def __init__(self):
        self.result = {}

    # Enter a parse tree produced by exprParser#prog.
    def enterProg(self, ctx:exprParser.ProgContext):
        pass

    # Exit a parse tree produced by exprParser#prog.
    def exitProg(self, ctx:exprParser.ProgContext):
        pass

    # Enter a parse tree produced by exprParser#expr.
    def enterExpr(self, ctx:exprParser.ExprContext):
        pass

    # Exit a parse tree produced by exprParser#expr.
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

继承exprListener创建我们自己的Listener，需要基于该Listener类遍历生成的AST，在这里只修改了exitExpr函数，从字面意思理解，该函数就是在遍历AST时离开某个节点时执行的函数，此时可以根据当前节点的孩子的个数执行不同的计算逻辑。

``` python
from antlr4 import CommonTokenStream
from antlr4 import ParseTreeWalker
from antlr4.InputStream import InputStream
from antlr4.Token import CommonToken

from grammer.exprParser import exprParser
from grammer.exprLexer import exprLexer
from Listener import Listener

if __name__ == '__main__':
    input_stream = InputStream("1+2*3\n")
    lexer = exprLexer(input_stream)
    token_stream = CommonTokenStream(lexer)

    parser = exprParser(token_stream)

    tree = parser.prog()
    listener = Listener()
    walker = ParseTreeWalker()
    walker.walk(listener, tree)

    print(listener.result)
```

### 5 总结

在实现一种语言或者规则时，首先需要解析语言或者规则，然后再对其中的单词或者语句进行处理，因此，在实际开发过程中，需要对输入进行分割然后再分析语义，而通过antlr4，可以自定义语言或者规则的构成，然后就可以通过antlr4的库得到一个AST的树，再利用antlr4的api遍历该树实现其他的业务逻辑，因此，基于antlr4可以简化我们的程序，帮助实现词法和语法的分析。

