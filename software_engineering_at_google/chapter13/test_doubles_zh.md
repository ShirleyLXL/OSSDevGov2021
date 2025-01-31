# 第十三章 测试问题

## 真实的实现

在谷歌，对真实的实现的偏爱是随着时间的推移而发展起来的，因为我们看到过度使用mocking框架有一种倾向，即用重复的代码污染测试，与真实实现不同步，使重构变得困难。我们将在本章后面更详细地讨论这个话题。

在测试中更倾向于真实的实现，这被称为*经典测试*。还有一种测试风格被称为mockist测试，在这种测试中，倾向于使用mocking框架而不是真实实现。尽管软体行业的一些人在进行mockist测试（包括第一个mocking框架的创造者），但在Google，我们发现这种测试方式很难扩展。它要求工程师在设计被测系统时遵循严格的准则，而谷歌大多数工程师的默认行为是以一种更适合经典测试风格的方式来编写代码。

### 偏爱现实主义而非孤立

对依赖关系使用真实的实现，使被测系统更加真实，因为这些真实实现中的所有代码都将在测试中执行。相反，利用双倍的测试将被测系统与其依赖关系隔离开来，这样测试就不会执行被测系统依赖关系中的代码。

我们更喜欢现实的测试，因为它们能给被测系统正常工作带来更多的信心。如果单元测试过于依赖双重测试，工程师可能需要运行集成测试或手动验证他们的功能是否按预期工作，以获得同样的信心。执行这些额外的任务可能会减慢开发速度甚至可能会让错误漏掉，如果工程师完全跳过这些任务，当这些任务太耗时了，与运行单元测试相比。

用测试替身替换类的所有依赖关系会任意地将被测系统与作者恰好直接放在该类中的实现隔离，并排除恰好在不同类中的实现。然而，一个好的测试应该是独立于实现的--它应该从被测试的API的角度来编写，而不是从实现的结构来编写。

如果真实实现中存在bug，使用真实的实现会导致你的测试失败。这是好的，如果你希望你的测试在这种情况下失败，因为这表明你的代码在生产中无法正常工作。有时，真实实现中的一个bug会导致测试失败，因为使用真实实现的其他测试也可能失败。但如果有好的开发者工具，比如持续集成（CI）系统，通常很容易追踪到导致失败的变化。

#### 案例研究: @DoNotMock

在Google，我们已经看到了足够多的过度依赖mocking框架的测试，这促使我们在Java中创建了@DoNotMock注解，它可以作为ErrorProne静态分析工具的一部分。这个注解是API所有者声明的一种方式，"这个类型不应该被mock，因为存在更好的替代方案"。

如果工程师试图使用mocking框架来创建一个被注解为@DoNotMock的类或接口的实例，如例13-10所示，他们会看到一个错误，指示他们使用更合适的测试策略，如真实的实现或假的实现。这个注解最常用于那些简单到可以按原样使用的价值对象，以及那些有精心设计的假的API。

例子 13-10. @DoNotMock的注释 

```java
@DoNotMock("Use SimpleQuery.create() instead of mocking.")
public abstract class Query {
public abstract String getQueryValue();
}
```

为什么API所有者会关心这个问题呢？简而言之，它严重限制了API所有者随着时间的推移对其实现进行修改的能力。正如我们在本章后面所探讨的，每当mocking框架被用于打桩或交互测试时，它都会重复API所提供的行为。

当API所有者想要改变他们的API时，他们可能会发现它已经在整个Google的代码库中被模拟了几千甚至几万次！这些测试替身很可能表现出违反被模拟类型的API要求的行为--例如，为一个永远不能返回空的方法返回空。如果测试使用的是真正的实现或假的，API所有者可以对他们的实现进行修改，而不需要先修复成千上万的有缺陷的测试。

### 如何决定何时使用真实的实施方案

