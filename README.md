# 操作系统

## 操作系统接口
## 前言
实验1需要我们调用unix操作系统保持出的接口，因此首先需要了解unix操作系统有关的知识。

## 操作系统（operating system）的功能
* 操作系统的任务是在多个程序之间共享一台计算机，并提供比单独的硬件所支持的更为有用的服务集。
* 操作系统管理和抽象化低级硬件，因此，例如，文字处理器不必担心自己正在使用哪种类型的硬件。
* 操作系统允许多个程序之间共享硬件，以便它们可以并发运行。
* 最后，操作系统提供了程序交互的方式，以便它们可以共享数据，协同工作。

* 操作系统为用户程序提供接口去调用。设计一个好的接口操作系统接口非常困难。
* 一方面希望简单，一方面又希望实现复杂的功能。
* 一种好的设计思路是接口之间可以通过某种机制组合起来以实现复杂的操作。

* 本实验中，使用了xv6 操作系统。其提供了unix的基本操作系统接口，并且模仿了Unix的内部设计。
* unix提供的接口很少，但是由于其可以组合的机制，提供了难以想象的通用性。该接口非常成功，以至于现代操作系统（BSD，Linux，Mac OS X，Solaris，甚至在较小程度上是Microsoft Windows）都具有类似Unix的接口。了解xv6是了解这些系统和许多其他系统的良好起点。

* 内核是操作系统的核心，为运行的程序提供服务。 每个正在运行的程序（称为进程）都具有包含指令，数据和堆栈的内存。
* 指令说明了程序的运行逻辑。数据是指令运行所需要的变量。堆栈组织程序的过程调用。
* 当进程需要调用内核服务时，它将通过操作系统提供的接口进行过程调用。 这样的过程称为系统调用。
* 内核使用CPU的硬件保护机制来确保在用户空间中执行的每个进程只能访问其自己的内存。
* 用户程序调用操作系统接口后，硬件提高权限级别，并开始在内核中执行预先安排的功能。

* shell是一个普通程序，可读取用户命令并执行命令。 shell是用户程序而不是内核的一部分，这一事实说明了操作系统接口的强大功能。shell没有什么特别之处，这也意味着shell易于更换；现代Unix系统有多种shell可供选择，每种shell都有其自己的用户界面和脚本功能。 xv6 shell是Unix Bourne shell的简单实现。 可以在（user / sh.c：1）中找到其实现。


## 操作系统接口
* xv6进程由用户空间内存（指令，数据和堆栈）和内核专有的每个进程状态组成。
* xv6保证进程的并发执行，在多个进程之间切换CPU能力。
* 当某个进程未执行时，xv6保存其CPU寄存器，并在下次运行该进程时恢复它们。 内核将进程标识符或pid与每个进程相关联。
* 进程可以使用fork系统调用来创建新进程。 Fork创建一个称为子进程的新进程，该进程与父进程的内存完全相同。
* Fork在子进程与父进程中都会返回。
* 在父进程中，fork返回子进程的pid； 在子进程中，它返回零。
例如，考虑以下用C编程语言编写的程序片段：
```
int pid = fork();
if(pid > 0){
    printf("parent: child=%d\en", pid);
    pid = wait(0);
    printf("child %d is done\en", pid);
} else if(pid == 0){
    printf("child: exiting\en");
    exit(0);
} else {
    printf("fork error\en");
}

```

* exit导致调用进程停止执行并释放资源，例如内存和打开的文件。
* exit接受一个整数状态参数，通常0表示成功，1表示失败。
* wait系统调用返回当前进程已退出子进程的pid，并将该子进程的退出状态传递给wait。
* 如果子进程都没有退出，一直会等待。
* 如果父进程不在乎子进程的退出状态，则可以传递状态0。

在下面的例子中，输出是：
* parent: child=1234
* child: exiting

也可能出现另外的情况，具体取决于父进程还是子进程首先进入其printf调用。

* 子进程退出后，父进程的wait返回，导致父进程打印出：
* parent: child 1234 is done。

* 尽管子进程最初具有与父进程相同的内存内容，但是父进程和子进程执行时使用的是不同的内存和不同的寄存器。
* 更改一个变量不会影响另一个变量。
* 例如，当wait的返回值在父进程中存储到pid中时，它不会更改子进程中的pid。 子进程中的pid的值仍为零。


