## Array 的变化侦测

实际上，Array的变化侦测与Object不同。

### 追踪变化

object可以通过Object.defineProperty的setter来追踪到变化，当数据发生变化时，会出发setter。

而在数组中，我们可以使用push等方法来修改数组，所以，当用户调用push等方法的时候，可以收集到数组的变化。

那么我们如何拦截push等操作数据的方法呢？聪明的程序猿用自定义的push方法覆盖原生的push方法，这样，在每次调用push方法的时候，都能拦截到数据的变化了。

### 拦截器

其实拦截器中的方法名称是和Array.prototype方法都是一模一样的，除了一些会改变数组的方法，我们加了一层处理。而这些方法包含7个，push、pop、
shift、unshift、splice、sort、reverse。

上代码：

    const arrayProto = Array.prototype
    export const arrayMethods = Object.create(arrayProto);

    ;['push','pop','shift','unshift','splice','sort','reverse'].forEach(function(method) {
        const original = arrayProto[method]
        Object.defineProperty(arrayMethods, method, {
            value: function mutator(...args) {
                return original.apply(this, args)
            },
            enumerable: false,
            writable: true,
            configurable: true
        })
    })

上面代码创建了一个新的对象，包含数组所有的方法。然后又重写了这个新对象中的某些方法，这些方法的特点是会修改数组的元素。而在调用这些方法的时候
都会进入mutator方法，这时候，我们就能拦截到会让数组元素变化时候的方法了，然后再做去一些其他的事情，比如发送变化的通知。

### 拦截器覆盖Array原型

有了拦截器，我们就需要用拦截器上的方法覆盖原生提供的方法，但是又只能覆盖响应式数据，不然会污染全局

而侦测数据的方法在Observer方法中，代码如下：

    export class Observer {
        constructor(value) {
            this.value = value

            if(Array.isArray(value)) {
                value.__proto__ = arrayMethods
            }
            else {
                this.walk(value)
            }
        }
    }

使用__proto__是有争议的，也不鼓励使用它。因为它从来没有被包括在EcmaScript语言规范中，但是现代浏览器都实现了它。__proto__属性已在
ECMAScript 6语言规范中标准化，用于确保Web浏览器的兼容性，因此它未来将被支持。它已被不推荐使用, 现在更推荐使用Object.getPrototypeOf
和Object.setPrototypeOf（尽管如此，设置对象的[[Prototype]]是一个缓慢的操作，如果性能是一个问题，应该避免）。

### __proto__不兼容的处理

__proto__在ES6中标准化，在ES6之前大部分浏览器也支持了，但是也有浏览器不支持，所有，我们需要处理不兼容的情况。

而Vue的做法就是直接把arrayMethods的方法设置到被侦测的数组上。

    import { arrayMethods } from './array'

    //__proto__是否可用
    const hasProto = '__proto__' in {}
    const arrayKeys = Object.getOwnPropertyNames(arrayMethods)

    function protoAugment(target, src, keys) {
        target.__proto__ = src
    }

    function copyAugment(target, src, keys) {
        for(let i = 0; i = keys.length; i ++) {
            const key = key[i]
            def(target, key, src[key])
        }
    }

    export class Observer {
        constructor(value) {
            this.value = value

            if(Array.isArray(value)) {
                const augment = hasProto ? protoAugment : copyAugment
                augment(value, arrayMethods, arrayKeys)
            }
            else {
                this.walk(value)
            }
        }
    }

我们可以对比，当对象发生变化，我们可以在Object.defineProperty的setter方法中知道，那么，数组则是在拦截器中，知道数组的变化，
我们又知道了，对象的依赖是在getter中收集的，那么，在数组的依赖在哪呢？

### 依赖的作用和收集

我们在讨论对象的时候，就知道，依赖是一个Watch，当数据发生变化的时候，我们会通知依赖列表里面的Watch，那么数组也是一样的，并且，数组也是在
Object.defineProperty的getter方法中收集依赖的。

为什么会进入getter方法，因为我们要用数组，必须要获取数组的值，获取数组的值就必须调用数组的getter方法。

### 依赖列表对象的声明

我们知道对象的依赖列表对象是在defineReactive类中声明的，在getter方法中添加依赖，在setter中触发依赖的更新方法。

但是在数组中，我们触发依赖应该是在拦截器中，收集却是在defineReactive中，所以我们既要在defineReactive中访问到依赖列表，又能在拦截器中访问
到依赖列表。所以，我们Vue选择把Array的依赖放在Observer中：

    export class Observer {
        constructor(value) {
            this.value = value
            //新增
            this.dep = new Dep()
            //新增，假设这个方法可以对第一个参数增加一个属性，这个属性指向Observer对象，先放一下
            def(value, '__ob__', this)
            if(Array.isArray(value)) {
                value.__proto__ = arrayMethods
            }
            else {
                this.walk(value)
            }
        }
    }

上面代码实现了新建依赖列表，def方法会在后面用到。这里标记一个问号了？

### 收集依赖

