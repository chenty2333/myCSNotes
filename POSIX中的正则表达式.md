# POSIX标准的正则表达式API

### C标准库

- POSIX标准的正则表达式API由操作系统中的C标准库实现。
- 在Linux系统中，有GNU C Library实现。 应用程序通过调用库函数实现正则表达式操作，无需与内核交互。

---

### recomp

`int regcomp(regex_t *preg, const char *pattern, int cflags);`将正则表达式编译成内部形式。

 **参数**：

- `preg`：指向 `regex_t` 结构的指针，用于存储编译后的正则表达式。
- `pattern`：表示要编译的正则表达式字符串。
- `cflags`：用于控制编译行为的标志。
  - `REG_EXTENDED`：使用扩展正则表达式语法。POSIX 定义了两种正则表达式语法：基本正则表达式（BRE）和扩展正则表达式（ERE）。默认情况下使用 BRE 语法，而设置 `REG_EXTENDED` 则启用更强大的 ERE 语法，支持更丰富的运算符，例如 `+`、`|` 和 `()`。
  - `REG_ICASE`：忽略大小写。启用此标志后，正则表达式在匹配时不区分大小写，如 `a` 会匹配 `A` 和 `a`。
  - `REG_NOSUB`：不需要存储匹配的子表达式。对于某些场景，仅需要知道是否匹配，不需要记录捕获的子表达式，此标志可以减少内存开销，提高效率。
  - `REG_NEWLINE`：改变 `.` 和 `[^...]` 的行为，使其不会匹配换行符 `\n`，同时会影响 `^` 和 `$`，分别匹配字符串的行首和行尾，而不是整个文本的首尾。

**返回值**：成功时返回0，否则返回非零值表示错误

| 标志名         | 说明                                                | 示例用途                                           |
| -------------- | --------------------------------------------------- | -------------------------------------------------- |
| `REG_EXTENDED` | 启用扩展正则表达式语法                              | 支持更复杂的表达式，如分支结构和多种量词。         |
| `REG_ICASE`    | 忽略大小写                                          | 检查不区分大小写的匹配，如验证一个名字。           |
| `REG_NOSUB`    | 不返回子表达式的匹配信息                            | 只关心是否匹配，而不关心捕获组，如检查简单的模式。 |
| `REG_NEWLINE`  | 支持行模式，`^` 和 `$` 作用于行首行尾，而非整个文本 | 处理多行文本中的行首行尾匹配，如逐行处理配置文件。 |

---

### regex_t

`regex_t`结构体用于存储编译后的正则表达式。包含编译后的正则表达式指针，编译标志``cflags`，字符分类表等。

类似于：

```c
typedef struct
{
  size_t re_nsub;          /* 子表达式数量 */
  re_dfa_t *re_dfa;        /* 指向已编译模式的指针 */
} regex_t;
```

其中re_dfa是一个指向``re_dfa_t`结构体的指针。

#### re_nsub

`re_nsub`记录正则表达式中子表达式的数量，如：

```regex
^(https?)://([^/]+)(/.*)?$
```

在该正则表达式中：

- `(https?)`：匹配协议部分，如 `http` 或 `https`。
- `([^/]+)`：匹配域名部分，确保不包含 `/`。
- `(/.*)?`：匹配路径部分，表示以 `/` 开头的任意字符序列，`?` 表示该部分是可选的。

这里，正则表达式包含三个子表达式，因此 `re_nsub` 的值为 3。

#### re_dfa_t

`re_dfa_t`形如：

```c
typedef struct re_dfa_t
{
  int re_magic;            /* 魔数，用于标识此结构体 */
  size_t re_nsub;          /* 子表达式数量 */
  unsigned char *re_guts;  /* 指向已编译模式的指针 */
  size_t re_guts_size;     /* 已编译模式的大小（字节） */
  int re_flags;            /* 编译时使用的标志 */
  int re_max_failures;     /* 在放弃匹配前允许的最大失败次数 */
  int re_num_states;       /* DFA 中的状态数量 */
  int re_num_transitions;  /* DFA 中的状态转换数量 */
  int re_start_state;      /* DFA 的起始状态 */
  int re_accept_state;     /* DFA 的接受状态 */
  int *re_state_table;     /* 状态转换表 */
  int *re_accept_table;    /* 接受状态表 */
  int *re_fail_table;      /* 失败状态表 */
} re_dfa_t;
```

