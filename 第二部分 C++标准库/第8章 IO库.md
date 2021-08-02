## 8.1 IO类

为了支持使用宽字符的语言，标准库定义了一组类型和对象来操纵`wchar_t`类型的数据（2.1.1节）。宽字符版本的类型和函数的名字以一个`w`开始。如`wcin`、`wcout`、`wcerr`等。宽字符版本的类型与其对应的普通`char`版本类型定义在同一头文件，如头文件`fstream`定义了`ifstream`和`wifstream`类型。

类型`ifstream`和`istringstream`都继承自`istream`类。

| 头文件     | 类型                                                         |
| ---------- | ------------------------------------------------------------ |
| `iostream` | `istream`、`wistream`，从流读取数据；`ostream`、`wostream`，向流写入数据；`iostream`、`wiostream`：读写流 |
| `fstream`  | `ifstream`、`wifstream`从文件读取数据；`ofstream`、`wofstream`向文件写入数据；`fstream`、`wfstream`读写文件 |
| `sstream`  | `istringstream`、`wistringstream`：从`string`读取数据；`ostringstream`、`wostringstream`：向`string`写入数据；`stringstream`、`wstringstream`：读写`string` |

### 8.1.1 IO对象无拷贝或赋值

IO对象不能拷贝，也不能赋值。传递时仅能使用引用。而且因为读写IO流会改变其状态，所以引用不能是`const`的。

### 8.1.2 条件状态

下表列出了IO类的一些函数和标志，帮助我们访问和操作流的条件状态（condition state）。

| 名称                | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| `strm::iostate`     | `strm`是一种IO类型，`iostate`是一种机器相关的类型，提供了表达条件状态的完整功能 |
| `strm::badbit`      | 用来指出流已经崩溃，表示系统级错误，此时流往往无法再使用了   |
| `strm::failbit`     | 用来指出一个IO操作失败了，这类问题通常可以修正，流仍可使用   |
| `strm::eofbit`      | 用来指出流到达了文件末尾。到达文件末尾时，`failbit`和`eofbit`将同时置位 |
| `strm::goodbit`     | 用来指出流未处于错误状态，此值保证为0，上面的三种状态位任何一个被置位都将导致`goodbit`被置位。 |
| `s.eof()`           | 若流s的`eofbit`置位，返回`true`                              |
| `s.fail()`          | 若流s的`failbit`或`badbit`置位，返回`true`                   |
| `s.bad()`           | 若流s的`badbit`置位，返回`true`                              |
| `s.good()`          | 若流s处于有效状态，返回`true`                                |
| `s.clear()`         | 将流s中所有条件状态位复位，将流的状态置为有效，返回`void`    |
| `s.clear(flags)`    | 根据给定的`flags`标志位，将流s中对应条件状态位**复位**。`flags`是`strm::iostate`类型。返回`void` |
| `s.setstate(flags)` | 将条件状态位**置位**。其他同上。                             |
| `s.rdstate()`       | 返回流s的当前状态，返回类型为`strm::iostate`                 |

#### 管理条件状态

```c++
auto old_state = cin.rdstate();		//记住cin的当前状态
cin.clear();				//使cin有效
process_input(cin);			//使用cin
cin.setstate(old_state);	//将cin置为原有状态

cin.clear(cin.rdstate() & ~cin.failbit & ~cin.badbit);	//复位failbit和badbit，其他标志位不变
```

### 8.1.3 管理输出缓冲

每个输出流都管理一个缓冲区。

导致缓冲区刷新的因素：

- 程序**正常**结束，缓冲刷新作为`main`的`return`语句的一部分被执行。程序崩溃不会导致缓冲区被刷新。即程序崩溃后可能仍有内容没有输出。
- 缓冲区满。
- 使用操作符如`endl`显式刷新。
- 在每个输出操作后，用操纵符`unitbuf`设置流的内部状态来清空缓冲区。`cerr`是默认设置`unitbuf`的，因此写到`cerr`的内容会立即刷新。
- 一个输出流可能被关联到另一个流。这样，当读写被关联的流时，关联到的流立即刷新。`cerr`和`cin`都默认关联到`cout`，因此读`cin`或写`cerr`都会导致`cout`被刷新。

#### 输出缓冲区刷新操作符

- `endl`：换行，刷新缓冲区
- `flush`：只刷新缓冲区
- `ends`：输出一个空字符，刷新缓冲区

#### `unitbuf`

```c++
cout << unitbuf;	//所有输出操作后都会立即刷新
cout << nounitbuf;	//回到正常的缓冲方式
```

#### 关联输入和输出流

可以使用IO类内置的`tie()`函数将两个流关联。`tie()`有两个版本：

1. 不带参数，返回一个指向输出流的指针。若当前流关联到了一个输出流，则返回的就是该输出流的指针；否则返回`nullptr`
2. 接受一个指向`ostream`的指针，从而将当前流关联到`ostream`。返回**原先**关联的`ostream`

```c++
cin.tie(&cout);		//仅仅用于展示：cin默认就是关联到cout的
ostream *old_tie = cin.tie(nullptr);	//cin不再与其他流关联
cin.tie(&cerr);		//将cin与cerr关联，这不是一个好主意
cin.tie(old_tie);	//恢复原有的关联
```

每个流最多关联到一个流，但多个流可以同时关联到同一个`ostream`。

## 8.2 文件输入输出

除去所有IO类共有的操作，文件IO类有一些特有的操作：

| 名称                      | 描述                                                         |
| ------------------------- | ------------------------------------------------------------ |
| `fstream fstrm`           | 创建一个未绑定的文件流。`fstream`是同名头文件中定义的三个类型之一 |
| `fstream fstrm(s);`       | 创建一个`fstream`，并打开名为`s`的文件。`s`可以是`string`(C++ 11)，也可以是`char *`。构造函数是`explicit`的。默认的`mode`依赖于`fstream`的类型 |
| `fstream fstrm(s, mode);` | 与前一个构造函数类似，但按指定`mode`打开文件                 |
| `fstrm.open(s);`          | 打开名为`s`的文件，并将其与`fstrm`绑定。其他同上。返回`void` |
| `fstrm.close();`          | 关闭与`fstrm`绑定的文件。返回`void`。`fstrm`被销毁（如离开其所在作用域）时，`close()`被自动调用 |
| `fstrm.is_open();`        | 若关联的文件成功打开且尚未关闭，返回`true`                   |

### 8.2.2 文件模式

| 名称     | 描述                         |
| -------- | ---------------------------- |
| `in`     | 以只读方式打开               |
| `out`    | 以只写方式打开，默认进行截断 |
| `app`    | 每次写操作前定位到文件末尾   |
| `ate`    | 打开文件后立即定位到文件末尾 |
| `trunc`  | 截断文件                     |
| `binary` | 以二进制方式进行IO           |

```c++
ofstream out1("file1", ofstream::out);	//默认截断
ofstream out2("file1", ofstream::out | ofstream::trunc);			//显式截断，效果与上面一样
ofstream app1("file2", ofstream::app);	//文件不清空
ofstream app2("file2", ofstream::app | ofstream::out);		//效果与上面一样
```