* exec系统调用使用从文件系统中存储的文件加载的新的内存映像替换调用进程的内存。
* 该文件必须具有特定的格式，该格式指定文件的哪一部分包含指令，哪一部分是数据，从哪条指令开始等。
* xv6使用ELF格式，第3章将对此进行详细讨论。
* 当exec成功执行时，它不会返回到调用程序。从文件加载的指令在ELF标头中声明的入口点开始执行。
* Exec接受两个参数：包含可执行文件的文件名和一个字符串参数数组。 例如：

```
char *argv[3];
argv[0] = "echo";
argv[1] = "hello";
argv[2] = 0;
exec("/bin/echo", argv);
printf("exec error\en");
```

该片段将调用程序以程序/ bin / echo的实例替换，参数列表为echo hello。 大多数程序会忽略第一个参数，这通常是程序的名称

* xv6 Shell使用上述调用执行用户运行的程序。
* xv6 Shell使用上述调用代表用户运行程序。
* shell的主要结构很简单； 参见main（user / sh.c：145）。
* 主循环使用getcmd从用户读取一行输入。 然后，它将调用fork，这将创建shell进程的副本。
* 父进程调用wait，而子进程运行命令。 例如，如果用户键入“ echo hello”给shell，则将以“ echo hello”作为参数调用runcmd。 runcmd（user / sh.c：58）运行实际命令
* 对于“ echo hello”，它将调用exec（user / sh.c：78）
* 如果exec成功，则子进程将从echo执行指令，而不是runcmd。
* 在某些时候，echo将调用exit，这将导致父进程从main（user / sh.c：145）的wait中返回。

* 您可能想知道为什么fork和exec不能在单个调用中组合？ 稍后我们将看到，用于创建进程和加载程序的单独调用在Shell中用于I / O重定向的用法很巧妙。
* 为了避免创建重复进程然后立即替换它的浪费，运行中的内核通过使用虚拟内存技术（如copy-on-write 写时复制）来针对此用例优化fork的实现。

* Xv6隐式分配了大多数用户空间内存：fork分配了子进程复制父进程所需的内存，而exec分配了足够的内存来保存可执行文件。
* 一个在运行时需要更多内存的进程（可能是malloc）可以调用sbrk（n）将其数据内存增加n个字节。 sbrk返回新内存的位置。
* Xv6没有提供用户概念来保护一个用户免受另一个用户侵害；用Unix术语，所有xv6进程都以root身份运行。

## I/O 与文件描述符
    * 文件描述符是一个小的整数，表示进程可以从中读取或写入的内核管理的对象。 进程可以通过打开文件，目录或设备，或通过创建管道，或通过复制现有描述符来获取文件描述符。 为简单起见，我们通常将文件描述符所指的对象称为“文件”； 文件描述符接口抽象了文件，管道和设备之间的差异，使它们看起来都像字节流。
    * 在内部，xv6内核使用文件描述符作为每个进程表的索引，因此每个进程都有一个从零开始的文件描述符专用空间。 按照惯例，进程从文件描述符0（标准输入）读取，将输出写入文件描述符1（标准输出），并将错误消息写入文件描述符2（标准错误）。 就像我们将看到的那样，shell利用约定来实现I / O重定向(redirection)和管道(pipelines)。 shell确保始终打开三个文件描述符（user / sh.c：151），默认情况下，这三个文件描述符是控制台(console)的文件描述符。
* read系统调用从文件描述符读取字节。
* write系统调用从文件描述符写入字节。
* 调用read（fd，buf，n）最多从文件描述符fd中读取n个字节，将它们复制到buf中，并返回读取的字节数。 引用文件的每个文件描述符都有一个与之关联的偏移量。read从当前文件偏移量读取数据，随着读取到的数据增加，文件的偏移量随之增加。当没有更多字节可以读取时，read将返回零以指示文件末尾。
* 调用write（fd，buf，n）将buf中的n个字节写入文件描述符fd，并返回写入的字节数。 仅在发生错误时才写入少于n个字节。 与读操作类似，写操作会在当前文件偏移量处写入数据，然后将偏移量增加写入的字节数：每次写操作都从上次中止的位置开始。

