# MPI的概念
Massage Passing Interface 是消息传递库的标准函数库的标准规范。
- 一种新的库描述，不是一种语言，共有上百个函数调用接口，提供与C和Fortran语言的绑定
- MPI是一种标准或规范的代表，而不是特指一个对它的具体实现
- MPI是一种消息传递编程模型，并成为这种编程模型的代表和事实上的标准

# MPI的特点

- 消息传递式并行程序设计
指用户必须通过显式地发送和接收消息来实现处理机间数据交换。  
再这种并行编程种，每个并行进程均有自己独立的地址空间，相互之间访问不能直接进行，必须通过显式的消息传递来实现  
这种编程方式是大规模并行处理机（MPP）和机群（Cluster）采用的主要编程方式。

- 并行计算粒度大，特别适合于大规模可拓展并行算法  
用户决定问题分解策略、进程间的数据交换策略，在挖掘潜在并行性方面更主动。并行地计算粒度大，特别适合于大规模可扩展并行算法
- 消息传递是当前并行计算领域一个非常重要的并行程序设计方式。

# MPI的通信机制

## MPI点对点的通信类别

- **通讯器**(communicator)。通讯器定义了一组能够互相发消息的进程。在这组进程中，每个进程会被分配一个序号，称作**秩**(rank)，进程间显性地通过指定秩来进行通信。

- 通信的基础建立在不同进程之间发送和接收操作。一个进程可以通过指定另一个进程的秩以及一个独一无二的消息标签(tag)来发送消息给另一个进程。接受者可以发送一个接收特定标签标记的消息的请求（或者也可以完全不管标签，接收任何消息），然后依次处理接收到的数据。类似这样的涉及一个发送者以及一个接受者的通信被称作**点对点(point-to-point)通信**

- 当然在很多情况下，某个进程可能需要跟所有其他进程通信。比如主进程想发一个广播给所有的从进程。在这种情况下，手动去写一个个进程点对点的信息传递就显得很笨拙。而且事实上这样会导致网络利用率低下。MPI由专门的接口来帮我们处理这类所有进程间的**集体性(collective)通信**

## 点对点的通信模型

MPI是一种基于消息传递的编程模型，不同进程间通过消息交换数据。  
所谓点对点的通信就是一个进程跟另一个进程的通信，而下面的聚合通信就是一个进程和多个进程的通信

### 标准模式：  
该模式下MPI可能先缓冲该消息，也可能直接发送，可理解为直接送信或通过邮局送信。是最常用的发送方式。  
由MPI决定是否缓冲消息  
1. 没有足够的系统缓冲区时或出于性能的考虑，MPI可能进行直接拷贝，仅当相应的接收完成后，发送语句才能返回。
2. MPI缓冲消息：发送语句在相应的接收语句完成前返回。

### 缓冲模式：

特征：通过用户定义的缓冲区传送消息  
通过用户指定的缓冲区传送消息，这里的缓冲区指的是应用缓冲区。需要用户程序事先申请一块足够大的缓冲区，用户定义的缓冲区只能用于缓存模式，一个进程一次只能绑定一块用户缓冲区，通过`MPI_Buffer_attach`实现  
发送是本地的：完成不依赖于与其匹配的接收操作。发送的结束仅表明消息进入用户指定的缓冲区中。  
`MPI_Buffer_detach`来回收申请的缓冲区，阻塞模式下该操作直到缓存区的数据传输完才返回。  
缓冲模式在相匹配的接收未开始的情况下，总是将送出的消息放在缓冲区内，这样发送者可以很快地继续计算，然后由系统处理放在缓冲区中的消息。占用额外内存，一次内存拷贝。
其函数调用形式为：`MPI_BSEND(...)`。B代表缓冲

### 同步模式
特征：要先进行握手，同步发送与接收  
本质特征：收方接收该消息的缓冲区已准备好，不需要附加系统缓冲区  
任意发出：发送请求可以不依赖收方的匹配的接收请求而任意发出  
成功结束：仅当收方已发出接收该消息的请求后才成功返回，否则将阻塞非本地的  
可用于实现进程同步  
函数调用：`MPI_SSEND(...)`,S代表同步  

