---
title: "一个简单的垃圾回收器（翻译）"
date: 2020-05-10T17:19:51+08:00
draft: true
---

垃圾回收是编程中鲨鱼出没最多的领域之一，但是在这篇文章中，我会给你一个不错的儿童游泳池，你可以在里面游泳。 (可能还有鲨鱼在里面，但至少它会比较浅。)

### 减少废物，循环再用，循环再造

垃圾回收背后的基本思想是，该语言(在大多数情况下)似乎可以访问无限内存。 开发人员只需要保持分配、分配和分配，就像魔术一样，它永远不会失败。

当然，机器没有无限的内存。 所以实现的方式是，当它正在运行时内存空间不够并需要分配一点内存，它会收集垃圾。

此上下文中的“垃圾”是指它以前分配的不需要再被使用的内存。 为了让无限记忆的想法实现，语言必须在处理“不再被使用”内存时非常安全。 如果随机对象在程序试图访问它们时就开始被回收，那就没有意思了。

为了能够实现可回收，这种语言必须确保程序无法再次使用这个对象。 如果它不能得到对象的引用，那么它显然不能再次使用它。 因此，“在用”的定义实际上非常简单:

```
任何由作用域中的变量引用的对象都在使用中。

任何被另一个正在使用的对象引用的对象都在使用中。
```

第二条规则是递归规则。 

```
如果对象 a 被一个变量引用，并且它有一些引用对象 b 的字段，那么 b 就被使用了，因为你可以通过 a 找到它。
```

最终的结果是一个可到达所有对象的引用图，你可以从一个变量开始遍历所有对象。 对于程序来说，任何不在这个图中的对象都是死的，它的内存已经过期，可以收获了。

### 标记和清扫

有许多不同的方法可以实现查找和回收所有未使用对象的过程，但有史以来为此发明的最简单也是最早的算法被称为“标记-清除”。 它是由 John McCarthy 发明的，他发明了 Lisp 和 beards，所以现在实现垃圾回收器就像是在和一个老神交流，但是希望不是以 Lovecraftian 的方式，以你的头脑和视网膜被炸得干干净净结束。

“标记-清除”的工作原理几乎和我们对可达性的定义一模一样:

从根开始，遍历整个对象图。 每次你到达一个对象时，在它上面设置一个“标记”位为 true。

完成之后，找到所有标记位未设置的对象并删除它们。

### A pair of objects

在我们开始实现这两个步骤之前，让我们先做一些准备工作。 我们实际上不会为某种语言实现一个解释器 ーー 没有解析功能、字节码或其他任何愚蠢的东西ーー但我们确实需要一些最少量的代码来创建一些垃圾来进行收集。

让我们假设我们正在为一种小语言写一个解释器。 它是动态类型的，有两种类型的对象: int 和 pairs。 这里有一个枚举来标识一个对象的类型:

{% highlight c %}
typedef enum {
  OBJ_INT,
  OBJ_PAIR
} ObjectType;
{% endhighlight %}

一个 pair 可以是任意的一对，两个 int、一个 int 和另一个 pair ... 。 仅凭这一点，你就可以走得非常远。 由于 VM 中的对象可以是这两种中的任何一种，因此 c 中实现它的典型方法是使用带标记的联合(union)。

我们就这样定义它:

{% highlight c %}
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
{% endhighlight %}

主对象结构有一个类型字段，用于标识它的值类型 ー int 或 pair。然后它有一个联合来保存 pair 或 int 的数据。 如果您对 c 语言不熟悉，那么 union 就相当于是一个结构体，其中字段在内存中重叠。 因为给定的对象只能是 int 或者 pair，所以没有理由同时在一个对象中为所有三个字段保留内存。  union 可以做到这一点，太棒了。

### 一个最小的虚拟机

现在我们可以将它包装在一个小的虚拟机结构中。它在这个故事中的角色是拥有一个堆栈，用于存储当前作用域中的变量。大多数语言虚拟机要么基于堆栈(如 JVM 和 CLR) ，要么基于寄存器(如 Lua)。在这两种情况下，实际上仍然有一个堆栈。 它用于存储表达式中间所需的局部变量和临时变量。

我们将这样简单明了地建模:

{% highlight c %}
#define STACK_MAX 256

typedef struct {
  Object* stack[STACK_MAX];
  int stackSize;
} VM;
{% endhighlight %}

