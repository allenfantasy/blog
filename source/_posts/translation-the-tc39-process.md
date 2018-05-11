title: "[译文] The TC39 Process"
date: 2018-05-11
tags:
- JavaScript
- Translation
---

提到 ECMAScript 规范制定的流程，目前网络上可以查到的中文资料，绝大多数是翻译于 Axel Rauschmayer 的 [The TC39 process for ECMAScript features](http://2ality.com/2015/11/tc39-process.html)。

<!--more-->

* https://www.jianshu.com/p/b0877d1fc2a4
* https://www.zhihu.com/question/24715618/answer/115283215

文章中详细解释 TC39 Process 的一些细节（写于2015年），比较重要的一些点有：

* 在 Draft 阶段，需要满足有两个对该规范的实验性实现，其中一个可以是类似 Babel 的转译器。
* 在 Candidate 阶段，需要有两个和规范兼容的实现。

距离这篇文章成文时间已经过去了3年多，目前的流程规范已经有了新的变化。本文是对 [The TC39 Process](https://tc39.github.io/process-document/) 的全文翻译。以下为正文部分：

---

ECMA TC39 委员会负责发展 ECMAScript 编程语言及编写语言规范。该委员会实行协商一致的原则，并在其觉得合适时酌情对规范进行修改。以下是对规范进行修改的通用流程：

### 开发

（TC39 委员会）通过一个流程来推动对语言的修改，该流程提供了如何（给规范）发展新增特性的指南：从一个点子到完整规范的，且同时具有可被接受的测试和多个具体实现的语言特性。（流程）有5个阶段：一个稻草人（strawman）阶段，和4个“成熟”阶段。TC39 委员会必须对（新特性的）每个阶段都通过验收。

 |**阶段**|**目的**|**准入标准**|**验收意义**|**规范的质量**|**验收后期望的修改**|**期望的实现方案***
 ---|---|---|---|---|---|---|---
 0|<div style="width:70px;">Strawman</div>|允许进入规范|无|N/A|N/A|N/A|N/A|
 1|Proposal|<ul style="width:120px"><li>提出这个新增特性的充分理由</li><li>描述解决方案的样子</li><li>确定潜在的挑战</li></ul>|<ul style="width:180px"><li>确认将会推动该新增特性的"主要负责人"人选（译注：可以理解为主要负责人）</li><li>简要概括问题或需求，以及一个大体上的解决方案形式</li><li>说明性的使用例子</li><li>高级 API</li><li>关于关键算法，抽象形式和语法的讨论</li><li>确认可能的 "cross-cutting" concerns[2] 和在实现中的挑战和复杂性</li></ul>|<div style="width:100px">委员会期望投入时间来检验问题空间，解决方案和 cross-cutting concerns</div>|<div style="width:80px;">无</div>|<div style="width:80px;">大幅度的修改</div>|<div style="width:105px;">Polyfill / demo</div>
2|Draft|使用标准的规范语言精确表达（该特性）的语法和语义|<ul><li>上述标准</li><li>初步的规范文本描述</li></ul>|委员会期望该特性进一步发展且最终被标准所包含|草稿: 包含了所有主要语义、语法和 API，同时期望有 TODO、预留位置（placeholders）和编辑方面事务（editorial issues）|增量修改|实验性的方案（Experimental）|
3|Candidate|表示：进一步的完善需要具体实现方案（译者注：指实现了该特性的库）和用户的反馈|<ul><li>上述标准</li><li>完整的规范文本描述</li><li>指定的审核人在当前的规范文本上签字通过</li><li>ECMAScript 编辑在当前规范文本上签字通过</li></ul>|当前这个解决方案是完整的，在没有具体实践经验、大规模使用和外部反馈之前，不需要额外的工作。|完整的：所有的语义、语法和 API 都有完整的描述|有限的：仅限于那些基于实践经验，被视作是关键修改的内容|和规范相符合（的具体实现）|
4|Finished|表示：该新增特性已经准备好，可以加入到正式的 ECMAScript 标准|<ul><li>上述标准</li><li>（关于该特性的）主流使用场景的验收测试代码已经编码完成并合并到 <a href="https://github.com/tc39/test262">Test262</a></li> 中<li>存在两个兼容规范，且通过验收测试的实现（库或工具）</li><li>显著有效地在实际开发中应用（关于该特性的）实现方案，比如应用两个不同VM（译者注：可能是浏览器和Node.js）提供的API</li><li>已向 <a href="https://github.com/tc39/ecma262">tc39/ecma262</a> 提交了一个带有完整规范文本的 pull request</li><li>ECMAScript 编辑已经在 pull request 上签字通过</li>|该特性将在最新一次标准修改时被包含进去|最终版：已经集成了所有根据实践经验反馈的结果|无|正在发布

\* 该列并不是必需的，只单纯是一个大体上的期望。

### 进入流程的输入

对 ECMAScript 编程语言的发展的意见以任何形式被接受。任何关于规范的修改或增补的讨论、意见提出或提案，在没有作为正式提案提交之前都被认为是 "strawman" (阶段0），不存在任何的验收要求。这样的（意见）提交必须要么来自 TC39 委员会成员提出，或者是来自 ECMA 国际注册的非委员会成员。

### 规范修订和行程安排

TC39 将在每年7月向 ECMA General Assembly[1] 提交一份（ECMAScript 的）规范以争取通过。以下是制定一份新的规范修订方案的大约时间线：

* 2月1日：完成 Candidate Draft
* 2月-3月：60 day royalty-free opt-out period.
* 3月份TC39会议：stage 4 的提案将被合并（到规范中），最终的语义将被通过，新的规范版本将会从 master 上切出一个分支。从这时候开始，只有校对文字的修改才会被接受。
* 4月-6月：ECMA CC 和 ECMA GA 复审阶段。
* 7月：ECMA General Assembly 将通过新的标准。

### 进行中的新增特性的状态

TC39 将在其 [Github 上](https://github.com/tc39/ecma262) 维护一个进行中的新增特性的列表，包括每项特性当前的成熟阶段。

### 规范文书

在 "draft" 阶段（第2阶段）及之后的阶段，新增特性的语义，API 和语法必须被描述成对最新发布的 ECMAScript 标准的编辑修改，使用同样的语言和约定。每个阶段中对规范文书质量的期望要求在前文中已有描述。
cribed above.

### 审核者

任何人都可以是审核者并就一个进行中的新增特性提交反馈。委员会应当在 "draft" 阶段（第2阶段）就确定指定的审核者进行验收工作。这些审核者必须在提案进入 "candidate" 阶段（第3阶段）之前签字同意（该方案）。指定的审核者不可以是新增特性的规范文书的作者，但同时要具备对（该特性）涉及的内容相关的专业知识。指定的审核者必须是由委员会选出，不能由该提案的主要负责人选出。

当审核者被指定后，需要确定一个以第3阶段为目标的会议。在会议开始的两周前，审核者需要将初步意见反馈给提案的主要负责人，以保证可以在会议前有一个来回（的讨论）。如果还没有准备好的话，主要负责人可以将第3阶段目标会议延迟到稍后的一个时间。

### 呼吁具体实现和反馈

当一个新增特性被通过到 "candidate"（第3阶段）级别时，意味着委员会认为（对规范的）设计工作已经完成了，需要具体实现、大量的使用和外部反馈才能有进一步的完善。

### Test262 测试

在第三阶段中，将编写 test262 测试并提交 pull request。一旦测试被恰当地审核过，就应该被合并（到主干代码中）以帮助实现规范的开发者在这一阶段提供反馈。（译者注：提供测试的好处时，实现规范的开发者，如Node或者浏览器的开发者可以利用测试来检查其实现是否遵循了提案中的规范细节）

### 忽略过程

在合适情况下，委员会在经过考虑后，可能会基于变化的范畴，（在处理某个新特性提案时）忽略上述的流程。

### 编辑的角色

对于在进行中的新增特性，很可能有一些规范文本是由（新增特性的提案的）一个主要负责人或者是一个委员会成员编写的，而不是由编辑编写；尽管有时候编辑也可能是某个具体新增功能的主要负责人，需要为其负责。编辑需要负责 ECMAScript 规范的整体结构和连贯性。编辑同时也扮演（向规范作者）提供指导和反馈的角色，以保证一个新增特性的规范在逐渐发展成熟的过程中，其规范文书的质量和完整度也在提升。编辑还需要将通过的 "完成的"（第4阶段）的新增特性规范，整合到整个规范的新修订版本中。

(全文完)

---

### Note

* [1] ECMA General Assembly：ECMA 里最权威的组织，相当于整个协会的控制核心。
* [2] [cross-cutting concern](https://en.wikipedia.org/wiki/Cross-cutting_concern)： AOP 中的一个概念, 比较抽象. [SO 上的这个解答](https://stackoverflow.com/questions/23700540/cross-cutting-concern-example?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa) 比较生动一些。

### Reference

* [The TC39 Process](https://tc39.github.io/process-document/)
* [The TC39 process for ECMAScript features](http://2ality.com/2015/11/tc39-process.html)
* [\[ECMAScript\] TC39 process](https://www.jianshu.com/p/b0877d1fc2a4)
* [ECMAScript 6 会重蹈 ECMAScript 4 的覆辙吗？](https://www.zhihu.com/question/24715618/answer/115283215)

### 彩蛋

贺老的一个知乎 Live：[如何学习和实践 ES201X?](https://www.zhihu.com/lives/883307634416054272)