** 简书篇幅有限，未完待续。下文请查看500 Lines or LessfunctionalDB函数式数据库（二） 
P.S. 还未发布**

存放着翻译的文章的我自己的github：https://github.com/fst034356/500LinesorLessToCN
我的GitHub上的是整个文章，不需要像简书这样分开。

原文：http://aosabook.org/en/500L/dbdb-dog-bed-database.html
原项目GitHub:https://github.com/aosabook/500lines/blob/master/README.md

由于英语功力实在有限，错误难免，欢迎各位大佬指正，顺便求个star。

* * * 
title: DBDB: Dog Bed Database
author: Taavi Burns

_As the newest bass (and sometimes tenor) in [Countermeasure](http://www.countermeasuremusic.com), Taavi strives to break the mould... sometimes just by ignoring its existence. This is certainly true through the diversity of workplaces in his career: IBM (doing C and Perl), FreshBooks (all the things), Points.com (doing Python), and now at PagerDuty (doing Scala).  Aside from that—when not gliding along on his Brompton folding bike—you might find him playing Minecraft with his son or engaging in parkour (or rock climbing, or other adventures) with his wife. He knits continental._

作为[Countermeasure](http://www.countermeasuremusic.com)中最新的低音（有时是男高音），Taavi努力打破模具...有时只是忽略它的存在。（译者：我不是很懂...）(以下他职业生涯中的多样化工作都是真的，IBM（做C和Perl），FreshBooks（所有的东西），Points.com（做Python），现在在PagerDuty（做Scala）。 除此之外 - 当他的Brompton折叠式自行车没有滑行时，你可能会发现他和他的儿子一起玩Minecraft，或者和他的妻子一起参加跑酷（或攀岩或其他冒险）。 He knits continental（译者：对不起...我翻译无能.....这是啥？）。

## Introduction 介绍

DBDB (Dog Bed Database) is a Python library that implements a simple key/value database.
It lets you associate a key with a value, and store that association on disk for later retrieval.

DBDB是一个实现一个简单的键/值数据库的Python库。
它允许您将键与值相关联，
并将该关联存储在磁盘上以供以后检索。

DBDB aims to preserve data in the face of computer crashes and error conditions. It also avoids holding all data in RAM at once so you can store more data than you have RAM.
DBDB旨在保护面对计算机崩溃和错误时的数据。它也避免了一次将所有数据保存在RAM中，以便您可以存储比RAM更多的数据。

## Memory 回忆

I remember the first time I was really stuck on a bug. When I finished
typing in my BASIC program and ran it, weird sparkly pixels showed up on the screen, and the program aborted early. When I went back to look at the code, the last few lines of the program were gone. 
我记得我第一次真的陷入了一个bug。 当我完成键入我的BASIC程序并运行它，奇怪的像素闪烁显示在屏幕上，并且程序提前中止。 当我回去看代码时，程序的最后几行已经执行完了。

One of my mom's friends knew how to program, so we set up a call. Within a few
minutes of speaking with her, I found the problem: the program was too big, and
had encroached onto video memory. Clearing the screen truncated the program,
and the sparkles were artifacts of Applesoft BASIC's behaviour of storing
program state in RAM just beyond the end of the program.
我妈妈的一个朋友知道如何编程，所以我们通一个电话。 在与她通话的几分钟之内，我发现了这个问题：程序太大了，已经侵入了视频内存。 清除屏幕并截断程序，并且闪光是Applesoft BASIC的存储在RAM中的程序超出行为的表象。
From that moment onwards, I cared about memory allocation.  I
learned about pointers and how to allocate memory with malloc. I learned how my
data structures were laid out in memory. And I learned to be very, very careful
about how I changed them.
从那时起，我关心内存分配。 我学习了指针，以及如何使用malloc分配内存。 我了解了我的数据结构在内存中的布局。 我学会了如何非常非常小心改变他们。
Some years later, while reading about a process-oriented language called
Erlang, I learned that it didn't actually have to copy data to send messages
between processes, because everything was immutable. I then discovered
immutable data structures in Clojure, and it really began to sink in. 
几年之后，在阅读一个名为Erlang的面向过程的语言的同时，我了解到，实际上它并不需要将数据复制到进程之间发送消息，因为一切都是不可变的。 然后，我在Clojure中发现了不可变数据结构，它真的开始沉入。
When I read about CouchDB in 2013, I just smiled and nodded,
recognising the structures and mechanisms for managing complex data as it
changes.
当我在2013年阅读CouchDB的时候，我只是微笑着点点头，认识到了管理复杂数据的结构和机制的变化。
I learned that you can design systems built around immutable data.
我了解到，您可以设计围绕不可变数据构建的系统。
Then I agreed to write a book chapter.
然后我同意写一章。
I thought that describing the core data storage concepts of CouchDB
(as I understood them) would be fun.
我认为描述CouchDB的核心数据存储概念（正如我所理解的）将是有趣的。
While trying to write a binary tree algorithm that mutated the tree in place, I got frustrated with how complicated things were getting. The number of edge cases and trying to reason about how changes in one part of the tree affected others was making my head hurt. I had no idea how I was going to explain all of this.
319/5000

在尝试编写一个二叉树算法时，将树的位置变异，令我感到沮丧的复杂的事情发生了。 边缘案件的数量以及试图推断一棵树的一部分变化如何影响到他人，使我的头很疼。 我不知道我将如何解释这一切。
Remembering lessons learned, I took a peek at a recursive algorithm for updating immutable binary trees and it turned out to be relatively straightforward.
回忆之前的经验教训，我仔细观察了递归算法来更新不可变二叉树，结果是比较简单。
I learned, once again, that it's easier to reason about things that don't change.
我再次认识到，对于不改变来说，更容易理解。
So starts the story.
让我们开始。


## Characterizing Failure

Databases are often characterized by how closely they adhere to
the ACID properties:
atomicity, consistency, isolation, and durability.
数据库的特征通常是它们与ACID属性的接近程度如何：原子性，一致性，隔离性和耐久性。
Updates in DBDB are atomic and durable,
two attributes which are described later in the chapter.
DBDB provides no consistency guarantees
as there are no constraints on the data stored.
Isolation is likewise not implemented.
DBDB中的更新是原子和持久的，本章后面将介绍这两个属性。 DBDB不提供一致性保证，因为对存储的数据没有约束。 隔离也没有实现。

Application code can, of course, impose its own consistency guarantees, but proper isolation requires a transaction manager. We won't attempt that here; however, you can learn more about transaction management in the [CircleDB chapter](http://aosabook.org/en/500L/an-archaeology-inspired-database.html). 

应用程序代码当然可以强加自己的一致性保证，但适当的隔离需要一个事务管理器。 我们不会在这里试图做这一点。但是，你可以在这里了解到更多有关事务管理的信息[CircleDB chapter](http://aosabook.org/en/500L/an-archaeology-inspired-database.html). 


We also have other system-maintenance problems to think about.  Stale data is not reclaimed in this implementation, so repeated updates (even to the same key) will eventually consume all disk space. (You will shortly discover why this is the case.) [PostgreSQL](http://www.postgresql.org/) calls this reclamation "vacuuming" (which makes old row space available for re-use), and [CouchDB](http://couchdb.apache.org/) calls it "compaction" (by rewriting the "live" parts of the data into a new file, and atomically moving it over the old one).
我们还有其他系统维护问题需要考虑。 在此实施中，不再收回过时的数据，因此重复更新（即使对于一样的KEY来说）将最终消耗所有磁盘空间。 （你会很快发现为什么会这样）。[PostgreSQL](http://www.postgresql.org/) 称这个为“vacuuming”（使旧row space可用于重用）， [CouchDB](http://couchdb.apache.org/) 称这个为"compaction"（通过将数据的“live”部分重写为新文件，
并原子地移动它离开旧的位置）。

DBDB could be enhanced to add a compaction feature,
but it is left as an exercise for the reader[^bonus]. 
DBDB可以添加压缩功能，但这作为读者的练习[^bonus].
[^bonus]: Bonus feature: Can you guarantee that the compacted tree structure is balanced?  This helps maintain performance over time.
您能保证压缩的树结构是平衡的吗？ 这有助于在一段时间内保持性能。


## The Architecture of DBDB

DBDB separates the concerns of "put this on disk somewhere"
(how data are laid out in a file; the physical layer)
from the logical structure of the data
(a binary tree in this example; the logical layer)
from the contents of the key/value store
(the association of key `a` to value `foo`; the public API).
DBDB将“将磁盘放在某处”与数据的逻辑结构从键/值存储的内容中分离出来。（如何在文件中布置数据;物理层）（本例中为二叉树;逻辑层）（key `a`与value foo`的关联;公共API）。

Many databases separate the logical and physical aspects
as it is is often useful to provide alternative implementations of each
to get different performance characteristics,
e.g. DB2's SMS (files in a filesystem) versus DMS (raw block device) tablespaces,
or MySQL's [alternative engine implementations](http://dev.mysql.com/doc/refman/5.7/en/storage-engines.html).
很多数据库分离逻辑层和物理层是因为提供每个替代实现以获得不同的性能特性通常是有用的，例如， DB2's SMS (files in a filesystem) versus DMS (raw block device) tablespaces,or MySQL's [alternative engine implementations](http://dev.mysql.com/doc/refman/5.7/en/storage-engines.html).

## Discovering the Design

Most of the chapters in this book describe how a program was built from
inception to completion. However, that is not how most of us interact with the
code we're working on.
We most often discover code that was written by others,
and figure out how to modify or extend it to do something different.
这本书的大部分章节都描述了一个程序是如何从头到完成的。 然而，这并不是我们大多数人经常进行的工作。 我们最经常的是找到由他人编写的代码，并找出如何修改或扩展它来做某些不同功能的代码方法。

In this chapter, we'll assume that DBDB is a completed project, and walk
through it to learn how it works. Let's explore the structure of the entire
project first.
在本章中，我们假设DBDB是一个已完成的项目，并通过它来了解它是如何工作的。 我们先来探讨整个项目的结构。


### Organisational Units 组织单元

Units are ordered here by distance from the end user; that is, the first module
is the one that a user of this program would likely need to know the most
about, while the last is something they should have very little interaction
with.
单元按照距离最终用户的距离排序; 也就是说，第一个模块是该程序的用户可能需要最了解的模块，而对于最后一个模块，用户应该有很少的交互。
* ``tool.py`` defines
    a command-line tool
    for exploring a database
    from a terminal window.
    一个用于从终端窗口浏览数据库的命令行工具。

* ``interface.py`` defines
    a class (``DBDB``)
    which implements the Python dictionary API
    using the concrete ``BinaryTree`` implementation.
    This is how you'd use DBDB inside a Python program.
一个类（“DBDB”），它使用具体的“BinaryTree”实现来实现Python字典API。 这是在Python程序中使用DBDB的方式。
* ``logical.py`` defines
    the logical layer.
    It's an abstract interface to a key/value store.
逻辑层。它是一个key/value存储的抽象接口。
    - ``LogicalBase`` provides the API for logical updates
        (like get, set, and commit)
        and defers to a concrete subclass
        to implement the updates themselves.
        It also manages storage locking
        and dereferencing internal nodes.
      提供用于逻辑更新的API（如get，set和commit），并提供一个具体的子类来实现更新本身。它还管理存储锁定和取消引用内部节点。
    - ``ValueRef`` is a Python object that refers to
        a binary blob stored in the database.
        The indirection lets us avoid loading
        the entire data store into memory all at once.
        是一个引用存储在数据库中的二进制blob的Python对象。 间接使我  们能够避免将整个数据存储区一次性加载到内存中。
* ``binary_tree.py`` defines
    a concrete binary tree algorithm
    underneath the logical interface.
    逻辑接口下面的具体的二叉树算法。
    - ``BinaryTree`` provides a concrete implementation
        of a binary tree, with methods for
        getting, inserting, and deleting key/value pairs.
        提供二叉树的具体实现，具有获取，插入和删除键/值对的方法。
        ``BinaryTree`` represents an immutable tree;
        updates are performed by returning a new tree
        which shares common structure with the old one.
        代表一棵不变的树; 通过返回与旧的树共享结构的新树来执行更新。
    - ``BinaryNode`` implements a node in the binary tree.
        在二叉树中的一个节点。
    - ``BinaryNodeRef`` is a specialised ``ValueRef``
        which knows how to serialise and deserialise
        a ``BinaryNode``.
        是一个专门的“ValueRef”，它知道如何序列化和反序列化“二进制节点”。
* ``physical.py`` defines
    the physical layer.
    The ``Storage`` class
    provides persistent, (mostly) append-only record storage.
    定义物理层。 “Storage”类提供了持久的（大部分）append-only的记录存储。
These modules grew from attempting
to give each class a single responsibility.
In other words,
each class should have only one reason to change.
这些模块从尝试给每个class一个 单一功能而增长。 换句话说，每个class 应该只有一个理由去改变。


### Reading a Value 读一个值

We'll start with the simplest case: reading a value from the database. Let's see what happens
when we try to get the value associated with key ``foo`` in ``example.db``:
我们从一个最简单的例子开始：从数据库中读取一个值。让我们看看当我们尝试从``example.db``中获取 key ``foo``的值的时候会发生什么。
```bash
$ python -m dbdb.tool example.db get foo
```

This runs the ``main()`` function from module ``dbdb.tool``:
```python
# dbdb/tool.py
def main(argv):
    if not (4 <= len(argv) <= 5):
        usage()
        return BAD_ARGS
    dbname, verb, key, value = (argv[1:] + [None])[:4]
    if verb not in {'get', 'set', 'delete'}:
        usage()
        return BAD_VERB
    db = dbdb.connect(dbname)          # CONNECT
    try:
        if verb == 'get':
            sys.stdout.write(db[key])  # GET VALUE
        elif verb == 'set':
            db[key] = value
            db.commit()
        else:
            del db[key]
            db.commit()
    except KeyError:
        print("Key not found", file=sys.stderr)
        return BAD_KEY
    return OK
```

The ``connect()`` function
opens the database file
(possibly creating it,
but never overwriting it)
and returns an instance of ``DBDB``:
``connect（）``函数打开数据库文件（可能是创建它，但不能覆盖它）并返回一个“DBDB”的实例：
```python
# dbdb/__init__.py
def connect(dbname):
    try:
        f = open(dbname, 'r+b')
    except IOError:
        fd = os.open(dbname, os.O_RDWR | os.O_CREAT)
        f = os.fdopen(fd, 'r+b')
    return DBDB(f)
```

```python
# dbdb/interface.py
class DBDB(object):

    def __init__(self, f):
        self._storage = Storage(f)
        self._tree = BinaryTree(self._storage)
```

We see right away that `DBDB` has a reference to an instance of `Storage`, but
it also shares that reference with `self._tree`. Why? Can't `self._tree`
manage access to the storage by itself? 
我们立刻看到，`DBDB`引用了一个`Storage`的实例，
但它也与`self._tree`共享该引用。 为什么？ “self._tree”不能自行管理对存储的访问吗？
The question of which objects "own" a resource is often an important one in a
design, because it gives us hints about what changes might be unsafe. Let's
keep that question in mind as we move on.
哪个对象“拥有”一个资源的问题在设计中通常是一个重要的问题，因为它提供了关于什么变化可能不安全的提示。 让我们继续关注这个问题。
Once we have a DBDB instance, getting the value at ``key`` is done via a
dictionary lookup (``db[key]``), which causes the Python interpreter to call
``DBDB.__getitem__()``.

一旦我们有一个DBDB实例，通过字典查找（``db[key]``）获取 ``key``的值，这将导致Python解释器调用``DBDB.__getitem__()``。
```python
# dbdb/interface.py
class DBDB(object):
# ...
    def __getitem__(self, key):
        self._assert_not_closed()
        return self._tree.get(key)

    def _assert_not_closed(self):
        if self._storage.closed:
            raise ValueError('Database closed.')
```

``__getitem__()`` ensures that the database is still open by calling
`_assert_not_closed`. Aha! Here we see at least one reason why `DBDB` needs
direct access to our `Storage` instance: so it can enforce preconditions.
(Do you agree with this design? Can you think of a different way that we could
do this?)

``__getitem __（）``通过调用`_assert_not_closed`确保数据库仍然是打开的。 阿哈！ 这里我们看到至少有一个“DBDB”需要直接访问我们的“存储”实例的原因：因此它可以强制执行前提条件。 （你同意这个设计吗？你能想出一个不同的我们可以做到方式吗？）

DBDB then retrieves the value associated with ``key`` on the internal ``_tree``
by calling ``_tree.get()``, which is provided by ``LogicalBase``:

DBDB然后通过调用``_tree.get（）``检索``key``与内部``_tree``相关联的值，由``LogicalBase``提供：
```python
# dbdb/logical.py
class LogicalBase(object):
# ...
    def get(self, key):
        if not self._storage.locked:
            self._refresh_tree_ref()
        return self._get(self._follow(self._tree_ref), key)
```

``get()`` checks if we have the storage locked. We're not 100% sure _why_
there might be a lock here, but we can guess that it probably exists to allow
writers to serialize access to the data. What happens if the storage isn't locked?

``get（）``检查我们是否锁定了存储。 我们不是100％肯定_why_可能有一个锁在这里，但我们可以猜到它可能存在允许作者序列化对数据的访问。 如果存储没有锁定会怎么样？
```python
# dbdb/logical.py
class LogicalBase(object):
# ...
def _refresh_tree_ref(self):
        self._tree_ref = self.node_ref_class(
            address=self._storage.get_root_address())
```

`_refresh_tree_ref` resets the tree's "view" of the data with what is currently
on disk, allowing us to perform a completely up-to-date read.

`_refresh_tree_ref`重新设置当前在磁盘上的数据的树的“视图”，允许我们执行一个完全最新的读取。

What if storage _is_ locked when we attempt a read? This means that some other
process is probably changing the data we want to read right now; our read is
not likely to be up-to-date with the current state of the data. This is
generally known as a "dirty read". This pattern allows many readers to access
data without ever worrying about blocking, at the expense of being slightly
out-of-date.
如果我们尝试读取时，storage 已经是_is_ locked怎么办？这意味着其他一些进程可能正在改变我们现在要读取的数据; 我们的读取不太可能是最新的数据的当前状态。 这通常被称为“脏读”。 这种模式允许许多读者访问数据，而不用担心阻塞。
For now, let's take a look at how we actually retrieve the data:
现在，我们来看看我们如何实际检索数据：
```python
# dbdb/binary_tree.py
class BinaryTree(LogicalBase):
# ...
    def _get(self, node, key):
        while node is not None:
            if key < node.key:
                node = self._follow(node.left_ref)
            elif node.key < key:
                node = self._follow(node.right_ref)
            else:
                return self._follow(node.value_ref)
        raise KeyError
```
This is a standard binary tree search, following refs to their nodes. We know
from reading the ``BinaryTree`` documentation that 
``Node``s and ``NodeRef``s are value objects:
they are immutable and their contents never change.
``Node``s are created
with an associated key and value,
and left and right children.
Those associations also never change.
The content of the whole ``BinaryTree`` only visibly changes
when the root node is replaced.
This means that we don't need to worry about the contents of our tree being
changed while we are performing the search. 

这是一个标准的二叉树搜索，以下是他们的节点。 我们阅读``BinaryTree``文档中知道，``Node``和``NodeRef``是值对象：它们是不可变的，它们的内容永远不会改变。``Node``将创建一个关联的键和值，以及左和右子节点。 那些关联也从不改变。 当根节点被替换时，整个``BinaryTree``的内容才会明显改变。 这意味着在执行搜索时，我们不需要担心我们的树的内容被改变。

Once the associated value is found, 
it is written to ``stdout`` by ``main()``
without adding any extra newlines,
to preserve the user's data exactly.

一旦找到关联的值，它将被写入``stdout`` by ``main()``而不添加任何额外的换行符，以完全保留用户的数据


#### Inserting and Updating

Now we'll set key ``foo`` to value ``bar`` in ``example.db``:
```bash
$ python -m dbdb.tool example.db set foo bar
```

Again, this runs the ``main()`` function from module ``dbdb.tool``. Since we've
seen this code before, we'll just highlight the important parts:
```python
# dbdb/tool.py
def main(argv):
    ...
    db = dbdb.connect(dbname)          # CONNECT
    try:
        ...
        elif verb == 'set':
            db[key] = value            # SET VALUE
            db.commit()                # COMMIT
        ...
    except KeyError:
        ...
```

This time we set the value with ``db[key] = value``
which calls ``DBDB.__setitem__()``.
```python
# dbdb/interface.py
class DBDB(object):
# ...
    def __setitem__(self, key, value):
        self._assert_not_closed()
        return self._tree.set(key, value)
```

``__setitem__`` ensures that the database is still open
and then stores the association from ``key`` to ``value``
on the internal ``_tree`` by calling ``_tree.set()``.

``_tree.set()`` is provided by ``LogicalBase``:
```python
# dbdb/logical.py
class LogicalBase(object):
# ...
    def set(self, key, value):
        if self._storage.lock():
            self._refresh_tree_ref()
        self._tree_ref = self._insert(
            self._follow(self._tree_ref), key, self.value_ref_class(value))
```

``set()`` first checks the storage lock:

```python
# dbdb/storage.py
class Storage(object):
    ...
    def lock(self):
        if not self.locked:
            portalocker.lock(self._f, portalocker.LOCK_EX)
            self.locked = True
            return True
        else:
            return False
```

There are two important things to note here: 

 - Our lock is provided by a 3rd-party file-locking library called
   [portalocker](https://pypi.python.org/pypi/portalocker).
 - `lock()` returns `False` if the database was already locked, and `True`
   otherwise.

Returning to `_tree.set()`, we can now understand why it checked the
return value of `lock()` in the first place: it lets us call
`_refresh_tree_ref` for the most recent root node reference
so we don't lose updates that another process may have made
since we last refreshed the tree from disk.
Then it replaces the root tree node
with a new tree containing the inserted (or updated) key/value.

Inserting or updating the tree doesn't mutate any nodes,
because ``_insert()`` returns a new tree.
The new tree shares unchanged parts with the previous tree
to save on memory and execution time.
It's natural to implement this recursively:
```python
# dbdb/binary_tree.py
class BinaryTree(LogicalBase):
# ...
    def _insert(self, node, key, value_ref):
        if node is None:
            new_node = BinaryNode(
                self.node_ref_class(), key, value_ref, self.node_ref_class(), 1)
        elif key < node.key:
            new_node = BinaryNode.from_node(
                node,
                left_ref=self._insert(
                    self._follow(node.left_ref), key, value_ref))
        elif node.key < key:
            new_node = BinaryNode.from_node(
                node,
                right_ref=self._insert(
                    self._follow(node.right_ref), key, value_ref))
        else:
            new_node = BinaryNode.from_node(node, value_ref=value_ref)
        return self.node_ref_class(referent=new_node)
```

Notice how we always return a new node
(wrapped in a ``NodeRef``).
Instead of updating a node to point to a new subtree,
we make a new node which shares the unchanged subtree.
This is what makes this binary tree an immutable data structure.

You may have noticed something strange here: we haven't made any changes to
anything on disk yet. All we've done is manipulate our view of the on-disk data
by moving tree nodes around.

In order to actually write these changes to disk, we need an explicit call to
`commit()`, which we saw as the second part of our `set` operation in `tool.py`
at the beginning of this section. 

Committing involves writing out all of the dirty state in memory,
and then saving the disk address of the tree's new root node. 

Starting from the API:
```python
# dbdb/interface.py
class DBDB(object):
# ...
    def commit(self):
        self._assert_not_closed()
        self._tree.commit()
```

The implementation of ``_tree.commit()`` comes from ``LogicalBase``:
```python
# dbdb/logical.py
class LogicalBase(object)
# ...
    def commit(self):
        self._tree_ref.store(self._storage)
        self._storage.commit_root_address(self._tree_ref.address)
```

All ``NodeRef``s know how to serialise themselves to disk
by first asking their children to serialise via ``prepare_to_store()``:
```python
# dbdb/logical.py
class ValueRef(object):
# ...
    def store(self, storage):
        if self._referent is not None and not self._address:
            self.prepare_to_store(storage)
            self._address = storage.write(self.referent_to_string(self._referent))
```

``self._tree_ref`` in ``LogicalBase`` is actually a ``BinaryNodeRef``
(a subclass of ``ValueRef``) in this case,
so the concrete implementation of ``prepare_to_store()`` is:
```python
# dbdb/binary_tree.py
class BinaryNodeRef(ValueRef):
    def prepare_to_store(self, storage):
        if self._referent:
            self._referent.store_refs(storage)
```

The ``BinaryNode`` in question, ``_referent``,
asks its refs to store themselves:
```python
# dbdb/binary_tree.py
class BinaryNode(object):
# ...
    def store_refs(self, storage):
        self.value_ref.store(storage)
        self.left_ref.store(storage)
        self.right_ref.store(storage)
```

This recurses all the way down for any ``NodeRef``
which has unwritten changes (i.e., no ``_address``).

Now we're back up the stack in `ValueRef`'s `store` method again. 
The last step of ``store()`` is to serialise this node
and save its storage address:
```python
# dbdb/logical.py
class ValueRef(object):
# ...
    def store(self, storage):
        if self._referent is not None and not self._address:
            self.prepare_to_store(storage)
            self._address = storage.write(self.referent_to_string(self._referent))
```

At this point
the ``NodeRef``'s ``_referent`` is guaranteed to have addresses available for all of its own refs,
so we serialise it by creating a bytestring representing this node:
```python
# dbdb/binary_tree.py
class BinaryNodeRef(ValueRef):
# ...
    @staticmethod
    def referent_to_string(referent):
        return pickle.dumps({
            'left': referent.left_ref.address,
            'key': referent.key,
            'value': referent.value_ref.address,
            'right': referent.right_ref.address,
            'length': referent.length,
        })
```

Updating the address in the ``store()`` method
is technically a mutation of the ``ValueRef``.
Because it has no effect on the user-visible value,
we can consider it to be immutable.

Once ``store()`` on the root ``_tree_ref`` is complete
(in ``LogicalBase.commit()``),
we know that all of the data are written to disk.
We can now commit the root address by calling:
```python
# dbdb/physical.py
class Storage(object):
# ...
    def commit_root_address(self, root_address):
        self.lock()
        self._f.flush()
        self._seek_superblock()
        self._write_integer(root_address)
        self._f.flush()
        self.unlock()
```

We ensure that the file handle is flushed
(so that the OS knows we want all the data saved to stable storage like an SSD)
and write out the address of the root node.
We know this last write is atomic because we store the disk address on a sector boundary.
It's the very first thing in the file,
so this is true regardless of sector size,
and single-sector disk writes are guaranteed to be atomic by the disk hardware.

Because the root node address has either the old or new value
(never a bit of old and a bit of new),
other processes can read from the database without getting a lock.
An external process might see the old or the new tree,
but never a mix of the two.
In this way, commits are atomic.

Because we write the new data to disk and call the ``fsync`` syscall[^fsync]
before we write the root node address,
uncommitted data are unreachable.
Conversely, once the root node address has been updated,
we know that all the data it references are also on disk.
In this way, commits are also durable.

[^fsync]: Calling ``fsync`` on a file descriptor
   asks the operating system and hard drive (or SSD)
   to write all buffered data immediately.
   Operating systems and drives don't usually write everything immediately
   in order to improve performance.

We're done!


### How NodeRefs Save Memory

To avoid keeping the entire tree structure in memory at the same time,
when a logical node is read in from disk
the disk address of its left and right children
(as well as its value)
are loaded into memory.
Accessing children and their values
requires one extra function call to ``NodeRef.get()``
to dereference ("really get") the data.

All we need to construct a ``NodeRef`` is an address:

    +---------+
    | NodeRef |
    | ------- |
    | addr=3  |
    | get()   |
    +---------+

Calling ``get()`` on it will return the concrete node,
along with that node's references as ``NodeRef``s:

    +---------+     +---------+     +---------+
    | NodeRef |     | Node    |     | NodeRef |
    | ------- |     | ------- | +-> | ------- |
    | addr=3  |     | key=A   | |   | addr=1  |
    | get() ------> | value=B | |   +---------+
    +---------+     | left  ----+
                    | right ----+   +---------+
                    +---------+ |   | NodeRef |
                                +-> | ------- |
                                    | addr=2  |
                                    +---------+

When changes to the tree are not committed,
they exist in memory
with references from the root down to the changed leaves.
The changes aren't saved to disk yet,
so the changed nodes contain concrete keys and values
and no disk addresses.
The process doing the writing can see uncommitted changes
and can make more changes before issuing a commit,
because ``NodeRef.get()`` will return the uncommitted value if it has one;
there is no difference between committed and uncommitted data
when accessed through the API.
All the updates will appear atomically to other readers
because changes aren't visible
until the new root node address is written to disk.
Concurrent updates are blocked by a lockfile on disk.
The lock is acquired on first update, and released after commit.


### Exercises for the Reader

DBDB allows many processes to read the same database at once without blocking;
the tradeoff is that readers can sometimes retrieve stale data.  What if we
needed to be able to read some data consistently?  A common use
case is reading a value and then updating it based on that value. How would you
write a method on `DBDB` to do this? What tradeoffs would you have to incur to
provide this functionality?

The algorithm used to update the data store
can be completely changed out
by replacing the string ``BinaryTree`` in ``interface.py``.
Data stores tend to use more complex types of search trees
such as B-trees, B+ trees, and others
to improve the performance.
While a balanced binary tree
(and this one isn't)
needs to do $O(log_2(n))$ random node reads to find a value,
a B+ tree needs many fewer, for example $O(log_{32}(n))$
because each node splits 32 ways instead of just 2.
This makes a huge different in practice,
since looking through 4 billion entries would go from
$log_2(2^{32}) = 32$ to $log_{32}(2^{32}) \approx 6.4$ lookups.
Each lookup is a random access,
which is incredibly expensive for hard disks with spinning platters.
SSDs help with the latency, but the savings in I/O still stand.

By default, values are stored by `ValueRef`
which expects bytes as values
(to be passed directly to `Storage`).
The binary tree nodes themselves
are just a sublcass of `ValueRef`.
Storing richer data
via <a href="http://json.org">json</a> or <a href="http://msgpack.org">msgpack</a> is a matter of writing your own
and setting it as the `value_ref_class`.
`BinaryNodeRef` is an example of using
[pickle](https://docs.python.org/3.4/library/pickle.html)
to serialise data.

Database compaction is another interesting exercise.
Compacting can be done via an infix-of-median traversal of the tree writing
things out as you go.
It's probably best if the tree nodes all go together,
since they're what's traversed
to find any piece of data.
Packing as many intermediate nodes as possible
into a disk sector
should improve read performance,
at least right after compaction.
There are some subtleties here
(for example, memory usage)
if you try to implement this yourself.
And remember:
always benchmark performance enhancements before and after!
You'll often be surprised by the results.

### Patterns and Principles 

Test interfaces, not implementation.
As part of developing DBDB,
I wrote a number of tests
that described how I wanted to be able to use it.
The first tests ran against an in-memory version of the database,
then I extended DBDB to persist to disk,
and even later added the concept of NodeRefs.
Most of the tests didn't have to change,
which gave me confidence that things were still working.

Respect the Single Responsibility Principle.
Classes should have at most one reason to change.
That's not strictly the case with DBDB,
but there are multiple avenues of extension
with only localised changes required.
Refactoring as I added features was a pleasure!


### Summary

DBDB is a simple database that makes simple guarantees, and yet
things still became complicated in a hurry. The most important thing I did to
manage this complexity was to implement an ostensibly mutable object with an
immutable data structure. I encourage you to consider this technique the next
time you find yourself in the middle of a tricky problem that seems to have
more edge cases than you can keep track of. 
