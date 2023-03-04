[toc]



https://releases.llvm.org/3.5.0/tools/clang/docs/ThreadSafetyAnalysis.html







# clang3.5线程安全分析



# 名词

Program point：程序点。程序执行的位置。

False negative：假阴 看起来没问题，其实有问题。

False positive：假阳 看起来有问题，其实没问题。

Inter-procedural：过程间的

Intra-procedural：过程内的



# 简介

Clang线程安全分析是c++的一个语言扩展功能，用来对代码中的竞态条件作出警告。分析完全是静态的（只消耗编译时间），没有运行时开销。这项功能由Google开发，虽然目前还在开发中，但已经足够成熟，完全可以在生产环境下使用，Google内部就广泛地使用着。

在多线程程序中，线程安全分析就像类型系统一样。除了声明数据的类型（比如说`int`, `float`)外，码农还可以声明如何控制数据的访问。打个例子，如果`foo`由互斥量`mu`守卫，在没有锁定`mu`的情况下读写`foo`，线程安全分析会报一个警告。同样地，如果存在一些特定的例程（routines），这些例程只能由GUI线程调用，而其他线程调用了这些例程，那线程安全分析也会报警告。



# 开始使用

```c++
#include "mutex.h"

class BankAccount {
private:
  Mutex mu;
  int   balance GUARDED_BY(mu);

  void depositImpl(int amount) {
    balance += amount;       // WARNING! Cannot write balance without locking mu.
  }

  void withdrawImpl(int amount) EXCLUSIVE_LOCKS_REQUIRED(mu) {
    balance -= amount;       // OK. Caller must have locked mu.
  }

public:
  void withdraw(int amount) {
    mu.Lock();
    withdrawImpl(amount);    // OK.  We've locked mu.
  }                          // WARNING!  Failed to unlock mu.

  void transferFrom(BankAccount& b, int amount) {
    mu.Lock();
    b.withdrawImpl(amount);  // WARNING!  Calling withdrawImpl() requires locking b.mu.
    depositImpl(amount);     // OK.  depositImpl() has no requirements.
    mu.Unlock();
  }
};
```

上面的例子演示了安全分析背后基本概念。`GUARDED_BY`属性声明：线程必须先锁定`mu`，才能读写`balance`，由此保证增减操作是原子性的。类似地，`EXCLUSIVE_LOCKS_REQUIRED`属性声明：线程必须先锁定`mu`，才能调用`withdrawImpl`。因为调用者锁定了`mu`，在方法（method）内就可以安全地修改`balance`。

<br />

`depositImpl()`方法没有`EXCLUSIVE_LOCKS_REQUIRED`属性，安全分析会报一个警告。因为安全分析不是过程间分析的（inter-procedural），所以调用要求必须显示声明。（意思是编译器不会联系上下文去分析这个方法需要加上什么属性，也不会自动帮你加上，所以需要显示地手动加上）`transferFrom()`里也会报一个警告，因为尽管锁定了`this->mu`，但并没有锁定`b.mu`，安全分析知道这是两个不同的互斥量。

<br />

`withdraw()`方法里也有一个警告，因为它没有解锁`mu`。每个锁定都必须有一个对应的解锁，安全分析会分析每对加锁/解锁。一个函数允许获取一个锁而不用释放这个锁（反过来也一样），但必须要标上属性，用`LOCK/UNLOCK_FUNCTION`。



# 运行分析

使用`-Wthread-safety`标志

```shell
clang -c -Wthread-safety example.cpp
```



# 基本概念：能力（Capabilities）

线程安全分析使用*能力*，提供了一种保护资源的方法。这里所说的资源可以是数据成员，或者一个访问底层资源的方法/函数。安全分析确保了调用线程不能读写数据，不能调用函数，除非线程有能力。

<br />

能力与C++命名对象关联，命名对象声明了特定的方法来获取和释放能力。对象的名字用来识别这种能力。最常见的例子就是互斥量`mutex`。如果`mu`是互斥量，数据由`mu`保护，调用`mu.Lock()`会使得线程获取访问数据的能力。同样地，调用`mu.Unlock()`则释放这种访问数据的能力。

<br />

一个线程可能持有排它或共享这两种能力之一。排它能力一次只能由一个线程持有，而共享能力则可以由多个线程在同一时间持有。这是一种多读单写机制。读取受保护的数据只需要共享能力，写则要求排它。

