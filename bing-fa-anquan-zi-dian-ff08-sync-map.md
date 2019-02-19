## 原生字典

Go语言自带的字典类型map，就是原生字典，并不是并发安全的。  
在使用原生字典的时候，应该在启动goroutine之前就完成字典的初始化和赋值。或者更高级的做法是，可以在goroutine中，在首次使用的时候通过sync.Once来并发安全的完成初始化和赋值的操作，达到一个延迟初始化的优化效果。之后在使用字典的时候，就只能获取其中的内容，不能再对其进行修改了。这个在讲sync.Once时，在最后有示例。  
如果是需要一个并发安全的，可以修改内容的原生字典，也不是太麻烦。加上互斥锁或读写锁就可以轻松实现了。下面就是一个自制的简易并发安全字典：

```go
package main

import (
    "fmt"
    "sync"
)

// 一个自制的简易并发安全字典
type ConcurrentMap struct {
    m map[interface{}]interface{}
    mu sync.RWMutex
}

func NewConcurrentMap() *ConcurrentMap {
    return &ConcurrentMap{
        m: make(map[interface{}]interface{}),
    }
}

func (cm *ConcurrentMap) Delete(key interface{}) {
    cm.mu.Lock()
    defer cm.mu.Unlock()
    delete(cm.m, key)
}

func (cm *ConcurrentMap) Load(key interface{}) (value interface{}, ok bool) {
    cm.mu.RLock()
    defer cm.mu.RUnlock()
    value, ok = cm.m[key]
    return
}

func (cm *ConcurrentMap) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool) {
    cm.mu.Lock()
    defer cm.mu.Unlock()
    actual, loaded = cm.m[key]
    if loaded {
        return
    }
    cm.m[key] = value
    actual = value
    return
}

func (cm *ConcurrentMap) Store(key, value interface{}) {
    cm.mu.Lock()
    defer cm.mu.Unlock()
    cm.m[key] = value
}

func (cm *ConcurrentMap) Range(f func(key, value interface{}) bool) {
    cm.mu.RLock()
    defer cm.mu.RUnlock()
    for k, v := range cm.m {
        if !f(k, v) {
            break
        }
    }
}

func main() {
    pairs := []struct{
        k string
        v int
    }{
        {"k1", 1},
        {"k2", 2},
        {"k3", 3},
        {"k4", 4},
    }

    {
        fmt.Println("创建map")
        cm := NewConcurrentMap()
        for i := range pairs {
            cm.Store(pairs[i].k, pairs[i].v)
        }
        fmt.Println(cm.m)

        fmt.Println("遍历输出map")
        cm.Range (func(k, v interface{}) bool {
            fmt.Printf("%v: %v\n", k, v)
            return true
        })

        fmt.Println("Load 和 LoadOrStore 方法")
        key := "k3"
        value, ok := cm.Load(key)
        fmt.Printf("Load %v: %v: %v\n", ok, key, value)
        value, ok = cm.LoadOrStore(key, 5)
        fmt.Printf("LoadOrStore %v: %v: %v\n", ok, key, value)
        key = "k5"
        value, ok = cm.Load(key)
        fmt.Printf("Load %v: %v: %v\n", ok, key, value)
        value, ok = cm.LoadOrStore(key, 5)
        fmt.Printf("LoadOrStore %v: %v: %v\n", ok, key, value)
        fmt.Println(cm.m)

        fmt.Println("Delete 方法")
        key = "k2"
        cm.Delete(key)
        fmt.Println(cm.m)
        key = "k21"
        cm.Delete(key)
        fmt.Println(cm.m)
    }
    fmt.Println("Over")
}
```

## 并发安全字典

