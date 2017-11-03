---
layout: post
title: "translate baby first garbage collector"
description: "a translation for baby's first garbage collector"
category: "theory"
tags: [translate,theory]
---

# 向垃圾回收迈出第一步

>本文参考[Baby's First Garbage Collector][1]，对它进行翻译并加上自己的理解。

最近在重拾C语言，无意中看到这篇博文，它对垃圾回收的讲解初步/直观，我呢，既可以找回下C语言的感觉，也可以了解下其它解释型语言的垃圾回收机制。

------

垃圾回收被认为是编程的一个危险的水域，不过在这里，我将给你精简的儿童泳池，让你在其中玩耍，不过里面还是有些小危险。

## 减少，重用，回收

垃圾回收的基本意图是让绝大多数编程语言看起来可以无限的访问内存，程序员可以不停的分配内存，内存似乎被施了无限扩大咒。

当然，计算器是不会有无限的内存的。实际上，当需要有内存被分配而内存被发现不够时，会有一个机制来收集内存中的垃圾。

垃圾在这里指的是原来被分配而现在却不用的内存。为了产生无限内存的假象，这个程序语言必须很小心的判别什么才是不被用到的内存。访问一个被回收的内存区域可是很危险的。

为了确保回收的成功进行，编程语言必须保证程序不会再次使用那个对象。如果无法获得这个对象的引用，那么很显然，它就不能再次被使用。由此可以得出何为对象在使用的定义：

 1. 被作用域内的变量引用的对象是被使用的
 2. 被其它对象引用的对象是被使用的
 
第二种情况是递归的。例如如果A对象被一个变量引用，同时它的成员引用了对象B，那么只要你能通过A获得B，则B就是被使用的。

经过上面的判断，最终我们得到的是一个图结构，图的节点是可达的对象。这些对象是以某一个变量为起点遍历得到的，那些不可达的对象对程序来说是死的，他们的内存是可以被回收的。

## 标记与清理

垃圾回收的实现有多种方式，最简单的也是第一个被发明的算法是"mark-sweep"。

它的执行方式正如它的名字一样：

 1. 从根元素开始，遍历整个对象图。每当达到一个对象，把它的mark变量的值设置为真。
 2. 第一步一旦完成，就找到所有标记不为真的对象然后删除它们。

## 一对对象

在实现上面提到的两步之前，先做些准备工作。我们不会去真的实现一个语言解释器，不过我们需要写一些能创造垃圾并回收的代码。

假设，我们在为一个语言写解释器。它的类型是动态的而且有两中类型：整数型（int）与成对型（pair），下面通过枚举定义他们

```c
typedef enum {
  OBJ_INT,
  OBJ_PAIR
} ObjectType;
```
 一个成对型的变量可以是类型中的任意两种，两个int型或者一个int型一个成对型都行。考虑到这个对象在虚拟机中可以是任意的两中，在C中可以通过标签联合体来实现。
 
**对象的声明 **
 
```c
 typedef struct sObject {
  ObjectType type;

  union {
    /* OBJ_INT */
    int value;

    /* OBJ_PAIR */
    struct {
      struct sObject* head;
      struct sObject* tail;
    };
  };
} Object;
```

## 一个小巧的虚拟机

现在，我们可以把上面的代码封装到一个虚拟机结构里。在这里，虚拟机是一个栈，里面存有当前作用域的变量。大多数的编程语言的虚拟机是基于栈或者寄存器的，不过，他们实际上都是栈结构。它被用来存储在表达式中会被使用的本地变量或者临时变量。

**虚拟机的声明**

```c
#define STACK_MAX 256

typedef struct {
  Object* stack[STACK_MAX];
  int stackSize;
} VM;
```

**初始化虚拟机**

```c
VM* newVM() {
  VM* vm = malloc(sizeof(VM));
  vm->stackSize = 0;
  return vm;
}
```


**虚拟机的操作**

```c
void push(VM* vm, Object* value) {
  assert(vm->stackSize < STACK_MAX, "Stack overflow!");
  vm->stack[vm->stackSize++] = value;
}

Object* pop(VM* vm) {
  assert(vm->stackSize > 0, "Stack underflow!");
  return vm->stack[--vm->stackSize];
}
```

**创建对象**

```c
Object* newObject(VM* vm, ObjectType type) {
  Object* object = malloc(sizeof(Object));
  object->type = type;
  return object;
}
```

**将对象加入虚拟机**

```c
void pushInt(VM* vm, int intValue) {
  Object* object = newObject(vm, OBJ_INT);
  object->value = intValue;
  push(vm, object);
}

Object* pushPair(VM* vm) {
  Object* object = newObject(vm, OBJ_PAIR);
  object->tail = pop(vm);
  object->head = pop(vm);

  push(vm, object);
  return object;
}
```

## 标记

第一步是标记，我们需要遍历所有可达的对象并做上标记。

我们需要给我们的Object类型加上一个标记成员。

```c
typedef struct sObject {
  unsigned char marked;
  /* Previous stuff... */
} Object;
```
当我们创建一个新对象时，我们需要在newObject()方法中加上标记的操作，及初始化标记变量为0.为了标记所有的可达的对象，我们需要遍历内存中的对象，及遍历栈中的。

```c
void mark(Object* object) {
  object->marked = 1;
}

void markAll(VM* vm)
{
  for (int i = 0; i < vm->stackSize; i++) {
    mark(vm->stack[i]);
  }
}
```
考虑到成对型的对象，我们需要递归的标记对象，及如下形式，注意，考虑到递归就有个结束的条件，不然会无限下去直到溢出。

```c
void mark(Object* object) {
  /* If already marked, we're done. Check this first
     to avoid recursing on cycles in the object graph. */
  if (object->marked) return;

  object->marked = 1;

  if (object->type == OBJ_PAIR) {
    mark(object->head);
    mark(object->tail);
  }
}
```

## 清理

第二步是清理，释放那些没有被标记的对象。但是这里有个问题，没有被标记的对象，从定义上讲是不可达的，那么我们怎样才能得到他们呢？

这个虚拟机实现了对象语义上的引用，及只保存了变量所指对象的指针。一旦对象不再被它们所指，则我们完全失去了它，这就造成了内存泄露。

解决这个问题的一个小技巧是让虚拟机自己保存一个指向这些对象的引用，就算栈里的引用消失，还是可以通过vm自己的引用找到该对象。

实现这个最简单的方法是让它自己维护一个对象的链表，在此，我们对Object的声明做下修改。

```c
typedef struct sObject {
  /* The next object in the list of all objects. */
  struct sObject* next;

  /* Previous stuff... */
} Object;
```

同时对vm做些修改

```c
typedef struct {
  /* The first object in the list of all objects. */
  Object* firstObject;

  /* Previous stuff... */
} VM;
```

在newVM()中，初始化的时候需要确保firstObject被初始化为NULL。每当创建个新对象的时候，需要把它加到链表中。

```c
Object* newObject(VM* vm, ObjectType type) {
  Object* object = malloc(sizeof(Object));
  object->type = type;
  object->marked = 0;

  /* Insert it into the list of allocated objects. */
  object->next = vm->firstObject;
  vm->firstObject = object;

  return object;
}
```

这样即使语言本身找不到对象，语言的实现还是可以的。遍历清理未标记的对象，我们只需要遍历链表。

```c
void sweep(VM* vm)
{
  Object** object = &vm->firstObject;
  while (*object) {
    if (!(*object)->marked) {
      /* This object wasn't reached, so remove it from the list
         and free it. */
      Object* unreached = *object;

      *object = unreached->next;
      free(unreached);
    } else {
      /* This object was reached, so unmark it (for the next GC)
         and move on to the next. */
      (*object)->marked = 0;
      object = &(*object)->next;
    }
  }
}
```

这段代码因为那个指向指针的指针，可能有点难读，不过如果你读懂后，会发现它很直接。它只是遍历整个链表，将没有被标记的对象从链表中删除然后释放该对象的空间。

这时，我们可以稍稍庆祝了，因为我们有了个垃圾回收器。这里还有一个块需要完善，将上面的两块内容整合下。

```c
void gc(VM* vm) {
  markAll(vm);
  sweep(vm);
}
```

这恐怕是最简单明了的基于标记清理算法的垃圾回收器了。不过最微妙的地方是何时调用它，考虑到现代的计算机有很大的虚拟内存，什么才是内存不足呢？

对于这个问题，没有绝对的对与错。它取决于你的虚拟机的作用以及运行它的硬件环境。为了保持这个例子的简单，我们会在超过一个确定的分配数值后运行垃圾回收。这个也是某些语言的做法。

在此，扩展下VM

```c
typedef struct {
  /* The total number of currently allocated objects. */
  int numObjects;

  /* The number of objects required to trigger a GC. */
  int maxObjects;

  /* Previous stuff... */
} VM;

VM* newVM() {
  /* Previous stuff... */

  vm->numObjects = 0;
  vm->maxObjects = INITIAL_GC_THRESHOLD;
  return vm;
}
```

INITIAL_GC_THRESHOLD就是上面说道的阈值。

每当我们创建一个对象的时候，我们就增加numObjects的值，当它到达上限时就执行垃圾回收。

```c
Object* newObject(VM* vm, ObjectType type) {
  if (vm->numObjects == vm->maxObjects) gc(vm);

  /* Create object... */

  vm->numObjects++;
  return object;
}
```

当然，如果在sweep()中清理了对象，它的数值得相应减少。最后，我们修改gc()中的代码来更新最大值。

```c
void gc(VM* vm) {
  int numObjects = vm->numObjects;

  markAll(vm);
  sweep(vm);

  vm->maxObjects = vm->numObjects * 2;
}
```

# 简单

如果你一路看过来，那么你对简单的垃圾回收算法就有了个基本了了解了，如果你想看看它们是如何配合的，可以看[完整的代码][2]。最后，我在这里想强调下，这个垃圾回收器虽然简单，但它不仅仅只是个玩具。

在此基础上，有很多优化可以做，同时这里的核心代码是真是的垃圾回收器。Ruby和Lua中的回收器和它类似。

------

## 我的想法

在整个阅读过程中，有下面两点让我疑惑了半天：

 - pushPair()的实现为什么是下面这种方式

```c
Object* object = newObject(vm, OBJ_PAIR);
  object->tail = pop(vm);
  object->head = pop(vm);

  push(vm, object);
```
 
  - 下面的代码在删除节点时，链表没有断吗(这个疑惑比较傻帽了，完全是由于对C的指针语法不熟所致)

```c
Object** object = &vm->firstObject;
  while (*object) {
    if (!(*object)->marked) {
      /* This object wasn't reached, so remove it from the list
         and free it. */
      Object* unreached = *object;

      *object = unreached->next;
      free(unreached);
    } else {
      /* This object was reached, so unmark it (for the next GC)
         and move on to the next. */
      (*object)->marked = 0;
      object = &(*object)->next;
    }
  }
```

**对于第一个问题，博客作者给了以下解答**

>Ah, great question! I meant to explain that. That avoids a nasty potential problem. If you call newObject() twice and store the results in local C variables, the second call to newObject() could trigger a GC. Since the first object isn't on the stack yet, the GC won't reach it and it will get collected.
Creating a pair like this ensures that the elements in the pair are on the stack as soon as they are created.

作者的回答从代码逻辑的角度解释了这个问题，一开始，我还是搞不明白为什么要这样，后来发现我和作者想问题的层次有差异。

首先，我们要明确的是，这里的代码并不是给用该语言的程序员用的，而是编写该语言的解释器的人写的。于是乎，这里的pushPair是声明pair变量的底层实现。

例如，假设这个语言声明这两种变量的方式如下（及使用这个语言的程序员写的）：

    int iValue = 1;
    pair pValue = (1,2);

那么，这个解释器在底层会是如下实现：

    pushInt(vm,1);
    pushInt(vm,1);
    pushInt(vm,2);
    pushPair(vm);
    
这样，从逻辑和实现层次上，我们就理解了代码为何要这样写。


**对于第二个问题，明确C语言指针的实质，及指针是地址**


下面用图像来说明：

下图模拟sweep()函数的过程，其中每个方块左上角的方块表示它的地址，有两个方块当然就是指针的指针了。

第一步：探索链表中的第一个对象，发现被标记，则执行相应的步骤。

![步骤1][3]

第二步：假设第二个元素是未被标记的，则需要被释放，最后的结果如下。

![步骤2][4]

这里很明显，由于指针灵活的操作，使得简单的两句代码就完成了链表中元素的删除。

最后，在博客的评论中，有人为sweep()提了一个优化方案：

>A new trick: in the sweep phase, you don't need to reset the mark. You can just use another integer as the "traversed" mark for the next traversal. You can increment the mark at each traversal, and if you're not comfortable with overflows wrap it around a fixed bound that must be higher than the total number of colors you need, plus one. This saves you one write per non-freed object, which is nice.


  [1]: http://journal.stuffwithstuff.com/2013/12/08/babys-first-garbage-collector/
  [2]: https://github.com/munificent/mark-sweep
  [3]: https://lh5.googleusercontent.com/-Tm3PMUn6DQg/UsAu5-vZUjI/AAAAAAAAAls/y-XH3uu3hAE/s0/1.png "1.png"
  [4]: https://lh3.googleusercontent.com/-Df-CoTKmvec/UsAvF78Hu7I/AAAAAAAAAl4/cAmcxwDz7J4/s0/2.png "2.png"