* 以下程序片段（cat命令的功能）将数据从其标准输入复制到其标准输出。 如果发生错误，它将向标准错误写入一条消息。
```
char buf[512];
int n;
for(;;){
    n = read(0, buf, sizeof buf);
    if(n == 0)
    break;
    if(n < 0){
        fprintf(2, "read error\en");
        exit();
    }
    if(write(1, buf, n) != n){
        fprintf(2, "write error\en");
        exit();
    }
}

```
    在代码片段中要注意的重要一点是cat不知道它是从文件，控制台还是管道中读取。 同样，cat不知道它是要打印到控制台，文件还是其他地方。 使用文件描述符以及文件描述符0是标准输入和输出文件描述符是标准输出的约定可以实现cat的简单实现。close系统调用将释放文件描述符，以供将来的open，pipe或dup系统调用重用。 新分配的文件描述符始终是当前进程中编号最小的未使用的描述符。

文件描述符和fork交互使I/O重定向易于实现。Fork会复制父文件的文件描述符表及其内存，以便子文件与父文件打开完全相同的文件。 exec系统调用替换了调用进程的内存，但保留了其文件表。 此行为允许Shell通过分叉，重新打开选定的文件描述符，然后exec新程序来实现I / O重定向。 这是shell为cat <input.txt命令运行的代码的简化版本：

```
char *argv[2];
    argv[0] = "cat";
    argv[1] = 0;
    if(fork() == 0) {
        close(0);
        open("input.txt", O_RDONLY);
        exec("cat", argv);
    }
```
* 当child关闭文件描述符0后，0 是最小的文件描述符。因此open操作将使文件描述符0（标准输入）指向文件input.txt.xv6 shell中的I / O重定向代码完全以这种方式工作（user / sh.c：82）。

* 现在应该清楚为什么将fork和exec分开调用是一个好主意？ 因为如果它们是分开的，则shell可以fork一个child，在该child中使用open，close，dup来更改标准输入和输出文件描述符，然后exec。 不需要更改正在执行的程序（在我们的示例中为cat）。 如果将fork和exec组合到单个系统调用中，则shell将需要一些其他（可能更复杂）的方案来重定向标准输入和输出，或者程序本身将必须了解如何重定向I / O。

* 尽管fork复制了文件描述符表，但每个潜在文件的偏移量在父级和子级之间共享。 考虑以下示例：
```
if(fork() == 0) {
    write(1, "hello ", 6);
    exit(0);
} else {
    wait(0);
    write(1, "world\en", 6);
}
```

* 在上例中，父进程和子进程都将写入文件描述符1.最后输出的数据是"hello world"
* 父进程的写入会等到子进程写入后进行(由于wait)。两个文件描述符共享一个偏移量。
* 此行为有助于从Shell命令序列产生顺序输出，例如（echo hello; echo world）> output.txt。
* dup系统调用复制了一个现有的文件描述符，并返回了一个新的文件描述符，该描述符引用了相同的潜在I/O对象。两个文件描述符共享一个偏移量，就像fork所复制的文件描述符一样。这是将hello world写入文件的另一种方法：
```
fd = dup(1);
write(1, "hello ", 6);
write(fd, "world\en", 6);
```
* 如果两个文件描述符是通过fork和dup调用序列从同一原始文件描述符派生的，则它们共享一个偏移量。 否则，文件描述符不共享偏移量，即使它们是对同一文件的open产生的。 Dup允许shell执行以下命令： ls existing-file non-existing-file > tmp1 2>&1。 2>＆1告诉shell将文件描述符2与描述符1相同。已存在文件的名称和文件不存在等错误消息都将显示在文件tmp1中。 xv6 Shell不支持错误文件描述符的I / O重定向，但是现在您知道如何实现它。

* 文件描述符是一种强大的抽象，因为它们隐藏了它们所连接的对象的详细信息：写入文件描述符1的进程可能正在写入文件，诸如控制台的设备或管道。

## 管道
* 管道是一个小的内核缓冲区，以一对文件描述符的形式暴露给进程，一个用于读取，一个用于写入。
* 将数据写入管道的一端可使该数据可从管道的另一端读取。 管道为流程进行通信提供了一种方法。 以下示例代码运行程序wc,使用标准输入连接到管道的读取端。

