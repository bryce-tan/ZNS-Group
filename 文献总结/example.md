# 示例：TVM: An Automated End-to-End Optimizing Compiler for Deep Learning
## 1 Knowledge
### 1.1 **Computational graphs**
 
These can be used for two different types of calculations:

1. Forward computation
2. Backward computation

Consider$Y=(a+b)*(b-c)$
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23157102/1664182390805-84bfd59a-c749-4868-a8a3-1ef7e88b00d2.png#averageHue=%23fbf4ef&clientId=u7e526509-0294-4&from=paste&height=174&id=u22ed7191&originHeight=348&originWidth=383&originalType=binary&ratio=1&rotation=0&showTitle=false&size=20887&status=done&style=none&taskId=ue748ea22-0374-4472-9e48-20bd84cc510&title=&width=191.5)
Forward computation
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23157102/1664182718093-6dd61a9f-3fa2-4c48-8cb7-f7a4ba15b936.png#averageHue=%23fbf6f3&clientId=u7e526509-0294-4&from=paste&height=206&id=u0bca18e1&originHeight=412&originWidth=486&originalType=binary&ratio=1&rotation=0&showTitle=false&size=26674&status=done&style=none&taskId=uaa5d24e2-884b-44d1-a508-1b399fa8dfe&title=&width=243)
Backward computation
### 1.2 Intermediate Representation (IR) 					 				
[IR - Stanford](https://suif.stanford.edu/dragonbook/lecture-notes/Stanford-CS143/16-Intermediate-Rep.pdf#:~:text=Most%20compilers%20translate%20the%20source%20program%20first%20to,is%20done%20on%20this%20form%20of%20the%20code.)
Most compilers translate the source program **first to some form of **_**intermediate representation **_and convert **from there into machine code.** The intermediate representation is a **machine- and language-independent** version of the original source code. 
Although converting the code twice introduces another step, use of an intermediate representation provides advantages in **increased abstraction**, **cleaner separation** between the front and back ends, and adds possibilities for re- targeting/cross-compilation. 
Intermediate representations **also lend themselves to supporting advanced compiler optimizations **and most optimization is done on this form of the code. 
 			![image.png](https://cdn.nlark.com/yuque/0/2022/png/23157102/1664873532250-4582a796-6aa3-46f5-80ec-c2dc22b1233f.png#averageHue=%23f6f6f6&clientId=u2df6587b-f171-4&from=paste&height=213&id=u90f86597&originHeight=426&originWidth=1868&originalType=binary&ratio=1&rotation=0&showTitle=false&size=74228&status=done&style=none&taskId=ub0eab209-7f77-4096-8715-646302c82da&title=&width=934)
###  1.3 Process and thread	
**_PROCESS_**_: _	A process is the **execution of a program** that allows you to perform the appropriate actions specified in a program. It can be defined as an execution unit where a program runs. The OS helps you to create, schedule, and terminates the processes which is used by CPU. The other processes created by the main process are called child process.
A process operations can be easily controlled with the help of** PCB(Process Control Block)**. You can consider it as the **brain of the process**, which contains all the crucial information related to processing like process id, priority, state, and contents CPU register, etc.
Here are the important **properties** of the process:

- **Creation of each process requires separate system calls for each process**.
- **It is an isolated execution entity and does not share data and information.**
- **Processes use the IPC(Inter-Process Communication) mechanism for communication that significantly increases the number of system calls.**
- Process management takes more system calls.
- **A process has its stack, heap memory with memory, and data map.**

**_THREAD: _** Thread is an** execution unit that is part of a process**. A process can have multiple threads, **all executing at the same time**. It is a unit of execution in concurrent programming. A thread is lightweight and can be managed independently by a scheduler. It helps you to improve the application performance using parallelism.
Multiple threads share information like data, code, files, etc. We can implement threads in three different ways: Kernel-level threads, User-level threads and Hybrid threads.
Here are important** properties of Thread**:

- **Single system call can create more than one thread**
- **Threads share data and information.**
- **Threads shares instruction, global, and heap regions. However, it has its register and stack.**
- **Thread management consumes very few, or no system calls because of communication between threads that can be achieved using shared memory.**

![image.png](https://cdn.nlark.com/yuque/0/2022/png/23157102/1665448934042-c03e33c4-a957-4e23-9897-78887b0fc34e.png#averageHue=%23f1f1f1&clientId=u4ff3207b-3de0-4&from=paste&height=606&id=u58580a49&originHeight=1212&originWidth=1866&originalType=binary&ratio=1&rotation=0&showTitle=false&size=192542&status=done&style=none&taskId=u4048216f-d612-4918-ae8b-172a4e1bf90&title=&width=933)
