# 还原被混淆的函数

> Demo



## 缘由

有一天做逆向看到如下代码:

```javascript
return function(e, t, n, s) {
    if (!o(t)) {
        var l = !1
        , d = [];
        if (o(e))
            l = !0,
                h(t, d);
        else {
            var c = r(e.nodeType);
            if (!c && tn(e, t))
                C(e, t, d, null, null, s);
            else {
                if (c) {
                    if (1 === e.nodeType && e.hasAttribute(R) && (e.removeAttribute(R),
                                                                  n = !0),
                        !0 === n && k(e, t, d))
                        return x(t, d, !0),
                            e;
                    u = e,
                        e = new ge(a.tagName(u).toLowerCase(),{},[],void 0,u)
                }
                n = e.elm;
                var u = a.parentNode(n);
                if (h(t, d, n._leaveCb ? null : u, a.nextSibling(n)),
                    r(t.parent))
                    for (var g = t.parent, p = m(t); g; ) {
                        for (var f = 0; f < i.destroy.length; ++f)
                            i.destroy[f](g);
                        if (g.elm = t.elm,
                            p) {
                            for (var v = 0; v < i.create.length; ++v)
                                i.create[v](Ji, g);
                            var b = g.data.hook.insert;
                            if (b.merged)
                                for (var _ = 1; _ < b.fns.length; _++)
                                    b.fns[_]()
                        } else
                            Qi(g);
                        g = g.parent
                    }
                r(u) ? y([e], 0, 0) : r(e.tag) && w(e)
            }
        }
        return x(t, d, l),
            t.elm
    }
    r(e) && w(e)
}
```

通过 github搜索 .data.hook.insert  关键字, 找到如下:

https://github.com/superfreeeee/Blog-code/blob/c226c083c948feb498af0c188eafccaf70f620aa/source_code_research/vue-2.6.12/src/core/vdom/patch/createPatchFunction.flat2.patch.js

https://github.com/vuejs/vue/blob/612fb89547711cacb030a3893a0065b785802860/dist/vue.runtime.common.dev.js

```javascript
return function patch (oldVnode, vnode, hydrating, removeOnly) {
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) { invokeDestroyHook(oldVnode); }
      return
    }

    var isInitialPatch = false;
    var insertedVnodeQueue = [];

    if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      isInitialPatch = true;
      createElm(vnode, insertedVnodeQueue);
    } else {
      var isRealElement = isDef(oldVnode.nodeType);
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly);
      } else {
        if (isRealElement) {
          // mounting to a real element
          // check if this is server-rendered content and if we can perform
          // a successful hydration.
          if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
            oldVnode.removeAttribute(SSR_ATTR);
            hydrating = true;
          }
          if (isTrue(hydrating)) {
            if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
              invokeInsertHook(vnode, insertedVnodeQueue, true);
              return oldVnode
            } else {
              warn(
                'The client-side rendered virtual DOM tree is not matching ' +
                'server-rendered content. This is likely caused by incorrect ' +
                'HTML markup, for example nesting block-level elements inside ' +
                '<p>, or missing <tbody>. Bailing hydration and performing ' +
                'full client-side render.'
              );
            }
          }
          // either not server-rendered, or hydration failed.
          // create an empty node and replace it
          oldVnode = emptyNodeAt(oldVnode);
        }

        // replacing existing element
        var oldElm = oldVnode.elm;
        var parentElm = nodeOps.parentNode(oldElm);

        // create new node
        createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        );

        // update parent placeholder node element, recursively
        if (isDef(vnode.parent)) {
          var ancestor = vnode.parent;
          var patchable = isPatchable(vnode);
          while (ancestor) {
            for (var i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor);
            }
            ancestor.elm = vnode.elm;
            if (patchable) {
              for (var i$1 = 0; i$1 < cbs.create.length; ++i$1) {
                cbs.create[i$1](emptyNode, ancestor);
              }
              // #6513
              // invoke insert hooks that may have been merged by create hooks.
              // e.g. for directives that uses the "inserted" hook.
              var insert = ancestor.data.hook.insert;
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (var i$2 = 1; i$2 < insert.fns.length; i$2++) {
                  insert.fns[i$2]();
                }
              }
            } else {
              registerRef(ancestor);
            }
            ancestor = ancestor.parent;
          }
        }

        // destroy old node
        if (isDef(parentElm)) {
          removeVnodes([oldVnode], 0, 0);
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode);
        }
      }
    }
```

这儿我们很快的就能知道项目使用Vue函数，并理解函数功能，方便快速还原堆栈上下文的函数，理解函数功能。



## 实现

和很多逆向工程的混淆破解一样，本项目实现一套机制提取公共函数的特征

对于需要分析的混淆函数，可以抽取特征，进行快速比对。



支持的语言：

* Javascript
* C
* C++



抽取特征:

1. 函数命名:  func_$$
2. 入参: inparam_list -> []
3. 变量命令:  var_$$
4. 常量，外部变量: constvar_$$
5. 固定函数/常量函数: constfunc_list -> []

描述特征:

1. 将函数做成一行，通过\n, \x20 等替换换行，空格
2. 替换原有函数，参等，为特征描述符； 
3. 存储， 比对相似性



```
one_feature = {
	"function_rawname": "",
	"function_oneline": "",
	"inparam_list": [],
	"constfunc_list": [],
	"len_var_": "",
	"len_constvar_": "",
}
```



## 模块

特征模块

入料区

使用区