```
int p[2];
char *argv[2];
argv[0] = "wc";
argv[1] = 0;
pipe(p);
if(fork() == 0) {
    close(0);
    dup(p[0]);
    close(p[0]);
    close(p[1]);
    exec("/bin/wc", argv);
} else {
    close(p[0]);
    write(p[1], "hello world\en", 12);
    close(p[1]);
}

```
    * 程序调用pipe创建一个新管道，并将读取和写入文件描述符记录在数组p中。 在fork之后，父进程和子进程都具有引用管道的文件描述符。 子进程将读取端复制到文件描述符0上，关闭p中的文件描述符，然后执行wc。 当wc从其标准输入中读取时，它将从管道中读取。 父进程关闭管道的读取侧，写入管道，然后关闭写入侧。
   *  如果没有可用数据，则在管道上进行读取以等待写入数据或所有引用写入端的文件描述符被关闭； 在后一种情况下，读取将返回0，就像到达数据文件的末尾一样。 读取管道会一直堵塞直到无法接受到数据。因此，对于子进程来说，在执行上述wc之前关闭管道的写端很重要：如果wc进程的文件描述符之一引用了管道的写端，则wc将永远等不到文件末尾 。

* xv6 shell实现了管道，例如grep fork sh.c | wc -l 类似于上面的代码（user / sh.c：100）。 子进程创建一个管道，以将管道的左端与右端连接起来。 然后，它在管道的左端调用fork和runcmd，在右端调用fork和runcmd，并等待两者都完成。 管道的右端可能是一个命令，该命令本身包括一个管道（例如a | b | c），该管道本身派生了两个新的子进程（一个用于b，一个用于c）。 因此，shell可以创建进程树。 该树的叶子是命令，内部节点是等待左右子节点完成的进程。 原则上，您可以让内部节点在管道的左端运行，但是这样做会使实现复杂化。

* 管道似乎没有临时文件强大：echo hello world | wc
* 可以在没有管道的情况下实现：
echo hello world >/tmp/xyz; wc </tmp/xyz

* 在这种情况下，管道比临时文件至少具有四个优点。 首先，管道会自动清理自己； 使用文件重定向，shell必须在完成后小心删除/ tmp / xyz。 其次，管道可以传递任意长的数据流，而文件重定向需要磁盘上有足够的可用空间来存储所有数据。 第三，管道允许并行执行管道阶段，而文件方法要求第一个程序在第二个程序启动之前完成。 第四，如果要实现进程间通信，则管道的读写锁比文件的 non-blocking语义更有效。

## 文件系统
* xv6文件系统提供了数据文件和目录，这些数据文件是原始的字节数组。目录包含对数据文件和其他目录的命名引用。 目录形成一棵树，从一个特殊的root目录开始。

* 类似于/a/b/c的路径是指根目录/中名为a的文件夹中名为b的文件夹中名为c的文件或文件夹。
* 不以/开头的路径是相对于调用进程的当前目录的。调用进程的当前目录可以通过chdir系统调用对其进行更改。

* 下面这两个程序片段都打开同一个文件（假设文件存在）
```
chdir("/a");
chdir("b");
open("c", O_RDONLY);

open("/a/b/c", O_RDONLY);
```
* 第一个代码片段将进程的当前目录更改为/a /b； 第二个既不引用也不更改进程的当前目录。
* 有多个操作系统接口来创建新文件或文件夹：mkdir创建新文件夹，使用O_CREATE标志调用open将创建新数据文件，而mknod将创建新设备文件。 如下例所示:
```
mkdir("/dir");
fd = open("/dir/file", O_CREATE|O_WRONLY);
close(fd);
mknod("/console", 1, 1);
```
Mknod在文件系统中创建一个文件，但是该文件没有内容。 但是，文件的元数据会将其标记为设备文件，并记录主设备号和次设备号（mknod的两个参数），它们唯一地标识内核设备。 当以后有一个进程打开文件时，内核会将read和write系统调用转换到内核设备的读写实现，而不是将它们转换到文件系统。

