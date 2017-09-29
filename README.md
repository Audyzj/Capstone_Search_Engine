# Capstone_Search_Engine_搜索引擎设计

#Part 1

什么是Token

Token是最小的有意义的语言单位。在作业中被定义为：

class Token {
  std::string term_;  // The normalized form.
  int begin_;         // Begin offset.
  int end_;           // End offset.
};

例子：
He is working in beijing.
转化为Token序列：
<he:0,2> <be:3,5> <work:6,13> <in:14,15> <beijing:16,23>
（实际输出可能会和这个不一样，因为标准化过程的实现不唯一）

例子中的每个Token由两个部分构成：Term和位置信息。如上述序列中的<work:6,13>，work是working的Term表示形式，6是working在原文中的起始偏移量，13是working在原文中结束偏移量。13-6正好就是working的长度7。

Term是每个Token的标准化形式，它包括：

小写化
提取词根
我们保存Term的原因在于：把每个Token的Term作为倒排索引的检索项，检索Term比检索Token的原字符串更为有效。比如：检索work，可以同时检索到works和working。

本次作业中，我们可以通过调用TokenStream::Normalize()

以流(Stream)读取Token

流是一种可顺序读取的数据序列。

比如：C++中定义了std::istream作为字符输入流，我们可以通过该类的对象std::cin，不停地从标准输入中顺序读入字符。

本次作业要求把std::istream封装成TextTokenStream，实现Token流的读入。TextTokenStream继承了TokenStream。同学们需要实现所有从TokenStream继承下来的函数。

class TokenStream {
 public:
  virtual ~TokenStream() = default;  // Judges whether it has next token.
  virtual bool HasNext() const = 0;
  // Gets the next token. Should make sure HasNext before calling.
  //
  // Supposed that we have tokens "<a> <b>", this function returns <b> on its
  // second call.
  virtual Token Next() = 0;
 protected:
  // Normalizes str.
  virtual void Normalize(std::string* str) const;
};

那么，为什么要以TokenStream的形式读入Token？原因有如下几个：

如果我们把std::istream封装成TokenStream，就可以像读入字符一样读入Token了。
TokenStream是一种适配器模式的设计模式，对于搜索引擎的索引模块来说，可以屏蔽底层对std::istream的操作。
方便以后对切词方法或词语标准化方法进行改进。
作业目标

实现文本的Token流，TextTokenStream，使之能以如下形式从std::istream中读入Token：

search::TokenStream* token_stream = analyzer.NewTextTokenStream(&std::cin);
while (token_stream->HasNext()) {
  search::Token token = token_stream->Next();
  // Do something to token.
}
delete token_stream;
Token的定义

本次作业把Token定义为文本中连续的字母数字串。

其中词的标准化形式Term，同学们应当通过调用TokenStream::Normalize来得到。

输入输出样例

输入：
-- She worked in A130 and B-8.
输出：

<she:3,6> <work:7,13> <in:14,16> <a130:17,21> <and:22,25> <b:26,27> <8:28,29>

考点

程序设计中的封装理念。
对C++输入流和字符串的操作。
对适配器模式的掌握。
如何完成作业

需要安装g++4.8以上版本和make程序。
强烈推荐在Linux系统下完成作业。
从作业网站中下载作业压缩包ex1.tar.gz。
解压后，找到所有包含TODO的地方，并进行实现。
通过make进行编译，并运行ex_main进行本地测试。
把修改后的代码打包上传到作业网站上。
评测方法

我们设计了多组测试，每通过一个大组的测试就会得到一定的分数，总分100分。

Tips

注意类的继承关系。
注意作业代码中已有的注释说明。特别是对指针Ownership的说明。
如果调用了New××××的函数，要记得通过某种方式析构。
参考ex_main.cc中对作业代码的调用方法。
运用valgrind对程序进行内存检查。


#Part 2

文档编号化

在搜索引擎中，每一个文档都被表示为一个系统赋予的编号。

编号	文档
0	第一个文档
1	第二个文档
……	……
N-1	第N个文档
于是我们需要在外存储上构建上述映射表的数据结构，使得我们可以很方便地检索到第i个文档的内容。

然而，我们面对的一个主要问题是：上表中每个记录不是定长的。

文档索引在外存储上的结构

所以，我们打算通过两个文件来解决记录不定长的问题。

存储文件Storage：顺序存储每个文档的内容
第一个文档	第二个文档	……	第N个文档
2. 存储索引文件StorageIndex：顺序存储每个文档在Storage中的偏移量。

第一个文档的在Storage中的偏移量
第二个文档的在Storage中的偏移量
……
第N-1个文档在Storage中的偏移量
这样，StorageIndex中的每个记录都是定长记录，进而就可以随机定位到某个文档的偏移量。

文档的写入和读取

我们把这部分模块称为Storage。

search::util::MemIO mem_io;
search::StorageWriter writer(&mem_io);
for (const std::string& item : data) {
  writer.AddStorage(item);
}
writer.Close();
search::StorageReader reader(&mem_io);
for (int i = 0; i < reader.num_docs(); ++i) {
  LOG(INFO) << "data[" << i << "]: " << reader.Stored(i);
}
通过调用StorageWriter::AddStorage对每个文档进行顺序写入，写入的时候该函数会对所加入文档顺序进行编号（从0开始）；

读取的时候，通过StorageReader::Stored以编号为参数读取。

作业目标

实现前文所提到的Storage模块。该作业需要操作两个文件来在外存储上构建这个Storage索引结构：StorageIndex和Storage。

上述两个文件要求通过调用util::IO的如下几个函数进行打开和关闭：


virtual std::istream* NewStorageIndexIn() const = 0;
virtual void CloseAndDeleteStorageIndexIn(std::istream* in) const = 0;
virtual std::ostream* NewStorageIndexOut() const = 0;
virtual void CloseAndDeleteStorageIndexOut(std::ostream* out) = 0;
virtual std::istream* NewStorageIn() const = 0;
virtual void CloseAndDeleteStorageIn(std::istream* in) const = 0;
virtual std::ostream* NewStorageOut() const = 0;
virtual void CloseAndDeleteStorageOut(std::ostream* out) = 0;
另外，本次作业只规定了两文件存储的大致方案，而具体的存储细节这里不作规定，同学们可以自由发挥。

考点

程序设计中的封装理念。
对流的读写操作。
对C++中字符串的灵活运用。
如何完成作业

需要安装g++4.8以上版本和make程序。
强烈推荐在Linux系统下完成作业。
从作业网站中下载作业压缩包ex1.tar.gz。
解压后，找到所有包含TODO的地方，并进行实现。
通过make进行编译，并运行ex_main进行本地测试。
把修改后的代码打包上传到作业网站上。
评测方法

我们设计了多组测试，每通过一个大组的测试就会得到一定的分数，总分100分。

Tips

注意类的继承关系。
注意作业代码中已有的注释说明。特别是对指针Ownership的说明。
如果调用了New××××的函数，要记得通过某种方式析构。
参考ex_main.cc中对作业代码的调用方法。
运用valgrind对程序进行内存检查。
