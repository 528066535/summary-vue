## Object 的变化侦测

实际上，Array的变化侦测与Object不同。

### 什么是变化侦测

变化侦测就是侦测一个数据的变化，只有知道了数据的变化，才能继续做其他事情。JS有两个办法可以侦测到对象的变化，一个是Object.defineProperty和
ES6的Proxy。vue2.x 版本使用的是前者。Vue3.x会用ES6的Proxy方法去重写这部分代码。

### Object.defineProperty

直接上代码：

    function defineReactive(data, key, val) {
        Object.defineProperty(data, key, {
            enumerable: true,
            configurable: true,
            get: function() {
                return val
            },
            set: function(newVal) {
                if(val === newVal) {
                    return
                }
                val = newVal
            }
        })
    }

这里把 Object.defineProperty 再次封装，其中get方法可以获取到data中的某个数据的值，set方法，可以知道数据的变化，参数只需要三个。

### 依赖收集和触发

我们知道，get可以获取到某个数据的值，而在get的过程中，我们可以做一些事情，把数据的变化收集起来，这个过程就是收集依赖，而当调用set方法的时候，我们需要
通知其他方法，数据发生了变化，这时候会触发在get中收集的依赖。

#### 依赖收集和触发的代码逻辑

那这些依赖，具体是怎么收集起来的呢？假设某个依赖是被存到了window.target上，并且假设window.target是一个函数，那么我们可以这么写。

上代码：

    function defineReactive(data, key, val) {

        let dep = [] //新增

        Object.defineProperty(data, key, {
            enumerable: true,
            configurable: true,
            get: function() {

                dep.push(window.target) //新增

                return val
            },
            set: function(newVal) {
                if(val === newVal) {
                    return
                }

                //新增
                for(let i = 0; i < dep.length; i++) {
                    dep[i](newVal, val)
                }

                val = newVal
            }
        })
    }

这里的dep是用来存储被收集到的依赖。set中循环dep出发收集到的依赖。

我们再封装一下Dep类，专门管理依赖，解耦合，上代码：

    export default class Dep {
        constructor() {
            this.subs = [];
        }

        addSub(sub) {
            this.subs.push(sub)
        }

        removeSub(sub) {
            remove(this.subs, sub)
        }

        depend() {
            if(window.target) {
                this.addSub(window.target)
            }
        }

        notify(){
            const subs = this.subs.slice();
            for(let i = 0, l = subs.length; i < l; i++) {
                sub[i].update();
            }
        }
    }

改造后的 defineReactive方法：

    function defineReactive(data, key, val) {

        let dep = new Dep() //修改

        Object.defineProperty(data, key, {
            enumerable: true,
            configurable: true,
            get: function() {

                dep.depend() //修改

                return val
            },
            set: function(newVal) {
                if(val === newVal) {
                    return
                }

                dep.notify() //修改

                val = newVal
            }
        })
    }

所以，依赖被收集在Dep中了。

#### 依赖是谁

在上面代码中，我们收集的依赖是window.target，而在Dep的notify方法中，调用了window.target的update方法通知，通知了什么呢？有可能是模板，
也有可能是用户的watch，而他的作用就很明显了，就是set的时候通知它，数据发生了变化，它在专门通知其他地方数据更新了。而它就叫做Watcher。

### Watcher是什么

Watcher就是一个中间角色，数据变化时会回调Watcher中的方法， Watcher再告诉其他地方，数据变化了，需要更新模板了等等。

Watcher的经典实用方式：

    vm.$watch('a.b',function(newVal, oldVal){

    })

当a.b属性发生变化时，会调用回调函数，那么我们如何实现呢？大概的思路是，侦测a.b数据的变化，把它存到依赖数组中，每次数据变化，告知watch去
调用update，并在update方法中执行这个回调函数。