现在我们已经准备好了基本的数据结构，让我们组合一些代码来创建一些东西。 首先，让我们编写一个函数来创建并初始化一个 VM:

{% highlight c %}
VM* newVM() {
  VM* vm = malloc(sizeof(VM));
  vm->stackSize = 0;
  return vm;
}
{% endhighlight %}

一旦我们有了一个 VM，我们需要能够操纵它的堆栈:

{% highlight c %}
void push(VM* vm, Object* value) {
  assert(vm->stackSize < STACK_MAX, "Stack overflow!");
  vm->stack[vm->stackSize++] = value;
}

Object* pop(VM* vm) {
  assert(vm->stackSize > 0, "Stack underflow!");
  return vm->stack[--vm->stackSize];
}
{% endhighlight %}

好，现在我们可以把东西放进堆栈中，我们需要能够真正创建对象。 首先是一个小小的帮助函数:

{% highlight c %}
Object* newObject(VM* vm, ObjectType type) {
  Object* object = malloc(sizeof(Object));
  object->type = type;
  return object;
}
{% endhighlight %}

它执行实际的内存分配并设置类型标记。我们稍后将重新讨论这个问题。使用这个，我们可以编写函数将每种类型的对象推送到 VM 的堆栈中:

{% highlight c %}
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
{% endhighlight %}

这就是我们的小 VM。 如果我们有一个解析器和一个解释器来调用这些函数，那么我们手头上就会有一个实实在在的语言。 而且，如果我们有无限的内存，它甚至可以运行真正的程序。 既然没有，我们就开始收集垃圾吧。

### Marky mark

第一个阶段是标记。我们需要遍历所有可到达的对象并设置它们的标记位。我们需要做的第一件事就是给 Object 添加一个标记位:

{% highlight c %}
typedef struct sObject {
  unsigned char marked;
  /* Previous stuff... */
} Object;
{% endhighlight %}

当我们创建一个新对象时，我们将修改 newObject() 函数将初始化标记设置为零。为了标记所有可访问的对象，我们从内存中的变量开始，这意味着遍历堆栈。类似是这样的:

{% highlight c %}
void markAll(VM* vm)
{
  for (int i = 0; i < vm->stackSize; i++) {
    mark(vm->stack[i]);
  }
}
{% endhighlight %}

我们将分阶段构建它。第一步:

{% highlight c %}
void mark(Object* object) {
  object->marked = 1;
}
{% endhighlight %}

毫不夸张地说，这是最重要的部分。我们已经将对象本身标记为可达的，但是请记住，我们还需要处理对象中的引用: 可达性是递归的。 如果对象是一对，它的两个字段也是可到达的。 处理这个问题很简单:

{% highlight c %}
void mark(Object* object) {
  object->marked = 1;

  if (object->type == OBJ_PAIR) {
    mark(object->head);
    mark(object->tail);
  }
}
{% endhighlight %}

但是这里有个漏洞。 你看到了吗？ 我们现在正在一个循环中，但是我们没有检查循环的机制。如果在一个循环中有一堆指向彼此的对，则会溢出堆栈并崩溃。

为了处理这个问题，我们只需要在到达一个已经处理过的对象时退出。所以完整的 mark() 函数的实现为:

{% highlight c %}
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
{% endhighlight %}

现在我们可以调用 markAll () ，它将正确地标记内存中每个可到达的对象！

### Sweepy sweep

下一步是扫描我们分配的所有对象，并释放任何没有标记的对象。但是这里有一个问题: 根据定义，所有未标记的物体都是不可到达的！ 我们接近不了他们！

Vm 已经为对象引用实现了语言的语义: 因此我们只在变量和 pair 元素中存储指向对象的指针。一旦一个对象不再被这些对象中的任何一个指向，我们就完全失去了它，并且实际上泄漏了内存。

解决这个问题的诀窍在于，VM 可以拥有自己的对象引用，这些对象与语言用户可见的语义不同。 换句话说，我们可以自己跟踪他们。

要做到这一点，最简单的方法就是维护我们分配的每个对象的链接列表。 我们将扩展 Object 本身作为列表中的节点:

{% highlight c %}
typedef struct sObject {
  /* The next object in the list of all objects. */
  struct sObject* next;

  /* Previous stuff... */
} Object;
The VM will keep track of the head of that list:
{% endhighlight %}

