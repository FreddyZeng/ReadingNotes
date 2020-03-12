# dealloc的实现

objc-709的源码的dealloc

## NSObject的dealloc

```objective-c
// Replaced by NSZombies
- (void)dealloc {
  _objc_rootDealloc(self);
}
```

```objective-c
void
_objc_rootDealloc(id obj)
{
    assert(obj);
    obj->rootDealloc(); // 调用NSObject实例的 rootDealloc()
}
```

```c++
调用objc runtime的  rootDealloc
inline void
objc_object::rootDealloc()
{
  
  if (fastpath(isa.nonpointer  &&  
                 !isa.weakly_referenced  &&  
                 !isa.has_assoc  &&  
                 !isa.has_cxx_dtor  &&  
                 !isa.has_sidetable_rc))
    {
        assert(!sidetable_present());
        free(this);
    } 
    else {
        object_dispose((id)this);
    }
}
```

## 我们先看看这5个东西，到底是怎么是什么，它们怎么设置值的
### nonpointer

  		判断self指针 与 _OBJC_TAG_MASK进行与运算，看看是否是 _OBJC_TAG_MASK。如果是，就不继续执行是否内存了，应该是debug工具使用的，比如调试测试某些情况下。
  		

```objective-c
((intptr_t)ptr & _OBJC_TAG_MASK) == _OBJC_TAG_MASK;
  		// These variables are always provided for debuggers.
			uintptr_t objc_debug_taggedpointer_mask = 0;
```

```objective-c
   if (isTaggedPointer()) return;  // fixme necessary?
```

   fastpath的表达式的值为1，就是true，而slowpath的表达式是0才是true。展开是 __builtin_expect，它是基于编译器对汇编进行优化条件分支的,并且提高准确，https://stackoverflow.com/questions/7346929/what-is-the-advantage-of-gccs-builtin-expect-in-if-else-statements  		

```objective-c
  		nonpointer
			objc_object::initIsa(Class cls) {
		    initIsa(cls, false, false);
		    OC实例对象和Class对象 isa.nonpointer == true，initIsa的时候是false。它的isa初始化的时候，会创建一个新的isa对象。从runtime-env.h的环境投文件中，也是可以看到可以配置的。可以修改这个行为,估计也是测试用的
			objc_object::initClassIsa(Class cls)
			{
			    if (DisableNonpointerIsa  ||  cls->instancesRequireRawIsa()) {
      			  initIsa(cls, false/*not nonpointer*/, false);
			...
  		
```

另外苹果引入了内存优化技术，64位机器的指针是8个字节，32位的指针是4个字节。TaggedPointer是使用前4个字节作为

```
在2013年9月，苹果推出了iPhone5s，与此同时，iPhone5s配备了首个采用64位架构的A7双核处理器，为了节省内存和提高执行效率，苹果提出了Tagged Pointer的概念。对于64位程序，引入Tagged Pointer后，相关逻辑能减少一半的内存占用，以及3倍的访问速度提升，100倍的创建、销毁速度提升。

https://halfrost.com/objc_runtime_isa_class/
```



###   		weakly_referenced

```
 		objc_object::setWeaklyReferenced_nolock()
      {
       ...
					if (slowpath(!newisa.nonpointer)) {
						ClearExclusive(&isa.bits);// 清除一些isa位信息，里面看了是汇编操作
						sidetable_setWeaklyReferenced_nolock();//对sidetable表修改弱引用属性 ，table.refcnts[this] |= SIDE_TABLE_WEAKLY_REFERENCED;
						return;
					}
					newisa.nonpointer = false
					if (newisa.weakly_referenced) {
						ClearExclusive(&isa.bits);
						return;
					}
					newisa.weakly_referenced = true// 可以看到是从这里标记弱引用的
					...
					}
```

					// sidetable_setWeaklyReferenced_nolock
	      objc_object::sidetable_setWeaklyReferenced_nolock()
	
	          SideTable& table = SideTables()[this];// 这里面是一个hash表，StripedMap.使用((addr >> 4) ^ (addr >> 9)) % StripeCount来获取this指针的值。
	
	          table.refcnts[this] |= SIDE_TABLE_WEAKLY_REFERENCED;// 修改弱引用
	      }
