## 简述： 

一个很好用的go代码检查工具，可以帮你扫除nil导致panic的隐患。uber开发的，这个是uber官方技术文章的混元翻译人工微调版。原文链接：https://www.uber.com/en-HK/blog/nilaway-practical-nil-panic-detection-for-go/

---

由于Go语言的高性能，Uber广泛采用Go作为实现后端服务和库的主要编程语言。Go 大仓是Uber最大的代码库，包含9000万行代码（并且还在增长）。这使得编写可靠的Go代码的工具成为我们开发基础设施的关键部分。

指针（持有其他变量的内存地址而不是它们的实际值的变量）是Go编程语言不可或缺的一部分，有助于高效的内存管理和有效的数据操作。因此，程序员在编写Go程序时会广泛使用指针，用于各种目的，如就地修改数据、并发编程、轻松共享数据、优化内存使用以及促进接口和多态。尽管指针功能强大且广泛使用，但谨慎和明智地使用它们至关重要，以避免常见的陷阱，如零指针解引用导致的Nil Panic。

## Nil Panic问题

Nil Panic是程序尝试解引用零指针时发生的运行时Panic。当指针为零时，意味着它不指向任何有效的内存地址，尝试访问它指向的值将导致Panic（即运行时错误），错误消息如图1所示。