### 就绪模式 
特征：接收必须先于发送，即有请求才有发送  
发送请求仅当有匹配的接收后才能发出，否则出错。在就绪模式下，系统默认与其相匹配的接收已经调用。接收必须先于发送  
它依赖于接收方的匹配的接收请求，不可以任意发出  
其函数调用形式为：`MPI_RSEND(...)`,R代表就绪。  
正常情况下可用标准模式替换，除可能影响性能外，不影响结果。


- 点对点通信的阻塞性分析  
上述的四种模式都有阻塞通信和非阻塞通信两个版本  
**阻塞通信**  
就是按照上面的流程进程自己要等待指定的操作实际完成，或者所涉及的数据被MPI系统安全备份后，函数才返回后才能进行下一步的操作。
**非阻塞通信**  
通信函数总是立即返回，实际操作由MPI后台进行，需要调用其他函数来查询通信是否完成，通信与计算可重叠。  
因为阻塞通信时保证了数据的安全性，所以通信还是返回后，其他的函数或者语句能够对这些数据资源直接访问。  
但是非阻塞通信函数立即返回，不能保证数据已经被传送出去或者被备份或者已经读入使用，所以即使你没有阻塞，后面的语句可以继续执行，如果你要操纵上面所说的数据，将会导致发送或接收产生错误。所以对于非阻塞操作，要先调用等待`MPI_Wait()`或者测试`MPI_Test()`函数来结束或者判断该请求，然后再向缓冲区写入新内容或读取新内容。  
**非阻塞函数调用**  
发送语句的前缀由MPI_改为`MPI_I`，I指的是immediate，即可改为非阻塞，示例如下：  
标准模式：MPI_Send(...)->MPI_Isend(...)  
Buffer模式：MPI_Bsend(...)->MPI_Ibsend(...)  

- 通信函数的返回和完成，通信的完成  
**通信函数的返回**  
在阻塞状态下通信函数的返回是在通信即将完成或者完成之后(根据不同的模式有不同设置，如缓冲模式发送函数的返回是在缓冲区的数据传输完之后返回)。非阻塞状态下，通信函数立即返回。  
**通信函数的完成**  
通信的完成=发送函数的完成+接收函数的完成  
发送的完成：代表发送缓冲区中的数据已送出，发送缓冲区可以重用。它并不代表数据已被接收方接收。数据有可能被缓冲；  
这里有一个特例：就是同步模式，它的发送完成==接收方已初始化接收，数据将被接收方接收。  
接收的完成：代表数据已经进入写入接收缓冲区。接收者可访问接收缓冲区，status对象已被释放。它并不代表相应的发送操作已结束（即操作函数可返回）。  

通信函数的返回和通信函数完成是不一样的。  
通过`MPI_Wait()`和`MPI_Test()`来判断通信是否已经完成。

## 点对点通信的常用函数  
- MPI程序的初始化  
任何一个MPI程序在调用MPI函数之前，首先调用的初始化函数`MPI_INIT`  
建立MPI的执行环境`int MPI_Init(int *argc,char ***argv)`  
- 获取通信域包含的进程数  
`MPI_Comm_Size(MPI_Comm,int *size)`  
- 获取当前进程标识  
`MPI_Comm_rank(MPI_Comm comm,int *rank)`  
- 消息的发送  
源程序将缓存的数据发送到目的进程  
阻塞发送:  
`MPI_Send(void* buf,int count,MPI_Datatype datatype,int dest)`  
非阻塞发送：
`MPI_Isend(void* buf,int count,MPI_Datatype datatype,int dest,int tag,MPI_Comm comm,MPI_Request *request)`  
非阻塞缓冲模:`MPI_Ibsend`   
非阻塞同步模式：`MPI_Issend`  
非阻塞就绪模式：`MPI_Irsend`  
    - buf：发送缓存的起始地址
    - count：发送的数据个数  
    - datatype：数据类型  
    - des：标识号  
    - Tag：消息标志  
    - comm：通信域  
    - request：非阻塞通信完成对象