Go言语官方是在Go 1.9中才正式加入了并发安全的字典类型sync.Map。所以在这之前，已经有很多库提供了类似的数据结构，其中有一些的性能在很大程度上有效的避免了的锁的依赖，性能相对会好一些。不过这些都是过去了，现在有了官方的并发安全的字典类型sync.Map可以使用。  
下面就是使用并发安全字典的示例

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    pairs := []struct{
        k string
        v int
    }{
        {"k1", 1},
        {"k2", 2},
        {"k3", 3},
        {"k4", 4},
    }

    {
        fmt.Println("创建map")
        cm := sync.Map{}
        for i := range pairs {
            cm.Store(pairs[i].k, pairs[i].v)
        }

        fmt.Println("遍历输出map")
        cm.Range (func(k, v interface{}) bool {
            fmt.Printf("%v: %v\n", k, v)
            return true
        })

        fmt.Println("Load 和 LoadOrStore 方法")
        key := "k3"
        value, ok := cm.Load(key)
        fmt.Printf("Load %v: %v: %v\n", ok, key, value)
        value, ok = cm.LoadOrStore(key, 5)
        fmt.Printf("LoadOrStore %v: %v: %v\n", ok, key, value)
        key = "k5"
        value, ok = cm.Load(key)
        fmt.Printf("Load %v: %v: %v\n", ok, key, value)
        value, ok = cm.LoadOrStore(key, 5)
        fmt.Printf("LoadOrStore %v: %v: %v\n", ok, key, value)

        fmt.Println("Delete 方法")
        key = "k2"
        cm.Delete(key)
        key = "k21"
        cm.Delete(key)

        fmt.Println("遍历输出map")
        cm.Range (func(k, v interface{}) bool {
            fmt.Printf("%v: %v\n", k, v)
            return true
        })
    }
    fmt.Println("Over")
}
```

## 并发安全字典与原生字典的比较

这里从3个方面对并发安全字典和原生字典进行了比较。  
**算法复杂度**  
这个字典类型提供了一些常用的键值存取操作方法，并保证了这些操作的并发安全。同时，它的存、取、删除等操作都可以基本保证在常数时间内执行完毕。就是说，它的算法复杂度与原生字典一样都是O\(1\)的。  
**效率和锁**  
与单纯使用原生字典和互斥锁的用法相比，使用并发安全字典可以显著减少锁的争夺。虽然并发安全字典本身也用到了锁，但是它在尽可能的避免使用锁。使用锁，就以为着要把一些并发的操作强制串行化，这就会降低程序的性能。因此，在讲原子操作的时候，就建议能用原子操作就不要用锁。不过当时也说了，原子操作支持的数据类型有局限性。  
**编译器的支持**  
无论在何种场景下使用并发安全字典，都需要记得，它与原生字典是明显不同的。并发安全字典只有Go语言标准库中的一员，而原生字典是语言层面的东西。正是因为这个原因，Go语言的编译器不会对它的键和值进行特殊的类型检查。它所有的方法涉及的键和值类型都是空接口。所以，必须在程序中自行保证它的键类型和值类型的正确性。

## 并发安全字典key类型的要求

原生字典的key不能是如下的类型：

* 不能是，函数类型
* 不能是，字典类型
* 不能是，切片类型

并发安全字典的key的类型也是一样的限制。  
在并发安全字典内部，使用的存储介质仍然是原生字典，上面这些类型都不能用。但是由于并发安全字典的key类型的实际类型是空接口，就是可以是任何类型，所以编译器是允许使用任何类型的。如果使用了上述key不支持的类型，编译阶段是发现不了的。只要在程序运行期间才能确定键的实际类型，此时如果是不正确的key值实际类型肯定会引发panic。  
因此，一定不要违反字典对key类型的限制。应该在每次操作并发安全字典的时候，都去显式的检查key值的实际类型。无论是存、取、删除都是如此。  
更好的做法是，把针对同一个并发安全字典的几种操作都集中起来，然后统一的编写检查代码。如果是把并发安全字典封装在一个结构体里，是一个很好的选择。  
总之，必须保证key的类型是可比较的（或者说是可判等的）。如果实在确定不了，那么可以先通过调用reflect.TypeOf函数得到一个键对应的反射类型值（即：reflect.Type类型），然后再调用这个值的Comparable方法，得到确切的判断结果。_Comparable方法返回一个布尔值，判断类型是否是可比较的，即：判断是否可以作为key的类型。_

# 保证key和value的类型正确

简单说，可以使用类型断言表达式或者反射操作来保证类型正确。为了进一步明确并发安全字典中key值的实际类型，这里大致有两个方案。

## 只能存储特定的类型

这个是方案一：让并发安全字典只能存储某个特定类型的key。  
指定key只能是某个类型，一旦完全确定了key的类型，就可以在存、取、删除的时候，使用类型断言表达式去对key的类型做检查了。一般这种检查并不繁琐，而且要是把并发安全字典封装在一个结构体里，就更方便了。这时Go语言编译器就可以帮助完成检查的工作。就像下面这样：

```go
// 这是一个key为字符串类型，value为数字的字典
type StrIntMap struct {
    m sync.Map
}

