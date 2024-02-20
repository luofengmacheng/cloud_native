## ANTLR4规则解析生成器(二)：编写规则文件

### 1 编译原理回顾

#### 1.1 编译的过程

* 词法分析：解析源代码中的每个单词，输入是源代码，输出是词法单元，也就是解析出来的单词
* 语法分析：分析多个单词组成的短语，输入是词法单元，输出是抽象语法树
* 语义分析：对抽象语法树进行类型检查和符号表填充
* 生成中间代码：生成接近机器语言的中间代码
* 代码优化：对中间代码进行优化，包括消除冗余等
* 生成目标程序：生成机器语言的代码，对于C语言也就是.o文件

#### 1.2 文法

文法的概念：自然语言或者编程语言都是由各种词语和句法构成，文法可以用来描述它们的结构，。形式文法通常由一组生成式组成，这些生成式定义了如何从一个起始符号通过一系列替换步骤生成语言中的所有合法字符串。

文法的类型：

* 0型文法：无限制文法，左侧至少包含1个非终结符
* 1型文法：上下文有关文法，进行生成式替换时，需要考虑左侧非终结符的上下文，左侧的符号个数不能多于右侧的符号个数
* 2型文法：上下文无关文法，进行生成式替换时，不需要考虑左侧非终结符的上下文
* 3型文法：正规文法，右侧要么是终结符，要么是终结符跟非终结符的组合

以上4种文法是逐级包含的关系，0包含1，1包含2，2包含3，3包含4。

### 2 ANTLR4中的文法规则

ANTLR4中使用的文法规则是基于正则表达式的上下文无关文法，在编写规则时只需要`采用递归`的形式描述`当前规则如何扩展`。

