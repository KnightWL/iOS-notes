#### block探究（一） block的本质是什么  
1. 创建一个新的命令行工程
2. 在main函数中定义一个int值age 与 block值myblock
```
int age = 50;
void(^myBlock)(int, int) = ^(int a, int b) {
    NSLog(@"age is %d", age);
    NSLog(@"a is %d", a);
    NSLog(@"b is %d", b);
};
```
3. 将main.m通过 `xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m`编译成cpp代码
4. 在cpp代码中查找myblock，发现上述代码转换成cpp代码如下  
```
int age = 50;
        void(*myBlock)(int, int) = ((void (*)(int, int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, age));

        ((void (*)(__block_impl *, int, int))((__block_impl *)myBlock)->FuncPtr)((__block_impl *)myBlock, 5, 10);
```  
    
> 结论1：**可以看出，定义myblock时，在cpp代码中myblock是一个 __main_block_impl_0指针指向的对象； 而调用myblock时，它是一个__block_impl** * **指针指向的对象。这两个数据类型的代码如下：**  
```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int age;
};
```  
  
```
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};
```  
> 结论2：综上，可以看出这两个数据类型都是结构体，且 *__main_block_impl_0* 包含了 *__block_impl*， 而 *__block_impl* 的第一条数据是一个 void* isa指针，由此可见 **__main_block_impl_0与__block_impl 在OC代码中均以对象形式存在**，并且__main_block_impl_0的最后一个成员变量是int age，所以block其实把用到的外部变量也拷贝了一份到自己的代码中。  
  
而在调用myblock时，使用了__block_impl结构体的 _void *FuncPtr_ 指针，所以可以猜测：_void *FuncPtr_ 指向的其实是内存中一段函数的地址。接下来我们来尝试证实。  
在main.m中添加如下代码，并在对应位置打上断点：  
![img](http://wl-ios-blog-images.oss-cn-beijing.aliyuncs.com/iOS%E5%BA%95%E5%B1%82/block_main_cpp.png)    

在运行到 `myBlock(5,10)`这个断点时，可以看到block结构为  
![img](http://wl-ios-blog-images.oss-cn-beijing.aliyuncs.com/iOS%E5%BA%95%E5%B1%82/block_main_struct.png)    

获得了FuncPtr的指向地址为0x0000000100000ed0，也验证了结论2。  
继续运行，走到代码中第一个断点时，xcode->debug->debug workflow->always Show Disassembly查看汇编代码，可以看到第一个断点即myBlock内部调用的第一行内存地址为0x0000000100000ed0  
![img](http://wl-ios-blog-images.oss-cn-beijing.aliyuncs.com/iOS%E5%BA%95%E5%B1%82/block_main_disasse.png)  
> 结论3：**_void *FuncPtr_ 指向的是block内部的代码实现地址。**  
  
>最终，可以得出， **block是封装了函数实现及其调用环境的对象。**
  
author: wanglei