func (ms *StrIntMap) Delete(key int) {
    ms.m.Delete(key)
}

func (ms *StrIntMap) Load(key int) (value int, ok bool) {
    v, ok := ms.m.Load(key)
    if v != nil {
        value = v.(int)
    }
    return
}

func (ms *StrIntMap) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool) {
    a, loaded := ms.m.LoadOrStore(key, value)
    actual = a.(int)
    return
}

func (ms *StrIntMap) Range(f func(key string, value int) bool) {
    f1 := func(key, value interface{}) bool {
        return f(key.(string), value.(int))
    }
    ms.m.Range(f1)
}

func (ms *StrIntMap) Store(key string, value int) {
    ms.m.Store(key, value)
}
```

把并发安全字典封装到结构体中之后，要把原本字典中公开的方法在结构体上再实现一遍。只要简单的调用并发安全字典里的方法就可以了。由于自己写的结构体的方法的参数列表里已经明确的定义了变量的类型，所以就不用再做类型检查了，并且使用编译器编写代码的时候类型错误也会有提示。基于上面的情况，使用这些方法取出key和value的时候，完全不用担心类型不正确，因为正确性在最初存入的时候就已经保证了。这里在取出值的时候，由于sync.Map返回的是空接口，所以需要用类型断言做一下类型转换。这里用类型断言的时候只使用一个返回值，直接转，不用担心会有错误。

## 使用反射

上一个方案虽然很好，但是有一点不方便。就是封装了一大堆内容，实现了一个key-value的类型。如果还是需要另外一个类型的字典，还得再封装同样的一大堆的内容。这样的话，在需求多样化之后，工作量反而更大，而且会有很多雷同的代码。  
这里需要一个这样效果的方案：既要保持sync.Map类型原有的灵活性，又可以约束key和value的类型。  
这个是方案二：就是通过反射来实现做类型的判断

### 结构体的类型

这次在设计结构体类型的时候，只包含sync.Map类型的字段就不够了，需要向下面这样：

```go
type ConcurrentMap struct {
    m         sync.Map
    keyType   reflect.Type
    valueType reflect.Type
}
```

这次多定义了2个字段keyType和valueType，分别用于保存key类型和value类型。类型都是reflect.Type，就是**反射类型**。这个类型可以代表Go语言的任何数据类型，并且类型的值也非常容易获得：就是通过reflect.TypeOf函数并把某个样本值传入即可，比如：`reflect.TypeOf(int(3))`，就是int类型的反射类型值。

### 构造函数

首先提供一个方法，让外部的代码可以创建一个结构体：

```go
func NewConcurrentMap(keyType, valueType reflect.Type) (*ConcurrentMap, error) {
    if keyType == nil {
        return nil, fmt.Errorf("key 类型为 nil")
    }
    if !keyType.Comparable() { // 判断类型是否可以比较，就是是否是内做key的类型
        return nil, fmt.Errorf("不可比较的类型: %s", keyType)
    }
    if valueType == nil {
        return nil, fmt.Errorf("value 类型为 nil")
    }
    cm := &ConcurrentMap{
        keyType:   keyType,
        valueType: valueType,
    }
    return cm, nil
```

创建成功，则返回结构体的指针和nil的错误类型。创建失败，则返回nil和具体的错误类型。  
先判断传入的类型是否为nil。另外对于key，基于对key类型的限制，必须是可比较的，这里用reflect包里的Comparable方法就可以进行判断了。

### Load方法

结构体有了，接下来就实现所有的方法。首先是Load方法：

```go
func (cm *ConcurrentMap) Load(key interface{}) (value interface{}, ok bool) {
    if reflect.TypeOf(key) != cm.keyType{
        return
    }
    return cm.m.Load(key)
}
```

这里的类型检查的代码非常简单。如果参数key的反射类型与keyTpye字段的反射类型不相等，那么就直接返回空值。_类型都不对，那么字典里是一定没有这个key的，所以就直接返回查询不到的结果。_这时，返回的是nil和false，完全符合Load方法原本的含义。

### Store方法

Stroe方法接收2个空接口类型，没有返回值：

```go
func (cm *ConcurrentMap) Store(key, value interface{}) {
    if reflect.TypeOf(key) != cm.keyType {
        panic(fmt.Errorf("key类型不符合: 实际类型: %v, 需要类型: %v\n", reflect.TypeOf(key), cm.keyType))
    }
    if reflect.TypeOf(value) != cm.valueType {
        panic(fmt.Errorf("value类型不符合: 实际类型: %v, 需要类型: %v\n", reflect.TypeOf(value), cm.valueType))
    }
}
```

这里的做法是一旦类型检查不符，就直接引发panic。这么做主要是由于Store方法没有结果声明，就是无返回值，所以在参数值有问题的时候，无法通过比较平和的方式告知调用方。这么做也是符合Store方法的原本含义的。  
也可以把结构体的Store方法改的和sysc.Map里的Store方法稍微不一样一点，就是加上结果声明，返回一个error类型。这样当参数值类型不正确的时候，就返回响应的错误，而不引发panic。作为示例，就先用panic的方式吧。在实际的应用场景里可以再做优化和改进。

### 完整的定义

其他的方法就都差不多了，下面展示了所有定义的方法：

```go
type ConcurrentMap struct {
    m         sync.Map
    keyType   reflect.Type
    valueType reflect.Type
}

func NewConcurrentMap(keyType, valueType reflect.Type) (*ConcurrentMap, error) {
    if keyType == nil {
        return nil, fmt.Errorf("key 类型为 nil")
    }
    if !keyType.Comparable() { // 判断类型是否可以比较，就是是否是内做key的类型
        return nil, fmt.Errorf("不可比较的类型: %s", keyType)
    }
    if valueType == nil {
        return nil, fmt.Errorf("value 类型为 nil")
    }
    cm := &ConcurrentMap{
        keyType:   keyType,
        valueType: valueType,
    }
    return cm, nil
}

func (cm *ConcurrentMap) Load(key interface{}) (value interface{}, ok bool) {
    if reflect.TypeOf(key) != cm.keyType {
        return
    }
    return cm.m.Load(key)
}

func (cm *ConcurrentMap) Store(key, value interface{}) {
    if reflect.TypeOf(key) != cm.keyType {
        panic(fmt.Errorf("key类型不符合: 实际类型: %v, 需要类型: %v\n", reflect.TypeOf(key), cm.keyType))
    }
    if reflect.TypeOf(value) != cm.valueType {
        panic(fmt.Errorf("value类型不符合: 实际类型: %v, 需要类型: %v\n", reflect.TypeOf(value), cm.valueType))
    }
    cm.m.Store(key, value)
}

func (cm *ConcurrentMap) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool) {
    if reflect.TypeOf(key) != cm.keyType {
        panic(fmt.Errorf("key类型不符合: 实际类型: %v, 需要类型: %v\n", reflect.TypeOf(key), cm.keyType))
    }
    if reflect.TypeOf(value) != cm.valueType {
        panic(fmt.Errorf("value类型不符合: 实际类型: %v, 需要类型: %v\n", reflect.TypeOf(value), cm.valueType))
    }
    actual, loaded = cm.m.LoadOrStore(key, value)
    return
}

