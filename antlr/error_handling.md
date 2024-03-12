## ANTLR4规则解析生成器(五)：错误处理

### 1 背景

当使用基于ANTLR4开发的程序处理用户的输入时，可能用户的输入不符合预期的，此时，用户需要知道哪里出现了问题，以便进行修改，因此，在开发基于ANTLR4的程序时，需要将错误也考虑进去，尽量输出用户可以看得明白的错误信息。

### 2 ErrorListener

语法解析器的类派生关系是：`UserParser`<-`antlr4.Parser`<-`antlr4.Recognizer`<-`object`，因此，用户的Parser也可以调用Recognizer中的方法。

当语法解析器在分析输入串出现错误时，会调用`antlr4.Recognizer`中的`getErrorListenerDispatch()`函数将错误信息分发出去，该函数返回`ProxyErrorListener`对象：

``` python
class ProxyErrorListener(ErrorListener):

    def __init__(self, delegates):
        super().__init__()
        if delegates is None:
            raise ReferenceError("delegates")
        self.delegates = delegates

    def syntaxError(self, recognizer, offendingSymbol, line, column, msg, e):
        for delegate in self.delegates:
            delegate.syntaxError(recognizer, offendingSymbol, line, column, msg, e)

    def reportAmbiguity(self, recognizer, dfa, startIndex, stopIndex, exact, ambigAlts, configs):
        for delegate in self.delegates:
            delegate.reportAmbiguity(recognizer, dfa, startIndex, stopIndex, exact, ambigAlts, configs)

    def reportAttemptingFullContext(self, recognizer, dfa, startIndex, stopIndex, conflictingAlts, configs):
        for delegate in self.delegates:
            delegate.reportAttemptingFullContext(recognizer, dfa, startIndex, stopIndex, conflictingAlts, configs)

    def reportContextSensitivity(self, recognizer, dfa, startIndex, stopIndex, prediction, configs):
        for delegate in self.delegates:
            delegate.reportContextSensitivity(recognizer, dfa, startIndex, stopIndex, prediction, configs)
```

`ProxyErrorListener`有4个方法，分别对应不同的错误类型：

* syntaxError：语法错误
* reportAmbiguity：歧义上报
* reportAttemptingFullContext：当从SLL模式切换到LL模式时上报
* reportContextSensitivity：与reportAttemptingFullContext类似，解析过程中需要完整的上下文才能继续解析

通常来说，下面的三个report函数是在开发过程中，由于规则不完善导致的，而syntaxError则是输入串出现不符合规则时触发的错误。

可以看到，`ProxyErrorListener`的`syntaxError()`方法只是遍历构造函数中的对象，调用每个对象的syntaxError()方法，构造函数中的对象来自于`Recognizer._listeners`，`Recognizer._listeners`是个数组，默认的成员是`ConsoleErrorListener.INSTANCE`：`ConsoleErrorListener.INSTANCE = ConsoleErrorListener()`，因此，默认情况下，调用的就是`ConsoleErrorListener`的syntaxError方法，也就是输出一行错误信息到标准出错：

``` python
class ConsoleErrorListener(ErrorListener):
    INSTANCE = None

    def syntaxError(self, recognizer, offendingSymbol, line, column, msg, e):
        print("line " + str(line) + ":" + str(column) + " " + msg, file=sys.stderr)

ConsoleErrorListener.INSTANCE = ConsoleErrorListener()
```

因此，对于默认情况下，如果输入串不符合规则，会打印类似`line 1:10 missing ')' at '\n'`的错误信息。

那么，有办法自定义错误信息吗？或者在错误信息中加入更多的上下问题吗？方法就是替换掉默认的`ErrorListener`。

首先定义一个我们自己的`ErrorListener`：

``` python
class exprException(ErrorListener):
    def syntaxError(
        self,
        recognizer: Recognizer,
        offendingSymbol: CommonToken,
        line: int,
        column: int,
        msg: str,
        e: RecognitionException,
    ):
        print("Something is wrong at {}(line):{}(column), errmsg={}".format(line, column, msg))
```

该ErrorListener只有一个syntaxError方法，该方法可以接收antlr4传过来的一些错误数据让我们自由拼接：

* recognizer：识别器，当前的解析器示例，其实就是我们的Parser对象
* offendingSymbol：触发错误的符号，包含错误的文本和位置
* line：错误的行号
* column：错误的列号
* msg：错误信息
* e：异常对象

不过根据上面这些字段看，最重要的也还是三个：line、column、msg，这三个就可以告诉用户，在哪里发生了什么样的错误，因此，上面的自定义ErrorListener还是基于这三个字段生成了错误信息，也可以在这里抛出异常，然后在调用解析器处理输入串时处理该异常。

当创建出我们自己的ErrorListener后，就需要将这个ErrorListener放到`Recognizer._listeners`数组中：

``` python
parser = exprParser(token_stream)
parser.removeErrorListeners()
parser.addErrorListener(exprException())
```

先调用`Recognizer.removeErrorListeners()`清空`_listeners`数组，然后将我们自己的ErrorListener放到`_listeners`数组中。

注意：虽然名称是`ErrorListener`，但是，在Listener和Visitor中都可以使用。

### 3 总结

antlr4在进行语法分析时，如果发现有不符合语法规则的语句时，默认情况下是会打印一行错误信息，通过创建基于`ErrorListener`的类，可以对错误信息自定义，使得错误信息的可读性更好。
