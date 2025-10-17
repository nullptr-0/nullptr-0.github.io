---
layout: post
title: "Clangd中的SelectionTree：概述和实现"
date:   2025-07-16
tags: [Compiler,LSP,Clangd,C++]
comments: true
author: nullptr-0
---

## **SelectionTree在Clangd的Tweak系统中的作用**

Clangd的**SelectionTree**是一个实用工具，用于将编辑器选择（或光标位置）映射到相应的AST节点。它被clangd的*Tweak*（code action）系统大量使用。当请求code action时，clangd会为请求的范围构建一个SelectionTree，并通过`Tweak::Selection`输入提供它。这允许每个 Tweak 快速确定光标或选择下的 AST 结构，而无需编写自己的 AST 遍历<sup>1,2</sup> 。在实践中，Tweak的`prepare()`方法将检查SelectionTree（通常通过`commonAncestor()`或通过导航父/子节点）以确定该作是否适用。例如，*MemberwiseConstructor* 调整通过获取公共祖先并查看它是否为`CXXRecordDecl`来检查所选内容是否在C++类定义内<sup>3</sup>。同样，*ExpandDeducedType*调整在选择下查找AutoTypeLoc<sup>4</sup>。这种设计将“what's under the cursor？”逻辑集中在SelectionTree中，因此单个Tweak（以及其他功能，如go-to-definition或hover）不需要自定义 AST 搜索<sup>1</sup>。

**不明确的光标位置：**LSP选择被指定为字符之间的半开字节范围，这在边界处可能是模糊的（例如，两个标记之间的光标在逻辑上可能属于左侧或右侧的标记）。SelectionTree通过在需要时考虑这两种可能性来处理此问题。在clangd的API中，`SelectionTree::createEach()`将为这种模棱两可的情况下为每个合理的解释生成一棵树（一个选择左侧的令牌，一个选择右侧的令牌）<sup>5</sup>。通常，clangd默认使用`SelectionTree::createRight()`（首选右侧标记）来解决歧义<sup>6</sup>。对于文件开始/结束和空格周围的空选择，还有特殊情况的逻辑，以选择最合理的标记<sup>7</sup>。

## **SelectionTree数据结构和节点**

一个**SelectionTree**表示修剪后的AST仅覆盖与所选内容相关的节点。在内部，它拥有一组 **Node**对象（存储在`std::deque`用于指针稳定性<sup>8</sup>）。每个`SelectionTree::Node`包含：

- 指向选择树（selection tree）中其**Parent**的指针（或对根，通常是`TranslationUnitDecl`，而言是 null）<sup>9</sup>。
- 位于选择子树（selection subtree）内的**Children**（指向 Node 的指针）的列表<sup>10</sup>。
- **DynTypedNode ASTNode**，其中包含实际的AST节点（可以是`Decl`、`Stmt`、`TypeLoc`、`NestedNameSpecifierLoc`或`CXXCtorInitializer`）<sup>11</sup>。在SelectionTree中仅表示这些AST节点类型，因为它们具有明确定义的源范围，并且`DynTypedNode`支持<sup>12</sup>。
- **Selected**标志/状态，指示节点的源范围与所选内容的关系：**Unselected**、**Partial**或**Complete**<sup>13,14</sup>。

**选定状态语义：**

- **Unselected**表示节点本身对选定范围没有贡献任何字符（任何重叠仅通过其子项）<sup>15</sup>。例如，如果选择仅涵盖if语句中的条件表达式，则`IfStmt`节点将被标记为Unselected，因为所有选定的字符都属于其子节点（条件表达式），而不是`if`关键字或`IfStmt`<sup>15</sup>的其他部分。（根据设计，在源中“不可见”的AST节点（如隐式强制转换）始终处于Unselected状态<sup>16</sup>。）

- **Partial**表示节点自身的范围与所选内容重叠，但未完全被所选内容覆盖<sup>14</sup>。换句话说，此AST节点的部分（但不是全部）字符被选中。例如，仅选择函数调用表达式的名称（而不是整个调用）会将调用的AST节点标记为Partial。