那么这个弱引用是在什么时候调用的呢？是当runtime设置变量的时候，根据内存管理调用存储的是什么类型

```
这个是runtime的函数
void object_setIvar(id obj, Ivar ivar, id value)
{
    return _object_setIvar(obj, ivar, value, false /*not strong default*/);
}
```

```
void _object_setIvar(id obj, Ivar ivar, id value, bool assumeStrong)
{
		...
    case objc_ivar_memoryWeak:       objc_storeWeak(location, value); break;// 储存弱引用
    case objc_ivar_memoryStrong:     objc_storeStrong(location, value); break;
    case objc_ivar_memoryUnretained: *location = value; break;
    case objc_ivar_memoryUnknown:    _objc_fatal("impossible");
    }

}
```

```
objc_storeWeak(id *location, id newObj)
{
    return storeWeak<DoHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object *)newObj);
}
```

```
storeWeak(id *location, objc_object *newObj)
{
	...
    // Assign new value, if any.
    if (haveNew) {
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected

        // Set is-weakly-referenced bit in refcount table.
        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced_nolock();// 只有新对象才会储存修改弱属性
        }
    ...
    return (id)newObj;
}
```



###  has_assoc

表示该对象是否有 C++ 或者 Objc 的析构器

has_assoc的赋值 跟 weakly_referenced逻辑大致一样，最上层函数，也是runtime函数

```
    objc_object::setHasAssociatedObjects()
{
 ...
  newisa.has_assoc = true;
  if (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)) goto retry;
}
```

  

```
void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy) {
    // retain the new value (if any) outside the lock.
    ObjcAssociation old_association(0, nil);
    id new_value = value ? acquireValue(value, policy) : nil;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        if (new_value) {
            // break any existing association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i != associations.end()) {
                // secondary table exists
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    j->second = ObjcAssociation(policy, new_value);
                } else {
                    (*refs)[key] = ObjcAssociation(policy, new_value);
                }
            } else {
                // create the new association (first time).
                ObjectAssociationMap *refs = new ObjectAssociationMap;
                associations[disguised_object] = refs;
                (*refs)[key] = ObjcAssociation(policy, new_value);
                object->setHasAssociatedObjects();
            }
        } 
}
```

### 	has_cxx_dtor 和has_sidetable_rc 赋值也是可以从代码中找到

has_cxx_dtor表示该对象是否有 C++ 或者 Objc 的析构器
has_sidetable_rc 是判断该对象的引用计数是否过大，如果过大则需要其他散列表来进行存储。

```
newisa.has_cxx_dtor = hasCxxDtor;
```

```
newisa.has_sidetable_rc = true;
```

## object_dispose

```
object_dispose(id obj)   {
  if (!obj) return nil;
  objc_destructInstance(obj);   // 
  free(obj);
  return nil; }
```

```
void *objc_destructInstance(id obj) {
  if (obj) {
​    // Read all of the flags at once for performance.
​    bool cxx = obj->hasCxxDtor();
​    bool assoc = obj->hasAssociatedObjects();
​    // This order is important.
​    if (cxx) object_cxxDestruct(obj);
​    if (assoc) _object_remove_assocations(obj);
​    obj->clearDeallocating();  
		}
  return obj;
}
```



#### object_cxxDestruct

```
object_cxxDestruct(obj);
```

它是会从当前实例对象的类开始向父类寻找SEL_cxx_destruct，如果有，传实例对象进去进行释放内存。如果找不到方法，它是会返回_objc_msgForward_impcache。代码如下