如果一个真实的实现是快速的、确定的，并且有简单的依赖关系，那么它就是首选。例如，一个真实的实现应该被用于一个*价值对象*。这方面的例子包括一笔钱、一个日期、一个地理位置，或者一个集合类，如列表或地图。

然而，对于更复杂的代码，使用真实实现往往是不可行的。考虑到要进行权衡，什么时候使用真实实现或测试替身可能没有一个确切的答案，所以你需要考虑以下因素。

#### 执行时间

单元测试最重要的要求之一是它们应该是快速的--你希望能够在开发过程中不断地运行它们，这样你就可以快速得到关于你的代码是否能运行的反馈（你也希望它们在CI系统中运行时能够快速完成）。因此，当真正的实现很慢时，测试替身可以非常有用。

对于一个单元测试来说，多慢才算慢？如果一个真正的实现在每个单独的测试用例的运行时间上增加一毫秒，很少有人会把它归类为慢。但如果它增加了10毫秒，100毫秒，1秒，等等呢？

这里没有确切的答案--它可能取决于工程师是否感觉到生产力的损失，以及有多少测试在使用真实的实现（如果有5个测试用例，每个测试用例多一秒钟可能是合理的，但如果有500个测试用例就不一样了）。对于边界情况，通常更简单的是使用真正的实现，直到它变得太慢，在这一点上，测试可以被更新为使用一个测试替身。

并行测试也可以帮助减少执行时间。在谷歌，我们的测试基础设施使得将测试套件中的测试拆分到多个服务器上执行变得非常简单。这增加了CPU时间的成本，但它可以为开发人员节省大量时间。我们将在第18章中进一步讨论这个问题。

鉴于测试需要构建真正的实现以及所有的依赖关系，使用真正的实现会导致构建时间的增加。使用像Bazel这样的高度可扩展的构建系统会有帮助，因为它可以缓存未改变的构建工件。

#### 决定论

如果对于被测系统的给定版本，运行测试的结果总是相同的，也就是说，测试要么总是通过，要么总是失败，那么这个测试就是*确定性的*。相反，如果一个测试的结果可以改变，即使被测系统保持不变，那么该测试就是*非确定性的*。

测试中的非确定性会导致浮动性--即使被测系统没有变化，测试也会偶尔失败。正如第11章所讨论的，如果开发人员开始不相信测试的结果并忽视失败，那么飘忽不定就会损害测试套件的健康。如果使用一个真实的实现很少会引起故障，它可能不值得回应，因为对工程师的干扰很小。但是，如果经常发生故障，可能是时候用一个测试替身来取代真实的实现了，因为这样做会提高测试的保真度。

真正的实现比测试替身复杂得多，这增加了它的非确定性的可能性。例如，如果被测系统的输出因线程执行的顺序不同而不同，利用多线程的真实实现可能偶尔会导致测试失败。

非确定性的一个常见原因是代码不是密封的；也就是说，它对外部服务有依赖性，而这些服务是在测试的控制之外的。例如，一个试图从HTTP服务器上读取网页内容的测试，如果服务器过载或网页内容改变，可能会失败。相反，应该使用一个测试替身来防止测试依赖于外部服务器。如果使用测试替身是不可行的，另一个选择是使用服务器的密封实例，它的生命周期由测试控制。下一章将更详细地讨论密封的实例。

非确定性的另一个例子是依赖于系统时钟的代码，因为被测系统的输出会因当前的时间而不同。一个测试可以使用一个测试替身来硬编码一个特定的时间来代替依赖系统时钟。

#### 依赖关系的构建

当使用一个真正的实现时，你需要构建它所有的依赖关系。例如，一个对象需要构建它的整个依赖树：它所依赖的所有对象，这些依赖对象所依赖的所有对象，等等。一个测试替身通常没有依赖关系，所以与构造一个真实的实现相比，构造一个测试替身可能要简单得多。

作为一个极端的例子，想象一下试图在测试中创建下面代码片断中的对象。要确定如何构建每个单独的对象是很耗时的。测试也将需要不断的维护，因为当这些对象的构造器的签名被修改时，它们需要被更新。

