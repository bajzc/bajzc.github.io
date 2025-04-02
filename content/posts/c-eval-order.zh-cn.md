---
title: C语言参数求值顺序
date: 2025-03-02
draft: false
tags:
    - Debug
    - C
categories:
    - Review
---

### TL;DR


> *"The order of evaluation of the function designator, the actual arguments,
and subexpressions within the actual arguments is unspecified, but there is a
sequence point before the actual call."*
[C99-6.5.2.2.10](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1256.pdf#%5B%7B%22num%22%3A164%2C%22gen%22%3A0%7D%2C%7B%22name%22%3A%22XYZ%22%7D%2C-27%2C816%2Cnull%5D):

所有依赖“求值顺序”的函数调用都是**“未定义行为”**。

# 案例 [See Commit](https://github.com/bajzc/tiger-in-c/commit/d2530a50ef6e0fa2eb50c4417f1421bd7cab7362)

故事开始于我想在一个老Thinkpad上跑一下之前写的[Tiger编译器](https://github.com/bajzc/tiger-in-c)。
目的就是在不同的CPU架构上发现一些BUG，详见某次debug[经历](http://localhost:1313/zh-cn/posts/2296229/#lec-2-syscall)。然后不出意外的出意外了。

```C {lineNoStart=63 hl_Lines=2}
static Temp_temp nthTemp(Temp_tempList list, int i) {
  assert(list);
  if (i == 0)
    return list->head;
  else
    return nthTemp(list->tail, i - 1);
} // https://github.com/bajzc/tiger-in-c/blob/d2530a50ef6e0fa2eb50c4417f1421bd7cab7362/assem.c#L63
```
`list`是`NULL`

这个函数的调用者是
```C {hl_Lines=13}
typedef struct Temp_tempList_ {
  Temp_temp head;
  Temp_tempList tail;
} *Temp_tempList;

static void format(char *result, char *assem, Temp_tempList dst,
                   Temp_tempList src, AS_targets jumps, Temp_map m) {
    ...
    } else if (*p == '`')
      switch (*(++p)) {
        case 's': {
          int n = ATOI(++p);
          string s = Temp_look(m, nthTemp(src, n));
          STRCPY(result + i, s);
          i += STRLEN(s);
        } break;
    ...
// https://github.com/bajzc/tiger-in-c/blob/d2530a50ef6e0fa2eb50c4417f1421bd7cab7362/assem.c#L102
```
`format`函数的作用和`printf`类似，将输入字符串`assem`中的指示符 (`d `, `s `)替换成`m`表里的寄存器名字。

比如对于``add `d0, `s0, `s1``, `format`会返回`add fp, fp, a0`

问题出在当输入是``add `d0, `s0, `s1``的时候，`src={some_ptr, NULL}`

被这个表象误导了，认为可能是在某个地方（比如coloring的时候）,不小心修改了`src`的内容，
我花了大概两个小时反复追踪`src`和`assem`的构造和修改的代码。

最后神奇地发现在codegen的地方，也就是构造`assem`的函数里
```C {hl_Lines=22}
#define L(h, t) Temp_TempList((Temp_temp) h, (Temp_tempList) t) // Constructor
#define S(format, ...) sprintf(buf, format, ##__VA_ARGS__)
static Temp_temp munchExp(T_exp e) {
  static char buf[80];
  Temp_temp r = Temp_newtemp();
  switch (e->kind) {
    case T_BINOP: {
      ...
      } else if ((e1->kind == T_CONST || e2->kind == T_CONST) &&
                 (op == T_plus || op == T_minus)) {
        /* BINOP(PLUS,CONST(i),e2) */
        /* BINOP(PLUS,e1,CONST(i)) */
        /* BINOP(MINUS,CONST(i),e2) */
        /* BINOP(MINUS,e1,CONST(i)) */
        T_exp var = e1->kind == T_CONST ? e2 : e1;
        int constt = e1->kind == T_CONST ? e1->u.CONST : e2->u.CONST;
        switch (op) {
          case T_plus: S("addi `d0, `s0, %d", constt); break;
          case T_minus: S("addi `d0, `s0, -%d", constt); break;
          default: assert(0);
        }
        emit(AS_Oper(STRDUP(buf), L(r, NULL), L(munchExp(var), NULL), NULL));
        return r;
      }
      ...
// https://github.com/bajzc/tiger-in-c/blob/02c247d4336ddc80234900628d5e9a1038a06e08/riscv_codegen.c#L228
```
当我**break**（很快就要考）到`AS_Oper()`里的时候，字符串竟然是以``add `d0...``开头的！

给我直接整的怀疑人生了，难道`sprintf()`的实现和我想的不一样，不是直接覆盖？难道是`STRDUP()`的实现有bug？

直到我想**step**（伏笔收回）进`AS_Oper()`，看看会不会是`L()`宏展开出了问题的时候：
我发现并没有进到`strdup()`里，而是开始调用`munchExp()`！

也就是说编译器并没有像“我想的”一样从左往右，从里到外的求值，而是倒着来的。
好巧不巧，调用的正好是自己，而且在`buf`上的副作用也因为它是静态的被体现出来。
这就导致了传给`AS_Oper()`的汇编和其他信息不匹配。

## 问题
如果你觉得上面的问题细节太多，可以看看下面这个抽象出来的问题：
```C
#include <stdio.h>
int eval2(int a, int b){
	printf("eval2 %d %d\n", a, b);
	return a + b + 1;
}
int eval(int a){
	printf("eval %d\n", a);
	return a + 1;
}
int main(){
	eval(eval2(eval(1), eval(eval(2))));
}
```

在Mac上Clang和GCC的运行结果一样：
```
eval 1
eval 2
eval 3
eval2 2 4
eval 7
```

也就是从左往右求值，但是在Thinkpad(gcc-x86-64-linux-gnu)上的结果是从右往左：
```
eval 2
eval 1
eval 3
eval2 2 4
eval 7
```

## 标准
C标准对求值顺序的规定是：**没有规定**

> *"The order of evaluation of the function designator, the actual arguments,
and subexpressions within the actual arguments is unspecified, but there is a
sequence point before the actual call."*
[C99-6.5.2.2.10](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1256.pdf#%5B%7B%22num%22%3A164%2C%22gen%22%3A0%7D%2C%7B%22name%22%3A%22XYZ%22%7D%2C-27%2C816%2Cnull%5D):

既然如此，那为什么同样是GCC，在不同的架构上求值的顺序会不一样呢？

在[godbolt](https://godbolt.org/z/P4Yc9Mrs9)上探索汇编一下发现，Clang在两种架构上表现相同，而GCC则不同。

## 实现

为了知其所以然，直接上GCC代码！
注意到求值顺序的不同在[Gimple](https://godbolt.org/z/865vf5daY)(GCC的IR)中就已经体现了出来
那么我们就去看从AST转到Gimple的[代码](https://github.com/gcc-mirror/gcc/blob/634215cdc3c569f9a9a247dcd4d9a4d6ce68ad57/gcc/gimplify.cc#L4805C1-L4831C6)：

```cpp{lineNostart=4805 hl_lines=[4,5,6]}
  // Gimplify the function arguments.  */
  if (nargs > 0)
    {
      for (i = (PUSH_ARGS_REVERSED ? nargs - 1 : 0);
	   PUSH_ARGS_REVERSED ? i >= 0 : i < nargs;
	   PUSH_ARGS_REVERSED ? i-- : i++)
	{
	  enum gimplify_status t;

	  /* Avoid gimplifying the second argument to va_start, which needs to
	     be the plain PARM_DECL.  */
	  if ((i != 1) || !builtin_va_start_p)
	    {
	      tree *arg_p = &CALL_EXPR_ARG (*expr_p, i);

	      if (gimplify_omp_ctxp && gimplify_omp_ctxp->code == OMP_DISPATCH)
		gimplify_omp_ctxp->in_call_args = true;
	      t = gimplify_arg (arg_p, pre_p, EXPR_LOCATION (*expr_p),
				!returns_twice);
	      if (gimplify_omp_ctxp && gimplify_omp_ctxp->code == OMP_DISPATCH)
		gimplify_omp_ctxp->in_call_args = false;

	      if t == GS_ERROR)
		ret = GS_ERROR;
	    }
	}
    ...
```
可以看到求值的顺序是正是反取决于`PUSH_ARGS_REVERSED`，它在每个架构的配置文件中被定义：
比如[i386](https://github.com/gcc-mirror/gcc/blob/574c59cfe6e29c9e5758988b75c2e7ab6edc37da/gcc/config/i386/i386.h#L1619)就是1，
而其他架构采用的就是[默认值](https://github.com/gcc-mirror/gcc/blob/e7912d4a81cf34e05c7ded70910069b691a8bb15/gcc/defaults.h#L808):0

让我们再进一步，为什么对于不同的架构需要指定不同的顺序呢？
参考2014年的[讨论](https://gcc.gnu.org/pipermail/gcc-patches/2014-March/384822.html)，
对于ARM来说，正序的压栈可以生成空间占用更小的二进制。

至于为什么会更小，就不知道了...

## 思考
很多时候自以为正确的代码是UB，这就是C语言！
比如
```C
// copy array
i = 0;
while(i < n)
    y[i] = x[i++];
```
代码假设`y[i]`先被求值，但这其实是没有保证的！

类似的错误数不胜数，编译器只用编译就行了，而程序员要考虑的就多了（我写了UB难道编译器就没有责任了吗？
