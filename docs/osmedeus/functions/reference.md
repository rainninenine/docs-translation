> ## Documentation Index
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# Functions Reference

按类别组织的190+个实用函数的完整参考。

## File Functions

文件和目录操作。

### file\_exists(path)

检查文件是否存在。

```javascript theme={null}
file_exists("/path/to/file.txt")  // 返回：true 或 false
```

### file\_length(path)

统计文件中的非空行数。

```javascript theme={null}
file_length("{{Output}}/hosts.txt")  // 返回：42
```

### dir\_length(path)

统计目录中的条目数。

```javascript theme={null}
dir_length("{{Output}}/screenshots")  // 返回：15
```

### file\_contains(path, pattern)

检查文件是否包含指定模式。

```javascript theme={null}
file_contains("{{Output}}/urls.txt", "admin")  // 返回：true 或 false
```

### regex\_extract(path, pattern)

从文件中提取匹配正则表达式的行。

```javascript theme={null}
regex_extract("{{Output}}/urls.txt", ".*api.*")  // 返回：["https://api.example.com", ...]
```

### read\_file(path)

读取整个文件内容。

```javascript theme={null}
read_file("{{Output}}/config.json")  // 返回：文件内容（字符串）
```

### read\_lines(path)

将文件读取为行数组。

```javascript theme={null}
read_lines("{{Output}}/subdomains.txt")  // 返回：["sub1.example.com", "sub2.example.com", ...]
```

### remove\_file(path)

删除文件。

```javascript theme={null}
remove_file("{{Output}}/temp.txt")  // 返回：true 或 false
```

### remove\_folder(path)

递归删除文件夹。

```javascript theme={null}
remove_folder("{{Output}}/cache")  // 返回：true 或 false
```

### rm\_rf(path)

递归删除文件或文件夹（类似 `rm -rf`）。

```javascript theme={null}
rm_rf("{{Output}}/tmp")  // 返回：true 或 false
```

### remove\_all\_except(folder, keep\_file)

删除文件夹下除指定文件外的所有内容。

```javascript theme={null}
remove_all_except("{{Output}}", "{{Output}}/keep.txt")  // 返回：true 或 false
```

### create\_folder(path)

递归创建文件夹（类似 `mkdir -p`）。

```javascript theme={null}
create_folder("{{Output}}/new-folder")  // 返回：true 或 false
```

### append\_file(dest, source)

将源文件内容追加到目标文件。

```javascript theme={null}
append_file("{{Output}}/all.txt", "{{Output}}/part.txt")  // 返回：true 或 false
```

### move\_file(source, dest)

将文件从源路径移动到目标路径。

```javascript theme={null}
move_file("{{Output}}/raw.txt", "{{Output}}/processed.txt")  // 返回：true 或 false
```

### glob(pattern)

列出匹配 glob 模式的文件名。

```javascript theme={null}
glob("{{Output}}/*.txt")  // 返回：["file1.txt", "file2.txt", ...]
```

### grep\_string\_to\_file(dest, source, str)

将包含指定字符串的行写入目标文件。

```javascript theme={null}
grep_string_to_file("{{Output}}/admin-urls.txt", "{{Output}}/urls.txt", "admin")  // 返回：true 或 false
```

### grep\_regex\_to\_file(dest, source, pattern)

将匹配正则表达式的行写入目标文件。

```javascript theme={null}
grep_regex_to_file("{{Output}}/api-urls.txt", "{{Output}}/urls.txt", ".*api.*")  // 返回：true 或 false
```

### grep\_string(source, str)

返回包含指定字符串的行。

```javascript theme={null}
grep_string("{{Output}}/urls.txt", "admin")  // 返回："https://example.com/admin\nhttps://example.com/admin/login"
```

### grep\_regex(source, pattern)

返回匹配正则表达式的行。

```javascript theme={null}
grep_regex("{{Output}}/urls.txt", ".*api.*")  // 返回：匹配行（字符串）
```

### remove\_blank\_lines(path)

原地删除文件中的空行。

```javascript theme={null}
remove_blank_lines("{{Output}}/urls.txt")  // 返回：true 或 false
```

### chunk\_file(input, lines\_per\_chunk, output)

将文件分割成每 N 行的块，写入编号的输出文件。

```javascript theme={null}
chunk_file("{{Output}}/urls.txt", 1000, "{{Output}}/chunks/urls")
// 创建：urls-0.txt, urls-1.txt, ...
// 返回：true 或 false
```

### cut\_to\_file(input\_file, delim, field, output\_file)

使用分隔符从每行中提取指定字段，并将结果写入文件。

```javascript theme={null}
cut_to_file("{{Output}}/data.csv", ",", 2, "{{Output}}/column2.txt")  // 返回：true 或 false
```