![avatar](https://blog.uber-cdn.com/cdn-cgi/image/width=1024,quality=80,onerror=redirect,format=auto/wp-content/uploads/2023/11/figure_1.jpg) 

Figure 1: Nil panic error message

图2显示了Go标准库实现中最近发现并解决的Nil Panic问题的示例，特别是其net包。在第1859行，由于直接调用RemoteAddr()方法返回值上的String()方法，假设它总是非零，导致了Panic，如图2所示。当接口类型net.Conn的字段c.rwc被分配了结构net.conn时，这是有问题的，因为如果连接c被发现不可用（如图3所示），RemoteAddr()的具体实现可以返回零值。具体来说，RemoteAddr()可以在L225返回零接口值，当在其上调用方法（这里是.String()）时，会导致Nil Panic，因为零值不包含指向任何可以调用的具体方法的指针。



![avatar](https://blog.uber-cdn.com/cdn-cgi/image/width=1024,quality=80,onerror=redirect,format=auto/wp-content/uploads/2023/11/figure_2.jpeg) 
Figure 2: Fix commit from golang/go fixing a nil panic in its net package (PR #60823). The nil panic is caused by calling method String() on the return of RemoteAddr() on L1859, which can be a nil interface value (as shown in Figure 3)



![avatar](https://blog.uber-cdn.com/cdn-cgi/image/width=1024,quality=80,onerror=redirect,format=auto/wp-content/uploads/2023/11/figure_3.jpg) 
Figure 3: Excerpt from net/net.go showing the net.Conn interface and the implementation of the RemoteAddr() method by the struct net.conn, which can return nil, if c.ok() is false

在Go程序中发现Nil Panic是一种特别普遍的运行时错误形式。Uber的Go 大仓也不例外，已经目睹了由于Nil Panic而在生产中发生的几起运行时错误，影响范围从错误的程序行为到应用程序中断，影响了Uber客户。因此，为了最大化可靠性和代码质量，对于Uber来说，使程序员能够在错误代码部署到生产之前及早检测和修复Nil Panic至关重要。

Nil Panic还可能导致拒绝服务攻击。例如，CVE-2020-29652是由于golang.org/x/crypto/ssh中的零指针解引用引起的，允许远程攻击者对SSH服务器发起拒绝服务攻击。

Go发行版提供了一个自动化工具nilness，用于检测Nil Panic。这个nilness检查器是一种轻量级的静态分析技术，只报告简单错误，例如明显的零解引用站点（例如，如果x == nil { print(*x) }）。然而，这些简单的检查无法捕捉到实际程序中的复杂零流，如图2所示。因此，我们需要一种执行严格分析并对生产代码有效的技术。

为了处理Java中的NullPointerExceptions（NPEs），Uber开发了NullAway。NullAway要求代码用@Nullable注解进行注释，以保证编译时的NPE自由。这限制了直接适应类似NullAway技术用于我们的目的的可行性，因为与Java不同，Go没有语言支持的注解。此外，注释大型代码库（例如，Uber的Go 大仓有9000万行代码）是一项繁琐的任务。此外，Go的各种独特特性和特点本身带来了独特的挑战。

我们克服这些限制的答案是NilAway。

我们设计并开发了NilAway，通过使用复杂的程序间静态分析和推理技术，自动检测Nil Panic。NilAway的设计目标是对开发者没有注释负担，保持对本地和CI构建时间的最小影响，并以对Go开发者自然的方式解决Go语言习惯用法带来的许多挑战。

## NilAway的核心思想

我们的主要思想是，代码中的零性流动可以建模为全局类型约束系统，然后可以使用2-SAT算法求解，以确定潜在的矛盾。在高层次上，我们在不同的程序站点捕获零和非零约束，用于结构字段、函数参数和返回值。零约束的一个例子是返回x，其中x是一个未初始化的指针，而解引用*x是非零约束的一个例子。然后我们构建一个全局蕴含图，对这些特定于程序站点的约束进行建模。最后，我们遍历蕴含图 - 向前传播已知的零值和向后传播已知的非零值 - 以找到矛盾。对于一个站点S，如果在蕴含图的程序路径中发现矛盾零性(S)^非零性(S)，则意味着见证了一个零值从零点流向站点S，从那里到达解引用点，很可能导致Nil Panic。NilAway收集并向开发者报告这些矛盾作为潜在Nil Panic。



![avatar](https://blog.uber-cdn.com/cdn-cgi/image/width=1024,quality=80,onerror=redirect,format=auto/wp-content/uploads/2023/11/figure_4.jpg) 
Figure 4: Excerpt from the implication graph built by NilAway representing the nil flow for the example in Figure 2.

图4显示了NilAway为图2所示示例构建的蕴含图中的零流路径。这里的节点是可能是零类型的程序站点，边是它们之间的零流。NilAway遍历蕴含图以找到不安全流，将它们建模为矛盾。如果发现见证的零值通过不同的程序路径流向目的地，而在这个目的地期望该值为非零，则流被认为是不安全的，例如在零值从具体实现net.conn.RemoteAddr()流向其解引用，通过接口声明net.Conn.RemoteAddr()上的方法调用的情况。NilAway报告了这个Nil Panic的详细错误消息（如图5所示），允许开发者通过确凿的零性到其解引用的确切零流轻松调试，并应用必要的修复以防止Nil Panic。

![avatar](https://blog.uber-cdn.com/cdn-cgi/image/width=1024,quality=80,onerror=redirect,format=auto/wp-content/uploads/2023/11/figure_5.jpg) 
Figure 5: Error message reported by NilAway for the unsafe flow in Figure 2

请注意，一般来说，对于实际的静态类型系统，无论是否具有全局类型推断，总会存在不满足有效静态类型检查的无错误程序。在NilAway的情况下，请注意，上述算法不会捕获程序执行中微妙的程序间不变式，这些不变式会阻止零到非零流动在运行时发生。例如，在图3中，可能设置了某些共享程序状态，以便每当从conn.RemoteAddr()调用c.ok()时，它总是返回true，在这种情况下，该代码中不存在Nil Panic。然而，实际上，NilAway的误报率很低，这种复杂的执行不变式本质上阻止推断适当的零性约束的情况往往与可能的代码异味相关联。

## NilAway的设计与实现

我们围绕以下四个关键要求设计和开发了NilAway，使其成为Uber规模的实用工具：

1、低延迟：NilAway在对大型Go代码库进行分析时应该只产生很低的额外开销。我们希望NilAway在开发人员引入潜在Nil Panic时立即提供反馈，从而要求NilAway足够快，能够在开发流程的每个阶段以低延迟运行，即使在本地构建期间也是如此。高开销意味着更高的延迟（延迟反馈），从而降低开发人员的工作效率。

2、高有效性：NilAway应该具有较低的误报率；检查误报的Nil Panic会浪费开发人员的时间。

3、完全自动化：NilAway应该是完全自动化的，不需要开发人员的额外输入（例如，像NullAway那样的注释，或人为的编码模式）。

4、针对Go的特殊性：NilAway应将Go的特殊性视为一等公民，并设计一个针对Go的系统。

![avatar](https://blog.uber-cdn.com/cdn-cgi/image/width=1024,quality=80,onerror=redirect,format=auto/wp-content/uploads/2023/11/figure_6.jpg) 
Figure 6: Architecture of NilAway.

NilAway是用Go实现的，并使用go/analysis框架来分析代码。图6显示了NilAway的架构概览。NilAway接受标准Go代码作为输入，以包含代码的目标包路径的形式，并通过其分析返回它识别的潜在Nil Panic错误。NilAway实现为一个分析器，可以作为独立工具使用，或者也可以轻松地集成到构建系统中，例如Bazel，使用现有的分析器驱动程序，如nogo。

广义上讲，NilAway的实现可以分为三个组件：分析器引擎、推理引擎和错误引擎。分析器引擎负责独立识别函数内的所有潜在零流（即内部程序），而推理引擎负责收集不同程序站点的见证零性值，并通过构建蕴含图通过程序间流传播此信息。最后，错误引擎累积来自分析器引擎和推理引擎的信息，并将每个潜在零流（内部和程序间）标记为安全或不安全。然后将不安全的零流报告给用户，作为潜在Nil Panic错误。

借助新颖的基于约束的方法来检测Nil Panic，NilAway恰当地满足了上述四个要求：

- NilAway速度快。分析器引擎中对每个函数的独立分析使其适合并行化，这是主要的性能增强器。此外，我们设计了NilAway，通过利用构建缓存逐步构建全局蕴含图，避免了昂贵的重新构建依赖项。这种仔细的工程设计使NilAway快速且可扩展，适用于大型代码库。在Uber的测量中，我们观察到NilAway仅给正常构建过程增加了很小的开销（不到5%）。

- NilAway实用。为了保持NilAway的精确性，分析器引擎设计并实现为支持许多常见的Go语言特性。我们的错误引擎也经过精心设计，只有在证据显示不安全零流时才报告错误。话虽如此，我们不声称我们的方法是完备或完整的，而是以实用的错误查找为我们的指导原则。NilAway可能会产生误报和漏报。然而，我们正在不断努力减少它们，使NilAway更加精确。在实际部署到Uber时，NilAway已被观察到在实践中表现良好（随后讨论），在新代码中捕获了大部分潜在Nil Panic，使NilAway在实用性和性能开销之间保持了良好的平衡。

- NilAway完全自动化。我们基于约束的方法使其非常适合推理，这允许NilAway以完全自动化的模式运行，无需注释。

## 在Uber使用NilAway

NilAway已集中部署在Go 大仓中，与Bazel+Nogo框架紧密集成，允许其在CI流水线的每次构建和本地构建中作为默认的linter运行。然而，错误报告是在测试阶段进行的，其中Nil Panic错误仅针对已加入NilAway的Go 大仓中的服务报告。

对于服务所有者，我们目前提供两种错误报告选项：（1）全面且阻塞，以及（2）止血且非阻塞。

- 在第一个选项中，如果发现任何错误，NilAway会导致构建失败（如果需要，可以通过//nolint:nilaway抑制）。NilAway全面报告所有代码的错误，无论是现有的还是新的。这个选项更可取，以确保没有Nil Panic的代码库。然而，它要求解决服务代码中报告的所有Nil Panic，然后才能允许任何构建通过。这可能会给服务的开发带来高昂的前期成本，从而引起服务所有者之间的摩擦。

- 为了解决上述问题，我们在第二个选项中提供了一个轻量级版本，在该版本中，我们仅报告服务中更改的代码的NilAway错误。这些错误以非阻塞方式直接在已加入服务的每个差异代码修订（即拉取请求）上报告。这种止血方法有助于防止新的Nil Panic被引入服务代码，同时允许团队逐步解决现有代码中的Nil Panic，而无需减慢开发的初始加入工作。

我们已经在Uber的几个服务上加入了NilAway，涵盖了这两种选项，我们从团队那里收到的总体反馈是积极的。一位满意的用户表示：“NilAway帮助他们的团队及早发现问题，防止了部署回滚”，另一位用户表示：“NilAway留下的评论非常可行，而且没有造成任何噪音。”用户还会积极报告他们可能遇到的误报，并提出我们积极改进的可用性建议。

### 有影响力的例子
我们现在讨论一个有趣的案例，其中NilAway报告了一个重要错误，该错误导致一个服务在生产代码中日志记录了超过3,000个Nil Panic。图7显示了导致Nil Panic的代码的简化和编辑摘录。这个示例使用了Go的消息传递构造，称为通道。在第16行，对t.s.foo(…)的函数调用返回一个通道ch，随后由变量a接收。不幸的是，Go允许从关闭的通道读取，在这种情况下，将返回零值（即零）。如果在函数foo中采取L7->L8->L5的代码路径，则会关闭通道，而不向其中写入任何内容。这将导致在第17行的解引用点a.Items[*id]处发生Nil Panic。NilAway正确地报告了这个错误，因为它见证了对可能从关闭通道接收的变量的不安全解引用。

![avatar](https://blog.uber-cdn.com/cdn-cgi/image/width=1024,quality=80,onerror=redirect,format=auto/wp-content/uploads/2023/11/figure_7.jpg) 
Figure 7: Simplified and redacted code excerpt from an internal Uber service logging 3000+ nil panics per day in production.

解决这个问题的方法是正确保护从关闭通道接收的内容，要么使用Go的ok构造（例如，if a, ok := <-t.s.foo(…); ok { … }），要么在L17解引用之前对结果变量a进行零性检查（例如，if a != nil { … }）。我们的开发人员在NilAway报告此错误后立即应用了零性检查修复，效果显著：服务从每天记录3,000多个Nil Panic变为0，如图8所示。

![avatar](https://blog.uber-cdn.com/cdn-cgi/image/width=1024,quality=80,onerror=redirect,format=auto/wp-content/uploads/2023/11/figure_8.jpg) 
Figure 8: Complete addressal of the 3000+ nil panics being logged per day in production

## 为您的代码使用NilAway

我们很高兴地宣布，NilAway现在已在https://github.com/uber-go/nilaway/上开源。我们相信NilAway对于任何实施Go代码并希望确保Nil Panic免费代码库的个人或团队都将是有用的。

设置NilAway相当简单。它可以作为独立检查器使用，也可以与现有驱动程序集成。有关更多详细信息，请参阅README和wiki。

立即尝试NilAway并告诉我们您的体验。我们也欢迎社区的贡献。


