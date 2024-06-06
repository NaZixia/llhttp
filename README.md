# llhttp
[![CI](https://github.com/nodejs/llhttp/workflows/CI/badge.svg)](https://github.com/nodejs/llhttp/actions?query=workflow%3ACI)

Port of [http_parser][0] to [llparse][1].

## Why?
让我们面对现实，[http_parser][0] 实际上几乎是无法维护的。即使是引入一个新的方法也会导致代码的显著变动。

该项目的目标是：

* 使其可维护
* 可验证
* 尽可能提高基准测试

更多详情请参阅 [Fedor Indutny 在 JSConf EU 2019 的演讲](https://youtu.be/x3k_5Mi66sY)

## How?
随着时间的推移，人们尝试了不同的方法来改进 [http_parser][0] 的代码库。然而，它们都因导致显著的性能下降而失败了。

这个项目是 [http_parser][0] 到 TypeScript 的移植。使用 [llparse][1] 来生成输出的 C 源文件，这些文件可以编译并与嵌入程序（如 [Node.js][7]）链接。

## Performance

到目前为止，llhttp 的性能优于 http_parser：

|                 | 输入大小  |  带宽         |  请求/秒      |  时间     |
|:----------------|-----------:|-------------:|-------------:|--------:|
| **llhttp**      | 8192.00 MB | 1777.24 MB/s | 3583799.39 req/sec | 4.61 s |
| **http_parser** | 8192.00 MB | 694.66 MB/s  | 1406180.33 req/sec | 11.79 s |

llhttp 的速度快了大约 **156%**。

## Maintenance
llhttp 项目大约有 1400 行 TypeScript 代码用于描述解析器本身，以及大约 450 行 C 代码和头文件用于提供辅助方法。整个 [http_parser][0] 大约由 2500 行 C 代码和 436 行头文件实现。

llhttp 中的所有优化和多字符匹配都是自动生成的，因此不会增加任何额外的维护成本。相反，http_parser 的大部分代码都是手工优化和展开的。与其描述“如何”解析 HTTP 请求/响应，维护者需要在 [http_parser][0] 中谨慎地实现新功能，考虑可能的性能下降并手动优化新代码。

## 验证

状态机图在 llhttp 中被明确编码。[llparse][1] 自动检查图中是否存在循环，并确保输入范围（如头部名称和值）的正确报告。未来，可以执行额外的检查，以实现对 llhttp 的更严格验证。

## Usage

```C
#include "stdio.h"
#include "llhttp.h"
#include "string.h"

int handle_on_message_complete(llhttp_t* parser) {
	fprintf(stdout, "Message completed!\n");
	return 0;
}

int main() {
	llhttp_t parser;
	llhttp_settings_t settings;

	/* 初始化用户回调函数和设置 */
	llhttp_settings_init(&settings);

	/*Set user callback */
	settings.on_message_complete = handle_on_message_complete;

	/* 在 HTTP_BOTH 模式下初始化解析器，意味着它将在读取第一个输入时自动选择
	* HTTP_REQUEST 和 HTTP_RESPONSE 解析。
	*/
	llhttp_init(&parser, HTTP_BOTH, &settings);

	/* 解析请求！ */
	const char* request = "GET / HTTP/1.1\r\n\r\n";
	int request_len = strlen(request);

	enum llhttp_errno err = llhttp_execute(&parser, request, request_len);
	if (err == HPE_OK) {
		fprintf(stdout, "Successfully parsed!\n");
	} else {
		fprintf(stderr, "Parse error: %s %s\n", llhttp_errno_name(err), parser.reason);
	}
}
```有关 API 使用的更多信息，请参阅 [src/native/api.h]
(https://github.com/nodejs/llhttp/blob/main/src/native/api.h)。
## API

### llhttp_settings_t

设置对象包含解析器将调用的回调函数列表。

以下回调函数可以返回 `0`（正常继续）、`-1`（错误）或 `HPE_PAUSED`（暂停解析器）：

* `on_message_begin`: 在新请求/响应开始时调用。
* `on_message_complete`: 在请求/响应完全解析完成时调用。
* `on_url_complete`: 在 URL 解析完成后调用。
* `on_method_complete`: 在 HTTP 方法解析完成后调用。
* `on_version_complete`: 在 HTTP 版本解析完成后调用。
* `on_status_complete`: 在状态码解析完成后调用。
* `on_header_field_complete`: 在头部名称解析完成后调用。
* `on_header_value_complete`: 在头部值解析完成后调用。
* `on_chunk_header`: 在开始新的块后调用。当前块长度存储在 `parser->content_length` 中。
* `on_chunk_extension_name_complete`: 在块扩展名开始后调用。
* `on_chunk_extension_value_complete`: 在块扩展值开始后调用。
* `on_chunk_complete`: 在接收到新块后调用。
* `on_reset`: 在接收到同一解析器上的新消息之后，调用 `on_message_complete` 之后和 `on_message_begin` 之前。对于解析器的第一条消息，不会调用此函数。以下回调函数可以返回 `0`（正常继续）、`-1`（错误）或 `HPE_USER`（来自回调函数的错误）：

* `on_url`: 当收到 URL 的另一个字符时调用。
* `on_status`: 当收到状态的另一个字符时调用。
* `on_method`: 当收到方法的另一个字符时调用。当解析器使用 `HTTP_BOTH` 创建并且输入为响应时，也会在第一条消息的 `HTTP/` 序列中调用此函数。
* `on_version`: 当收到版本的另一个字符时调用。
* `on_header_field`: 当收到头部名称的另一个字符时调用。
* `on_header_value`: 当收到头部值的另一个字符时调用。
* `on_chunk_extension_name`: 当收到块扩展名的另一个字符时调用。
* `on_chunk_extension_value`: 当收到扩展值的另一个字符时调用。

当头部完成时调用的 `on_headers_complete` 回调函数可以返回：

* `0`: 正常继续。
* `1`: 假定请求/响应没有主体，并继续解析下一条消息。
* `2`: 假定没有主体（如上所述），并使 `llhttp_execute()` 返回 `HPE_PAUSED_UPGRADE`。
* `-1`: 错误
* `HPE_PAUSED`: 暂停解析器。
### `void llhttp_init(llhttp_t* parser, llhttp_type_t type, const llhttp_settings_t* settings)`

使用特定类型和用户设置初始化解析器。

### `uint8_t llhttp_get_type(llhttp_t* parser)`

返回解析器的类型。

### `uint8_t llhttp_get_http_major(llhttp_t* parser)`

返回当前请求/响应的 HTTP 协议的主版本号。

### `uint8_t llhttp_get_http_minor(llhttp_t* parser)`

返回当前请求/响应的 HTTP 协议的次版本号。

### `uint8_t llhttp_get_method(llhttp_t* parser)`

返回当前请求的方法。

### `int llhttp_get_status_code(llhttp_t* parser)`

返回当前响应的状态码。### `uint8_t llhttp_get_upgrade(llhttp_t* parser)`

如果请求包含 `Connection: upgrade` 头，则返回 `1`。

### `void llhttp_reset(llhttp_t* parser)`

将已经初始化的解析器重置回起始状态，保留现有的解析器类型、回调设置、用户数据和宽松标志。

### `void llhttp_settings_init(llhttp_settings_t* settings)`

初始化设置对象。

### `llhttp_errno_t llhttp_execute(llhttp_t* parser, const char* data, size_t len)`


`llhttp_execute` 在给定的数据上执行解析器。解析器将逐步解析数据，并根据解析的结果调用相应的回调函数。解析完成后，将返回相应的错误代码（如果有）。解析完整或部分的请求/响应，在解析过程中调用用户回调函数。

如果任何 `llhttp_data_cb` 返回的 errno 不等于 `HPE_OK`，解析过程将中断，并从 `llhttp_execute()` 返回该 errno。如果 `HPE_PAUSED` 作为 errno 使用，可以通过调用 `llhttp_resume()` 来恢复执行。

在 CONNECT/Upgrade 请求/响应的特殊情况下，在完全解析请求/响应后，将返回 `HPE_PAUSED_UPGRADE`。如果用户希望继续解析，他们需要调用 `llhttp_resume_after_upgrade()`。

**如果此函数返回非暂停类型的错误，它将继续在每次成功调用之前返回相同的错误，直到调用 `llhttp_init()`。**

### `llhttp_errno_t llhttp_finish(llhttp_t* parser)`
此方法应在另一端没有进一步发送字节时调用（例如，关闭了 TCP 连接的可读端）。

没有 `Content-Length` 和其他消息的请求可能需要将所有传入的字节视为消息体的一部分，直到连接的最后一个字节。

如果请求已安全终止，则此方法将调用 `on_message_complete()` 回调。否则，将返回错误代码。

### `int llhttp_message_needs_eof(const llhttp_t* parser)`

如果传入消息已解析到最后一个字节，并且必须通过在 EOF 上调用 `llhttp_finish()` 来完成，则返回 `1`。

### `int llhttp_should_keep_alive(const llhttp_t* parser)`

如果可能还有其他消息跟随上次成功解析的消息，则返回 `1`。

### `void llhttp_pause(llhttp_t* parser)`使进一步调用 `llhttp_execute()` 返回 `HPE_PAUSED`，并设置相应的错误原因。

**不要从用户回调中调用此函数！如果需要暂停，则用户回调必须返回 `HPE_PAUSED`。**

### `void llhttp_resume(llhttp_t* parser)`

在用户回调中暂停后，可能会调用此函数以恢复执行。

有关详细信息，请参阅上面的 `llhttp_execute()`。

**仅在 `llhttp_execute()` 返回 `HPE_PAUSED` 时调用此函数。**

### `void llhttp_resume_after_upgrade(llhttp_t* parser)`

在用户回调中暂停后，可能会调用此函数以恢复执行。

有关详细信息，请参阅上面的 `llhttp_execute()`。

**仅在 `llhttp_execute()` 返回 `HPE_PAUSED_UPGRADE` 时调用此函数。**

### `llhttp_errno_t llhttp_get_errno(const llhttp_t* parser)`

返回最新的错误。

### `const char* llhttp_get_error_reason(const llhttp_t* parser)`

返回最新返回的错误的文本解释。

**当返回错误时，用户回调应设置错误原因。参见 `llhttp_set_error_reason()` 获取详细信息。**

### `void llhttp_set_error_reason(llhttp_t* parser, const char* reason)`

将文本描述分配给返回的错误。必须在用户回调中，在返回 errno 之前调用此函数。

**在用户回调中，`HPE_USER` 错误代码可能很有用。**
### `const char* llhttp_get_error_pos(const llhttp_t* parser)`

返回返回的错误之前最后解析的字节的指针。指针相对于 `llhttp_execute()` 的 `data` 参数。

**此方法可能对计算已解析字节的数量很有用。**

### `const char* llhttp_errno_name(llhttp_errno_t err)`

返回错误代码的文本名称。

### `const char* llhttp_method_name(llhttp_method_t method)`

返回 HTTP 方法的文本名称。

### `const char* llhttp_status_name(llhttp_status_t status)`

返回 HTTP 状态的文本名称。

### `void llhttp_set_lenient_headers(llhttp_t* parser, int enabled)`

启用/禁用宽松的头部值解析（默认情况下禁用）。
宽松解析禁用头部值令牌检查，扩展了 llhttp 解析器的功能。's
protocol support to highly non-compliant clients/server. 

No `HPE_INVALID_HEADER_TOKEN` will be raised for incorrect header values when
lenient parsing is "on".

**Enabling this flag can pose a security issue since you will be exposed to request smuggling attacks. USE WITH CAUTION!**

### `void llhttp_set_lenient_chunked_length(llhttp_t* parser, int enabled)`

Enables/disables lenient handling of conflicting `Transfer-Encoding` and
`Content-Length` headers (disabled by default).

Normally `llhttp` would error when `Transfer-Encoding` is present in
conjunction with `Content-Length`. 

This error is important to prevent HTTP request smuggling, but may be less desirable
for small number of cases involving legacy servers.

**Enabling this flag can pose a security issue since you will be exposed to request smuggling attacks. USE WITH CAUTION!**

### `void llhttp_set_lenient_keep_alive(llhttp_t* parser, int enabled)`

Enables/disables lenient handling of `Connection: close` and HTTP/1.0
requests responses.

Normally `llhttp` would error the HTTP request/response 
after the request/response with `Connection: close` and `Content-Length`. 

This is important to prevent cache poisoning attacks,
but might interact badly with outdated and insecure clients. 

With this flag the extra request/response will be parsed normally.

**Enabling this flag can pose a security issue since you will be exposed to poisoning attacks. USE WITH CAUTION!**

### `void llhttp_set_lenient_transfer_encoding(llhttp_t* parser, int enabled)`

Enables/disables lenient handling of `Transfer-Encoding` header.

Normally `llhttp` would error when a `Transfer-Encoding` has `chunked` value
and another value after it (either in a single header or in multiple
headers whose value are internally joined using `, `).

This is mandated by the spec to reliably determine request body size and thus
avoid request smuggling.

With this flag the extra value will be parsed normally.

**Enabling this flag can pose a security issue since you will be exposed to request smuggling attacks. USE WITH CAUTION!**

### `void llhttp_set_lenient_version(llhttp_t* parser, int enabled)`

Enables/disables lenient handling of HTTP version.

Normally `llhttp` would error when the HTTP version in the request or status line
is not `0.9`, `1.0`, `1.1` or `2.0`.
With this flag the extra value will be parsed normally.

**Enabling this flag can pose a security issue since you will allow unsupported HTTP versions. USE WITH CAUTION!**

### `void llhttp_set_lenient_data_after_close(llhttp_t* parser, int enabled)`

Enables/disables lenient handling of additional data received after a message ends
and keep-alive is disabled.

Normally `llhttp` would error when additional unexpected data is received if the message
contains the `Connection` header with `close` value.
With this flag the extra data will discarded without throwing an error.

**Enabling this flag can pose a security issue since you will be exposed to poisoning attacks. USE WITH CAUTION!**

### `void llhttp_set_lenient_optional_lf_after_cr(llhttp_t* parser, int enabled)`

Enables/disables lenient handling of incomplete CRLF sequences.

Normally `llhttp` would error when a CR is not followed by LF when terminating the
request line, the status line, the headers or a chunk header.
With this flag only a CR is required to terminate such sections.

**Enabling this flag can pose a security issue since you will be exposed to request smuggling attacks. USE WITH CAUTION!**

### `void llhttp_set_lenient_optional_cr_before_lf(llhttp_t* parser, int enabled)`

Enables/disables lenient handling of line separators.

Normally `llhttp` would error when a LF is not preceded by CR when terminating the
request line, the status line, the headers, a chunk header or a chunk data.
With this flag only a LF is required to terminate such sections.

**Enabling this flag can pose a security issue since you will be exposed to request smuggling attacks. USE WITH CAUTION!**

### `void llhttp_set_lenient_optional_crlf_after_chunk(llhttp_t* parser, int enabled)`

Enables/disables lenient handling of chunks not separated via CRLF.

Normally `llhttp` would error when after a chunk data a CRLF is missing before
starting a new chunk.
With this flag the new chunk can start immediately after the previous one.

**Enabling this flag can pose a security issue since you will be exposed to request smuggling attacks. USE WITH CAUTION!**

### `void llhttp_set_lenient_spaces_after_chunk_size(llhttp_t* parser, int enabled)`

Enables/disables lenient handling of spaces after chunk size.

Normally `llhttp` would error when after a chunk size is followed by one or more spaces are present instead of a CRLF or `;`.
With this flag this check is disabled.

**Enabling this flag can pose a security issue since you will be exposed to request smuggling attacks. USE WITH CAUTION!**

## Build Instructions

Make sure you have [Node.js](https://nodejs.org/), npm and npx installed. Then under project directory run:

```sh
npm ci
make
```

---

### Bindings to other languages

* Lua: [MunifTanjim/llhttp.lua][11]
* Python: [pallas/pyllhttp][8]
* Ruby: [metabahn/llhttp][9]
* Rust: [JackLiar/rust-llhttp][10]

### 使用 CMake

如果您想将此库用作 CMake 项目中的共享库，您可以使用下面的代码片段。

```cmake
FetchContent_Declare(llhttp
  URL "https://github.com/nodejs/llhttp/archive/refs/tags/release/v8.1.0.tar.gz")

FetchContent_MakeAvailable(llhttp)

# Link with the llhttp_shared target
target_link_libraries(${EXAMPLE_PROJECT_NAME} ${PROJECT_LIBRARIES} llhttp_shared ${PROJECT_NAME})
```

如果您想在 CMake 项目中将此库用作静态库，您可以首先设置一些缓存变量。

```cmake
FetchContent_Declare(llhttp
  URL "https://github.com/nodejs/llhttp/archive/refs/tags/release/v8.1.0.tar.gz")

set(BUILD_SHARED_LIBS OFF CACHE INTERNAL "")
set(BUILD_STATIC_LIBS ON CACHE INTERNAL "")
FetchContent_MakeAvailable(llhttp)

# Link with the llhttp_static target
target_link_libraries(${EXAMPLE_PROJECT_NAME} ${PROJECT_LIBRARIES} llhttp_static ${PROJECT_NAME})
```

请注意，直接使用 git 存储库（例如，通过 git 存储库 URL 和标签）将无法使用 FetchContent_Declare，因为 [CMakeLists.txt](./CMakeLists.txt) 需要进行字符串替换（例如，`_RELEASE_`）才能构建。

## Building on Windows

### 安装

* `choco install git`
* `choco install node`
* `choco install llvm`（或者从 Visual Studio 2019 安装程序中安装 `C++ Clang 工具` 可选包）
* `choco install make`（或者如果你已经安装了 MinGW，它已经包含在其中）

1. 确保 `Clang` 和 `make` 已添加到系统路径中。
2. 使用 Git Bash，将存储库克隆到您喜欢的位置。
3. 切换到克隆的目录并运行 `npm ci`。
5. 运行 `make`。
6. 您的 `repo/build` 目录现在应该包含 `libllhttp.a` 和 `libllhttp.so` 静态和动态库。
7. 在构建可执行文件时，您可以链接到这些库。确保在构建时将构建文件夹设置为包含路径，以便您可以引用 `repo/build/llhttp.h` 中的声明。

### 链接库的简单示例：

假设您在当前工作目录中有一个可执行文件 `main.cpp`，您可以运行：`clang++ -Os -g3 -Wall -Wextra -Wno-unused-parameter -I/path/to/llhttp/build main.cpp /path/to/llhttp/build/libllhttp.a -o main.exe`。

如果您遇到 `unresolved external symbol` 链接器错误，很可能是您试图构建 `llhttp.c`，但没有将其与来自 `api.c` 和 `http.c` 的对象文件链接在一起。

#### 许可证

本软件根据 MIT 许可证获得许可。

版权所有 Fedor Indutny，2018。

特此授予任何获得本软件及相关文档文件（以下简称"软件"）副本的人免费许可，以处理本软件而无需限制，包括但不限于使用、复制、修改、合并、发布、分发、再许可和/或销售软件的副本，以及允许使用本软件的人员这样做，但需遵守以下条件：

在所有副本或重要部分的软件中必须包含上述版权声明和本许可声明。

本软件按"原样"提供，不提供任何形式的明示或暗示担保，包括但不限于适销性、特定用途适用性和非侵权性的保证。在任何情况下，作者或版权持有人对任何索赔、损害或其他责任不承担责任，无论是在合同诉讼、侵权行为或其他情况下，都与软件或软件的使用或其他交易有关。

[0]: https://github.com/nodejs/http-parser
[1]: https://github.com/nodejs/llparse
[2]: https://en.wikipedia.org/wiki/Register_allocation#Spilling
[3]: https://en.wikipedia.org/wiki/Tail_call
[4]: https://llvm.org/docs/LangRef.html
[5]: https://llvm.org/docs/LangRef.html#call-instruction
[6]: https://clang.llvm.org/
[7]: https://github.com/nodejs/node
[8]: https://github.com/pallas/pyllhttp
[9]: https://github.com/metabahn/llhttp
[10]: https://github.com/JackLiar/rust-llhttp
[11]: https://github.com/MunifTanjim/llhttp.lua