Foo foo = new Foo(new A(new B(new C()), new D()), new E(), ..., new Z());

使用测试替身是很诱人的，因为构建一个测试替身是很简单的。例如，在使用Mockito嘲讽框架时，只需要构建测试替身：

 @Mock Foo mockFoo;

 尽管创建这个测试替身要简单得多，但使用真正的实现有很大的好处，正如本节前面所讨论的。以这种方式过度使用测试替身往往也有很大的弊端，我们在本章后面会看一下。所以，在考虑是使用真实实现还是测试替身时，需要做一个权衡。

与其在测试中手动构建对象，理想的解决方案是使用生产代码中使用的相同对象构建代码，如工厂方法或自动依赖注入。为了支持测试的使用情况，对象构造代码需要足够灵活，以便能够使用测试替身，而不是用于生产的实现方式的硬编码。

## 假的

如果在测试中使用一个真实的实现是不可行的，最好的选择往往是使用一个假的来代替它。假的比其他测试的测试替身更受欢迎，因为它的行为与真实的实现相似：被测试的系统甚至不应该能够分辨出它是在与真实的实现还是假的进行交互。例13-11说明了一个假的文件系统。

例子 13-11. 一个假的文件系统

```java
// 这个假的实现了FileSystem接口。真正的实现也使用该接口。
public class FakeFileSystem implements FileSystem {
// 存储一个文件名到文件内容的映射。文件被存储在内存中，而不是磁盘上，因为测试不需要做磁盘I/O。
private Map<String, String> files = new HashMap<>();
@Override
public void writeFile(String fileName, String contents) {
// 将文件名和内容添加到map上。
files.add(fileName, contents);
}
@Override
public String readFile(String fileName) {
String contents = files.get(fileName);
// 如果没有找到文件，真正的实现会抛出这个异常，所以假的也必须抛出这个异常。
if (contents == null) { throw new FileNotFoundException(fileName); }
return contents;
}
}
```

### 为什么fakes很重要？

fakes可以是一个强大的测试工具：它们执行得很快，并允许你有效地测试你的代码，而没有使用真实实现的缺点。

一个fakes就能从根本上改善一个API的测试体验。如果你把它扩展到各种API的大量假的测试，fakes可以为整个软件组织的工程速度提供巨大的提升。

另一种做法，在一个很少有假的软件组织中，速度会比较慢，因为工程师最终会在使用真实的实现中挣扎，从而导致缓慢和不稳定的测试。或者，工程师们可能会求助于其他测试替身的技术，如插桩或交互测试，正如我们在本章后面要研究的那样，这可能会导致测试不明确、脆弱和低效率。

### 什么时候应该写fakes？

创建一个fake需要更多的努力和更多的领域经验，因为它需要与真实的实现有类似的行为。fake也需要维护：当真实实现的行为发生变化时，fake也必须被更新以适应这种行为。正因为如此，拥有真实实现的团队应该编写和维护一个fake。

如果一个团队正在考虑编写一个fake，就需要权衡使用假的所带来的生产力的提高是否超过编写和维护它的成本。如果只有少数几个用户，可能不值得他们花费时间，而如果有几百个用户，则可能会带来明显的生产力提高。

为了减少需要维护的fake的数量，通常应该只在不会在测试中使用的代码的根部创建一个fake。例如，如果一个数据库不能在测试中使用，那么数据库API本身应该存在一个fake，而不是每个调用数据库API的类。

如果fake服务的实现需要在不同的编程语言中重复，那么维护fake服务的工作就会很繁重，例如对于允许从不同语言调用服务的客户端库的服务。这种情况下的一个解决方案是创建一个单一的fake服务，并让测试配置客户端库来向这个fake服务发送请求。这种方法与完全写在内存中的fake服务相比更加沉重，因为它要求测试跨进程通信。然而，只要测试仍能快速执行，这可能是一个合理的权衡。