虚拟机将跟踪这个列表的头部:

{% highlight c %}
typedef struct {
  /* The first object in the list of all objects. */
  Object* firstObject;

  /* Previous stuff... */
} VM;
{% endhighlight %}

在 newVM() 函数中，我们将确保初始化 firstObject 为 NULL。 每当我们创建一个对象，我们把它添加到列表中:

{% highlight c %}
Object* newObject(VM* vm, ObjectType type) {
  Object* object = malloc(sizeof(Object));
  object->type = type;
  object->marked = 0;

  /* Insert it into the list of allocated objects. */
  object->next = vm->firstObject;
  vm->firstObject = object;

  return object;
}
{% endhighlight %}

这样，即使语言找不到对象，语言的实现仍然可以。 要清除和删除未标记的对象，我们只需遍历列表:

{% highlight c %}
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
{% endhighlight %}

由于那个指向指针的指针，这段代码有点难以阅读，但是如果你仔细研究一下，就会发现它非常简单。它只是遍历整个链表。每当它命中一个没有标记的对象时，它就释放内存并将其从列表中删除。当这样做，我们将删除每一个不可到达的对象。

祝贺你！我们有一个垃圾回收器！只是还少了一点: 真正的调用它。首先让我们把这两个阶段结合在一起:

{% highlight c %}
void gc(VM* vm) {
  markAll(vm);
  sweep(vm);
}
{% endhighlight %}

您不能要求更明显的标记-清除实现。 最棘手的部分是确定什么时候调用这个函数。 “内存不足”到底是什么意思，尤其是在具有近乎无限虚拟内存的现代计算机上？

事实证明，这个问题没有明确的正确或错误答案。这实际上取决于您使用 VM 的目的是什么，以及它运行在什么样的硬件上。为了保持这个示例的简单性，我们只需要在一定数量的分配之后收集。这实际上就是一些语言实现的工作方式，而且很容易实现。

我们将扩展 VM 来跟踪我们创建了多少对象:

{% highlight c %}
typedef struct {
  /* The total number of currently allocated objects. */
  int numObjects;

  /* The number of objects required to trigger a GC. */
  int maxObjects;

  /* Previous stuff... */
} VM;
{% endhighlight %}

然后初始化它们:

{% highlight c %}
VM* newVM() {
  /* Previous stuff... */

  vm->numObjects = 0;
  vm->maxObjects = INITIAL_GC_THRESHOLD;
  return vm;
}
{% endhighlight %}

INITIAL_GC_THRESHOLD 是将要启动第一个 gc() 函数时的对象数。越小的数字在内存方面越保守，越大的数字花在垃圾收集上的时间越少。根据需求调整。

每当我们创建一个对象，我们都会增加 numObjects ，如果它是否达到最大值就运行一次回收:

{% highlight c %}
Object* newObject(VM* vm, ObjectType type) {
  if (vm->numObjects == vm->maxObjects) gc(vm);

  /* Create object... */

  vm->numObjects++;
  return object;
}
{% endhighlight %}

我不打算显示它，但是我们也会在每次释放一个对象时调整 sweep() 以减少 numObjects 最后，我们修改 gc() 来更新 max:

{% highlight c %}
void gc(VM* vm) {
  int numObjects = vm->numObjects;

  markAll(vm);
  sweep(vm);

  vm->maxObjects = vm->numObjects * 2;
}
{% endhighlight %}

在每次垃圾回收之后，我们根据回收之后剩余的活动对象数量更新 maxObjects。那里的倍增器让我们的堆随着活动对象数量的增加而增长。同样，如果一堆对象最终被释放，它也会自动收缩。

### Simple

你成功了！如果您遵循了所有这些步骤，那么现在已经掌握了一个简单的垃圾回收器算法。如果你想看到它的全貌，这里是[完整的代码](https://github.com/munificent/mark-sweep)。让我在这里强调，虽然这个垃圾回收器是简单的，它不是一个玩具。

您可以在此基础上构建大量的优化(在 GC 和编程语言等领域，优化占了90% 的工作量)，但是这里的核心代码是一个合法的真正的 GC 。它与 Ruby 和 Lua 中的垃圾回收器非常相似。您可以发布使用类似这样的东西的生产代码。现在去建造一些很棒的东西吧！

[原文链接](http://journal.stuffwithstuff.com/2013/12/08/babys-first-garbage-collector/)