- 消息的探测  
`MPI_Probe()`和`MPI_Iprober()`函数探测接收消息的内容。用户根据探测到的消息内容决定如何接收这些消息，如根据消息大小分配缓冲区等。前者为阻塞方式，即只有探测到匹配的消息才返回；后者为非阻塞，即无论探测到与否均立即返回。  
阻塞式消息探测：`MPI_Probe(int source,int tag,MPI_Comm comm,MPI_Status* status)`  
非阻塞式消息探测：`MPI_Iprobe(int source,int tag,MPI_Comm comm,int* flag,MPI_Status* status)`
    - source：数据源的rank，可以是MPI_ANY_SOURCE 
    - tag：数据标签，可以是MPI_ANY_TAG  
    - comm：通信空间/通信域  
    - flag 布尔值。表示探测到与否（只用于非阻塞方式）  
    - status：status对象，包含探测消息的内容

- 消息的探测  
`MPI_Probe()`和`MPI——Iprobe()`函数探测接受消息的内容。用户根据到的消息内容决定如何接收这些消息，如果根据消息大小分配缓冲区等。前者为阻塞方式，即只有探测到匹配的消息才返回，后者为非阻塞。即无论探测到与否均立即返回。 
阻塞式消息探测：`MPI_probe(int source,int tag,MPI_Comm comm,MPI_Status* status)` 
非阻塞式消息探测：`MPI_Iprobe(int source,int tag,MPI_Comm comm,int* flag,MPI_Status)`  
    - source数据源的rank，可以是MPI_ANY_SOURCE
    - tag数据标签，可以是MPI_ANY_TAG
    - comm 通信空间/通信域
    - flag布尔值：表示探测到与否（只用于非阻塞方式）
    - status status对象，包含探测到消息的内容
- 通信的检查  
必须等待指定的通信请求完成后才能返回，成功返回时，status中包含关于所完成的通信消息，相应的通信请求被释放，即request被置成MPI_REQUEST_NULL  
`MPI_Wait(MPI_Request* request,int *flag,MPI_Status* status)`  
不论通信是否完成都立刻返回，flag为1表示通信完成  
`MPI_Test(MPI_Request* request,int* flag,MPI_Status* status)`  
其他通信函数

```f90
int MPI_Waitany(int count,MPI_Request *array_of_requests,int *index, MPI_Status *status)
int MPI_Waitall(int count,MPI_Request *array_of_requests,MPI_Status *array_of_statuses)
int MPI_Waitsome(int incount,MPI_Request *array_of_requests,int *outcount,
				 int *array_of_indices,MPI_Status *array_of_statuses)
int MPI_Testany(int count,MPI_Request *array_of_requests,int *index, int *flag,MPI_Status *status)
int MPI_Testall(int count,MPI_Request *array_of_requests,int *flag,MPI_Status *array_of_statuses)
int MPI_Testsome(int incount,MPI_Request *array_of_requests,int *outcount,
				int *array_of_indices,MPI_Status *array_of_statuses)

```

- 请求的释放和撤销  
`MPI_Request_free(MPI_Request request)`  
释放指定的通信请求（及所占用的内存资源）  
若该通信请求相关联的通信操作尚未完成，则等待通信的完成，因此通信请求的释放并不影响该通信的完成  
该函数成功返回后request被置为MPI_REQUEST_NULL  
一旦执行了释放操作，该通信请求就无法再通过其他任何的调用访问  
`MPI_Cancel(MPI_Request request)`  
非阻塞型，用于取消一个尚未完成的通信请求  
在MPI系统中设置一个取消该通信请求的标志后立即返回，具体的取消操作由MPI系统在后台完成。  
MPI_CANCEL允许取消已经调用的通信请求，但不意味着相应的通信一定会被取消：若相应的通信请求已经开始，则它会正常完成，不受取消操作的影响；若相应的通信请求还没开始，则可以释放通信占用的资源。  
调用MPI_CANCEL后，仍需用MPI_WAIT.MPI_REQUEST_FREE来释放该通信请求。  
`MPI_Test_cancelled(MPI_Status status,int* flag)`  
检测通信请求是否被取消  

- MPI程序结束
`MPI_Finalize()`结束MPI程序的执行  
