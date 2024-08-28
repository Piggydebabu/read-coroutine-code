# 云风coroutine源码分析

源码地址：https://github.com/cloudwu/coroutine.git

## 主要数据结构

`schedule`：记录协程公共数据的结构体

```c
struct schedule {
    // 协程公共栈大小 1024K
	char stack[STACK_SIZE];
    // 主上下文
	ucontext_t main;
    // 已使用的个数
	int nco;
    // 最大容量
	int cap;
    // 记录当前正在执行的协程
	int running;
    // 协程信息, 保存了一系列协程
	struct coroutine **co
}
```

`coroutine`: 记录协程数据的结构体

```c
struct coroutine {
    // 协程包裹的函数
    coroutine_func func;
    // 用户数据, 执行func时传入
    void *ud;
    // 保存执行前的上下文
    ucontext_t ctx;
    // 所属的schedule
    struct schedule *sch;
    // 当前栈的最大容量
    ptrdiff_t cap;
    // 当前栈的大小(已使用的大小)
    ptrdiff_t size;
    // 协程状态
    // 有四种: 
    // #define COROUTINE_DEAD 0
	// #define COROUTINE_READY 1
	// #define COROUTINE_RUNNING 2
	// #define COROUTINE_SUSPEND 3
    int status;
    // 协程的栈顶指针, 协程私有栈
    char *stack;
}
```

## 主要接口

coroutine_open: 创建调度器, 准备开始执行协程

```c
struct schedule * 
coroutine_open(void) {
	struct schedule *S = malloc(sizeof(*S));
    // 协程个数
	S->nco = 0;
    // 最大协程容量, 默认16
	S->cap = DEFAULT_COROUTINE;
    // 初始化时没有协程在运行
	S->running = -1;
    // 分配了一个结构体指针数组, 指向协程结构体的信息
	S->co = malloc(sizeof(struct coroutine *) * S->cap);
	memset(S->co, 0, sizeof(struct coroutine *) * S->cap);
	return S;
}
```

其实就是申请了公共栈大小的空间，里面放了协程调度器和协程集合

coroutine_new: 创建新协程并加入到调度器中, 返回协程id

```c
int 
coroutine_new(struct schedule *S, coroutine_func func, void *ud) {
    // 调用内部函数创建coroutine结构体
	struct coroutine *co = _co_new(S, func , ud);
    // 协程数达到上限则扩容
	if (S->nco >= S->cap) {
		int id = S->cap;
        // 扩容为原来2倍
		S->co = realloc(S->co, S->cap * 2 * sizeof(struct coroutine *));
		memset(S->co + S->cap , 0 , sizeof(struct coroutine *) * S->cap);
		// 将新创建的协程添加到数组的末尾
        S->co[S->cap] = co;
		S->cap *= 2;
		++S->nco;
		return id;
	} else {
		int i;
        // 循环遍历协程数组，寻找空位
		for (i=0;i<S->cap;i++) {
            // 循环索引, 确保索引在数组范围内
			int id = (i+S->nco) % S->cap;
            // 如果找到空位，将新创建的协程添加到该位置，增加协程数量，并返回索引
			if (S->co[id] == NULL) {
				S->co[id] = co;
				++S->nco;
				return id;
			}
		}
	}
	assert(0);
	return -1;
}
// 填充coroutine结构体, 设置协程状态为ready
struct coroutine * 
_co_new(struct schedule *S , coroutine_func func, void *ud) {
	struct coroutine * co = malloc(sizeof(*co));
	co->func = func;
	co->ud = ud;
	co->sch = S;
	co->cap = 0;
	co->size = 0;
	co->status = COROUTINE_READY;
	co->stack = NULL;
	return co;
}
```

协程的切换

```c
void
coroutine_yield(struct schedule * S) {
	int id = S->running;
	assert(id >= 0);
	struct coroutine * C = S->co[id];
	assert((char *)&C > S->stack);
    // 保存当前栈的信息
	_save_stack(C,S->stack + STACK_SIZE);
    // 当前协程被挂起
	C->status = COROUTINE_SUSPEND;
    // 没有协程正在执行
	S->running = -1;
    // 切换到main执行
	swapcontext(&C->ctx , &S->main);
}
// 协程执行的时候使用的是一个公共的栈，即所有协程共享的。
// 那么如果协程让出执行权后，其他协程执行时就会覆盖栈里的信息。
// 所以让出执行权之前，协程有个很重要的事情要做，那就是保存当前栈的信息，以便再次执行时，能恢复栈的上下文
// 保存当前栈信息，top是协程的栈顶最大值
static void
_save_stack(struct coroutine *C, char *top) {
    // dummy用于计算出当前的esp，即栈顶地址
    char dummy = 0;
    assert(top - &dummy <= STACK_SIZE);
    // top-&dummy算出协程当前的栈上下文有多大，如果比当前的容量大，则需要扩容
    if (C->cap < top - &dummy) {
        free(C->stack);
        C->cap = top-&dummy;
        C->stack = malloc(C->cap);
    }
    // 记录当前的栈大小
    C->size = top - &dummy;
    // 复制公共栈的数据到私有栈
    memcpy(C->stack, &dummy, C->size);
}
```

```c
// 协程的恢复
void 
coroutine_resume(struct schedule * S, int id) {
	assert(S->running == -1);
	assert(id >=0 && id < S->cap);
    // 通过id获取协程
	struct coroutine *C = S->co[id];
	if (C == NULL)
		return;
	int status = C->status;
	switch(status) {
	case COROUTINE_READY:
        // 获取当前上下文存入协程上下文
		getcontext(&C->ctx);
        // 设置协程上下文的栈参数
		C->ctx.uc_stack.ss_sp = S->stack;
		C->ctx.uc_stack.ss_size = STACK_SIZE;
		// 设置协程上下文的链接指向主上下文
        C->ctx.uc_link = &S->main;
		S->running = id;
		C->status = COROUTINE_RUNNING;
		uintptr_t ptr = (uintptr_t)S;
        // 设置协程的上下文，使其在恢复时执行 mainfunc 函数，并传递调度器的指针 ptr 作为参数
		makecontext(&C->ctx, (void (*)(void)) mainfunc, 2, (uint32_t)ptr, (uint32_t)(ptr>>32));
        // 从主上下文切换到协程上下文
		swapcontext(&S->main, &C->ctx);
		break;
	case COROUTINE_SUSPEND:
        // 被yield的协程恢复执行时, 
        // 把私有栈C->stack开始, 大小为C->size的字节复制到公共栈中, 达到恢复上下文的目的
		memcpy(S->stack + STACK_SIZE - C->size, C->stack, C->size);
		S->running = id;
		C->status = COROUTINE_RUNNING;
		swapcontext(&S->main, &C->ctx);
		break;
	default:
		assert(0);
	}
}
```