## String Functions

字符串操作函数。

### trim(str)

去除首尾空白字符。

```javascript theme={null}
trim("  hello world  ")  // 返回："hello world"
```

### split(str, delim)

按分隔符将字符串分割为数组。

```javascript theme={null}
split("a,b,c", ",")  // 返回：["a", "b", "c"]
```

### join(arr, delim)

用分隔符连接数组元素。

```javascript theme={null}
join(["a", "b", "c"], "-")  // 返回："a-b-c"
```

### replace(str, old, new)

将所有出现的 old 替换为 new。

```javascript theme={null}
replace("hello world", "world", "there")  // 返回："hello there"
```

### contains(str, substr)

检查字符串是否包含子串。

```javascript theme={null}
contains("hello world", "world")  // 返回：true
```

### starts\_with(str, prefix)

检查字符串是否以指定前缀开头。

```javascript theme={null}
starts_with("hello", "hel")  // 返回：true
```

### ends\_with(str, suffix)

检查字符串是否以指定后缀结尾。

```javascript theme={null}
ends_with("hello.txt", ".txt")  // 返回：true
```

### to\_lower\_case(str)

转换为小写。

```javascript theme={null}
to_lower_case("HELLO")  // 返回："hello"
```

### to\_upper\_case(str)

转换为大写。

```javascript theme={null}
to_upper_case("hello")  // 返回："HELLO"
```

### match(str, pattern)

检查字符串是否匹配正则表达式模式。

```javascript theme={null}
match("test123", "[0-9]+")  // 返回：true
```

### regex\_match(pattern, str)

检查字符串是否匹配正则表达式（模式在前）。

```javascript theme={null}
regex_match("[0-9]+", "test123")  // 返回：true
```

### cut\_with\_delim(input, delim, field)

按分隔符提取字段（从1开始索引，类似 `cut`）。

```javascript theme={null}
cut_with_delim("a:b:c", ":", 2)  // 返回："b"
```

### normalize\_path(input)

将特殊字符（/ | : 等）替换为下划线。

```javascript theme={null}
normalize_path("test/path:file")  // 返回："test_path_file"
```

### normal\_path(input)

规范化为路径友好格式（与 `{{TargetSpace}}` 相同）。

```javascript theme={null}
normal_path("https://example.com/path")  // 返回："example.com_path"
```

### clean\_sub(path, target?)

清理并去重文件中的子域名，可选地按目标域名过滤。

```javascript theme={null}
clean_sub("{{Output}}/subdomains.txt", "example.com")  // 返回：true 或 false
```

### trim\_left(input, substring)

从字符串左侧/开头修剪子串。

```javascript theme={null}
trim_left("https://example.com", "https://")  // 返回："example.com"
```

### trim\_right(input, substring)

从字符串右侧/结尾修剪子串。

```javascript theme={null}
trim_right("example.com/", "/")  // 返回："example.com"
```

### trim\_string(input, substring)

从字符串两端修剪子串。

```javascript theme={null}
trim_string("---hello---", "---")  // 返回："hello"
```

### cut\_space(input, field)

按空白字符分割字符串并提取字段（从1开始索引）。

```javascript theme={null}
cut_space("hello world foo", 2)  // 返回："world"
```

### get\_target\_space(input)

清理并截断输入，用作项目空间安全的目标名称（与 `{{TargetSpace}}` 相同）。

```javascript theme={null}
get_target_space("https://example.com/path")  // 返回："example.com_path"
```

### pick\_valid(v1, v2, ..., v10)

从给定参数中返回第一个非空值（最多10个）。

```javascript theme={null}
pick_valid("", "", "fallback")  // 返回："fallback"
pick_valid(target, "default.com")  // 返回：target 若非空，否则 "default.com"
```

**别名：** `cut` 是 `cut_with_delim` 的别名。`bash` 是 `exec_cmd` 的别名。

## Type Conversion Functions

数据类型转换函数。

### parse\_int(str)

将字符串解析为整数。

```javascript theme={null}
parse_int("42")  // 返回：42
```

### parse\_float(str)

将字符串解析为浮点数。

```javascript theme={null}
parse_float("3.14")  // 返回：3.14
```

### to\_string(val)

将值转换为字符串。

```javascript theme={null}
to_string(123)  // 返回："123"
```

### to\_boolean(val)

将值转换为布尔值。

```javascript theme={null}
to_boolean("true")  // 返回：true
to_boolean(1)       // 返回：true
```

## Type Detection Functions

输入类型检测函数。

### get\_types(input)

检测输入类型：file、folder、cidr、ip、url、domain 或 string。

