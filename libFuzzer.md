#### 安装

```shell
sudo apt update
# libfuzzer集成在clang中
sudo apt install clang
```

#### fuzz paozhu的url_decode

1. 定位函数：

```txt
vendor/httpserver/include/urlcode.h

vendor/httpserver/src/urlcode.cpp
```

2. 创建fuzz目录，fuzz目录在项目目录下，因为编译依赖项目的头文件和源文件

```txt
├── vendor/
├── src/
├── fuzz/
│   |
│   └── fuzz_url_decode.cpp
```

3. 编写fuzz harness

```c
#include <stdint.h>
#include <stddef.h>

#include "urlcode.h"


// extern "C" int LLVMFuzzerTestOneInput(... libfuzzer规定的入口函数
// libfuzzer喂入的参数固定是const uint8_t *data和size_t size，如果被fuzz的函数不适合直接传入这两个参数，就得自己写逻辑拆分
extern "C"
int LLVMFuzzerTestOneInput(
    const uint8_t *data,
    size_t size)
{

    // 被fuzz函数前的一些处理规则，过滤掉一些fuzz输入，这里防止超长输入拖慢fuzz速度
    if(size > 64 * 1024)
        return 0;


    // url_decode是被fuzz函数
    auto result =
        http::url_decode(
            (const char*)data,
            size
        );


    return 0;
}
```

4. 编译

```shell
clang++ \
  -std=c++20 \
  -O1 \
  -g \
  -fno-omit-frame-pointer \
  -fsanitize=fuzzer,address \
  fuzz/fuzz_url_decode.cpp \
  vendor/httpserver/src/urlcode.cpp \
  -Ivendor/httpserver/include \
  -o fuzz_url_decode
```

5. seed的编写

每种seed一个文件

Seed 不需要很多，通常十几个到几十个足够。好的 seed 应满足：

- 尽量短，便于 fuzzer 修改。
- 每个 seed 覆盖一种不同结构。
- 同时包含合法和非法输入。
- 包含边界情况，例如空输入、截断编码、NUL。
- 不要放大量重复样本。
- 不要一开始放几 MB 的文件。
- 不要放纯随机文件，libFuzzer 自己会生成随机变异。

```shell
printf 'hello' > corpus/url_decode/plain
printf 'hello%%20world' > corpus/url_decode/space
printf 'a%%2Fb%%3Fc%%3Dd' > corpus/url_decode/reserved
printf 'name+value' > corpus/url_decode/plus
printf '%%41%%42%%43' > corpus/url_decode/hex

printf '%%' > corpus/url_decode/percent_only
printf '%%A' > corpus/url_decode/one_hex
printf '%%GG' > corpus/url_decode/non_hex
printf '%%%%%%' > corpus/url_decode/many_percent
printf 'abc%%2' > corpus/url_decode/truncated
printf 'abc%%ZZdef' > corpus/url_decode/invalid_middle

printf '../' > corpus/url_decode/dotdot
printf '%%2e%%2e%%2f' > corpus/url_decode/encoded_traversal
printf '%%2E%%2E%%2F' > corpus/url_decode/uppercase_traversal
printf '%%252e%%252e%%252f' > corpus/url_decode/double_encoded
printf '..%%5coutside' > corpus/url_decode/backslash

printf 'abc%%00def' > corpus/url_decode/encoded_nul

python3 - <<'PY'
from pathlib import Path
Path("corpus/url_decode/raw_nul").write_bytes(b"abc\x00def")
PY

python3 - <<'PY'
from pathlib import Path
Path("corpus/url_decode/binary").write_bytes(b"\xff\xfe%41\x80")
PY

touch corpus/url_decode/empty
```



6. 运行

```shell
# 使用corpus/url_decode下seed，最长输入65536，生成异常样本前缀artifacts/url_decode-
./fuzz_url_decode corpus/url_decode \
  -max_len=65536 \
  -artifact_prefix=artifacts/url_decode-
 

# -fork=4：同时运行最多 4 个子进程；-ignore_crashes=1：某个子进程崩溃后保存样本，继续 fuzz；-max_total_time=3600：运行一小时；-timeout=2：单个输入运行超过两秒，视为超时
# -runs=1000000该参数可以限制运行次数，默认是-1，无限运行
./fuzz_url_decode corpus/url_decode \
  -fork=4 \
  -ignore_crashes=1 \
  -max_total_time=3600 \
  -max_len=4096 \
  -timeout=2 \
  -artifact_prefix=artifacts/url_decode-
```

7. 
