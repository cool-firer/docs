# nodejs delete cache问题

在test/建三个文件：

test/parentA.js

```javascript
'use strict';

const add = require('./child');

function testA() {
    return add(1, 1);
}

module.exports.version = 'A';
module.exports.testA = testA;
```

test/parentB.js

```javascript
'use strict';

const add = require('./child');

function testB() {
    return add(2, 2);
}

module.exports.version = 'B';
module.exports.testB = testB;
```

test/child.js

```javascript
'use strict';

function test(a, b) {
    return a + b;
}

module.exports = test;
module.exports.version = 1;
```

在REPL下，先require parentA, 再require parentB, 看看父子关系：

```shell
test> node
elcome to Node.js v12.18.0.
Type ".help" for more information.

# 现在是空的
> require.cache
[Object: nul prototype] {}

# 先parentA
# A require了child, 当前cache没有child.js, 先new一个child Module
# child的parent就指向了A, 同时在A的children加入child
> require('./parentA')
{ version: 'A' }

# 此时的关系是 A.children包含了child
# child.parent指向A
> require.cache
[Object: null prototype] {
  'D:\\test\\parentA.js': Module {
    id: 'D:\\test\\parentA.js',
    exports: { version: 'A', testA: [Function: testA] },
    parent: Module {
      id: '<repl>',
    },
    children: [ [Module] ], # 包含了child
  },
  'D:\\test\\child.js': Module {
    id: 'D:\\test\\child.js',
    exports: [Function: test] { version: 1 },
    # 指向A
    parent: Module {
      id: 'D:\\test\\parentA.js',
    },
    children: [],
  }
}

# 再require B
# B require了child, 此时cache里已经有了child, 因此不会去new Module,
# 只会在B.children加入child, child.parent指向不变, 还是A
> require('./parentB')
{ version: 'B' }


# 再看看此时关系 
> require.cache
[Object: null prototype] {
  'D:\\test\\parentA.js': Module {
    id: 'D:\\test\\parentA.js',
    exports: { version: 'A', testA: [Function: testA] },
    parent: Module {
      id: '<repl>',
    },
    children: [ [Module] ],
  },
  'D:\\test\\child.js': Module {
    id: 'D:\\test\\child.js',
    exports: [Function: test] { version: 1 },
    # 虽然A B同时require了child, 但child只会保留第一次的parent
    parent: Module {
      id: 'D:\\test\\parentA.js',
    },
    children: [],
  },
  'D:\\test\\parentB.js': Module {
    id: 'D:\\test\\parentB.js',
    exports: { version: 'B', testB: [Function: testB] },
    parent: Module {
      id: '<repl>',
    },
    children: [ [Module] ],
  }
}
> let A = require.cache[require.resolve('./parentA.js')]
> let B = require.cache[require.resolve('./parentB')]
> let C = require.cache[require.resolve('./child')]
> A.exports.version # 'A'
> A.exports.testA() # 2
```

此时的依赖关系如图：

![nodejs_cache1.png](E:\chat\docs\images\nodejs_cache1.png)

此时我要热更C, 如果只delete C：

```javascript
delete require.cache[require.resolve('./child.js')]
```

是完全不行的，只是删除了cache里的记录，AB的依赖还是旧的。

那么把A B的children删除呢？

```javascript
> A.children.splice(A.children.indexOf(C), 1)
> 或者 A.splice(0, 1)

> A.children
[]

# 还是可以正常调用
> A.exports.testA()
2
```

如果需要热更，需要清楚的知道每个module的依赖关系，才能确保不出现内存泄露。



参考资料： https://zhuanlan.zhihu.com/p/34702356