```javascript theme={null}
get_types("192.168.1.0/24")     // 返回："cidr"
get_types("example.com")        // 返回："domain"
get_types("https://example.com") // 返回："url"
get_types("/etc/passwd")        // 返回："file"
```

### is\_file(path)

检查路径是否为现有文件。

```javascript theme={null}
is_file("/tmp/data.txt")  // 返回：true 或 false
```

### is\_dir(path)

检查路径是否为现有目录。

```javascript theme={null}
is_dir("/tmp/output")  // 返回：true 或 false
```

### is\_git(path)

检查路径是否在 git 仓库内。

```javascript theme={null}
is_git("/path/to/project")  // 返回：true 或 false
```

### is\_url(input)

检查输入是否为有效的 URL。

```javascript theme={null}
is_url("https://example.com")  // 返回：true
is_url("not-a-url")            // 返回：false
```

### is\_compress(path)

检查路径是否为压缩归档文件（.zip、.tar.gz、.tgz 等）。

```javascript theme={null}
is_compress("archive.tar.gz")  // 返回：true
is_compress("file.txt")        // 返回：false
```

### detect\_language(path)

检测源代码文件夹的主要编程语言（支持26+种语言）。

```javascript theme={null}
detect_language("/path/to/project")  // 返回："javascript"
detect_language("{{Output}}/repo")   // 返回："python"
```

## Utility Functions

通用工具函数。

### len(val)

获取字符串或数组的长度。

```javascript theme={null}
len("hello")   // 返回：5
len([1, 2, 3]) // 返回：3
```

### is\_empty(val)

检查值是否为空。

```javascript theme={null}
is_empty("")      // 返回：true
is_empty("hello") // 返回：false
```

### is\_not\_empty(val)

检查值是否不为空。

```javascript theme={null}
is_not_empty("test")  // 返回：true
is_not_empty("")      // 返回：false
```

### printf(message)

将消息打印到标准输出。

```javascript theme={null}
printf("Scan started for " + target)
```

### cat\_file(path)

将文件内容打印到标准输出。

```javascript theme={null}
cat_file("{{Output}}/results.txt")
```

### exit(code)

以指定代码退出扫描。

```javascript theme={null}
exit(0)  // 成功
exit(1)  // 错误
```

### exec\_cmd(command)

执行 bash 命令并返回输出。

```javascript theme={null}
exec_cmd("whoami")  // 返回："root"
exec_cmd("date")    // 返回："Mon Jan 20 10:30:00 UTC 2025"
```

### sleep(seconds)

暂停执行 n 秒。

```javascript theme={null}
sleep(5)  // 暂停5秒
```

### command\_exists(command)

检查命令是否存在于 PATH 中。

```javascript theme={null}
command_exists("nmap")   // 返回：true 或 false
command_exists("nuclei") // 返回：true 或 false
```

## Logging Functions

带级别前缀的日志消息函数。

### log\_debug(message)

以 `[DEBUG]` 前缀记录调试消息。

```javascript theme={null}
log_debug("Processing target: " + target)
```

### log\_info(message)

以 `[INFO]` 前缀记录信息消息。

```javascript theme={null}
log_info("Scan completed successfully")
```

### log\_warn(message)

以 `[WARN]` 前缀记录警告消息。

```javascript theme={null}
log_warn("Rate limit approaching")
```

### log\_error(message)

以 `[ERROR]` 前缀记录错误消息。

```javascript theme={null}
log_error("Failed to connect to target")
```

## Color Printing Functions

带颜色输出的打印消息函数。

### print\_green(message)

以绿色打印消息。

```javascript theme={null}
print_green("Success!")
```

### print\_blue(message)

以蓝色打印消息。

```javascript theme={null}
print_blue("Processing {{Target}}")
```

### print\_yellow(message)

以黄色打印消息。

```javascript theme={null}
print_yellow("Warning: Rate limit hit")
```

### print\_red(message)

以红色打印消息。

```javascript theme={null}
print_red("Error occurred")
```

## Runtime Variable Functions

运行时设置和获取变量。

### set\_var(name, value)

设置运行时变量以供后续检索。

```javascript theme={null}
set_var("api_url", "https://api.example.com")
```

### get\_var(name)

获取运行时变量的值。

```javascript theme={null}
get_var("api_url")  // 返回："https://api.example.com"
```

## HTTP Functions

HTTP 请求和网络操作函数。

### http\_request(url, method, headers, body)

完全控制地发起 HTTP 请求。

```javascript theme={null}
http_request("https://api.example.com/data", "POST",
  {"Authorization": "Bearer token", "Content-Type": "application/json"},
  '{"key":"value"}')
// 返回：{statusCode: 200, body: "...", headers: {...}}
```

### http\_get(url)

HTTP GET 请求，返回结构化响应。

