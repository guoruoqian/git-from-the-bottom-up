# 仓库：目录内容的跟踪

就像我们在前面章节所讲述的，Git所做的工作其实非常简单：维护着“工作目录”工作过程中的快照。Git的大多数设计都可以使用这个最基本的任务来理解。

Git仓库的设计理念与Unix文件系统的设计非常相似：Unix文件系统从一个根目录\(root\)开始，典型的文件系统包含了一些目录，大多数的目录还有自己的子目录，或者是包含了真正需要保存的数据的“文件”，即叶子节点。这些文件的元信息，例如文件大小，类型，文件权限等信息，存储在他们所在的目录中，也存储在他们对应的i-node中（在Unix中i-node是文件内容的引用）。每一个i-node都有一个独一无二的数字在唯一标识这个文件的内容。当然，可能存在多个文件指向同一个i-node的情况（典型的如硬链接）。可以说，真正“拥有”这些存在磁盘上文件内容的其实是这些i-node。

在Git的实现内部，与Unix文件系统的结构有着惊人的相似性，尽管存在着一两个重要的不同之处。首先，Git使用一种叫blob的东西来存储文件的内容，就像文件是目录树的叶子节点一样，blobs也是tree（在Git中用于表示目录的数据结构）的的叶子节点。就像i-node使用一个数字来唯一标示文件，blob使用一个SHA1的hash值作为这个文件内容的代表，而这个SHA1值是使用其对应文件的内容和文件大小计算出来的。虽然如此，SHA1值并没有实际的意义，对于大多数的使用场景而言，我们只需要把这个数字当做一个唯一标识，就像i-node的编号一样就可以了。不过一个文件的SHA1值有着两个比较重要的作用：1. SHA1码保证了这个文件绝对不会被修改（HASH校验码的功能），2. 但凡内容相同的文件（尽管命名或创建时间可能不同），不管他们出现在什么地方，我们永远可以使用同一个blob来表示，不管是处于不同commit，不同仓库，乃至于整个互联网，我们都可以用一个唯一的SHA1来表示同一个文件。这就像一个硬链接一样，只要我们的仓库中有至少一个快照中包含了引用了这个blob，这个blob就永远不会消失。

Blob存储方式与Unix文件系统还有一个区别就在于，blob不会存储文件的元信息，而所有的这些源信息都存储在tree中（译者注：下文我们把tree译作“树”）。我们现在假设两个文件“foo”和“bar”包含完全相同的内容，一棵”树“可能知道他包含的“foo”是在2004年8月创建的，而另外一棵树却记录着他拥有的文件“bar”创建于“foo”的五年后。然而，在一个标准的Unix文件系统中，两个内容相同，但是元信息不同的文件被看做是两个独立而不同的文件（如果他们不是硬链接关系），那么为什么会有这种区别呢？我们知道，Unix系统被设计为支持这些文件变动的，而Git却是为不变而设计的。事实上，一旦一个文件被保存在仓库中，那么他就是**不可变的，**因此就需要不同于在Git这种设计就变得合理多了。在这种设计的前提下，我们的存储会变的更加紧凑，因为所有相同内容的文件都只会被存储一次，而不管他在历史快照中出现了多少次。