我们把Dep类放在了Observer中后，接下来需要收集依赖了，而收集依赖是在defineReactive方法中了：

    function defineReactive(data, key, val) {
            /** 去掉并修改
            *if(typeof val === 'object') {
            *    new Observer(val)
            *}
            **/
            let childOb = observe(val)

            let dep = new Dep()

            Object.defineProperty(data, key, {
                enumerable: true,
                configurable: true,
                get: function() {
                    dep.depend()

                    //新增数组的依赖添加
                    if(childOb) {
                        if (Array.isArray(value)) {
                            childOb.dep.depend()
                        }
                    }

                    return val
                },
                set: function(newVal) {
                    if(val === newVal) {
                        return
                    }

                    dep.notify()

                    val = newVal
                }
            })
        }

        /** 去掉并修改
        *   observe用于返回一个Observer实例
        *   而Observer的构造方法在遍历对象的时候会在每个对象上增加一个__ob__属性，如果有这个属性，则说明这个对象已经遍历过了
        **/
        export function observe(value, asRootData) {
            if(!isObject(value)) {
                return
            }

            let ob
            if(hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
                ob = value.__ob__
            } else {
                ob = new Observer(value)
            }
            return ob
        }

这有两个有点绕的点，第一个是__ob__，这个属性是在新建Observer的时候，通过def方法加的，他是指向Observer实例的，目的是是为了防止
重复创建Observer，可以在observe方法中判断，如果当前对象存在__ob__，就不用再次新建Observer实例了。这里再给出def方法的实现：

    function def(obj, key, val, enumerable) {
        Object.defineProperty(obj, key, {
            value: val,
            enumerable: !!enumerable,
            writable: true,
            configurable: true
        })
    }

第二点绕点是在get方法中，分别调用了dep.depend()和childOb.dep.depend()两次依赖，如果是对象当调用dep.depend()会有window.target标记
Watch，所以可以出发依赖的添加，而数组则没有window.target，所以不会触发，直接进入childOb.dep.depend()，这里加了数组的判断，所以对象也
进不来。

### 向依赖发送通知

我们收集依赖后，如何触发依赖呢，之前讲过依赖是在拦截器中触发，而在Observer的构造函数中，我们把dep放在了Observer的实例上，又把Observer的
实例指向对象的__ob__属性，那么我们的代码可以这么实现：

    ;['push','pop','shift','unshift','splice','sort','reverse'].forEach(function(method) {
        const original = arrayProto[method]

        def(arrayMethods, method, function mutator(...args){
            const result = original.apply(this, args)
            const ob = this.__ob__
            ob.dep.notify()
        })
    })

这样，我们就可以通知到Watch了。

### 侦测数组中的元素变化

对象需要侦测对象上每个属性，每个属性的属性...的变化，对应的数组也要侦测数组内，所有元素的变化，所以我们需要增加一个递归，来侦测数组中所有的
元素变化。

    export class Observer {
        constructor(value) {
            this.value = value
            this.dep = new Dep()
            def(value, '__ob__', this)

            if(Array.isArray(value)) {
                value.__proto__ = arrayMethods
                this.observerArray(value)
            }
            else {
                this.walk(value)
            }
        }

        observerArray(items){
            for(let i = 0; i = items.length; i < 1; i ++) {
                observe(item[i])
            }
        }
    }

observe方法已经讲过，作用是返回一个Observer实例，如果存在则直接返回，不存在则创建。

### 侦测新增元素的变化

上面我们讨论的都是数组不变的情况下，那么，当数组新增了一个元素，我们该怎么侦测呢？先上代码：

;['push','pop','shift','unshift','splice','sort','reverse'].forEach(function(method) {
        const original = arrayProto[method]

        def(arrayMethods, method, function mutator(...args){
            const result = original.apply(this, args)
            const ob = this.__ob__

            let inserted

            switch(method) {
                case 'push':
                case 'unshift':
                    inserted = args
                    break
                case 'splice':
                    inserted = args.slice(2)
                    break
            }
            if(inserted) ob.observeArray(inserted)
            ob.dep.notify()
        })
    })

我们可以找到，增加的方法，并获取增加的参数，然后再把这些方法执行一遍observerArray方法。

### 无法侦测的情况

所有，数组检测不到的一种情况是直接根据下标修改数组的值，比如

    this.list[1] = 2

Vue 提供了$set 方法帮助我们修改不能侦测到的情况

    vm.$set(vm.items, indexOfItem, newValue)

再比如直接修改数组的长度

    this.list.length = 0

可以用这种方法来解决

    // Array.prototype.splice
    this.list.items.splice(0, this.list.length)

### 总结

我们知道object是在getter中收集依赖，在setter中，发送通知给依赖，而数组也是在getter中收集依赖，但是实在拦截中，发送通知给依赖。

我们为了防止拦截器污染全局数组方法，只设置响应数据的数组方法，这种方式的实现是覆盖实例的__proto__来实现，但是这样会有兼容性问题，如果不兼容，
我们直接遍历方法，并覆盖。

另外我们的依赖列表要收集又要通知，所以必须在getter方法中和拦截器中同时都能获取到，所以我们放在了Observer类中，而作为辅助我们用一个方法observe
用来返回Observer实例（如果存在，则返回，不存在，则新建），注意这里会把Observer的实例放在对象的__ob__属性上，这样方便获取。

整个流程大概是在defineReactive中创建Observer实例并返回（如果存在着直接返回）。接下来可以在getter中收集，由于dep依赖列表是放在Observer的实例
上了，所以我们可以直接调用Observer实例的dep属性达到收集依赖的目的。而在Observer创建的过程中，我们也已经把拦截器覆盖到了需要响应的数组上，这样
在调用会修改数组元素的方法的时候，我们可以通知依赖，数组已经发生了变化，另外我们需要注意的是，新增的元素我们要重新设置为响应的数据。而新增的情况
只有调用push、unshift、splice的第三个参数存在的情况下。

最后需要注意直接修改下标或者长度并不能拦截到数组元素的变化。