- **Complete**表示选择完全覆盖节点的源范围（可能完全覆盖，甚至超出该范围）<sup>17</sup>。例如，如果整个变量声明`int x = 42;`正好高亮显示， 则`VarDecl`节点可能为Complete。

这些状态让clangd启发式地决定用户想要什么 “东西”。通常，选择树的**commonAncestor()**是起点。它返回包含所有选定节点的最低节点，并且本身是非平凡的（选定节点或具有多个选定子节点）<sup>18</sup>。实际上，`commonAncestor()`以合理的方式给出了“**under the cursor**”的AST节点<sup>19,20</sup>。Code action通常使用它来获取感兴趣的主节点。

值得注意的是，SelectionTree*排除*了与所选内容无关的AST节点。该树**仅**包含至少部分覆盖的节点或用作覆盖节点<sup>21</sup>的父节点的节点。这将产生一个紧凑的树，而不是整个AST。使用deque来存储节点可以保证在添加新节点时指向Node对象的指针保持有效<sup>8</sup> – 这很重要，因为父指针和子指针都指向此容器。

SelectionTree拥有其Node实例，但它们包装的AST节点（在`ASTNode`中）*不会*被复制 - 它们指向原始`ASTContext`的AST。这意味着SelectionTree是一个轻量级视图，不会改变或复制真实的AST<sup>22</sup>。

## **构建SelectionTree：实现细节**

SelectionTree是通过遍历*主文件*的AST并选择相关节点来构建的。Clangd通过Selection.cpp中的自定义递归AST访问者（`SelectionVisitor`）实现此目的，该访问者遍历AST并决定要包含哪些节点： 

- **遍历和修剪**：访问者修剪完全位于选择范围之外的子树以提高效率。在访问节点的子项之前，它会根据选择边界检查节点的源范围。如果节点的范围完全在选择之前或之后，它可以跳过该节点及其内部的所有内容<sup>23</sup>。此检查使用SourceManager来比较字节偏移量：例如`B.second >= SelEnd || E.second < SelBeginTokenStart`表示“节点在选择结束之后开始，或在选择开始之前结束”，因此可以跳过<sup>23</sup>。（如果选择在令牌中间开始，这里`SelBeginTokenStart`将是一个对齐到令牌的开头后的选择开始位置<sup>24</sup>。）此优化（`canSafelySkipNode`）避免遍历大型不相关的AST区域<sup>25,26</sup>。
- **压入节点**：当遇到*可能*与所选内容重叠的节点时，访问者会将新的SelectionTree节点“压入”到表示当前路径的堆栈上。这涉及在deque中创建一个Node并使用AST节点（`DynTypedNode`）初始化它，并将其链接到堆栈上的父节点<sup>27</sup>。例如，在进入未跳过的Stmt或Decl时，它们会执行以下操作：
```cpp
Nodes.emplace_back();
Nodes.back().ASTNode = DynTypedNode::create(*X);
Nodes.back().Parent = Stack.top();
Nodes.back().Selected = SelectionTree::Unselected;
Stack.push(&Nodes.back());
```
<sup>27</sup>。最初，堆栈使用虚拟根节点为`TranslationUnitDecl`作为基础<sup>28</sup>。