```javascript theme={null}
http_get("https://api.example.com/data")
// 返回：{statusCode: 200, body: "...", headers: {...}}
```

### http\_post(url, body)

HTTP POST 请求，返回结构化响应。

```javascript theme={null}
http_post("https://api.example.com/submit", '{"key":"value"}')
// 返回：{statusCode: 200, body: "...", headers: {...}}
```

### get\_ip(domain\_or\_url)

将域名或 URL 解析为 IP 地址。

```javascript theme={null}
get_ip("example.com")              // 返回："93.184.216.34"
get_ip("https://example.com/path") // 返回："93.184.216.34"（自动解析 URL）
```

## Generation Functions

生成随机值。

### random\_string(length)

生成随机字母数字字符串。

```javascript theme={null}
random_string(16)  // 返回："aB3xY9kLm2nP7qRs"
```

### uuid()

生成 UUID v4。

```javascript theme={null}
uuid()  // 返回："550e8400-e29b-41d4-a716-446655440000"
```

## Encoding Functions

编码和解码数据。

### base64\_encode(str)

将字符串编码为 base64。

```javascript theme={null}
base64_encode("hello")  // 返回："aGVsbG8="
```

### base64\_decode(str)

解码 base64 字符串。

```javascript theme={null}
base64_decode("aGVsbG8=")  // 返回："hello"
```

## Data Query Functions

查询结构化数据。

### jq(jsonData, query)

使用 jq 语法提取数据。

```javascript theme={null}
jq('{"name":"test","version":"1.0"}', '.name')  // 返回："test"
jq('{"items":[1,2,3]}', '.items[]')             // 返回：[1, 2, 3]
jq('{"a":{"b":"c"}}', '.a.b')                   // 返回："c"
```

### jq\_from\_file(path, query)

从 JSON 文件中使用 jq 提取数据。

```javascript theme={null}
jq_from_file("{{Output}}/data.json", ".results[].url")
```

## Notification Functions

通过多种渠道发送通知。

### notify\_telegram(message)

向配置的 Telegram 聊天发送消息。

```javascript theme={null}
notify_telegram("Scan completed for {{Target}}")  // 返回：true 或 false
```

### send\_telegram\_file(path, caption?)

向 Telegram 发送文件，可选标题。

```javascript theme={null}
send_telegram_file("{{Output}}/report.pdf", "Scan report for {{Target}}")  // 返回：true 或 false
```

### notify\_webhook(message)

向所有配置的 webhook 发送消息。

```javascript theme={null}
notify_webhook("Scan completed for {{Target}}")  // 返回：true 或 false
```

### send\_webhook\_event(eventType, data)

向所有 webhook 发送结构化事件。

```javascript theme={null}
send_webhook_event("scan_complete", {target: "{{Target}}", status: "success"})  // 返回：true 或 false
```

### notify\_telegram\_channel(channel, message)

向特定 Telegram 频道发送消息。

```javascript theme={null}
notify_telegram_channel("alerts", "Critical finding on {{Target}}")  // 返回：true 或 false
```

### send\_telegram\_file\_channel(channel, path, caption?)

向特定 Telegram 频道发送文件，可选标题。

```javascript theme={null}
send_telegram_file_channel("reports", "{{Output}}/report.pdf", "Report for {{Target}}")  // 返回：true 或 false
```

### notify\_message\_as\_file\_telegram(path)

将文件内容作为文本文件发送到 Telegram。

```javascript theme={null}
notify_message_as_file_telegram("{{Output}}/summary.txt")  // 返回：true 或 false
```

### notify\_message\_as\_file\_telegram\_channel(channel, path)

将文件内容作为文本文件发送到特定 Telegram 频道。

```javascript theme={null}
notify_message_as_file_telegram_channel("reports", "{{Output}}/summary.txt")  // 返回：true 或 false
```

## Event Generation Functions

为事件系统生成结构化事件。

### generate\_event(workspace, topic, source, data\_type, data)

生成结构化事件并发送到服务器/webhook。

```javascript theme={null}
// 简单字符串数据
generate_event("{{Workspace}}", "discovery", "subdomain-scan", "domain", "api.example.com")

// 复杂对象数据
generate_event("{{Workspace}}", "vulnerability", "nuclei", "finding", {
  url: "https://example.com/admin",
  severity: "critical",
  template: "CVE-2024-1234"
})
// 返回：true（始终为 true - 如果服务器不可用，事件将排队）
```

**参数：**

* `workspace` - 项目空间名称（必填，使用 `{{Workspace}}`）
* `topic` - 事件类别（例如 "discovery"、"vulnerability"）
* `source` - 事件来源（例如 "amass"、"nuclei"）
* `data_type` - 数据类型（例如 "domain"、"url"、"finding"）
* `data` - T