<br />

在程序执行的过程中，线程持有一连串能力（比如锁定的所有互斥量），这些能力就像一串钥匙，线程拿着这些钥匙打开访问资源的大门。线程不能拷贝这些能力，也不能销毁它们，而只能从另一个线程那里获取这些能力，释放这些能力给另一个线程。标注并不知道这套获释能力的机制具体是怎么实现的，它假定了底层实现（互斥量的实现）以一种合适的方式转移能力。（意思是，标注不管实现，具体锁是怎么转移的，由`Mutex`实现，标注只管用就行了）

<br />

在程序运行过程中，线程持有的一连串能力是一个运行时概念。静态的安全分析计算出这些能力的一个大概值，叫做能力环境。在每个程序点，都会计算一下能力环境，并且描述那些静态持有或者不持有的能力。这个能力环境就构成了线程运行时的最小能力簇。（意思大概是，安全分析只能静态分析线程持有哪些`mutex`，分析不了动态持有的，所有这些静态分析得到的`mutex`就是线程运行时最少要获取的）



# 参考指南

线程安全分析使用属性来声明线程约束。属性必须附于命名的声明语句，例如类`class`、方法`method`和数据成员`data member`。强烈建议码农使用宏定义不同的属性，宏定义的例子能在[*mutex.h*](https://releases.llvm.org/3.5.0/tools/clang/docs/ThreadSafetyAnalysis.html#mutexheader)找到。下文都假定用宏定义。

## GUARDED_BY(c) 和 PT_GUARDED_BY(c)

`GUARDED_BY`属性用在数据成员上，声明该数据成员受给定的能力保护。读该数据需要共享访问，写操作需要排它访问。

`PT_GUARDED_BY`属性作用类似，只不过是用在普通指针和智能指针。指针本身没有约束，指针指向的数据受能力保护。

```c++
Mutex mu;
int *p1            GUARDED_BY(mu); // p1受mu保护
int *p2            PT_GUARDED_BY(mu); // p2指向的数据受mu保护
unique_ptr<int> p3 PT_GUARDED_BY(mu); // p3指向的数据受mu保护

void test() {
  p1 = 0;             // Warning! p1本身的值受mu保护，需要锁定mu才行

  p2 = new int;       // OK.   p2随便可以更改指向，没问题
  *p2 = 42;           // Warning!  但修改指向的数据就不行

  p3.reset(new int);  // OK.   p3与p2一样
  *p3 = 42;           // Warning!
}
```



## EXCLUSIVE_LOCKS_REQUIRED(...), SHARED_LOCKS_REQUIRED(...)

`EXCLUSIVE_LOCKS_REQUIRED`属性用在方法和函数上，表明调用线程必须拥有排它能力。可以指定多个能力。在进入函数前，必须拥有所有指定的排它能力，并且退出函数时依然持有这些能力。

`SHARED_LOCKS_REQUIRED`属性只要求共享访问能力。

```c++
Mutex mu1, mu2;
int a GUARDED_BY(mu1);
int b GUARDED_BY(mu2);

// 调用这个函数需要锁定mu1、mu2
void foo() EXCLUSIVE_LOCKS_REQUIRED(mu1, mu2) {
  a = 0;
  b = 0;
}

void test() {
  mu1.Lock();
  
  // 锁定了mu1, 但没锁定mu2，报警告
  foo();         // Warning!  Requires mu2.
  
  // 退出函数会依然持有mu1、mu2, 所以需要手动解锁
  mu1.Unlock();
}
```



## EXCLUSIVE_LOCK_FUNCTION(...), SHARED_LOCK_FUNCTION(...), UNLOCK_FUNCTION(...)

`EXCLUSIVE_LOCK_FUNCTION`属性用在方法和函数上，表明函数体内获取能力，且不释放。调用者在进入函数前不能持有该能力，退出时会持有该能力。`SHARED_LOCK_FUNCTION`类似。

`UNLOCK_FUNCTION`属性表明函数释放能力。调用者在进入函数前必须持有能力，函数退出时不再持有该能力，不管能力是共享还是排它，调用都都不再持有。

```c++
Mutex mu;
MyClass myObject GUARDED_BY(mu);

// 属性表明: 调用前不能锁定mu，在函数体内锁定mu，函数退出不释放mu
// 满足属性
void lockAndInit() EXCLUSIVE_LOCK_FUNCTION(mu) {
  mu.Lock();
  myObject.init();
}

// 属性表明: 调用前已经锁定了mu, 函数退出应该释放能力
// 这里没有释放, 不满足属性, 报警
void cleanupAndUnlock() UNLOCK_FUNCTION(mu) {
  myObject.cleanup();
}  // Warning!  Need to unlock mu.

void test() {
  lockAndInit();
  myObject.doSomething();
  cleanupAndUnlock();
  
  // 假设前面都没有警告, 执行到这里, 线程已经释放了mu, 访问受mu保护的myObject会报警
  myObject.doSomething();  // Warning, mu is not locked.
}
```

如果没有往`EXCLUSIVE_LOCK_FUNCTION`、`UNLOCK_FUNCTION`传参数，那默认参数是`this`，此时安全分析不会检查函数体。这种场景适合类用抽象接口来隐藏锁的细节。

```c++
template <class T>
class LOCKABLE Container {
private:
  Mutex mu;
  T* data;

public:
  // Hide mu from public interface.
  void Lock() EXCLUSIVE_LOCK_FUNCTION() { mu.Lock(); }
  void Unlock() UNLOCK_FUNCTION() { mu.Unlock(); }

  T& getElem(int i) { return data[i]; }
};

void test() {
  Container<int> c;
  c.Lock();
  int i = c.getElem(0);
  c.Unlock();
}
```



## LOCKS_EXCLUDED(...)

`LOCKS_EXCLUDED`属性用在方法和函数上，表明调用者在进入函数时，不能持有该能力。这个属性用来防止死锁。大部分互斥量都不是可重入的，所以如果函数再次尝试获取`mu`，则可能发生死锁。

```c++
Mutex mu;
int a GUARDED_BY(mu);

// 至于函数体内是怎么样的, 则不管
void clear() LOCKS_EXCLUDED(mu) {
  mu.Lock();
  a = 0;
  mu.Unlock();
}

void reset() {
  mu.Lock();
  
  // 调用函数前, 不能锁定mu，报警
  clear();     // Warning!  Caller cannot hold 'mu'.
  mu.Unlock();
}
```

不像LOCKS_REQUIRED，LOCKS_EXCLUDED是可选的。



## NO_THREAD_SAFETY_ANALYSIS

`NO_THREAD_SAFETY_ANALYSIS`属性用在方法和函数上，用来关闭该函数的线程安全检查。如果线程安全函数或线程不安全函数逻辑太复杂，不想让这个函数执行安全分析，那就用这个属性。

```c++
class Counter {
  Mutex mu;
  int a GUARDED_BY(mu);

  void unsafeIncrement() NO_THREAD_SAFETY_ANALYSIS { a++; }
};
```



## LOCK_RETURNED(c)

`LOCK_RETURNED`属性用在方法和函数上，用来返回一个能力的引用。

```c++
class MyClass {
private:
  Mutex mu;
  int a GUARDED_BY(mu);

public:
  Mutex* getMu() LOCK_RETURNED(mu) { return &mu; }

  // analysis knows that getMu() == mu
  void clear() EXCLUSIVE_LOCKS_REQUIRED(getMu()) { a = 0; }
};
```



## ACQUIRED_BEFORE(...), ACQUIRED_AFTER(...)

`ACQUIRED_BEFORE`和`ACQUIRED_AFTER`属性用在成员声明上，特别是用在互斥量的声明上。属性规定了互斥量必须以特定的顺序获取，以防死锁。

```c++
Mutex m1;
Mutex m2 ACQUIRED_AFTER(m1);

// Alternative declaration
// Mutex m2;
// Mutex m1 ACQUIRED_BEFORE(m2);

void foo() {
  m2.Lock();
  // m1应该先于m2获取, 报警. 注意报警位置, 是报在m1, 没有报在m2.Lock()
  m1.Lock();  // Warning!  m2 must be acquired after m1.
  
  m1.Unlock();
  m2.Unlock();
}
```



## LOCKABLE

`LOCKABLE`属性用在类上，表明该类可用作能力(也就是互斥量`mutex`的作用)。参考上面的`Container`例子，或者[*mutex.h*](https://releases.llvm.org/3.5.0/tools/clang/docs/ThreadSafetyAnalysis.html#mutexheader)里的`Mutex class`。



## SCOPED_LOCKABLE

`SCOPED_LOCKABLE`属性用在实现了RAII风格（Resource Acquire Is Initialization 获取资源即初始化）锁的类上，在这样的类实现里，能力在构造函数里获取，在析构函数里释放。这样的类需要特殊处理，因为构造函数和析构函数通过不同的名字引用能力。可参考[*mutex.h*](https://releases.llvm.org/3.5.0/tools/clang/docs/ThreadSafetyAnalysis.html#mutexheader)里的`MutexLocker class`。



## EXCLUSIVE_TRYLOCK_FUNCTION(<bool>, ...), SHARED_TRYLOCK_FUNCTION(<bool>, ...)

用在方法和函数上，表明方法函数尝试去获取能力，并返回一个`bool`值表示获取成功或失败。第一个参数必须是`true`或`false`，用来指定获取成功的返回值，剩下的参数就跟`(UN)LOCK_FUNCTION`属性一样。使用例子参考[*mutex.h*](https://releases.llvm.org/3.5.0/tools/clang/docs/ThreadSafetyAnalysis.html#mutexheader)。



## ASSERT_EXCLUSIVE_LOCK(...) and ASSERT_SHARED_LOCK(...)

用在方法和函数上，用来做运行时测试，看调用线程是否持有能力。如果线程未持有能力，函数调用失败(无返回)。例子参考[*mutex.h*](https://releases.llvm.org/3.5.0/tools/clang/docs/ThreadSafetyAnalysis.html#mutexheader)。



## GUARDED_VAR and PT_GUARDED_VAR

弃用了，不用看。



## 警告标志(Warning flags)

- -Wthread-safety`: 伞标，相当于打开了下面三个标志：
  - `-Wthread-safety-attributes`: 检查属性完整性;
  - `-Wthread-safety-analysis`: 执行安全分析;
  - `-Wthread-safety-precise`: 要求互斥量表达式精确匹配，如果代码里有很多别名，这个警告能被禁用;

当有新功能、新的检查被加到安全分析，也会有产生新的警告，这些一开始是作为beta警告，之后才会加入到正式版本的安全分析。

- -Wthread-safety-beta: 新功能，默认是off;



# FAQ

Q. 属性是放在头文件里好，还是.cc、.cpp、.cxx文件里好？

A. 属性呆在头文件里好。

Q. ”互斥量并不锁定在穿过这里的每条道路上(*Mutex is not locked on every path through here*)“ 这句话什么意思？

A. 参看[*No conditionally held locks.*](https://releases.llvm.org/3.5.0/tools/clang/docs/ThreadSafetyAnalysis.html#conditional-locks)。



# 安全分析的局限

## 词法范围

线程安全属性包含常见的c++表达式，因此遵循表达式的范围规则。也就是说，互斥量`mutex`必须先声明，再在属性上使用。在一个单独的类里，互斥量可以在声明前使用，因为属性是和方法体一起解析的，而c++会滞后解析方法体，直到类的定义结束，此时编译器其实掌握了类的所有信息了，自然就包括了互斥量的声明，这也没违背先声明再使用原则。在类之间，则不能声明前使用。

```c++
// 声明类, 类的定义在Bar的后面
class Foo;

// 编译器此时没有Foo的类详细信息, 不能访问其成员
class Bar {
  void bar(Foo* f) EXCLUSIVE_LOCKS_REQUIRED(f->mu);  // Error: mu undeclared.
};

class Foo {
  Mutex mu;
};
```



## 私有互斥量

好的软件工程实践要求互斥量应该是私有成员，因为线程安全类使用的锁机制是这个类的内部实现，不应该公开。然而，私有的互斥量有时候会通过公开接口泄露。线程安全属性遵循c++访问限制，如果`mu`是对象`c`的一个私有成员，那么在属性上写`c.mu`会报错。

一个迂回办法是：用`LOCK_RETURNED`属性起一个公开的方法，实际不暴露底层的互斥量。

```c++
class MyClass {
private:
  Mutex mu;

public:
  // For thread safety analysis only.  Does not actually return mu.
  Mutex* getMu() LOCK_RETURNED(mu) { return 0; }

  void doSomething() EXCLUSIVE_LOCKS_REQUIRED(mu);
};

void doSomethingTwice(MyClass& c) EXCLUSIVE_LOCKS_REQUIRED(c.getMu()) {
  // The analysis thinks that c.getMu() == c.mu
  c.doSomething();
  c.doSomething();
}
```

调用`doSomethingTwice()`时，要求锁定`c.mu`，但不能直接写`c.mu`，因为`c.mu`是私有的。不鼓励用这样的方法，因为这违背了封装原则，但有时候又是必要的，特别是在现有的代码库里加标注时。总之，这个迂回方法就是定义一个`getMu`()作为一个伪`getter`方法，仅仅是用来做线程安全分析。（译者注：仅仅是用了让安全分析通过，实际代码运行不会去锁定`c.mu`）



## 引用传递假阴（False negatives on pass by reference)

当前版本的安全分析只检查通过名称直接访问数据成员的操作，如果通过指针、引用间接访问数据成员，将不会有警告产生。因此下面的代码将不会产生警告：

```c++
Mutex mu;
int a GUARDED_BY(mu);

void clear(int& ra) { ra = 0; }

void test() {
  int *p = &a;
  *p = 0;       // No warning.  *p is an alias to a.

  clear(a);     // No warning.  'a' is passed by reference.
}
```

在当前版本，这是导致假阴最大的原因。根本原因是：标注是附属在数据成员上，而不是在类型上。`&a`的类型应该是`int GUARDED_BY(mu)*`，而不是`int*`，也因此语句`p = &a`应该报错。然而，标注属性如果附属在类型上，那么将会侵入c++类型系统，造成大改，后果就是影响模板实例化、函数重载等。对于这个问题的完整解决办法暂时还没有。

未来版本的安全分析会为指针和别名提供更好的支持，同时对受保卫的数据成员的类型进行有限的检查，减少假阴次数。



## 条件分支下不持有锁(No conditionally held locks)

在每个程序点，安全分析必须能够判断一个锁是否被持有着。下面这段代码中，锁可能被持有，因此将产生一个虚假警告：（译者注：时刻注意，安全分析是静态分析，此时它判断不了b的值，也就不知道mu是否被持有，与安全分析规定相违背，所以会有警告。警告是虚假的，意思是这实际上不是问题）

```c++
void foo() {
  bool b = needsToLock();
  if (b) mu.Lock();
  ...  // Warning!  Mutex 'mu' is not held on every path through here.
  if (b) mu.Unlock();
}
```



## 不检查构造析构

安全分析不对构造、析构函数内部做检查，换句话说，这两个函数就像是标注了`NO_THREAD_SAFETY_ANALYSIS`一样。之所以不检查，是因为在初始化期间，只有一个线程有正在初始化对象的访问权限，因此不用获取锁就初始化受保卫的成员，这是安全的做法(也是惯例做法)，析构函数也是一样。

理想情况下，安全分析允许在初始化、要销毁的对象中进行对受保卫成员的初始化，而仍然要求对其他的一切要有访问限制。然而实际上很难限制，因为在复杂的基于指针的数据结构中，很难决定这个封装的对象拥有什么样的数据。



## 不内联

跟类型检查一样，安全分析是严格地逐过程分析的。它只依赖在函数上声明的属性，不会步渐式的分析，也不会内联函数调用。（译者注：静态过程分析，不会像调用函数那样动态地分析）

```c++
template<class T>
class AutoCleanup {
  T* object;
  void (T::*mp)();

public:
  AutoCleanup(T* obj, void (T::*imp)()) : object(obj), mp(imp) { }
  ~AutoCleanup() { (object->*mp)(); }
};

Mutex mu;
void foo() {
  mu.Lock();
  AutoCleanup<Mutex>(&mu, &Mutex::Unlock);
  ...
}  // Warning, mu is not unlocked.
```

在上面这个例子，`Autocleanup`的析构函数会调用`mu.Unlock()`释放`mu`，所以报的那个警告其实是不对的。然而，安全分析不会内联展开析构函数，自然看不到`Unlock()`操作，而且它也没办法标注构造函数，因为析构函数调用了一个不能从静态分析得来的未知函数。



### LOCKS_EXCLUDED不传递

函数调用标注了`LOCKS_EXCLUDED`的函数，并不要求在调用函数上也标注`LOCKS_EXCLUDED`，这将导致假阴。在这点上，`LOCKS_EXCLUDED`就跟`LOCKS_REQUIRED`不同，

```c++
class Foo {
  Mutex mu;

  void foo() {
    mu.Lock();
    // 按理说这里应该报警
    bar();                // No warning
    mu.Unlock();
  }

  void bar() { baz(); }   // No warning.  (Should have LOCKS_EXCLUDED(mu).)

  void baz() LOCKS_EXCLUDED(mu);
};
```

之所以`LOCKS_EXCLUDED`不传递，是因为它很容易破坏封装。要求函数列出恰好在内部获取的私有锁的名称是一个坏主意。（译者注：刚才那个例子，要想让`foo`函数正常报警，就要像`baz()`一样，这样写：

`void foo() LOCKS_EXCLUDED(mu);` `mu`在`foo`函数自己内部获取，而这其实是强制码农必须清楚`foo`函数的逻辑，破坏了封装。所以安全分析就没做传递）



## 不检查别名

安全分析目前不检查指针别名。因此如果两个指针指向同一个互斥量，将会假阳。

```c++
class MutexUnlocker {
  Mutex* mu;

public:
  // 调用者在进入函数前必须持有能力，函数退出时不再持有该能力
  MutexUnlocker(Mutex* m) UNLOCK_FUNCTION(m) : mu(m)  { mu->Unlock(); }
  
  // 函数体内获取能力，且不释放。调用者在进入函数前不能持有该能力，退出时会持有该能力
  ~MutexUnlocker() EXCLUSIVE_LOCK_FUNCTION(mu) { mu->Lock(); }
};

Mutex mutex;
// 在进入函数前，必须拥有所有指定的排它能力，并且退出函数时依然持有这些能力。
void test() EXCLUSIVE_LOCKS_REQUIRED(mutex) {
  {
    MutexUnlocker munl(&mutex);  // unlocks mutex
    doSomeIO();
  }                              // Warning: locks munl.mu
}
```

`MutexUnlocker`理应是是[*mutex.h*](https://releases.llvm.org/3.5.0/tools/clang/docs/ThreadSafetyAnalysis.html#mutexheader)里`MutexLocker`是二元类。然而却并不是，因为安全分析并不知道`mun1.mu == mutex`。`SCOPED_LOCKABLE`属性是处理别名的。

（译者注：把自己想像成安全分析，去静态分析那段代码，进入`test()`前，已经锁定了`mutex`，初始化对象`MutexUnlocker munl(&mutex);` 调用构造函数，因为构造函数是这样写的：`MutexUnlocker(Mutex* m) UNLOCK_FUNCTION(m) : mu(m)  { mu->Unlock(); }` 此时 `m`和`mu`两个指针指向了同一个互斥量了，调用其中一个进行解锁：`mu->Unlock()`。到现在这句为止没报警，说明安全分析对指针别名的`Unlock()`是可以识别的，一个指针`Unlock()`了，可以识别到另一个指针也`Unlock()`了。之后`test()`函数结束时，析构`MutexUnlocker()`：调用`mu->Lock()`，`mun1`对象成为垃圾。之后报警。报警的原因我猜是对象销毁，理应释放掉所有的互斥量，但`mun1`没有释放，才报的警。而其实，这个报警是没必要的，因为调用完`test()`，线程还要继续持有`mutex`。因为安全分析并不知道`mun1.mu`是传入的`mutex`，所以在销毁`mun1`时，机械地认为`mun1.mu`理应一起销毁，才导致报一个不该报的警告，即假阳）



## ACQUIRED_BEFORE(...) and ACQUIRED_AFTER(...) 

目前暂未实现，后续的版本更新



# mutex.h

线程安全分析能与任何的线程库一起使用，但它要求线程API包裹在标注了相应的属性的类和方法里。`mutex.h`是一个例子，作为底层实现的一个办法。

```c++
#ifndef THREAD_SAFETY_ANALYSIS_MUTEX_H
#define THREAD_SAFETY_ANALYSIS_MUTEX_H

// Enable thread safety attributes only with clang.
// The attributes can be safely erased when compiling with other compilers.
#if defined(__clang__) && (!defined(SWIG))
#define THREAD_ANNOTATION_ATTRIBUTE__(x)   __attribute__((x))
#else
#define THREAD_ANNOTATION_ATTRIBUTE__(x)   // no-op
#endif

#define THREAD_ANNOTATION_ATTRIBUTE__(x)   __attribute__((x))

#define GUARDED_BY(x) \
  THREAD_ANNOTATION_ATTRIBUTE__(guarded_by(x))

#define GUARDED_VAR \
  THREAD_ANNOTATION_ATTRIBUTE__(guarded)

#define PT_GUARDED_BY(x) \
  THREAD_ANNOTATION_ATTRIBUTE__(pt_guarded_by(x))

#define PT_GUARDED_VAR \
  THREAD_ANNOTATION_ATTRIBUTE__(pt_guarded)

#define ACQUIRED_AFTER(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(acquired_after(__VA_ARGS__))

#define ACQUIRED_BEFORE(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(acquired_before(__VA_ARGS__))

#define EXCLUSIVE_LOCKS_REQUIRED(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(exclusive_locks_required(__VA_ARGS__))

#define SHARED_LOCKS_REQUIRED(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(shared_locks_required(__VA_ARGS__))

#define LOCKS_EXCLUDED(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(locks_excluded(__VA_ARGS__))

#define LOCK_RETURNED(x) \
  THREAD_ANNOTATION_ATTRIBUTE__(lock_returned(x))

#define LOCKABLE \
  THREAD_ANNOTATION_ATTRIBUTE__(lockable)

#define SCOPED_LOCKABLE \
  THREAD_ANNOTATION_ATTRIBUTE__(scoped_lockable)

#define EXCLUSIVE_LOCK_FUNCTION(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(exclusive_lock_function(__VA_ARGS__))

#define SHARED_LOCK_FUNCTION(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(shared_lock_function(__VA_ARGS__))

#define ASSERT_EXCLUSIVE_LOCK(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(assert_exclusive_lock(__VA_ARGS__))

#define ASSERT_SHARED_LOCK(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(assert_shared_lock(__VA_ARGS__))

#define EXCLUSIVE_TRYLOCK_FUNCTION(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(exclusive_trylock_function(__VA_ARGS__))

#define SHARED_TRYLOCK_FUNCTION(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(shared_trylock_function(__VA_ARGS__))

#define UNLOCK_FUNCTION(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(unlock_function(__VA_ARGS__))

#define NO_THREAD_SAFETY_ANALYSIS \
  THREAD_ANNOTATION_ATTRIBUTE__(no_thread_safety_analysis)


// Defines an annotated interface for mutexes.
// These methods can be implemented to use any internal mutex implementation.
class LOCKABLE Mutex {
public:
  // Acquire/lock this mutex exclusively.  Only one thread can have exclusive
  // access at any one time.  Write operations to guarded data require an
  // exclusive lock.
  void Lock() EXCLUSIVE_LOCK_FUNCTION();

  // Acquire/lock this mutex for read operations, which require only a shared
  // lock.  This assumes a multiple-reader, single writer semantics.  Multiple
  // threads may acquire the mutex simultaneously as readers, but a writer must
  // wait for all of them to release the mutex before it can acquire it
  // exclusively.
  void ReaderLock() SHARED_LOCK_FUNCTION();

  // Release/unlock the mutex, regardless of whether it is exclusive or shared.
  void Unlock() UNLOCK_FUNCTION();

  // Try to acquire the mutex.  Returns true on success, and false on failure.
  bool TryLock() EXCLUSIVE_TRYLOCK_FUNCTION(true);

  // Try to acquire the mutex for read operations.
  bool ReaderTryLock() SHARED_TRYLOCK_FUNCTION(true);

  // Assert that this mutex is currently held by the calling thread.
  void AssertHeld() ASSERT_EXCLUSIVE_LOCK();

  // Assert that is mutex is currently held for read operations.
  void AssertReaderHeld() ASSERT_SHARED_LOCK();
};


// MutexLocker is an RAII class that acquires a mutex in its constructor, and
// releases it in its destructor.
class SCOPED_LOCKABLE MutexLocker {
private:
  Mutex* mut;

public:
  MutexLocker(Mutex *mu) EXCLUSIVE_LOCK_FUNCTION(mu) : mut(mu) {
    mu->Lock();
  }
  ~MutexLocker() UNLOCK_FUNCTION() {
    mut->Unlock();
  }
};

#endif  // THREAD_SAFETY_ANALYSIS_MUTEX_H
```