- **访问子项**：访问者继续进入子项（例如，对于`Decl`，`TraverseDecl`在传递给`traverseNode`辅助程序的lambda中调用`Base::TraverseDecl`<sup>29</sup>）。对于语句，它使用`RecursiveASTVisitor`的数据递归优化：一个`dataTraverseStmtPre()`压入节点，一个匹配的`dataTraverseStmtPost()`弹出它<sup>25,30</sup>。只有某些节点类型会生成SelectionTree节点 —— 主要是`Decls`、`Stmts`、`TypeLocs`、`NestedNameSpecifierLocs`和`CXXCtorInitializer` – 如文档中所列<sup>21</sup>。其他AST构造（例如没有`loc`的`NestedNameSpecifier`、`QualType`等的）被忽略，或遍历但不会添加为节点<sup>31</sup>。
- **弹出节点并确定选择状态**：随着遍历的展开，访问者会将节点从堆栈中“弹出”并最终确定其选择状态。这发生在`pop()`例程中访问节点的子项之后<sup>32</sup>。在弹出时，clangd通过检查节点的源范围如何与所选内容相交以及子项是否与所选内容有关来计算节点的Selected状态：
  - 它获取节点的源范围（针对宏进行调整 – 使用`SM.getTopMacroCallerLoc`，以便如果选择位于宏参数中，则该范围将映射到主文件中的调用点）<sup>33</sup>。如果节点的范围不在主文件中，则该节点将被视为Unselected<sup>34</sup>。然后，它会检查与选择项是否有任何重叠：如果节点的开始在选择之后，或者它的结束在选择开始之前，则为Unselected（无重叠）<sup>35</sup>。
  - 如果存在重叠，则它会检查*整个重叠部分*是否被子节点覆盖。这将使用辅助程序`nodesCoverRange(children,  Begin, End)`来查看所选子项范围的并集是否覆盖父项内的选择范围<sup>36</sup>。例如，如果`IfStmt`下的所有选定字符都属于其子表达式和正文节点，则`IfStmt`本身没有“直接”选择。在这种情况下，节点保持Unselected（意味着选择“通过”了它，但不包括该节点本身的任何字符）<sup>36</sup>。如果子节点没有覆盖整个重叠，则意味着此节点为选择贡献了一些字符（例如，节点的关键字或标点符号可能在高亮显示的范围内）。那么，该节点被视为*已选中*。
  - 最后，它区分**Partial和Complete**：如果节点的源范围完全位于选择范围内，则标记为Complete;否则为Partial<sup>37</sup>。在代码中：`return (B.second >= SelBegin & E.second <= SelEnd) ? SelectionTree::Complete : SelectionTree::Partial;`<sup>37</sup>。因此，如果选择从头到尾覆盖节点（即使选择超出节点），则为Complete。如果所选内容仅覆盖节点的一部分，则为 Partial。

`computeSelection()`函数封装了这个逻辑<sup>38,35</sup>。每个节点在弹出时都会调用它，因此它针对常见情况进行了优化（无重叠时快速退出）。值得注意的是，它扩展了节点的结束位置以覆盖完整令牌（使用 Lexer 测量令牌长度或查找下一个令牌），以便在文件偏移量中的半开范围[begin， end]上进行比较<sup>39,40</sup>。 

- **附加或丢弃节点**：计算选择状态后，`pop()`将节点附加到树或丢弃它<sup>41</sup>。如果节点被**选中**（Partial/Complete）**或**它有任何最终被选中的子节点，则它应保留在SelectionTree中。在这种情况下，它会将此节点附加到其父节点的Children列表<sup>42</sup>中。相反，如果节点本身是Unselected*并且*没有保留任何子项（意味着这个子树与选择无关），那么节点会被弹出并立即丢弃：`Nodes.pop_back()`（这是安全的，因为它是最后一个元素）<sup>43</sup>。这确保了最终的树不包含伪节点：SelectionTree中的每个节点要么直接被选择覆盖，要么是某个东西的祖先。

例如，考虑在代码中选择函数调用`foo()`的名称。`foo()`的`CallExpr`节点可能标记为Partial（因为选择涵盖`foo`但不包括括号），并且`foo`名称的子`DeclRefExpr`将为Complete（选择完全覆盖标识符）。除了子令牌覆盖的字符外，这个`CallExpr`没有选择自己的字符，因此在计算后，它仍然是Partial（因为`(`或`)`可能没有被完全覆盖）。两个节点都保留在树中：`DeclRefExpr`作为`CallExpr`的子节点。如果改为选择整个调用`foo();`，则`CallExpr`最终将变为Complete（选择涵盖其所有标记）<sup>37</sup>，并且父语句节点也可能变为Complete，依此类推。

