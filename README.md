### Block的本质
#### 一.block截获自动变量
    //申明一个blcok类型的变量 其可做以下使用:自动变量 函数参数 静态变量  静态全局变量 全局变量
    const char *text = "hello";
    //在现在的block中,截获自动变量的方法并没有实现对C语言的数据的截获,可以使用指针可以解决该问题
    //自动变量的截获
    void (^block)(void) = ^{
        printf("%c\n",text[2]);
    };
    block();
     
    NSMutableArray *arr = [NSMutableArray array];
    void (^block)(void) = ^{
        [arr addObject:[[NSObject alloc] init]];//这样是没有问题的
        //arr = [NSMutableArray array]; //向截获的变量array赋值则会产生编译错误
    };
    block();
#### 二.block的本质
    block是带有自动变量的匿名函数
    clang -rewrite-objc 文件名
    
    int main(int argc, const char * argv[]) {
    void (^blk)() = ^{
        printf("djdj");
    };
    blk();
    return 0;
    }
    
    编译成main.cpp 如下:
    
    struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
    };
    
    struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;//用于初始化__block_impl结构体的isa成员
    impl.Flags = flags;
    #将__main_block_func_0函数指针赋值给成员变量FuncOtr
    impl.FuncPtr = fp;
    Desc = desc;
     }
    };
    
    static void __main_block_func_0(struct __main_block_impl_0 *__cself) {

        printf("djdj");
    }
    
    static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
    } __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
    
    #第一个参数是由block语法转换的C语言函数指针 __main_block_impl_0 
    #第二个参数是作为静态全局变量初始化的__main_block_desc_0结构体实例指针
    void (*blk)() = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
    
    #去掉转换部分  (*blk->FuncPtr)(blk); 使用函数指针调用函数
    ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
    
    
    return 0;
    }
    
#### 三.block获取自动变量
    struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    int a;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _a, int flags=0) : a(_a) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
     }
    };
    在__main_block_impl_0结构体实例中,自动变量值会被截获.
    所谓"截获自动变量值"是指在执行block语法时,block语法表达式所使用的自动变量值被保存到block的结构体实例中
    
#### 四.__block说明符
     int a = 10;
    void (^blk)() = ^{
        a = 100;
        printf("djdj %d",a);
    };
    blk();
    在编译器在编译过程中检出给被截获自动变量赋值的操作时,便产生编译错误
    解决这个问题有两种方法第一种:C语言中有一个变量,允许block改写值,具体如下:
     *静态变量
     *静态全局变量
     *全局变量
    
    如下该源码使用了block改写静态变量static_val,静态全局变量static_global_val,全局变量global_val
    
    int global_val = 1;
    static int static_global_val = 2;

    int main(int argc, const char * argv[]) {
    static int static_val = 3;
    void (^blk)() = ^{
        global_val *=1;
        static_global_val *=2;
        static_val *=3;
    };
    blk();
    printf("%d,%d,%d\n",global_val,static_global_val,static_val);
    return 0;
    }
    该源码转换后:
    static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    int *static_val = __cself->static_val; // bound by copy

        global_val *=1;
        static_global_val *=2;
        (*static_val) *=3;
        printf("djdj %d",global_val);
    }
    
    int main(int argc, const char * argv[]) {
    static int static_val = 3;
    void (*blk)() = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, &static_val));
    ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
    return 0;
    }
    
    对静态全局变量static_global_val和全局变量global_val的访问与转换前完全相同,
    但静态变量static_val的指针对其进行访问(这是超出作用域使用变量的最简单方法)
    实际上,在block语法生成的值blcok上,可以存有超过其变量作用域的被截获对象的自动变量.
    变量作用域结束的同时,原来的自动变量被废弃,因此block中超过变量作用域而存在的变量
    同静态变量一样,将不能通过指针访问原来自动变量
    
    __block存储域类型说明符
    //C语言中有以下存储域类型说明符:typedef extern static auto register 它们用于将指定变量值设置到哪个存储域中
    
    __block int a = 10;
    源代码转换为:
    struct __Block_byref_a_0 {
    void *__isa;
    __Block_byref_a_0 *__forwarding;//持有指向该实例自身的指针,通过__forwarding访问成员变量a
    int __flags;
    int __size;
    int a;
    };
    变成了一个结构体实例,__block变量也同Block一样变成__Block_byref_a_0结构体类型的自动变量

    struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    __Block_byref_a_0 *a; // __main_block_impl_0持有__Block_byref_a_0结构体实例的指针
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_a_0 *_a, int flags=0) : a(_a->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
     }
    };
    
    note:__Block_byref_a_0 结构体并不在block的__main_block_impl_0结构体,中这样做的原因是因为多个block变量使用
    