### Fakes的正确性

也许围绕着创建fake的最重要的概念是正确性；换句话说，fake的行为与真实实现的行为有多密切。如果fake的行为与真实实现的行为不匹配，那么使用该fake的测试就没有用处--当使用该fake时，测试可能会通过，但同样的代码路径在真实实现中可能无法正常工作。

完美的保真并不总是可行的。毕竟，fake是必要的，因为真正的实现在某种程度上并不适合。例如，在硬盘存储方面，一个假的数据库通常不会与真正的数据库保真，因为假的数据库会把所有东西都存储在内存中。

然而，主要的是，假的应该保持对真实实现的API要求的一致性。对于API的任何给定的输入，假的应该返回相同的输出，并执行其相应的真实实现的相同的状态变化。例如，对于数据库.save(itemId)的真实实现，如果一个项目在其ID不存在的情况下被成功地保存，但在ID已经存在的情况下会产生错误，那么假的必须符合这种相同的行为。

一种思考方式是，假的必须对真实的实现有完美的保真度，但只是从测试的角度来看。例如，一个假的散列API不需要保证给定输入的散列值与真实实现产生的散列值完全相同--测试可能不关心具体的散列值，只关心给定输入的散列值是唯一的。如果散列API的要求没有对返回的具体散列值作出保证，那么即使假的散列值没有完全和真正的实现一致，因为它仍然符合API要求的规定。

其他典型的完美保真例子可能对fake没有用的，如延迟和资源消耗。然而，如果你需要明确地测试这些约束条件（例如，验证函数调用延迟的性能测试），就不能使用fake，所以你需要借助其他机制，例如使用真正的实现而不是fake。

一个fake可能不需要拥有其对应的真实实现的100%的功能，特别是如果这种行为不是大多数测试所需要的（例如，罕见的边缘情况的错误处理代码）。在这种情况下，最好是让fake快速失败；例如，如果执行了不支持的代码路径，就会引发一个错误。这种失败向工程师表明，在这种情况下，fake是不合适的。

### Fakes应该被测试

一个fake必须有自己的测试，以确保它符合其对应的真实实现的API。没有测试的fake最初可能会提供真实的行为，但如果没有测试，随着时间的推移，这种行为会随着真实实现的发展而发生变化。

为fake编写测试的一种方法是针对API的公共接口编写测试，并针对真正的实现和fake运行这些测试（这些被称为*契约测试*）。针对真实实现运行的测试可能会比较慢，但是他们的缺点被最小化了，因为他们只需要由fake的所有者运行。

### 如果没有fake，该怎么办？

如果没有fake，首先要求API的所有者创建一个。所有者可能不熟悉fake的概念，或者他们可能没有意识到fake对API用户的好处。

如果一个API的所有者不愿意或不能创建一个fake，你也许可以自己写一个。一种方法是将所有对API的调用都封装在一个单一的类中，然后创建一个不与API对话的假的类版本。这样做也比为整个API创建一个假的要简单得多，因为无论如何你往往只需要使用API的一个子集的行为。在谷歌，一些团队甚至将他们的fake贡献给了API的所有者，这使得其他团队能够从fake中受益。

最后，你可以决定使用一个真实的实现（并处理本章前面提到的真实实现的权衡问题），或者采用其他测试替身技术（并处理本章后面提到的权衡问题）。

在某些情况下，你可以把fake看作是一种优化：如果使用真实实现的测试太慢，你可以创建一个fake来使其运行更快。但是，如果fake速度提高并不超过创建和维护fake的工作，那么最好还是坚持使用真实的实现。

## 打桩

正如本章前面所讨论的，打桩是一种测试的方式，为一个函数硬编码行为，否则它本身就没有行为。它通常是一种快速而简单的方法来替代测试中的真实实现。例如，例13-12中的代码使用打桩来模拟信用卡服务器的响应