- **对宏的考虑**：SelectionTree尝试对宏进行合理操作，尽管它没有显式地对宏扩展图进行建模<sup>20</sup>。使用`getTopMacroCallerLoc`意味着，如果选择源自宏参数的文本，则SelectionTree会将该选择归类于宏调用中的AST节点（因此，获得的是与调用方中的参数扩展相对应的节点）<sup>33</sup>。它不会为每个展开的宏扩展事件生成单独的节点 - 只有该参数的第一个扩展被视为已选中<sup>44</sup>。如果一个节点完全来自不属于宏参数的宏扩展，它可能不会被计为选中（因为主文件范围检查将失败）<sup>34</sup>。这是一种有意识的简化，以避免构建一个分布在扩展中的选择的“森林”<sup>44</sup>。实际上，这意味着SelectionTree适用于宏参数或内联宏等内容，但可能会忽略仅存在于扩展宏正文中的选择。

构建后，客户端（如Tweak）可以查询SelectionTree。最常见的查询是`SelectionTree::commonAncestor()`，它返回所有选定节点（不包括手动加入的TU根）的最低公共祖先的Node<sup>18</sup>。对于典型的单范围选择，这实际上是完全包含选择（或具有多个不同的选定子项）的最小AST节点。调用方将其用作主要的“被选中的AST节点”。例如，许多调整可以：
```cpp
if (const SelectionTree::Node *N = Inputs.ASTSelection.commonAncestor()) {
  if (auto *D = N->ASTNode.get<SomeASTType>()) {...}
}
```
以获取光标下的特定AST节点类型。例如，`if (auto *N = Inputs.ASTSelection.commonAncestor()) Class = N->ASTNode.get<CXXRecordDecl>();`接着检查`Class`是否为定义，然后再继续<sup>3</sup>。

SelectionTree的Node还提供了导航或解释树的辅助程序：

- `Node::Parent`和子列表允许在选定的结构中向上或向下移动。
- `Node::getDeclContext()`给出节点的周围声明上下文（词法范围）。在内部，它会在父链中向上爬升，直到找到一个`Decl`或`Lambda`节点，并返回适当的`DeclContext`<sup>45,46</sup>。值得注意的是，对于lambda内的节点，它将返回lambda的调用运算符（闭包的函数调用运算符`operator()`），而不是外部函数或TU，以首选最内层的词法范围<sup>46</sup>。（这是最近clangd版本中的一项改进，以更好地处理lambda范围<sup>47,48</sup>。如果未找到合适的祖先（TU除外），则返回TU的`DeclContext`。这对于需要知道在何处插入新声明或在选择上下文中评估名称查找的调整非常有用。
- `Node::ignoreImplicit()`和`Node::outerImplicit()`处理隐式AST包装器，如隐式强制转换节点。`ignoreImplicit()`*向下*跳过隐式节点链，返回具有不同（更窄）源范围的第一个后代<sup>49</sup>。例如，如果所选内容是一个表达式，该表达式在AST中将`ImplicitCastExpr`作为其父级，则`ignoreImplicit()`将跳过强制转换并给出真正的子节点（这可以防止隐式转换迷惑只关心底层表达式的操作）。相反，`outerImplicit()`在与此节点具有相同源范围的任何父节点中*向上*移动（本质上是前面向下过程的逆过程）<sup>50</sup>。这些辅助程序确保go-to-definition或某些重构等功能可以忽略AST组件，并对用户认为的“selected expression”进行操作。例如，code action可能会调用`selection.commonAncestor()->ignoreImplicit()`来获取用户突出显示的真实节点，忽略任何编译器生成的没有直接源表示的AST节点。

最后，SelectionTree支持带缩进调试打印（`operator<<`）转储选定节点的树，用`*`标记完全选定的节点，用`.`标记部分选定的节点<sup>51,52</sup>。这对于日志记录或测试非常有用。实际上，有一些单元测试（SelectionTests.cpp）从带注释的代码构造SelectionTrees以验证它是否捕获了预期的节点。

## **Clangd代码库中的实现**

