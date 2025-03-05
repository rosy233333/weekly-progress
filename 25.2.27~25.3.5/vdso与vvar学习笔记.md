# vdso与vvar学习笔记

时间：2025/3/1

## vdso初始化过程

[参考链接](https://tinylab.org/riscv-syscall-part4-vdso-implementation/)

描述整个vdso的对象`vdso_info`：

```C
struct __vdso_info {
    const char *name;
    const char *vdso_code_start;  // vdso 代码起始地址
    const char *vdso_code_end;    // vdso 代码结束地址
    unsigned long vdso_pages;     // vdso 代码部分所占内存页数
    /* Data Mapping */
    struct vm_special_mapping *dm;
    /* Code Mapping */
    struct vm_special_mapping *cm;
};

extern char vdso_start[], vdso_end[];

static struct __vdso_info vdso_info __ro_after_init = {
    .name = "vdso",
    .vdso_code_start = vdso_start,
    .vdso_code_end = vdso_end,
};
```

描述vvar区域的对象`vdso_data`：

```C
static union {
    struct vdso_data    data;
    u8            page[PAGE_SIZE];
} vdso_data_store __page_aligned_data;

struct vdso_data *vdso_data = &vdso_data_store.data;
```

两个`vm_special_mapping`的初始化（关于`vm_special_mapping`可参考[此链接](https://blog.csdn.net/xiaozhiwise/article/details/144261920)）：

```C
static struct vm_special_mapping rv_vdso_maps[] __ro_after_init = {
    [RV_VDSO_MAP_VVAR] = {
        .name   = "[vvar]",
        .fault = vvar_fault,
    },
    [RV_VDSO_MAP_VDSO] = {
        .name   = "[vdso]",
        .mremap = vdso_mremap,
    },
};

static int __init vdso_init(void)
{
    vdso_info.dm = &rv_vdso_maps[RV_VDSO_MAP_VVAR];
    vdso_info.cm = &rv_vdso_maps[RV_VDSO_MAP_VDSO];
    return __vdso_init();
}

static int __init __vdso_init(void)
{
    unsigned int i;
    struct page **vdso_pagelist;

    // ...
    
    vdso_pagelist = kcalloc(vdso_info.vdso_pages,
                sizeof(struct page *),
                GFP_KERNEL);
    if (vdso_pagelist == NULL)
        return -ENOMEM;
    /* Grab the vDSO code pages. */
    pfn = sym_to_pfn(vdso_info.vdso_code_start);
    for (i = 0; i < vdso_info.vdso_pages; i++)
        vdso_pagelist[i] = pfn_to_page(pfn + i);
    vdso_info.cm->pages = vdso_pagelist;
    return 0;
}
```

注意：对于`vdso_info.cm`的初始化，不只设置了`name`和`mremap`，还在`__vdso_init`函数中设置了`pages`字段为内核中的vdso代码区域（即`vdso_start` ~ `vdso_end`所在的内存页）。

而`vdso_info.dm`（vvar区域）只设置了`name`和`fault`，其在`fault`函数中映射`vdso_data`中的数据。

```C
static vm_fault_t vvar_fault(const struct vm_special_mapping *sm,
                 struct vm_area_struct *vma, struct vm_fault *vmf)
{
  // ...
  pfn = sym_to_pfn(vdso_data);
  // ...
}
```

这部分调研中，我得知了vvar在内核中的存储位置（`vdso_data`），但仍然不知道其内容和vdso访问vvar的方法。

## 进一步调研

在Linux源码中，发现其对`vdso_data`结构体类型做了定义：

```C
struct vdso_data {
	u32			seq;

	s32			clock_mode;
	u64			cycle_last;
#ifdef CONFIG_GENERIC_VDSO_OVERFLOW_PROTECT
	u64			max_cycles;
#endif
	u64			mask;
	u32			mult;
	u32			shift;

	union {
		struct vdso_timestamp	basetime[VDSO_BASES];
		struct timens_offset	offset[VDSO_BASES];
	};

	s32			tz_minuteswest;
	s32			tz_dsttime;
	u32			hrtimer_res;
	u32			__unused;

	struct arch_vdso_time_data arch_data;
};
```

因此，根据前文`vdso_data_store`联合类型的定义，对象`vdso_data`可以用于以`vdso_data`类型访问其中的各种字段，而其在内存上的存储布局由`vdso_data_store`的`page`描述。

在vdso相关函数调用过程中，最终调用了此函数获取数据（具体的调用链为：`__vdso_clock_gettime`（vdso函数） -> `__cvdso_clock_gettime` -> `__arch_get_vdso_data`）：

```C
static __always_inline const struct vdso_data *__arch_get_vdso_data(void)
{
	return _vdso_data;
}
```

而其中的`_vdso_data`符号，定义在`vdso.lds.S`中：

```C
SECTIONS
{
	PROVIDE(_vdso_data = . - __VVAR_PAGES * PAGE_SIZE);
    // ...
}
```

因此，vdso代码通过访问代码段之前一页或数页的位置获得数据段的起始地址，并将其转换为`vdso_data`类型，从而访问数据段中的数据。