```
for ( ; cls; cls = cls->superclass) {
        if (!cls->hasCxxDtor()) return; 
        dtor = (void(*)(id))
            lookupMethodInClassAndLoadCache(cls, SEL_cxx_destruct);
        if (dtor != (void(*)(id))_objc_msgForward_impcache) {
            if (PrintCxxCtors) {
                _objc_inform("CXX: calling C++ destructors for class %s", 
                             cls->nameForLogging());
            }
            (*dtor)(obj);
        }
    }
```

#### _object_remove_assocations

```
void _object_remove_assocations(id object) {
    vector< ObjcAssociation,ObjcAllocator<ObjcAssociation> > elements;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        if (associations.size() == 0) return;
        disguised_ptr_t disguised_object = DISGUISE(object);// 这里面是对指针进行了取反，应该是为了保护object对象
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            // copy all of the associations that need to be removed.
            ObjectAssociationMap *refs = i->second;
            for (ObjectAssociationMap::iterator j = refs->begin(), end = refs->end(); j != end; ++j) {
                elements.push_back(j->second);
            }
            // remove the secondary table.
            delete refs;
            associations.erase(i);
        }
    }
    // the calls to releaseValue() happen outside of the lock.
    for_each(elements.begin(), elements.end(), ReleaseValue());
}
```

首先 ObjectAssociationMap的模板是

```
 template <class _Key, class _Tp, class _Compare = less<_Key>,

   class _Allocator = allocator<pair<const _Key, _Tp> > >
```

 AssociationsHashMap的模板是

```
template <class _Key, class _Tp, class _Hash = hash<_Key>, class _Pred = equal_to<_Key>,
          class _Alloc = allocator<pair<const _Key, _Tp> > >
```

通过AssociationsHashMap::iterator i = associations.find(disguised_object);，找到AssociationsHashMap

再通过AssociationsHashMap的value，利用 ObjectAssociationMap *refs = i->second; 找到 ObjectAssociationMap，

```
class ObjcAssociation {
        uintptr_t _policy;
        id _value;// 这个就是对象的地址值，后面会对着发送((id(*)(id, SEL))objc_msgSend)(value, SEL_release);
    public:
        ObjcAssociation(uintptr_t policy, id value) : _policy(policy), _value(value) {}
        ObjcAssociation() : _policy(0), _value(nil) {}

        uintptr_t policy() const { return _policy; }
        id value() const { return _value; }
        
        bool hasValue() { return _value != nil; }
    };
```

再把这个ObjectAssociationMap的所有值，保存在elements中，稍后删除

删除ObjectAssociationMap， delete refs;

删除相关的 AssociationsHashMap， associations.erase(i);

然后遍历删除，对value调用((id(*)(id, SEL))objc_msgSend)(value, SEL_release);

```
for_each(elements.begin(), elements.end(), ReleaseValue());
```

```
obj->clearDeallocating();// 把对象从内存管理的表中删除
```



objc_destructInstance 是不会free释放内存的，他是在object_dispose(id obj)里面释放内存

```
object_dispose(id obj)
{
    if (!obj) return nil;

    objc_destructInstance(obj);    
    free(obj);

    return nil;
}
```



阅读这个delloc的收获

1. 知道内存的释放有debug时不释放内存的nonpointer
2. 有weakly_referenced弱引用的设置逻辑
3. 是否拥有关联值has_assoc
4. dealloc的大致释放逻辑object_cxxDestruct是先从当前对象是否，遍历到父类（如果找到析构函数的话），从_object_remove_assocations知道，释放关联，需要从他的两个表中寻找，并且调动Release。最后就是从sidetable表中，清楚实例对象自己。

### 不足：

runtime有很多汇编的代码，我只是了解r0作为返回值，r1-r4是参数。而且asm 是使用r12进行传值进去的，而且会r8的值（这个是在看别人hook objc_msgSend看到的（ARM的汇编对可变函数是r寄存器+栈混合使用的，当结构之大于16字节，就会使用栈），并且自己使用asm发现，它确实会改，r12会传值给r8）。

顺便也了解到，fish hook的原理。就是对符号地址的替换。

如果上面的总结有错误，请指教，谢谢

参考 https://halfrost.com/objc_runtime_isa_class/