在[grammars-v4](https://github.com/antlr/grammars-v4)中保存了很多语言或者中间件的g4文件，下面以[sqlite的g4文件](https://github.com/antlr/grammars-v4/tree/master/sql/sqlite)为例讲解ANTLR4中的规则。

sqlite的g4文件分成两个，一个是词法的g4文件(SQLiteLexer.g4)，里面定义了sqlite的sql语句中可用的一些标识符，一个是语法的g4文件(SQLiteParser.g4)，里面定义了sqlite的sql语句的语法结构。

```
// SQLiteLexer.g4
lexer grammar SQLiteLexer;

// 声明一些选项，这里表名在进行词法分析时不区分大小写
// 可用的选项可以在https://github.com/antlr/antlr4/blob/master/doc/options.md查看
options {
    caseInsensitive = true;
}

// 定义sql语句中的特殊字符，如括号和算符
SCOL      : ';';
DOT       : '.';
OPEN_PAR  : '(';
CLOSE_PAR : ')';
COMMA     : ',';
ASSIGN    : '=';
STAR      : '*';
PLUS      : '+';
MINUS     : '-';

// 定义关键字
ABORT_             : 'ABORT';
ACTION_            : 'ACTION';
ADD_               : 'ADD';
AFTER_             : 'AFTER';
ALL_               : 'ALL';
ALTER_             : 'ALTER';
ANALYZE_           : 'ANALYZE';
AND_               : 'AND';
AS_                : 'AS';
ASC_               : 'ASC';
ATTACH_            : 'ATTACH';
AUTOINCREMENT_     : 'AUTOINCREMENT';
BEFORE_            : 'BEFORE';
BEGIN_             : 'BEGIN';
BETWEEN_           : 'BETWEEN';
BY_                : 'BY';
CASCADE_           : 'CASCADE';
CASE_              : 'CASE';
CAST_              : 'CAST';
```

```
// SQLiteParser.g4
parser grammar SQLiteParser;

// 指定词法分析器名
options {
    tokenVocab = SQLiteLexer;
}

// parse是语法规则的起始
// 冒号右边表示合法的输入是若干sql语句
// 然后是输入流的结束符EOF，EOF是ANTLR4内置的特殊的符号，不需要显示定义
parse
    : (sql_stmt_list)* EOF
;

// sql表达式列表的定义
// 可以发现其中只有两个元素：sql_stmt和SCOL(;)
// 所以这条规则就是用于处理sql语句中的分号，能够让sql语句在一定程度上具有容错能力
sql_stmt_list
    : SCOL* sql_stmt (SCOL+ sql_stmt)* SCOL*
;

// 单条sql语句的定义
// ?在正则表达式中可以用于懒惰匹配，也就是0次或者1次匹配
// 也就是说?前面的内容有和没有都可以成功匹配
// 正确的sql语句有三种形式：
// 1 explain 常规的sql语句：分析sql语句的执行计划，输出表的读取顺序、使用的索引、预计查询的行数等
// 2 explain query plan 常规的sql语句：与explain相比提供了更高层次的查询策略描述
// 3 常规的sql语句：普通的查询语句，例如create、drop等
sql_stmt
    : (EXPLAIN_ (QUERY_ PLAN_)?)? (
        alter_table_stmt
        | analyze_stmt
        | attach_stmt
        | begin_stmt
        | commit_stmt
        | create_index_stmt
        | create_table_stmt
        | create_trigger_stmt
        | create_view_stmt
        | create_virtual_table_stmt
        | delete_stmt
        | delete_stmt_limited
        | detach_stmt
        | drop_stmt
        | insert_stmt
        | pragma_stmt
        | reindex_stmt
        | release_stmt
        | rollback_stmt
        | savepoint_stmt
        | select_stmt
        | update_stmt
        | update_stmt_limited
        | vacuum_stmt
    )
;

// create table语句的定义
// 1 语句以create table开头，并且table关键字前可以加TEMP或者TEMPORARY关键字，表示创建临时表，临时表是只在当前数据库连接存在时有效
// 2 在table关机键案子后面可以加if not exists，表示当表不存在时创建，如果表存在则直接结束而不打印错误
// 3 表名可以有两种形式，一种是直接用表名，例如，student，另一种是加上数据库的名称，例如，school.student，schema_name就是数据库名，table_name就是表名，中间以点号DOT分割，而schema_name和table_name的定义都是下面的any_name，含义基本就是可用的标识符
// 4 表可以通过两种形式创建，一种是直接定义表的字段，另一种是通过select子查询创建，这就对应了table_name后面的括号中的|两边的形式
// 5 表的字段放在小括号中，规则匹配的小括号是OPEN_PAR和CLOSE_PAR
// 6 表的字段由column_def定义，并且可以包含多个字段，然后加上表的约束
create_table_stmt
    : CREATE_ (TEMP_ | TEMPORARY_)? TABLE_ (IF_ NOT_ EXISTS_)? (schema_name DOT)? table_name (
        OPEN_PAR column_def (COMMA column_def)*? (COMMA table_constraint)* CLOSE_PAR (
            WITHOUT_ row_ROW_ID = IDENTIFIER
        )?
        | AS_ select_stmt
    )
;

// 表的列的定义
// 1 列名，也是下面的any_name
// 2 列的类型，如果不指定列的类型，则根据插入数据的类型自动推导出类的类型
// 3 列的约束
column_def
    : column_name type_name? column_constraint*
;

// 列的类型
// 会先有一个类型的名称，也就是这里的name，它也是个any_name
// 后面可能需要指定列的位数，例如，字符串的字符个数，CHAR(50)
type_name
    : name+? (
        OPEN_PAR signed_number CLOSE_PAR
        | OPEN_PAR signed_number COMMA signed_number CLOSE_PAR
    )?
;

// 类的约束
// 1 约束的名称，可能没有
// 2 primary key，指定列的主键
// 3 null/not null/unique
// 4 check，对列增加检查，例如，CHECK (age > 0)
// 5 default，设置列的默认值
// 6 collate，指定列的排序规则
// 7 foreign key，设置表的外键
// 8 generated always，创建生成列
column_constraint
    : (CONSTRAINT_ name)? (
        (PRIMARY_ KEY_ asc_desc? conflict_clause? AUTOINCREMENT_?)
        | (NOT_? NULL_ | UNIQUE_) conflict_clause?
        | CHECK_ OPEN_PAR expr CLOSE_PAR
        | DEFAULT_ (signed_number | literal_value | OPEN_PAR expr CLOSE_PAR)
        | COLLATE_ collation_name
        | foreign_key_clause
        | (GENERATED_ ALWAYS_)? AS_ OPEN_PAR expr CLOSE_PAR (STORED_ | VIRTUAL_)?
    )
;

// 表的约束
// 1 约束的名称，可能没有
// 2 primary key，指定表的主键列
// 3 check
// 4 foreign key
table_constraint
    : (CONSTRAINT_ name)? (
        (PRIMARY_ KEY_ | UNIQUE_) OPEN_PAR indexed_column (COMMA indexed_column)* CLOSE_PAR conflict_clause?
        | CHECK_ OPEN_PAR expr CLOSE_PAR
        | FOREIGN_ KEY_ OPEN_PAR column_name (COMMA column_name)* CLOSE_PAR foreign_key_clause
    )
;

// select语句的定义
// select语句应该是sqlite中比较复杂的，这里将它抽象为几个部分
// 1 select_core，select语句的核心结构，主要就是select/from/where/group等
// 2 compound_operator，集合运算符，对表的数据求交集、并集、差集
// 3 对集合运算的结果增加order by和limit
select_stmt
    : common_table_stmt? select_core (compound_operator select_core)* order_by_stmt? limit_stmt?
;

// select语句的核心定义
// 1 select distinct/all
// 2 result_column (COMMA result_column)*，指定查询的若干列
// 3 from，后面是表明或者子查询
// 4 where
// 5 group by 
// 6 window
select_core
    : (
        SELECT_ (DISTINCT_ | ALL_)? result_column (COMMA result_column)* (
            FROM_ (table_or_subquery (COMMA table_or_subquery)* | join_clause)
        )? (WHERE_ whereExpr = expr)? (
            GROUP_ BY_ groupByExpr += expr (COMMA groupByExpr += expr)* (
                HAVING_ havingExpr = expr
            )?
        )? (WINDOW_ window_name AS_ window_defn ( COMMA window_name AS_ window_defn)*)?
    )
    | values_clause
;

// 所有可用的字符串
any_name
    : IDENTIFIER
    | keyword
    | STRING_LITERAL
    | OPEN_PAR any_name CLOSE_PAR
;
```

可以看到，要描述一个完整的语言需要熟悉两部分的内容，一个是语言中允许出现的格式，另一个是正则表达式。

从以上sqlite中的sql可以看到使用正则表达式编写g4常见的一些套路：

* 尽量将单条规则拆分成多个可以独立描述的部分，每个部再分别编写规则
* 虽然表名和视图名都可以用相同的规则描述，例如any_name，但是为了表达其含义，尽量将不同含义的词法规则用不同的名称，并且名称可以代表含义，例如，table_name、view_name
* 关键字和特殊符号用大写
* 如果这部分在语句中属于可选的部分，可以使用`?`，例如`(DISTINCT_ | ALL_)?`表示在查询表的数据时可以加distinct关键字或者all关键字
* 如果这部分类似可变参数，可以使用`*`，例如`result_column (COMMA result_column)*`表示读取表的一列或者多列
* 用`(|)`表示对于多个正则表达式的选择，而`|`则表示文法规则的多个选择，也就是说如果是表示从多个文法规则中选择某个规则，则不需要加括号，如果是正则表达式中的多个语句的选择，则需要加括号

### 3 总结

本文简单回顾了编译原理中的一些名词解释，然后对sqlite中的sql的g4文件的部分内容进行了解析，大概可以了解到g4文件的编写方式，其实就是用正则表达式描述所有支持的文本结构。