那么，我们先下Watch看代码吧：

    export default class Watcher {
        constructor(vm, expOrFn, cb) {
            this.vm = vm;
            //执行this.getter(),可以读取对应的值
            this.getter = parsePath(expOrFn)
            this.cb = cb
            this.value = this.get();
        }

        get(){
            window.target = this;
            let value = this.getter.call(this.vm, this.vm)
            window.target = undefined
            return value
        }

        update() {
            const oldValue = this.value
            this.value = this.get();
            this.cb.call(this.vm, this.value, this.oldValue)
        }
    }

那么，这么少的代码是如何包含刚刚说的，那么复杂的逻辑呢？首先，我们在defineReactive方法中，侦测了a.b数据，接下来会调用新建一个Watcher的实例，
在新建Watcher的实例中，我们会调用get方法。

在get方法中，我们设置了window.target的值是当前Watcher，然后再去获取当前值，在调用getter方法获取当前值的过程中，我们会调用Object.defineProperty
中的get方法，而get方法会讲window.target加入到dep的依赖数组中，这样下次数据再发生变化，就能调用到update方法并执行回调函数了。

get方法的最后，我们再把window.target置空，并返回当前值。

这里给出parsePath获取对象中对应key的值的实现代码：

    const bailRe = /[^\w.$]/
    export function parsePath(path) {
        if(bail.test(path)) {
            return
        }
        const segments = path.split(',')
        return function(obj) {

            for(let i = 0; i < segments.length; i++) {
                if(!obj) return

                obj = obj[segments[i]]
            }
            return obj
        }
    }

### 侦测所有的数据

现在我们需要再加一个Observer类，可以侦测所有需要侦测的数据。

上代码：

    export class Observer {
        constructor(value) {
            this.value = value

            if(!Array.isArray(value)) {
                this.walk(value)
            }
        }

        walk(obj) {
            const keys = Object.keys(obj)
            for(let i = 0; i < keys.length; i++) {
                defineReactive(obj, keys[i], obj[keys[i]])
            }
        }
    }

    function defineReactive(data, key, val) {
        //新增
        if(typeof val === 'object') {
            new Observer(val)
        }

        let dep = new Dep()

        Object.defineProperty(data, key, {
            enumerable: true,
            configurable: true,
            get: function() {
                dep.depend()
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

这样，我们就会对所有的data，增加侦测的功能了。当data中的任意属性发生变化时，这个属性对应的依赖（Watcher）就会接收到通知，并且发送给到其他地方。

### 无法侦测的数据

我们知道了变化侦测的原理是利用Object.defineProperty的get方法来增加依赖，set方法来通知Watcher，那么根据这个基本的原理我们可以推断出，有
些操作，并不会调用到get或者set方法，比如向object新增一个属性，这时候，并不会调用get或者set，又比如，我们调用delete方法，删除object的某个
属性，也不会调用到这两个方法。另外Object.assign()也不支持。

为了解决这个问题，Vue提供了两个Api，vm.$set 方法和 vm.$delete方法。而Object.assign可以用下面代码代替。

    this.someObject = Object.assign({}, this.someObject, { a: 1, b: 2 })


### 总结

![关系图](/img/vue-zc-1.png)

好不容易在百度上找到一个符合的图，所以就忽略解析指令的这部分吧。

首先，我们会创建一个Vue实例，这时候会新建一个Observer实例，这时候把所有的data下的属性都遍历一遍，并且调用defineReactive方法，defineReactive
方法会检测当前要侦测的数据是否是对象，如果是的话，再继续新建一个Observer实例，这样就能把data中所有的属性都遍历到了。

接下来，defineReactive方法调用了Object.defineProperty，劫持了所有属性，当调用到get方法的时候，会增加一个依赖（Watcher），当调用到set
方法的时候，会通知依赖（Watcher），让Watcher去通知其他地方。

而当外界读取数据的时候，都会触发Watcher的getter方法，这时候又会把Watcher放入依赖列表Dep中，而Dep通知Watcher数据发生变化后，Watcher又会
调用update方法，并在这个方法中，再通知其他地方。