func (cm *ConcurrentMap) Delete(key interface{}) {
    if reflect.TypeOf(key) != cm.keyType {
        return
    }
    cm.m.Delete(key)
}

func (cm *ConcurrentMap) Range(f func(key, value interface{}) bool) {
    cm.m.Range(f)
}
```

## 总结

第一种方案，适用于可以完全确定key和value的具体类型的情况。这是，可以利用Go语言的编译器去做类型检查，并用类型断言表达式作为辅助。缺陷是一次定义只能满足一种字典类型，一旦字典类型多样化，就需要编写大量的重复代码。不过还有一个好处，就是在编译阶段就可以发现类型不正确的问题。  
第二种方案，在程序运行之前不用明确key和value的类型，只要在初始化并发安全字典的时候，动态的给定key和value的类型即可。主要是用到了reflect包中的方法和数据类型。灵活性高了，一次就能满足所有的字典类型。但是那些反射操作或多或少都会降低程序的性能。而且如果有类型不符合的情况，无法在编译阶段发现。

# 并发安全字典内部的实现

保证并发安全，还要保证效率，主要的问题就是要尽量避免使用锁。  
sync.Map类型在内部使用了大量的原子操作来存取key和value，并使用了两个原生的字典作为存储介质。

## 只读字典

其中一个原生的字典是sync.Map的read字段，类型是sync/atomic.Value类型。这个原生字典可以被看作一个快照，总会在条件满足时，去重新保存所属的sync.Map值中包含的所有key-value对。这里叫它**只读字典**，它虽然不会增减其中的key，但是允许改变其中的key所对应的value。所以这个只读特性只是对于所有的key的。  
因为是sync/atomic.Value类型，所以都是原子操作，不用锁。另外，这个只读字典在存储key-value对的时候，对value还做了一层封装。先把值转换为了unsafe.Pointer类型的值，然后再把unsafe.Pointer也封装，并存储在其中的原生字典中。如此一来，在变更某个key所对应的value的时候，就可以使用原子操作了。

## 脏字典

另一个原生字典是sync.Map的dirty字段，它的key类型是空接口。它存储key-value对的方式与read字段中的原生字典一致，并且也是把value的值先做转换和封装后再进行存储的。这里叫它**脏字典**。  
有点不太好理解，看下源码里的类型定义帮助理解只读字典和脏字典的类型：

```go
type Map struct {
    mu Mutex
    read atomic.Value // readOnly
    dirty map[interface{}]*entry
    misses int
}