SelectionTree实现位于**clang-tools-extra/clangd/Selection.h**和**Selection.cpp**中。类及其静态生成器（`createEach`、`createRight`）在Selection.h<sup>53,6</sup>中声明，详细逻辑（`SelectionVisitor`、`Node`方法等）在Selection.cpp 中定义。Tweak系统在**Tweak.h/Tweak.cpp**中使用SelectionTree：`Tweak::Selection`结构包含一个`SelectionTree ASTSelection`成员<sup>54,55</sup>，该成员是根据`ParsedAST`和编辑器的选择范围构建的。例如，`Tweak::Selection`的构造函数调用`ASTSelection = SelectionTree(AST.getASTContext(), RangeBegin, RangeEnd)`来构建树<sup>56</sup>。然后，每个具体的Tweak（在clangd/refactor/tweaks/目录中）都会检查`Inputs.ASTSelection`，如前所述。

**实现要点：**

- *Selection.h*中的Clangd SelectionTree设计注释<sup>57,58</sup>
- SelectionTree节点定义和选择状态<sup>11,13</sup>
- *Selection.cpp*中的SelectionTree构造和跳过逻辑<sup>26,27</sup>
- 选择计算和子项覆盖率检查<sup>35,37</sup>
- `SelectionTree::commonAncestor()`实现<sup>59</sup>
- Tweak中的示例用法（`MemberwiseConstructor`）<sup>3</sup>
- 上下文节点和隐式节点的`Node`实用工具方法<sup>60,49</sup>

总之，**SelectionTree是clangd用来将编辑器的选定文本映射到C++ AST的机制**，并能够消除用户可能意图的歧义。它为简单的用例提供了一个高级节点（`commonAncestor`）（例如“光标处是什么符号？”，用于转到定义<sup>1</sup>），以及用于高级分析的丰富的AST节点子树（例如，可能需要考虑父节点或子节点的复杂code action）。它的内部架构 - 使用带有范围检查的集中AST遍历、稳定的节点容器以及对宏和隐式节点的仔细处理 - 确保这种映射高效且健壮。 


## 参考文献

[1] clangd code walkthrough
<https://clangd.llvm.org/design/code>

[2] [56] ⚙ D57570 [clangd] Expose SelectionTree to code tweaks, and use it for swap if branches.
<https://reviews.llvm.org/D57570>

[3] clang-tools: MemberwiseConstructor.cpp Source File
<https://clang.llvm.org/extra/doxygen/MemberwiseConstructor_8cpp_source.html>

[4] clang-tools: ExpandDeducedType.cpp Source File
<https://clang.llvm.org/extra/doxygen/ExpandDeducedType_8cpp_source.html>

[5] [6] [9] [10] [11] [12] [13] [14] [15] [16] [17] [18] [19] [20] [21] [22] [44] [53] [57] [58] clang-tools: Selection.h Source File
<https://clang.llvm.org/extra/doxygen/Selection_8h_source.html>

[7] [8] [23] [24] [25] [26] [27] [28] [29] [30] [31] [32] [33] [34] [35] [36] [37] [38] [39] [40] [41] [42] [43] [51] [52] [59] ⚙ D57562 [clangd] Lib to compute and represent selection under cursor.
<https://reviews.llvm.org/D57562>

[45] [46] [49] [50] [60] clang-tools: Selection.cpp Source File
<https://clang.llvm.org/extra/doxygen/Selection_8cpp_source.html>

[47] [48] clang-tools-extra/clangd/Selection.cpp · 690a30f3fd68a1a97bd3f8b13470abf9d34a61df · llvm-doe / llvm-project · GitLab
<https://code.ornl.gov/llvm-doe/llvm-project/-/blob/690a30f3fd68a1a97bd3f8b13470abf9d34a61df/clang-tools-extra/clangd/Selection.cpp>

[54] [55] clang-tools: clang::clangd::Tweak::Selection Struct Reference
<https://clang.llvm.org/extra/doxygen/structclang_1_1clangd_1_1Tweak_1_1Selection.html>
