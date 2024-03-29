我们的Lua类绑定机制
======

[TOC]

前言
------

最近一个人搞后台，框架底层+逻辑功能茫茫多，扛得比较辛苦，一直没抽出空来写点东西。

空闲的时间，完善了[LLVM+Clang+libc++和libc++abi的编译脚本](https://www.owent.net/?p=1149)。不得不说clang的编译脚本质量比gcc差不是一点点。为了在centos下尽可能编译出更多的功能和llvm的子项目，要各种改各种试才终于自举通过，并且这种情况编出来的二进制大得夸张。不过总算是告一段落。

因为项目需要，买了本[《Redis的设计与实现》](http://www.duokan.com/book/53962)的电子版，顺便给这本书的作者贡献了一丢丢的Redis文档的翻译。这本书通篇贴代码，有点看blog的感觉，写得还是很简单易懂的。（不过谁告诉我为毛我花了30大洋买完不到半个月就变成10块了，坑爹么不是。T_T）

另外还零零散散地看了些[《程序员的自我修养-链接、装载与库》](http://book.douban.com/subject/3652388/) 这本书[云风](http://blog.codingnow.com)之前推荐过。并且在我看得这些里国人写得书里，这确实是一本值得推荐，并且不可多得的佳作。

还是回到正题

为什么要重写Lua类绑定？
------

早先我们用得都是tolua++，但是tolua++貌似很久没有更新了，而且不支持lua大于5.1的版本。并且在使用的过程中发现了一些坑，比较隐晦+恶心。总不能让所有人时时刻刻记住这些坑吧？再加上服务器的lua脚本会比较轻量级，索性就用一个简单高效的Lua绑定机制。

即便如此，本来的第一选择是去找了个好像叫**LuaBridge**的项目。但是使用的时候发现，一是并不是很方便，另外就是也适配的不好，所以索性自己搞一个算了。

函数绑定的接口形式
------

先看我们函数绑定的最终成果 ，要绑定一个类和类成员，只要在cpp文件中加入类似下面的代码即可：
```cpp
// 这个FightBullet名字可以随意，只要保证全局唯一并且符合c++标识符规则即可，类似gtest
LUA_BIND_OBJECT(FightBullet) {
    // 这里面的代码会在Lua绑定管理器初始化时自动执行
    lua_State* L = script::lua::LuaEngine::Instance()->getLuaState();

    // 声明一个类，命名空间是game.logic，类名是FightBullet
    script::lua::LuaBindingClass<FightBullet> clazz("FightBullet", "game.logic", L);

    // 绑定类成员函数或static函数，将会自动推断函数、返回值和参数类型
    {
        clazz.addMethod("getId", &FightBullet::getId);
        clazz.addMethod("getTypeId", &FightBullet::getTypeId);
        clazz.addMethod("getFightRoomId", &FightBullet::getFightRoomId);
        clazz.addMethod("addActionTimer", &FightBullet::addActionTimer);
        clazz.addMethod("setNextActionFrame", &FightBullet::setNextActionFrame);
        clazz.addMethod("getNextActionFrame", &FightBullet::getNextActionFrame);
        clazz.addMethod("skill", &FightBullet::skill);
    }
}
```

是不是很简单？


面向对象的实现原理
------

### Lua层面向对象(模拟继承和覆盖)
Lua原生并不支持面向对象设计,所以我们这里使用了一个简单的方法对面向对象的特性做一些模拟。

根据Lua的特性，对一个table而言，rawger找不到的东西会去查找metatable的index。类似这种查找法

```lua
function gettable_event (table, key)
  local h
  if type(table) == "table" then
    local v = rawget(table, key)
    if v ~= nil then return v end
    h = metatable(table).__index
    if h == nil then return nil end
  else
    h = metatable(table).__index
    if h == nil then
      error(···)
    end
  end
  if type(h) == "function" then
    return (h(table, key))     -- call the handler
  else return h[key]           -- or repeat operation on it
  end
end
```

于是，我们可以用这种特性来模拟继承和覆盖。把table的metatable设为自身，__index设为父类的table。即：

```lua
local father = {}
local child = {}
setmetatable(child, child)
child.__index = father
```

为什么不新建一个table作为metatable，并把__index设为父类呢？

```lua
local father = {}
local child = {}
setmetatable(child, {__index = father})
```

如上面代码所示，不是更干净吗？而且还能更快地回收内存？原因有两个，一是不想空创建一个table，因为每一次创建table都意味着lua虚拟机的malloc操作。另一方面我们的类会设置一些通用的方法，比如说设置__tostring函数用于把对象转换成字符串，并实现对tostring方法的转换。这时候，我们会打印出当前类实例的实际类型。然而这个__tostring方法是写在基类里的，如果通过上面第一种方式，我们只要按某种规则把类名保存在table的某个字段里，基类直接读取这个字段即可。如果用第二种方法，由于执行__tostring的时候，父类函数里的self对象并不是子类对象，所以就获取不到子类信息。

比如:
```lua
-- tostring方法
function class.object:__tostring()
    if self then
        local ret = '[' .. self.__type .. ' : ' .. self.__classname .. ']'
        if self.__instance_id then
            ret = ret .. ' @' .. tostring(self.__instance_id)
        end
        return ret
    end
    return '[class : object] WARNING: you should use obj:__tostring but obj.__tostring'
end
```
这个tostring方法会打印出当前类的类型、类名和ID。这些数据都会按照链式的查找规则，一层一层地向上查找。这里的ID是指我们每创建一个类实例都会分配一个唯一ID，而**类型**在类里都是class，而类实例里都是object，其他的类型后面会提到。

### C++ binding层面向对象

C++类型绑定以后，也是走得上面的机制，所不一样的地方是，像诸如__tostring方法我们在本地（也就是C++）实现，并且把**类型**设为*native code*。

另外就是lua里保存C++对象一定要把metatable设成预定义好的元表。为了保存C++的成员函数，静态函数。我们采用了如下结构：

[![类图](res/2015/img01.png)](http://yuml.me/diagram/scruffy/class/edit///%20Cool%20Class%20Diagram,%20%5B%E7%BB%91%E5%AE%9A%E7%B1%BB%E5%90%8D%E7%9A%84%E5%85%83%E8%A1%A8%5D%3C%3E-%3E%5B%E7%B1%BB%E5%AE%9E%E4%BE%8B%5D,%20%5B%E6%88%90%E5%91%98%E5%87%BD%E6%95%B0%E5%8F%8A%E5%8F%98%E9%87%8F%E8%A1%A8%5D%3C%3E-%3E%5B%E7%BB%91%E5%AE%9A%E7%B1%BB%E5%90%8D%E7%9A%84%E5%85%83%E8%A1%A8%5D,%20%5B%E9%9D%99%E6%80%81%E6%88%90%E5%91%98%E8%A1%A8%5D%3C%3E-%3E%5B%E6%88%90%E5%91%98%E5%87%BD%E6%95%B0%E5%8F%8A%E5%8F%98%E9%87%8F%E8%A1%A8%5D)

然后把相应的函数设置在对应的table上就完事儿。

### 单例及命名空间

很多情况下都会用到单例，于是我们就干脆抽离出来。单例和其他类不同的地方在于，单例的new方法不会创建类实例，会直接返回自身。并且会增加一个instance方法为new方法的别名。

```lua
-- 单例基类
do
    class.singleton = class.register('utils.class.singleton')
    rawset(class.singleton, 'new', function(self, inst)
        if nil == self then
            log_error('singleton class must using class:new or class:instance(class.new and class.instance is unavailable)')
            return nil
        end
        return self
    end)

    rawset(class.singleton, 'instance', class.singleton.new)
end
```

为了方便管理和区分，注册类的时候给出的是类似 a.b.c.d 的路径，那么这时候，a、a.b和a.b.c就都是命名空间（如果事前没有注册过为类）。命名空间也是一个单例。

```lua
-- 命名空间
class.namespace = class.register('utils.class.namespace', class.singleton)
```

### 保护父类数据和const模拟

在我们实际应用中，为了防止一些误操作，需要一些方法来保护一些数据。比如说，a.b.c = 123。由于Lua的数据传递都是引用的方式，如果这时候 b 是基类里的table，就会使得基类里的东西被改动。为了防止这种情况，我们加了一个函数用于保护父类数据。

具体做法是，在table引用其他数据之前增加一个table，设置__newindex用于保存数据。然后采用类似继承的方式来读数据。

```lua
-- 保护table数据
function class.protect(obj, r)
    -- 和readonly混用的优化
    local tmb = getmetatable(obj)
    if tmb and class.readonly__newindex == tmb.__newindex then
        obj = tmb.__index
    end

    local wrapper = {}
    local wrapper_metatable = {}
    setmetatable(wrapper, wrapper_metatable)

    for k, v in pairs(obj) do
        if r and 'table' == type(v) or 'userdata' == type(v) then
            rawset(wrapper, k, class.protect(v, r))
        else
            rawset(wrapper, k, v)
        end
    end

    rawset(wrapper_metatable, '__index', obj)
    rawset(wrapper_metatable, '__newindex', function(tb, k, v)
        rawset(tb, k, v)
    end)
    return wrapper
end
```

另外，特别是配置表，为了防止使用者对其修改。我们还需要一个设置成*readonly*的功能，也就是**模拟const**功能。

readonly的实现原理和对象数据保护的方法类似。也是设置__newindex方法并且报错。不过有一个已知的问题，采用这种方法以后，next、unpack、pair和ipair函数的行为失效了。所以我们加了几个函数来处理这个问题。

```lua
-- readonly 重载__newindex函数
function class.readonly__newindex(tb, key, value)
    log_error('table %s is readonly, set key %s is invalid', tostring(tb), tostring(key))
end

-- 设置table为readonly
function class.set_readonly(obj)
    local tmb = getmetatable(obj)
    if not tmb or class.readonly__newindex ~= tmb.__newindex then
        local wrapper = {
            __index = obj,
            __newindex = class.readonly__newindex,
        }

        -- 设置readonly后 # 操作符失效的解决方案
        function wrapper:table_len() 
            return #obj
        end

        -- 设置readonly后pairs失效的解决方案
        function wrapper:table_pairs() 
            return pairs(obj)
        end

        -- 设置readonly后ipairs失效的解决方案
        function wrapper:table_ipairs() 
            return ipairs(obj)
        end
        
        -- 设置readonly后next失效的解决方案
        function wrapper:table_next(index) 
            return next(obj, index)
        end

        -- 设置readonly后unpack失效的解决方案
        function wrapper:table_unpack(index) 
            return table.unpack(obj, index)
        end

        -- 原始table
        function wrapper:table_raw() 
            return obj
        end

        -- 复制可写表
        function wrapper:table_make_writable() 
            local ret = table.extend(obj)
            for k, v in pairs(ret) do
                if 'table' == type(v) then
                    rawset(ret, k, v:table_make_writable())
                else
                    rawset(ret, k, v)
                end
            end

            return ret
        end

        setmetatable(wrapper, wrapper)

        for k, v in pairs(obj) do
            if 'table' == type(v) or 'userdata' == type(v) then
                rawset(obj, k, class.set_readonly(v))
            end
        end
        return wrapper
    end
    return obj
end
```

如果谁有更好的方法希望能不吝赐教。

对象的生命周期管理
------

最初，我们的C++对象的生命周期由Lua管理。即Lua的table释放的时候，在__gc方法里对C++对象执行delete。但是实际的使用过程中发现一个很严重的问题，就是Lua的内存回收时机是不确定的。这意味着反复多次执行进入场景的时候，有大量的不需要的对象没有释放掉。浪费了很多内存。

然而如果每次强制Lua进行垃圾回收会显著降低性能，所以后来我们采取了另一种方法。在Lua中记录C++对象的弱引用，在本地代码中使用管理器来管理这些对象。

实际上我们给Lua绑定的C++对象传入的是一个weak_ptr，在本地代码管理器中保存的对象的shared_ptr。调用成员函数时，如果对象已经被释放，则会报错并调用失败。

对于本地对象生命周期管理，我们采用了类似cocos的做法。即本次主循环会把shared_ptr放到一个缓存池（即引用计数+1），然后再在下一个主循环的时候清空。这样，在lua层创建的对象初始只有一个引用在缓存池里，如果创建出来以后没有添加到其他模块中，下一次主循环的时候即会销毁。如果被添加到了其他模块中，则回收工作就转移给了那个模块。

```lua
local ut = unit.new(unit_id) -- 这里这个对象引用计数为1，在缓存池里。如果没有缓存池，引用计数为0，就会被销毁
-- ut 只有一次弱引用，不会影响实际的对象回收
```


函数类型和函数参数的自动判定
------

Lua绑定C++函数的时候，有可能出现各种函数类型。为了减少代码，我们大量使用了C++11的特性（主要是function、lambda表达式、type_traits和动态模板参数）。利用C++的模板参数推导规则来自动分析参数。比如成员函数：

```cpp
/**
 * 给类添加方法，自动推断类型
 *
 * @tparam  TF  Type of the tf.
 * @param   func_name   Name of the function.
 * @param   fn          The function.
 */
template<typename R, typename... TParam>
self_type& addMethod(const char* func_name, R(*fn)(TParam... param)) {
    lua_State* state = getLuaState();
    lua_pushstring(state, func_name);
    lua_pushlightuserdata(state, reinterpret_cast<void*>(fn));
    lua_pushcclosure(state, detail::unwraper_static_fn<R, TParam...>::LuaCFunction, 1);
    lua_settable(state, getStaticClassTable());

    return (*this);
}
```

这里会把函数指针推入一个lua闭包，并设定**detail::unwraper_static_fn<R, TParam...>::LuaCFunction**为闭包函数。在这个函数里，使用了一些小技巧把Lua传入的参数按C++函数的参数次序导出转换并调用这个函数指针。

拿静态函数举个例子就是：
```cpp
/*************************************\
|* 静态函数Lua绑定，动态参数个数          *|
\*************************************/
template<typename Tr, typename... TParam>
struct unwraper_static_fn : public unwraper_static_fn_base<Tr> {
    typedef unwraper_static_fn_base<Tr> base_type;
    typedef Tr (*value_type)(TParam...);

     
    // 动态参数个数
    static int LuaCFunction(lua_State* L) {
        value_type fn = reinterpret_cast<value_type>(lua_touserdata(L, lua_upvalueindex(1)));
        if (nullptr == fn) {
            // 找不到函数
            return 0;
        }

        return base_type::template run_fn<value_type, std::tuple<
            typename std::remove_cv<typename std::remove_reference<TParam>::type>::type...
        > >(L,
            fn,
            typename build_args_index<TParam...>::index_seq_type()
        );
    }
};
```

在这里，把函数类型和顺序放到了**tuple<...>**里并移除了引用和常量标识，移除引用和常量标识的原因是我们在最终构建参数的时候会用一个右值，如果有这些标识会导致C++的类型检查不通过。（比如临时变量-也就是右值-不能转为左值引用）。并使用**build_args_index<TParam...>::index_seq_type()**来实现枚举常量数字，1,2...参数个数。也就这个*build_args_index*使用了一点奇技淫巧（type_traits）。

```cpp
template<int... _index>
struct index_seq_list{ typedef index_seq_list<_index..., sizeof...(_index)> next_type; };

template <typename... TP>
struct build_args_index;

template <>
struct build_args_index<>
{
    typedef index_seq_list<> index_seq_type;
};

template <typename TP1, typename... TP>
struct build_args_index<TP1, TP...>
{
    typedef typename build_args_index<TP...>::index_seq_type::next_type index_seq_type;
};
```

最后在**run_fn**里把参数解引用出来就完事啦。run_fn参考如下:

```cpp
template<typename Tfn, class TupleT, int... N >
static int run_fn(lua_State* L, Tfn fn, index_seq_list<N...>) {
    fn(
        unwraper_var<typename std::tuple_element<N, TupleT>::type>::unwraper(
            L, 
            N + 1
        )...
    );
    return 0;
}
```

再利用C++模板推导的规则定制不同的类型转换函数(unwraper_var::unwraper)并把指定index的数据导出来并返回作为函数的参数即可。

C++和Lua的数据类型转换
------

上面有提到*利用C++模板推导的规则定制不同的类型转换函数*，实际上我们除了有把数据从Lua导出来传给C++函数以外还有从把C++数据传给Lua，所以除了上面提到的unwraper_var::unwraper外还有wraper_var::wraper。

它们的实现原理都一样，就是利用C++的偏特化和模板类型匹配规则。而且我们除了对基本数据类型、数组和枚举类型做了适配以外，还对一些常用的STL库容器做了适配，比如std::string、std::array、std::vector、std::pair，拿vector举个例子：

```cpp
// C++类型转Lua table
template<typename Ty, typename... Tl>
struct wraper_var<std::vector<Ty>, Tl...> {
    static int wraper(lua_State* L, const std::vector<Ty>& v) {
        lua_Unsigned res = 1;
        lua_newtable(L);
        int tb = lua_gettop(L);
        for (const Ty& ele : v) {
            // 目前只支持一个值
            lua_pushunsigned(L, res);
            wraper_var<Ty>::wraper(L, ele);
            lua_settable(L, -3);

            ++res;
        }
        lua_settop(L, tb);
        return 1;
    }
};

// Lua数据转C++对象
template<typename Ty, typename... Tl>
struct unwraper_var<std::vector<Ty>, Tl...> {
    static std::vector<Ty> unwraper(lua_State* L, int index) {
        std::vector<Ty> ret;
        LUA_CHECK_TYPE_AND_RET(table, L, index, ret);

        LUA_GET_TABLE_LEN(size_t len, L, index);
        ret.reserve(len);

        lua_pushvalue(L, index);
        for (size_t i = 1; i <= len; ++i) {
            lua_pushinteger(L, static_cast<lua_Unsigned>(i));
            lua_gettable(L, -2);
            ret.push_back(unwraper_var<Ty>::unwraper(L, -1));
            lua_pop(L, 1);
        }
        lua_pop(L, 1);

        return ret;
    }
};
```

通过这种方式，我们可以很容易地使用stl作为函数参数，并实现绑定机制的自动类型判定。并且如果想自定义增加类型，只要写**unwraper_var和wraper_var对这个类型的偏特化**即可，并且可以和所有已经绑定的类型之间搭配或嵌套使用。（比如: *std::vector&lt;std::string&gt;*）


另外为了方便调用Lua函数，我们还写了一个自动分析参数并传入lua函数的接口，

```cpp
/**
 * @brief 自动打包调用lua函数
 * @return 参数个数，如果返回值小于0则不会改变参数个数
 */
template<typename... TParams>
int auto_call(lua_State* L, int index, TParams&&... params);
```

其实现原理和前面绑定类成员函数的一样，就是功能反过来而已。

自动激活
------

为了让代码更简单，我们使用了有点像gtest里的自动激活的设计。

**LUA_BIND_OBJECT**这个宏会定义一个函数和一个statis的全局变量，因为全局变量的启动规则是程序启动后，进入main函数前，为了保证一些前置数据有效，我们在这个全局变量的构造函数内把这个函数指针添加到Lua绑定的管理器（**LuaBindingMgr**）中，并在管理器初始化函数（*LuaBindingMgr::init*）的时候执行这些函数。以完成命名空间和类的绑定操作。

这样不同模块的开发者不需要写额外的代码，并且不需要去频繁改动上层的Lua绑定管理器。可以认为是一种依赖反转的做法。

后记
------

我们的Lua绑定机制核心的部分大致上就这么多，目前这个绑定机制并不完整，但是功能上已经能满足目前的所有需求，如果以后有强烈的需求的时候可以再加。功能缺失的部分包括暂不支持成员变量、自动分析重载的函数等。

上面绘制的图并不标准，因为stackeditor的markdown绘图功能有限。

本文中提及的结构及所有代码都在托管在 https://github.com/owent-utils/lua 有兴趣的童鞋可自取。

> Written with [StackEdit](https://stackedit.io/).