fstat系统调用得到有关文件描述符引用的对象的信息。此对象信息返回结构体stat，定义在
stat.h (kernel/stat.h)：
```
#define T_DIR 1 // Directory
#define T_FILE 2 // File
#define T_DEVICE 3 // Device
struct stat {
    int dev; // File system’s disk device
    uint ino; // Inode number
    short type; // Type of file
    short nlink; // Number of links to file
    uint64 size; // Size of file in bytes
};
```
文件名与文件不同； 同一个文件（称为inode）可以具有多个名称（称为links）。
link系统调用将创建另一个文件名称，该名称引用与现有文件相同的inode。 下面的程序片段创建了一个名为a又为b的新文件。

```
open("a", O_CREATE|O_WRONLY);
link("a", "b");
```

读取,写入a与读取,写入到b相同。 每个inode由唯一的inode编号标识。 在上面的代码片段之后，可以通过检查fstat的结果确定a和b是否引用相同的文件：两者将返回相同的inode编号（ino），并且nlink将变为2。

unlink系统调用从文件系统中删除一个名称。 仅当文件的link计数为零且没有文件描述符引用该文件时， 才会将inode和其所在的磁盘空间清除。

因此当执行了
```
unlink("a");
```
之后，使用名称b任然能够访问文件。

下面的程序片段是一种惯用的方式创建一个临时inode。
```
fd = open("/tmp/xyz", O_CREATE|O_RDWR);
unlink("/tmp/xyz");
```
* 当fd文件描述符被关闭后，临时的inode将会被清除。
* 用于文件系统操作的Shell命令是作为用户级程序（例如mkdir，ln，rm等）实现的。该设计允许任何人通过添加新的用户程序扩展Shell。在事后看来，似乎是理所当然的。
* 但和Unix同时期的其他系统设计，通常将这样的命令构建到shell中（并将shell构建到内核中）。
* cd是一个例外，它内置在shell中（user / sh.c：160）。 cd必须更改shell本身的当前工作目录。 如果cd以常规命令运行，那么shell将派生一个子进程，该子进程将运行cd，而cd会更改该子进程的工作目录。 父进程(即shell)的工作目录不会更改。

## 总结
Unix结合了文件描述符，管道和方便的shell语法以对其进行操作，这是编写通用可复用程序的重大进步。这是Unix的强大功能和广泛使用的原因，外壳程序是第一种所谓的“脚本语言”。Unixit系统调用接口在BSD,Linux,和Mac OSX 上广泛使用。

* Unix系统调用接口已通过可移植操作系统接口（POSIX）标准进行了标准化。 Xv6不兼容POSIX。 它抛弃一些了系统调用（包括诸如lseek之类的基本调用），仅部分实现了系统调用以及其他差异。 xv6的主要目标是简单性和清晰度，同时提供简单的类UNIX系统调用接口。 为了运行基本的Unix程序，一些人用更多的系统调用和一个简单的C库扩展了xv6。 但是，与xv6相比，现代内核提供了更多的系统调用和内核服务。 例如，它们支持联网，窗口系统，用户级线程，许多设备的驱动程序等。 现代内核不断快速发展，并提供了POSIX以外的许多功能。
* 在很大程度上，现代Unix派生的操作系统没有遵循早期的Unix模型，即将设备公开为特殊文件，例如上面讨论的控制台设备文件。Unix的作者继续构建Plan9，将“资源即文件”概念应用于现代设施，将网络，图形和其他资源表示为文件或文件树。
* 文件系统和文件描述符是强大的抽象。 即使这样，也存在其他模型。 Multics是Unix的前身，它以一种类似于内存的方式抽象了文件存储，从而产生了截然不同的界面风格。 Multics设计的复杂性直接影响了Unix的设计师，后者试图构建更简单的东西。
* 本书探讨了xv6如何实现其类似Unix的接口，但是这些思想和概念不仅适用于Unix。 任何操作系统都必须将进程多路复用到基础硬件上，将进程彼此隔离，并提供用于受控的进程间通信的机制。 研究xv6之后，您应该能够查看其他更复杂的操作系统，



## 参考资料
https://pdos.csail.mit.edu/6.828/2019/labs/util.html
https://pdos.csail.mit.edu/6.828/2019/xv6/book-riscv-rev0.pdf
https://pdos.csail.mit.edu/6.828/2019/lec/l-overview.txt