#### 五.Block存储域
    Block转换为Block的结构体类型的自动变量 ,__block变量转换成__block变量的结构体类型的自动变量,
    "结构体类型的自动变量:即栈上生成的该结构体的实例"??
    Block也是Objective-C对象 ,该Block的类为_NSConcreteStackBlock(还有的类如:_NSConcreteGlobalBlock(全局),_NSConcreteMallocBlock(堆))
    
    "__block变量用结构体成员变量__forwarding可以实现无论__block变量配置在栈上还是堆上时都能正确访问__block变量"
    
    *将Block作为函数返回值时,编译器会自动生成复制到堆上的代码
     1.将通过Block语法生成的Block,即配置在栈上的Block用结构体实例赋值给相当于Block类型的变量tmp中
     2.tmp = _Block_copy(tmp)
       _Block_copy函数 将栈上的Block复制到堆上,复制后,将堆上的地址作为指针赋值给变量tmp
    
     3.return objc_autoreleaseReturnValue(tmp)
       将堆上的Block作为Objective-C对象 注册到autoreleasepool中,然后返回该对象
    
    *向方法或函数的参数中传递Block时
    
   名称 | 实质
   ---|---
   Block |栈上Block的结构体实例
   __block | 栈上 __block的结构体实例
   
   
   Block的副本
   Block的类 | 副本源的配置储存域 | 复制效果
   ---|--- |----
   _NSConcreteStackBlock  |    栈 | 从栈复制到堆
   _NSConcreteGlobalBlock |  程序的数据区 | 不做任何事
   _NSConcreteMallocBlock | 堆  | 引用计数增加
   
#### 六.__block变量存储域
    1.当一个Block钟使用__block变量 ,当Block变量从栈复制到堆时,使用的所有的__block变量也会从栈上复制到堆上
    2.在多个Block中使用__block变量时,当任何一个Block从栈复制到堆区时,__block变量也会一并从栈复制到堆并被该Block所持有,
    当剩下的Block从栈复制到堆时,被复制的Block持有__block变量,并增加__block变量的引用计数
![image](https://raw.githubusercontent.com/CathyLy/imageForSource/master/%E5%A4%8D%E5%88%B6__block%E5%8F%98%E9%87%8F.png)

#### 七.截获对象
    blk_t blk;
    {
        id array = [NSMutableArray array];
        blk = [^(id obj){
            [array addObject:obj];
            NSLog(@"array count = %ld",[array count]);
        } copy];
    }
    
    blk([[NSObject alloc] init]);
    blk([[NSObject alloc] init]);
    
    终端输出如下:
    2017-03-24 16:55:30.922 Object-C 多线程和内存管理[82607:1623320] array count = 1
    2017-03-24 16:55:30.923 Object-C 多线程和内存管理[82607:1623320] array count = 2
    
    意味着赋值给array的NSMutableArray类的对象在该源代码最后BLock的执行部分超出其变量作用域而存在
    struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    id __strong array;
    };
    
    含有__strong修饰符的对象类型变量array 
    在__main_block_desc_0中增加了一下两个结构体:
    static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {
    _Block_object_assign(&dst->array, src->a, BLOCK_FIELD_IS_BYREF); //相当于retain实例方法的函数 ,将对象赋值在对象类型的结构体成员变量中
    }
    
    static void __main_block_dispose_0(struct __main_block_impl_0*src) {
    _Block_object_dispose((void*)src->a, BLOCK_FIELD_IS_BYREF);//用来释放在Block用结构体成员变量array中的对象,相当于调用release
    }
 
    下面情况会将栈上的Block复制到堆上:
    1.调用Block的copy实例方法
    2.Block作为函数返回值返回时
    3.将Block 赋值给附有__strong修饰id类型的类或Block类型的成员变量
    4.在方法命中含有usingBlock的Cocoa框架方法或者GCD的API中传递Block
   
 八.Block循环引用
    typedef void (^blk_t) (id);
    @interface MyObject ()
    {
        blk_t blk;
    }
    @end
    @implementation MyObject
    - (void) test{
        id array = [NSMutableArray array];
        blk = [^(id obj){
            [array addObject:obj];
            NSLog(@"%@",self);
        } copy];
    
    blk([[NSObject alloc] init]);
    blk([[NSObject alloc] init]);
    }
   
    上面的源码会导致循环引用
    可以使用如下的方式解决循环引用:
    1.id __weak weakSelf = self;
      id __unsafe_unretained weakSelf = self;
    2.还可以使用__block避免循环引用
      使用__block的变量的有点如下:
      a.通过__block变量可控制对象的持有期间
      b.在不能使用__weak修饰符的环境中不使用__unsafe_unretained即可(不必担心悬垂指针)
      c.在执行Block时可动态地决定是否将nil或其他对象赋值在__block变量中
      d.为避免循环引用必须执行Block

    在ARC无效的时候.__block说明符被用来避免Block中的循环引用,这是因为从栈复制到堆上,
    若使用Block使用的变量为附有__block说明符的对象类型的自动变量,不会被retain
   
   

    
#### 类与对象
    Class 为objc_class结构体的指针类型 
    typedef struct objc_class *Class; 
    
    struct objc_class {
    Class isa ;
    }
    
    struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
    };
    
    objc_object和objc_class结构体归根到底都是在各个对象和类的实现中使用最基本的结构体
    各类的结构体就是基于objc_class结构体的class_t结构体
    在Objective-C中,比如NSObject的class_t结构体实例以及NSMutableArray的class_t结构体实例等,
    均生成并保持各个类的class_t结构体实例,该实例持有声明的成员变量,方法的名称,方法的实现(即函数指针),
    属性以及父类的指针并被Object_C运行时库所使用
    
    

    
    

    



