tolua++内存释放坑
============

[TOC]

前言
------
本来想参考下tolua++的对象生命周期维护方式。一不小心发现了一个坑。

代码追踪
------
我这里用得是tolua++ 1.0.93版本。

tolua++在new一个类的时候，会把类指针作为userdata传入lua，建立metatable并通过**tolua_classevents**函数给metatable注册魔术方法。

这里面可以看到gc方法被设成了 _G.tolua_gc_event
```cpp
    lua_pushstring(L,"__gc");
    lua_pushstring(L, "tolua_gc_event");
    lua_rawget(L, LUA_REGISTRYINDEX);
    /*lua_pushcfunction(L,class_gc_event);*/
    lua_rawset(L,-3);
```

那么关键是这个**_G.tolua_gc_event**被设成了什么，在**tolua_open**函数里可以找到
```cpp
        /* create gc_event closure */
        lua_pushstring(L, "tolua_gc_event");
        lua_pushstring(L, "tolua_gc");
        lua_rawget(L, LUA_REGISTRYINDEX);
        lua_pushstring(L, "tolua_super");
        lua_rawget(L, LUA_REGISTRYINDEX);
        lua_pushcclosure(L, class_gc_event, 2);
        lua_rawset(L, LUA_REGISTRYINDEX);
```

垃圾回收函数使用的是**class_gc_event**并且添加了两个闭包参数**_G.tolua_gc,_G.tolua_super**。其中前一个用于设置全局metatable缓存。后一个貌似是基类。

比如新建一个Class, 指针是p，lua对象是t。那么相当于在lua里会设置
```lua
_G.tolua_gc[p] = getmetatable(t)
```

具体见**tolua_register_gc**函数

```cpp
TOLUA_API int class_gc_event (lua_State* L)
{
    void* u = *((void**)lua_touserdata(L,1));
    int top;
    /*fprintf(stderr, "collecting: looking at %p\n", u);*/
    /*
    lua_pushstring(L,"tolua_gc");
    lua_rawget(L,LUA_REGISTRYINDEX); 
    */
    lua_pushvalue(L, lua_upvalueindex(1)); // 这里是拿到 _G.tolua_gc
    lua_pushlightuserdata(L,u);
    lua_rawget(L,-2);            /* stack: gc umt    */ // 这里是拿到上文中提到的 _G.tolua_gc[p]
    lua_getmetatable(L,1);       /* stack: gc umt mt */ 
    /*fprintf(stderr, "checking type\n");*/
    top = lua_gettop(L);
    if (tolua_fast_isa(L,top,top-1, lua_upvalueindex(2))) /* make sure we collect correct type */ // 这个是类型检查
    {
        /*fprintf(stderr, "Found type!\n");*/
        /* get gc function */
        lua_pushliteral(L,".collector");
        lua_rawget(L,-2);           /* stack: gc umt mt collector */
        if (lua_isfunction(L,-1)) { // .collector有数据时调用.collector函数
            /*fprintf(stderr, "Found .collector!\n");*/
        }
        else { // .collector没有数据时调用tolua_default_collect函数
            lua_pop(L,1);
            /*fprintf(stderr, "Using default cleanup\n");*/
            lua_pushcfunction(L,tolua_default_collect);
        }

        lua_pushvalue(L,1);         /* stack: gc umt mt collector u */
        lua_call(L,1,0); // 这里开始调用tolua_default_collect函数

        lua_pushlightuserdata(L,u); /* stack: gc umt mt u */
        lua_pushnil(L);             /* stack: gc umt mt u nil */
        lua_rawset(L,-5);           /* stack: gc umt mt */
    }
    lua_pop(L,3);
    return 0;
}
```

然后坑爹的就来了**tolua_default_collect**这么实现的

```cpp
/* Default collect function
*/
TOLUA_API int tolua_default_collect (lua_State* tolua_S)
{
    void* self = tolua_tousertype(tolua_S,1,0);
    free(self);
    return 0;
}
```

这个很2B的坑
------

这么搞问题就来了，默认tolua++是没有设置**.collector**函数的（new一个自定义class之后调用**push_collector**的传入了空指针），然后释放的时候就华丽丽free掉了，且不说标准C++并不保证new的分配内存和malloc一样（虽然现在大部分编译器的实现确实一样），它竟然**没调用析构函数**。这意味着如果类里面有使用stl或者其他依赖析构来释放资源的成员类对象的话，就华丽丽地内存泄露了。

另外，网上随便搜了一下，也找到其他人也有发现这个问题。http://www.cnblogs.com/egmkang/archive/2012/07/01/2572064.html

无力吐槽了，让我晕一会...


> Written with [StackEdit](https://stackedit.io/).

