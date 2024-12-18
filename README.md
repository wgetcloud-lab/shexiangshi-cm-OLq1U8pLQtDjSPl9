[合集 \- .Net Core 学习笔记(10\)](https://github.com)[1\..NET Core  泛型(Generic)底层原理浅谈11\-07](https://github.com/lmy5215006/p/18529501)[2\..NET Core 委托(Delegate)底层原理浅谈11\-13](https://github.com/lmy5215006/p/18534896)[3\..NET Core 反射(Reflection)底层原理浅谈11\-15](https://github.com/lmy5215006/p/18545334)[4\..NET Core 特性(Attribute)底层原理浅谈11\-19](https://github.com/lmy5215006/p/18551715)[5\..NET Core 线程(Thread)底层原理浅谈11\-22](https://github.com/lmy5215006/p/18556052)[6\..NET Core 线程池(ThreadPool)底层原理浅谈11\-25](https://github.com/lmy5215006/p/18566995):[楚门加速器官网](https://chuanggeye.com)[7\..NET Core 异步(Async)底层原理浅谈11\-29](https://github.com/lmy5215006/p/18571532)[8\..NET Core 锁(Lock)底层原理浅谈12\-05](https://github.com/lmy5215006/p/18585588)[9\..NET Core 堆结构(Heap)底层原理浅谈12\-11](https://github.com/lmy5215006/p/18583743)10\..NET Core 异常(Exception)底层原理浅谈12\-17收起
# 中断与异常模型图


![image](https://img2024.cnblogs.com/blog/1084317/202412/1084317-20241217095330933-1442979344.png)


1. 内中断
内中断是由 CPU 内部事件引起的中断，通常是在程序执行过程中由于 CPU 自身检测到某些异常情况而产生的。例如，当执行除法运算时除数为零，或者访问了不存在的内存地址，CPU 就会产生内中断。


	1. 硬件异常
	CPU内部产生的异常事件
		1. 故障Fault
		故障是在指令执行过程中检测到的错误情况导致的内中断,比如空指针，除0异常，缺页中断等
		2. 自陷Trap
		这是一种有意的内中断，是由软件预先设定的特殊指令或操作引起的。比如syscall,int 3这种故意设定的陷阱
		3. 终止abort
		终止是一种比较严重的内中断，通常是由于不可恢复的硬件错误或者软件严重错误导致的，比如内存硬件损坏、Cache 错误等
	2. 用户异常
	软件模拟出的异常，比如操作系统的SEH，.NET的OutOfMemoryException
2. 外中断
外中断是由 CPU 外部的设备或事件引起的中断。比如键盘，鼠标，主板定时器。这些外部设备通过向 CPU 发送中断请求信号来通知 CPU 需要处理某个事件。外中断是计算机系统与外部设备进行交互的重要方式，使得 CPU 能够及时响应外部设备的请求，提高系统的整体性能和响应能力。


	1. NMI（Non \- Maskable Interrupt，非屏蔽中断）
	NMI 是一种特殊类型的中断，它不能被 CPU 屏蔽。与普通中断（可以通过设置中断屏蔽位来阻止 CPU 响应）不同，NMI 一旦被触发，CPU 必须立即响应并处理。这种特性使得 NMI 通常用于处理非常紧急且至关重要的事件，这些事件的优先级高于任何其他可屏蔽中断。
	2. INTR（Interrupt Request，中断请求）
	INTR 是 CPU 用于接收外部中断请求的引脚（在硬件层面）或者信号机制（在软件层面）。外部设备（如磁盘驱动器、键盘、鼠标等）通过向 CPU 的 INTR 引脚发送信号来请求 CPU 中断当前任务，为其提供服务。这是计算机系统实现设备交互和多任务处理的关键机制之一。


# 用户异常


C\#的异常，在Windows平台下是完全围绕SEH处理框架来展开。其开销并不低，内部走了很多流程。



```
        static void Main(string[] args)
        {
            try
            {
                var num = Convert.ToInt32("a");
            }
            catch (Exception ex)
            {
                Debugger.Break();
                Console.WriteLine(ex.Message);
            }

            Console.ReadLine();
        }

```

![image](https://img2024.cnblogs.com/blog/1084317/202412/1084317-20241217103817363-698779035.png)


## 眼见为实：用户Execption的调用栈


![image](https://img2024.cnblogs.com/blog/1084317/202412/1084317-20241217104120060-1169445986.png)


# 硬件异常


硬件异常指CPU执行机器码出现异常后，由CPU通知操作系统，操作系统再通知进程触发的异常。
比如：


1. 内核模式切换：syscall
2. 访问违例：AccessViolationException
3. visual studio中F9中断：int 3



```
        static void Main(string[] args)
        {
            try
            {
                string str = null;
                var len = str.Length;

                Console.WriteLine(len);
            }
            catch (Exception ex)
            {
                Debugger.Break();
                Console.WriteLine(ex.ToString());
            }

            Console.ReadLine();
        }

```

![image](https://img2024.cnblogs.com/blog/1084317/202412/1084317-20241217134856028-1763141306.png)


与用户异常不同的是，异常的发起点在CPU上，并且CLR为了统一处理。会先将硬件异常转换成用户异常。以此来复用后续逻辑。所以相比用户异常，硬件异常的开销更大


## 眼见为实：硬件Execption的调用栈


![image](https://img2024.cnblogs.com/blog/1084317/202412/1084317-20241217134327081-1805910994.png)


## 硬件异常如何与用户异常绑定?


上面说到，CLR会先将硬件异常转换成用户异常。那么在抛出异常的时候，如何正确抛出一个托管堆认识的异常呢？
以空指针异常为例
![image](https://img2024.cnblogs.com/blog/1084317/202412/1084317-20241217144427362-1784058711.png)


核心逻辑在ProcessCLRException中，它会判断 Thread 是否挂了异常？没有的话就会通过MapWin32FaultToCOMPlusException来转换，然后通过 pThread.SafeSetThrowables 塞入到线程里。从而实现了硬件异常在托管堆上的映射。


### 眼见为实


上源码
[https://github.com/dotnet/runtime/blob/main/src/coreclr/vm/excep.cpp](https://github.com)
![image](https://img2024.cnblogs.com/blog/1084317/202412/1084317-20241217144838561-1601033874.png)


# .NET 异常处理流程


对.NET Runtime来说，主要实现以下四个操作


1. 捕获异常并抛出异常的位置
2. 通过线程栈空间获取异常调用栈
线程的栈空间维护了整个调用栈，扫描整个栈空间即可获取。



> windbg的k系列命令就是参考此原理。


3. 获取元数据的异常处理表
一旦方法中有try\-catch语句块时，JIT会将try\-catch的适用范围记录下来，并整理成异常处理表(Execption Handling Table , EH Table)



C\# 代码

```
    public class ExceptionEmample
    {
        public static void Example()
        {
			try
			{
                Console.WriteLine("Try outer");
				try
				{
                    Console.WriteLine("Try inner");
                }
				catch (Exception)
				{ 
                    Console.WriteLine("Catch Expception inner");
                }
            }
			catch (ArgumentException)
			{
                Console.WriteLine("Catch ArgumentException outer");
            }
            catch (Exception)
            {
                Console.WriteLine("Catch Exception outer");
            }
            finally
            {
                Console.WriteLine("Finally outer");
            }
        }
    }

```



IL代码

```
.method public hidebysig static void  Example() cil managed
{
  // Code size       96 (0x60)
  .maxstack  1
  IL_0000:  nop
  IL_0001:  nop
  IL_0002:  ldstr      "Try outer"
  IL_0007:  call       void [System.Console]System.Console::WriteLine(string)
  IL_000c:  nop
  IL_000d:  nop
  IL_000e:  ldstr      "Try inner"
  IL_0013:  call       void [System.Console]System.Console::WriteLine(string)
  IL_0018:  nop
  IL_0019:  nop
  IL_001a:  leave.s    IL_002c
  IL_001c:  pop
  IL_001d:  nop
  IL_001e:  ldstr      "Catch Expception inner"
  IL_0023:  call       void [System.Console]System.Console::WriteLine(string)
  IL_0028:  nop
  IL_0029:  nop
  IL_002a:  leave.s    IL_002c
  IL_002c:  nop
  IL_002d:  leave.s    IL_004f
  IL_002f:  pop
  IL_0030:  nop
  IL_0031:  ldstr      "Catch ArgumentException outer"
  IL_0036:  call       void [System.Console]System.Console::WriteLine(string)
  IL_003b:  nop
  IL_003c:  nop
  IL_003d:  leave.s    IL_004f
  IL_003f:  pop
  IL_0040:  nop
  IL_0041:  ldstr      "Catch Exception outer"
  IL_0046:  call       void [System.Console]System.Console::WriteLine(string)
  IL_004b:  nop
  IL_004c:  nop
  IL_004d:  leave.s    IL_004f
  IL_004f:  leave.s    IL_005f
  IL_0051:  nop
  IL_0052:  ldstr      "Finally outer"
  IL_0057:  call       void [System.Console]System.Console::WriteLine(string)
  IL_005c:  nop
  IL_005d:  nop
  IL_005e:  endfinally
  IL_005f:  ret
  IL_0060:  
  // Exception count 4
  .try IL_000d to IL_001c catch [System.Runtime]System.Exception handler IL_001c to IL_002c
  .try IL_0001 to IL_002f catch [System.Runtime]System.ArgumentException handler IL_002f to IL_003f
  .try IL_0001 to IL_002f catch [System.Runtime]System.Exception handler IL_003f to IL_004f
  .try IL_0001 to IL_0051 finally handler IL_0051 to IL_005f
} // end of method ExceptionEmample::Example

```


IL代码中最后4行就代表了方法的异常处理表。



```
1. IL_000d to IL_001c 之间代码发生的Exception异常由IL_001c to IL_002c 之间的代码处理
2. IL_0001 to IL_002f 之间发生的ArgumentException异常由IL_002f to IL_003f之间的代码处理
3. IL_0001 to IL_002f 之间发生的Exception异常由IL_003f to IL_004f之间的代码处理
4. IL_0001 to IL_0051 之间无论发生什么，结束后都要执行IL_0051 to IL_005f之间的代码

```

4. 枚举异常处理表，调用对应的catch块与finally块
当异常发生时，Runtime会枚举EH Table，找出并调用对应的catch块与finally块。
核心方法为ProcessManagedCallFrame：
![image](https://img2024.cnblogs.com/blog/1084317/202412/1084317-20241217154337944-2105781699.png)



> [https://github.com/dotnet/runtime/blob/main/src/coreclr/vm/exceptionhandling.cpp](https://github.com)


需要注意的是，一旦CLR找到catch块，就会先执行内层所有finally块中的代码，再等到当前catch块中的代码执行完毕finally才会执行


5. 重新抛出异常
在执行catch,finally的过程中，如果又抛出了异常。程序会再次进入ProcessCLRException中走重复流程。
但是调用链会消失，如果想要防止调用链丢失，需要特殊处理。



```
        static void Main(string[] args)
        {
            try
            {
                Test();
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex);
            }
        }

		private static void Test()
		{
            try
            {
                throw new Exception("test");
            }
            catch (Exception ex)
            {
                //throw ex; //会丢失调用链,找不到真正的异常所在
                //throw; //调用链完整
                //ExceptionDispatchInfo.Capture(ex).Throw();//调用链更完整，显示了重新抛出异常所在的位置。
            }
        }

```


> 我在这里踩过大坑，使用throw ex重新抛出异常，结果丢失了异常真正的触发点，日志跟没记一样。


## finally一定会执行吗?


常规情况下，finally是保证会执行的代码，但如果直接用win32函数TerminateThread杀死线程，或使用System.Environment的Failfast杀死进程，finally块不会执行。


## 先执行return还是先执行finally?



C\#代码

```
~~~
        public static int Example2()
        {
            try
            {
                return 100+100;
            }
            finally
            {
                Console.WriteLine("finally");
            }
        }
~~~

```



IL代码

```
.method public hidebysig static int32  Example2() cil managed
{
  // Code size       22 (0x16)
  .maxstack  1
  .locals init (int32 V_0)
  IL_0000:  nop
  IL_0001:  nop
  IL_0002:  ldc.i4.1  //将100+100的值，压入Evaluation Stack
  IL_0003:  stloc.0   //从Evaluation Stack出栈，保存到序号为0的本地变量
  IL_0004:  leave.s   IL_0014 //退出代码保护区域，并跳转到指定内存区域IL_0014， 指令 leave.s 清空计算堆栈并确保执行相应的周围 finally 块。
  IL_0006:  nop
  IL_0007:  ldstr      "finally"
  IL_000c:  call       void [System.Console]System.Console::WriteLine(string)
  IL_0011:  nop
  IL_0012:  nop
  IL_0013:  endfinally
  IL_0014:  ldloc.0 //读取序号0的本地变量并存入Evaluation Stack
  IL_0015:  ret  //从方法返回，返回值从Evaluation Stack中获取
  IL_0016: 
  // Exception count 1
  .try IL_0001 to IL_0006 finally handler IL_0006 to IL_0014
} // end of method ExceptionEmample::Example2

```


从IL中可以看到，当try中包含return语句时，编译器会生成一个临时变量将返回值保存起来。然后再执行finally块。最后再return 临时变量。这个过程称为局部展开(local unwind)


再举一个例子



C\#代码

```
        public static int Test()
        {
			int result = 1;
			try
			{
				return result;
			}
			finally
			{
				result = 3;
			}
        }

```



IL代码

```
.method public hidebysig static int32  Test() cil managed
{
  // 代码大小       15 (0xf)
  .maxstack  1
  .locals init (int32 V_0,
           int32 V_1)
  IL_0000:  nop
  IL_0001:  ldc.i4.1  //将常量1压栈
  IL_0002:  stloc.0   //将序号0出栈，赋值给result
  IL_0003:  nop
  IL_0004:  ldloc.0  //将当前方法序号0的变量，也就是result,压入栈中。
  IL_0005:  stloc.1  //将序号1的值出栈，保存到一个临时变量中。也就是return的值
  IL_0006:  leave.s    IL_000d   //跳转到对应行, 指令 leave.s 清空计算堆栈并确保执行相应的周围 finally 块。
  IL_0008:  nop
  IL_0009:  ldc.i4.3   
  IL_000a:  stloc.0
  IL_000b:  nop
  IL_000c:  endfinally
  IL_000d:  ldloc.1  //将return的值 入栈
  IL_000e:  ret  //执行return
  IL_000f:  
  // Exception count 1
  .try IL_0003 to IL_0008 finally handler IL_0008 to IL_000d
} // end of method Class1::Test



```


虽然在finally块中修改了result的值，但是return语句已经确定了要返回的值，finally块中的修改不会改变这个返回值。不过，如果返回的是引用类型），在finally块中修改引用类型对象的内容是会生效的
# 异常对性能的影响


引用别人的数据，自己就不班门弄斧了


1. 大佬的研究
[https://github.com/huangxincheng/p/12866824\.html](https://github.com)
2. \<.NET Core底层入门\>
![image](https://img2024.cnblogs.com/blog/1084317/202409/1084317-20240904101045039-12905218.png)



> 总体来说，只要进入内核态。就没有开销低的。


# CLS与非CLS异常(历史包袱)


在CLR的2\.0版本之前，CLR只能捕捉CLS相容的异常。如果一个C\#方法调用了其他编程语言写的方法，且抛出一个非CLS相容的异常。那么C\#无法捕获到该异常。
在后续版本中，CLR引入了RuntimeWrappedException类。当非CLS相容的异常被抛出时,CLR会自动构造RuntimeWrappedException实例。使之与与CLS兼容



```
        public static void Example2()
        {
            try
            {

            }
            catch(Exception)
            {
                //c# 2.0之前这个块只能捕捉CLS相容的异常
            }
            catch
            {
                //这个块可以捕获所有异常
            }
        }

```

