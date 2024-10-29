# fprintf和vfprintf

### fprintf

简单来说，`vfprint`是`fprintf`的底层实现，比如这样(以下是一个简化版）：

```c
#include <stdio.h>
#include <stdarg.h>

int my_fprintf(FILE *stream, const char *format, ...) 
{
    va_list args;
    
    // 初始化 va_list，获取变长参数
    va_start(args, format);

    // 调用 vfprintf 进行格式化输出
    int result = vfprintf(stream, format, args);
	
    // 清理 va_list
    va_end(args);
    
    // 返回写入字符的数量
    return result;
}

int main() 
{
    my_fprintf(stdout, "Hello, %s! You scored %d points.\n", "Diana", 100);
    return 0;
}
```

具体可以参考[fprintf 代码](https://github.com/bminor/glibc/blob/master/stdio-common/fprintf.c)

所以`fprintf` 实际上是一个“包装器”，它调用了 `vfprintf`，并传递给它一个已知的参数列表。`vfprintf` 才是底层真正处理变长参数、格式化输出的核心函数。

`fprintf`是一个“更方便使用”的`vfprintf`。

那么什么是`vfprintf`呢？

---

### vfprintf

在真正的 `glibc` 实现中，`vfprintf` 是核心函数。 `vfprintf` 会包含更多的错误处理、缓冲管理和优化，我们先来看一个简单的`vfprintf`实现吧：

```c
#include <stdarg.h>
#include <stdio.h>
#include <string.h>

int my_vfprintf(FILE *stream, const char *format, va_list args) 
{
    int total_chars = 0; // 记录输出的总字符数

    for (const char *p = format; *p != '\0'; p++) 
    {
        if (*p == '%' && *(p + 1)) 
        { 
            // 检测格式符
            p++; 
            // 跳过 '%'
            if (*p == 'd') 
            { 
                // 处理整数
                int i = va_arg(args, int);
                char buffer[20];
                snprintf(buffer, sizeof(buffer), "%d", i); // 将整数转换为字符串
                fputs(buffer, stream); // 输出到文件流
                total_chars += strlen(buffer);
            } 
            else if (*p == 'c') 
            { 
                // 处理字符
                char c = (char)va_arg(args, int);
                fputc(c, stream);
                total_chars++;
            } 
            else if (*p == 's') 
            { 
                // 处理字符串
                const char *s = va_arg(args, const char *);
                fputs(s, stream);
                total_chars += strlen(s);
            } 
            else 
            { 
                // 处理未知格式符
                fputc('%', stream);
                fputc(*p, stream);
                total_chars += 2;
            }
        } 
        else 
        { 
            // 非格式化字符直接输出
            fputc(*p, stream);
            total_chars++;
        }
    }

    return total_chars; // 返回总输出字符数
}

// 简单测试函数
int my_printf(const char *format, ...) 
{
    va_list args;
    va_start(args, format);
    int result = my_vfprintf(stdout, format, args);
    va_end(args);
    return result;
}

int main() 
{
    my_printf("Hello %s, your score is %d%%!\n", "Diana", 95);
    return 0;
}
```

这个代码简化了`vfprintf`的工作流程。在`my_printf(const char *format, ...)`调用了我们的`my_vfprintf`：

`va_list`、`va_start`、`va_end` 这些宏都是由 C 标准库（libc）提供的，它们在头文件 `<stdarg.h>` 中定义，专门用于处理变长参数：

1. **`va_list`**：这是一个类型，用于定义变长参数列表变量。可以把它看作是一个“容器”，用来存放传递给函数的变参。

   ```c
   va_list args;  // 定义变参列表
   ```

2. **`va_start`**：初始化 `va_list` 变量，并将其指向第一个变长参数。它的第一个参数是 `va_list` 变量，第二个参数是可变参数列表之前的最后一个已知参数。

   ```
   va_start(args, format);  // 初始化 args，使其指向第一个变参
   ```

3. **`va_end`**：用于清理 `va_list` 变量，结束变长参数的处理。虽然不总是必须的，但按照标准规范使用 `va_end` 可以确保没有资源泄露。

   ```c
   va_end(args);  // 清理变参列表
   ```

通过这些宏，我们可以灵活地访问和操作变长参数，实现像 `printf` 这样的自定义变参函数。

---

在 GNU C Library也就是`glibc`中，`vfprintf` 的核心实现依赖于 `vfprintf-internal`，也就是这样：

```c
/* Copyright (C) 1991-2024 Free Software Foundation, Inc.
   This file is part of the GNU C Library.

   The GNU C Library is free software; you can redistribute it and/or
   modify it under the terms of the GNU Lesser General Public
   License as published by the Free Software Foundation; either
   version 2.1 of the License, or (at your option) any later version.

   The GNU C Library is distributed in the hope that it will be useful,
   but WITHOUT ANY WARRANTY; without even the implied warranty of
   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
   Lesser General Public License for more details.

   You should have received a copy of the GNU Lesser General Public
   License along with the GNU C Library; if not, see
   <https://www.gnu.org/licenses/>.  */

#include <libio/libioP.h>

extern int
__vfprintf (FILE *fp, const char *format, va_list ap)
{
  return __vfprintf_internal (fp, format, ap, 0);
}
ldbl_strong_alias (__vfprintf, _IO_vfprintf);
ldbl_strong_alias (__vfprintf, vfprintf);
ldbl_hidden_def (__vfprintf, vfprintf)
```

具体实现可以看[vfprintf-internal 的实现](https://github.com/bminor/glibc/blob/master/stdio-common/vfprintf-internal.c)。

`vfprintf-internal` 负责解析 `format` 字符串，识别各种格式说明符（如 `%d`、`%s` 等）。

在 `vfprintf-internal` 识别出特定格式说明符后，会调用 `vfprintf-process-arg`（[具体实现可以看这里](https://github.com/bminor/glibc/blob/master/stdio-common/vfprintf-process-arg.c)） 来处理这些参数，将它们正确格式化并插入到输出流中。例如，当遇到 `%s` 时，`vfprintf-process-arg` 会将变参中的字符串插入到输出位置，或在遇到 `%d` 时将整数转换为字符串再插入。

这种调用关系的设计是为了让 `vfprintf-internal` 专注于整体的格式字符串解析流程，而 `vfprintf-process-arg` 则专门负责具体参数的读取、转换和格式化。