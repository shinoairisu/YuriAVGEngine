﻿## 文法设计

### 上下文有关的Yuriri文法
本小节将展开介绍Yuri Engine的脚本语言**Yuriri**的文法设计。<br/>
Yuri AVG Engine采用**Yuriri**脚本语言作为自己的编码手段，使用可视化开发环境**Halation**进行游戏开发的本质就是将可视化布局信息翻译为脚本语言并交由后端虚拟机执行演绎的过程。为此，引擎设计了Yuriri Script脚本语言（下称**Yuriri**脚本）作为引擎的输入。<br/>
Yuriri脚本是一种**上下文有关文法**，因为对于Yuriri的形式文法 **Gy = (N, Σ, P, S)**，存在候选式：

> **α**A**β** → **αγβ**

即它存在一些语法结构要求在上下文中必须配对出现，以避免**二义性**情况。而对于绝大多数的AVG游戏a动作命令则以行为单位进行编码，命令行的基本格式如下：

> LineItem → @Action ParameterName = ParameterValueExpression, ...<br/>
                      | # Notation<br/>
                      | Dialog<br/>

符号“**@**”表示当前行是**可执行命令**，**Action**是命令名称，**ParameterName**是命令参数的名字，**ParameterValueExpression**是要赋值给等号左侧参数的表达式，省略号表示一个命令既可以没有**<参数, 值>**对，也可以有多个**<参数, 值>**对。注意到，一个命令如果带有多个参数时，参数是没有先后顺序要求的；而符号“**#**”表明当前行是注释，编译器在做语法分析时将略过它；推导符号**Dialog**代表在游戏执行过程中要显示的文本，这是AVG游戏使用频率最高的命令，由于文本的显示存在跨行的情况，因此它以一种上下文有关文法来表示，其文法如：

> Dialog  →  [<br/>
                     DialogContext<br/>
                     ]<br/>

显示文本的文法以符号**[**和**换行**开始，以换行和符号]结束，中间所包含的**DialogContext**即为要显示的对话字符串，可以跨行分布。Yuriri脚本的动作文法规则将在下一节中详述。<br/>

### 表达式的LL(1)上下文无关推导
Yuriri脚本语言和它的运行时环境类似于C-Machine，是非上下文无关的，它不能够简单地使用LL(1)递归下降分析法来构造语法树。<br/>
在这里提出一种解决方案：使用非LL(1)的预测推导队列和LL(1)递归下降两种方式相互结合的语法分析算法：对输入的Token流，如果是命令Token流，则跳过符号**@**的Token，看第二个Token，即命令的类型，随后按顺序为后面的参数Token构造一个待推导项目并加入待推导队列，待推导项目包含了参数名和参数等号后的表达式Token流。其中表达式的文法可以被收缩为**上下文无关文法**，即对表达式文法做**左递归替换**和**二义性消除**后，此时可以使用**LL(1)递归下降算法**对表达式构造AST抽象语法树。很容易注意到，这个行为是可并行的。<br/>
需要注意的第一点是，对于左递归的情况，考虑直接左递归的产生式格式为：

>	A → A**α** | **β**

可以通过将式(2.1)等价转换为式(2.2)、式(2.3)两个产生式来消除左递归： 

> ①   A → **β**A’<br/>
②   A’ → **α**A’ | **ε**

第二点，二义性的消除。当一条产生式的左边看到下一个Token后发现有不止一条候选式，那么这个文法是有二义性的，也就是说，对于一个具有二义性的文法的产生式A → **α** | **β**，存在终结符号**a**使得**α**和**β**都能够推导出以**a**开头的串。通过消二义性规则，将不需要的语法分析树抛弃，只为每个句子留下**一**棵语法分析树。<br/>
继续考虑表达式文法**Gexpr**的LL(1)递归下降推导过程。在自顶向下语法分析过程中，**FIRST**集与**FOLLOW**集的引入使得编译时可以根据向前看的下一个Token来选择应用哪个产生式。FIRST(**α**)被定义为可从α推导得到的串的第一个符号的集合，其中**α**是任意的文法符号串；FOLLOW(A)被定义为可能在某些句型中紧跟在A右边的终结符号的集合。利用FIRST集与FOLLOW集去计算出**SELECT**集， SELECT集就是需要构造的LL(1)预测分析表M，其原理和构造方法为：

- ① 对于FIRST(**α**) 中的每个终结符号**a**，将A→**α**加入到M[A, **a**]中。
- ② 如果ε在FIRST(α)中，那么对FOLLOW(A)中的每个终结符号**b**，将A→**α**加入到M[A, **b**]中。如果**ε**在FIRST(**α**)中，且语句终止符号**#**在FOLLOW(A) 中，也将A→**α**加入到M[A, **#**]中。
- ③ 在完成上面的步骤之后，如果M[A, **a**]中没有产生式，那么将M[A, **a**]设置为**语法错误**，用一个空条目表示。

对于文法**Gexpr**做递归下降的过程，就是不断地取匹配栈顶的符号，在Token流中向前看一个Token的类型后，查预测分析表M获得唯一的产生式来处理语法树当前节点，如果当前节点为非终结符，那么弹出匹配栈顶，展开当前节点，**自左向右插入子节点，自右向左将子节点类型压入匹配栈**，这就是下降的过程；如果当前节点为终结符，则弹出匹配栈的栈顶元素，但无需展开节点。最后，还需确定下一个展开节点。如果当前节点有**亲妹妹**节点，则选择**最大的亲妹妹**，即排位在自己之后的离自己最近的那个节点为后继；如果没有亲妹妹节点，则选择**母亲**节点的**最大妹妹节点**，即母亲同辈份在母亲节点之后的排位最大节点，使用这个规则**递归**地下降生长语法分析树。递归终点是，当前节点指针回到语法树根节点，此时语法树生成完毕，匹配工作结束，将这棵语法树的根节点作为这一脚本行的语法树节点的参数字典中对应项目的键值，供后续语义分析继续处理。

### 表达式的LL(1)文法
``` markdown
disjunct    →  conjunct disjunctPi
disjunctPi  →  || conjunct disjunctPi
               |ε
conjunct    →  bool conjunctPi
conjunctPi  →  && bool conjunctPi
               |ε
bool        →  ! bool 
               | comp
comp        →  wexpr rop wexpr
rop         →  ==
               | <>
               | >
               | <
               | >=
               | <=
               | ε
wexpr       →  wmulti wexprPi
               | ε
wexprPi     →  wplus wexprPi
               | ε
wplus       →  + wmulti
               | - wmulti
wmulti      →  wmulti wmultiOpt
wmultiOpt   →  * wunit wmultiOpt
               | / wunit wmultiOpt
               | ε
wunit       →   number
               | id
               | - wunit
               | + wunit
               | (disjunct)
```