这些结构体在 `regcomp` 函数中被初始化，并在 `regexec` 函数中用于指导正则表达式的匹配过程。

`regcomp`大致的代码：

```c
int
regcomp (regex_t *preg, const char *pattern, int cflags)
{
  reg_errcode_t err;
  size_t pattern_len = strlen (pattern);

  // 初始化 regex_t 结构体
  preg->re_nsub = 0;
  preg->re_dfa = NULL;

  // 解析正则表达式
  err = re_compile_internal (preg, pattern, pattern_len, cflags);
  if (err != REG_NOERROR)
    {
      regfree (preg);
      return err;
    }

  // 优化 DFA
  err = analyze (preg);
  if (err != REG_NOERROR)
    {
      regfree (preg);
      return err;
    }

  return REG_NOERROR;
}
```

完整代码请参阅 [glibc 的 regexec.c 实现](https://github.com/bminor/glibc/blob/master/posix/regexec.c)。

---

### regexec

`int regexec(const regex_t *preg, const char *string, size_t nmatch, regmatch_t pmatch[], int eflags);`执行正则表达式匹配

 **参数**：

- `preg`：指向编译好的正则表达式（即 `regcomp` 生成的 `regex_t`）。
- `string`：目标字符串。
- `nmatch`：`pmatch` 数组中能存储的最大匹配结果数量。
- `pmatch`：用于存储匹配结果的位置和长度信息。`regmatch_t` 结构会包含起始位置和结束位置。
- `eflags`：控制匹配行为的标志，例如 `REG_NOTBOL` 表示字符串不是行首，`REG_NOTEOL` 表示不是行尾。

**返回值**：0表示匹配成功，`REG_NOMATCH` 表示不匹配，其他值则表示错误。

以下是一个`regexec`大致实现

```c
int
regexec (const regex_t *preg, const char *string, size_t nmatch,
         regmatch_t pmatch[], int eflags)
{
  reg_errcode_t err;
  re_match_context_t mctx;

  // 初始化匹配上下文
  mctx.preg = preg;
  mctx.input = string;
  mctx.nmatch = nmatch;
  mctx.pmatch = pmatch;
  mctx.eflags = eflags;

  // 执行匹配
  err = re_search (&mctx, 0, strlen (string));
  if (err != REG_NOERROR)
    return err;

  return REG_NOERROR;
}
```

其中`re_search (re_match_context_t *mctx, size_t start, size_t length)`负责在指定的字符串范围内搜索与预编译的正则表达式模式相匹配的子串。

完整代码请参阅 [glibc 的 regfree.c 实现](https://github.com/bminor/glibc/blob/master/posix/regfree.c)。

#### re_search

**参数说明**：

- `mctx`：指向匹配上下文的指针，包含了预编译的正则表达式、目标字符串等信息。
- `start`：搜索的起始位置索引。
- `length`：从起始位置开始搜索的长度。

以下是`re_search`简化后的示意：

```c
reg_errcode_t
re_search (re_match_context_t *mctx, size_t start, size_t length)
{
  const char *input = mctx->input + start;
  size_t remaining = length;
  int state = mctx->preg->re_dfa->re_start_state;
  int *state_table = mctx->preg->re_dfa->re_state_table;

  while (remaining > 0)
    {
      char ch = *input++;
      remaining--;

      int next_state = state_table[state * 256 + (unsigned char)ch];
      if (next_state == -1)
        return REG_NOMATCH;

      state = next_state;
    }

  if (state == mctx->preg->re_dfa->re_accept_state)
    {
      // 记录匹配信息
      mctx->pmatch[0].rm_so = start;
      mctx->pmatch[0].rm_eo = start + length;
      return REG_NOERROR;
    }
  else
    return REG_NOMATCH;
}

```

`re_search` 的核心逻辑是使用正则表达式的有限状态自动机 (DFA) 来逐字符扫描目标字符串，判断是否符合预编译的正则表达式模式。

`re_search`中的while循环，实现：

**读取字符**：从 `input` 中取出当前字符 `ch`，并向前移动指针 `input++`。

**查找转换状态**：`next_state` 在状态表中查找 `state * 256 + (unsigned char)ch`。通过 `state` 和当前字符 `ch` 计算表索引。

- 如果 `next_state == -1`，意味着在当前状态下，无法匹配此字符（失败），因此返回 `REG_NOMATCH`。

**状态更新**：将当前状态更新为 `next_state`，继续读取下一个字符并更新状态。

---

### * DFA

每个正则表达式都可以转换为等价的DFA，用于识别特定的字符串模式。DFA通过状态转换来匹配输入字符串，正如正则表达式通过模式描述来匹配字符串一样。

**示例：匹配三个连续的'1'**

DFA来精确识别`111`。

**DFA设计**

1. **状态集（States）**：

   - `q0`：初始状态，未开始匹配。
   - `q1`：匹配到一个'1'。
   - `q2`：匹配到两个连续的'1'。
   - `q3`：匹配到三个连续的'1'（接受状态）。
   - `q4`：匹配失败状态。

2. **输入字母表（Alphabet）**：

   - `{0, 1}`

3. **转移函数（Transition Function）**：

   | 当前状态 | 输入符号 | 下一状态 |
   | -------- | -------- | -------- |
   | `q0`     | `1`      | `q1`     |
   | `q1`     | `1`      | `q2`     |
   | `q2`     | `1`      | `q3`     |
   | `q3`     | `0`      | `q4`     |
   | `q4`     | `0`      | `q4`     |
   | `q4`     | `1`      | `q4`     |
   | `q0`     | `0`      | `q0`     |
   | `q1`     | `0`      | `q0`     |
   | `q2`     | `0`      | `q0`     |
   | `q3`     | `1`      | `q3`     |

4. **初始状态（Start State）**：

   - `q0`

5. **接受状态集（Accept States）**：

   - `{q3}`

**工作流程**

- 从初始状态`q0`开始。
- 读取输入字符串的每个字符，根据转移函数移动到相应的状态。
- 当读取完所有字符后，若当前状态是接受状态`q3`，则表示输入字符串包含**恰好三个连续的'1'**；否则，表示不匹配。

**示例**

- **输入字符串**：`111`
  - `q0` --`1`--> `q1`
  - `q1` --`1`--> `q2`
  - `q2` --`1`--> `q3`
  - 最终状态为`q3`，因此输入字符串`111`被接受，表示匹配成功。
- **输入字符串**：`0110111`
  - `q0` --`0`--> `q0`
  - `q0` --`1`--> `q1`
  - `q1` --`1`--> `q2`
  - `q2` --`1`--> `q3`
  - `q3` --`0`--> `q4`（匹配失败）
  - `q4` --`1`--> `q4`
  - `q4` --`1`--> `q4`
  - `q4` --`1`--> `q4`
  - 最终状态为`q4`，因此输入字符串`0110111`被拒绝，表示不匹配。

---

### regerror

`size_t regerror(int errcode, const regex_t *preg, char *errbuf, size_t errbuf_size);`函数用于转换正则表达式函数的错误码，生成对应的错误信息。

一个具体的使用例子：

```c
#include <stdio.h>
#include <regex.h>
#include <stdlib.h>

int main() {
    regex_t regex;
    int result;
    char error_message[100];

    // 尝试编译一个无效的正则表达式
    result = regcomp(&regex, "(", REG_EXTENDED);
    
    if (result != 0) {
        // 如果编译失败，调用 regerror 获取错误信息
        regerror(result, &regex, error_message, sizeof(error_message));
        printf("Regex compilation failed: %s\n", error_message);
    } else {
        printf("Regex compiled successfully.\n");
        // 成功编译后释放内存
        regfree(&regex);
    }

    return 0;
}
```

`regerror` 函数会根据不同的 `result` 返回值，将对应的错误信息填充到 `error_message` 中。如果 `result` 为 0，表示函数执行成功，此时 `error_message` 数组中将包含空字符串，即 `""`.

在 GNU C Library（glibc）的源代码中，正则表达式相关函数的返回值（即 `result` 的不同值）被定义为一组枚举常量，代表各种可能的错误类型：

```c
/* Error codes for the POSIX regex functions */
typedef enum {
  REG_NOERROR = 0,      /* No error */
  REG_NOMATCH,          /* No match */
  REG_BADPAT,           /* Invalid regular expression */
  REG_ECOLLATE,         /* Invalid collating element */
  REG_ECTYPE,           /* Invalid character class name */
  REG_EESCAPE,          /* Trailing backslash */
  REG_ESUBREG,          /* Invalid back reference */
  REG_EBRACK,           /* Unmatched '[' or ']' */
  REG_EPAREN,           /* Unmatched '(' or ')' */
  REG_EBRACE,           /* Unmatched '{' or '}' */
  REG_ERANGE,           /* Invalid range end */
  REG_ESPACE,           /* Out of memory */
  REG_BADBR,            /* Invalid content of '{...}' */
  REG_BADRPT,           /* Invalid use of repetition operator */
  REG_EEND,             /* Nonspecific error */
  REG_ESIZE,            /* Regular expression too big */
  REG_ERPAREN,          /* Unmatched ')' or '\\)' */
  REG_ECONTEXT          /* Invalid use of context operator */
} reg_errcode_t;
```

中文版：

```c
/* POSIX 正则表达式函数的错误码 */
typedef enum {
  REG_NOERROR = 0,      /* 无错误 */
  REG_NOMATCH,          /* 无匹配项 */
  REG_BADPAT,           /* 无效的正则表达式 */
  REG_ECOLLATE,         /* 无效的排序元素 */
  REG_ECTYPE,           /* 无效的字符类名称 */
  REG_EESCAPE,          /* 以反斜杠结尾 */
  REG_ESUBREG,          /* 无效的反向引用 */
  REG_EBRACK,           /* 未匹配的 '[' 或 ']' */
  REG_EPAREN,           /* 未匹配的 '(' 或 ')' */
  REG_EBRACE,           /* 未匹配的 '{' 或 '}' */
  REG_ERANGE,           /* 无效的范围结束 */
  REG_ESPACE,           /* 内存不足 */
  REG_BADBR,            /* '{...}' 中的内容无效 */
  REG_BADRPT,           /* 无效的重复运算符使用 */
  REG_EEND,             /* 非特定错误 */
  REG_ESIZE,            /* 正则表达式过大 */
  REG_ERPAREN,          /* 未匹配的 ')' 或 '\\)' */
  REG_ECONTEXT          /* 无效的上下文运算符使用 */
} reg_errcode_t;
```

[完整代码参考](https://github.com/bminor/glibc/blob/master/posix/regex.c)

---

### regfree

用于释放由 `regcomp` 分配的内存资源，如：

- **状态机缓冲区**：`regcomp` 会为正则表达式创建一个状态机（如 DFA），并将其存储在 `regex_t` 的 `buffer` 字段中。 

  [GitHub](https://github.com/bminor/glibc/blob/master/posix/regcomp.c)

- **快速映射表（Fastmap）**：用于加速匹配过程的字符映射表，存储在 `regex_t` 的 `fastmap` 字段中。 

  [GitHub](https://github.com/bminor/glibc/blob/master/posix/regcomp.c)

大致代码：

```c
void regfree(regex_t *preg) {
    if (preg != NULL) {
        // 释放状态机缓冲区
        if (preg->buffer != NULL) {
            free(preg->buffer);
            preg->buffer = NULL;
        }
        // 释放快速映射表
        if (preg->fastmap != NULL) {
            free(preg->fastmap);
            preg->fastmap = NULL;
        }
        // 释放翻译表
        if (preg->translate != NULL) {
            free(preg->translate);
            preg->translate = NULL;
        }
        // 其他资源的释放
        // ...
    }
}
```

[完整代码参考](https://github.com/bminor/glibc/blob/master/posix/regex.c)