// readOnly is an immutable struct stored atomically in the Map.read field.
type readOnly struct {
    m       map[interface{}]*entry
    amended bool // true if the dirty map contains some key not in m.
}

type entry struct {
    p unsafe.Pointer // *interface{}
}
```

## 存取值的过程

了解下sync.Map是如何巧妙的使用它的2个原生字典的。分析各个方法在执行时具体进行的操作，再判断这些操作对性能的影响。

**获取值的过程**  
sync.Map在查找指定的key所对应的value的时候，会先去只读字典中查找，只读字典是原子操作，不要要锁。只有当只读字典里没有，再去脏字典里查找，访问脏字典需要加锁。

**存储值的过程**  
在存key-value的时候，只要只读字典中已经存在这个key了，并且未被标记为删除，就会把新值直接存到value里然后直接返回，还是不需要加锁。否则，才会加锁后，把新的key-value加到脏字典里。这个时候还要抹去只读字典里的删除标记。

**删除标记**  
当一个kye-value对应该被删除，但是仍然存在于只读字典中的时候，会用标记来标记删除，但是还不会直接武林删除。  
这种情况会在重建脏字典以后的一段时间内出现。不过，过不了多久，还是会被真正的删除掉的。在查找和遍历的时候，有删除标记的key-value对会忽略的。

**删除的过程**  
删除key-value对，sync.Map会先检查只读字典中是否有对应的key。如果没有，可能会在脏字典中，那就加锁然后试图从脏字典里删掉该key-value对。最后，sync.Map会把key-value对中指向value的那个指针置为nil，这是另一种逻辑删除的方式，脏字点重建之后自然就真正删除了。

**重建脏字典**  
只读字典和脏字典之间是会互相转换的。在脏字典中查找key-value对足够次数时，sync.Map会把脏字典直接作为只读字典，保存到read字段中。然后把dirty字段置为nil。之后一旦再有新的key-value存入，就会依据只读字典去重建脏字典，会把只读字典标记为删除的key-value对过滤掉。这些转换操作都有脏字典，所以都需要加锁。

**小结**  
sync.Map的只读字典和脏字典中的key-value对并不是实时同步的。由于只读字典的key不能改变，所以其中的key-value对可能是不全的。而脏字典中的key-value对总是全的，不多也不少。新的key-value对只读字典里没有；已经标记删除的key-value对物理上还在只读字典里，只是加了标记。  
在读操作为主，而写操作很少的情况下，并发安全字典的性能会更好。在几个写操作中，新增key-value对的操作对并发安全字典的性能影响最大，其次是删除操作。修改操作影响略小，只读字典里如果有这个key的话，并且没有标记删除，直接在只读字典里把value改掉，不用加锁，对性能影响就会很小。

