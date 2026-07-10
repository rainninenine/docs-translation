> ## Documentation Index
> Fetch the complete documentation index at: https://docs.osmedeus.org/llms.txt
> Use this file to discover all available pages before exploring further.

# Functions Reference

Complete reference for all 190+ utility functions organized by category.

## File Functions

Operations on files and directories.

### file\_exists(path)

Check if a file exists.

```javascript theme={null}
file_exists("/path/to/file.txt")  // Returns: true or false
```

### file\_length(path)

Count non-empty lines in a file.

```javascript theme={null}
file_length("{{Output}}/hosts.txt")  // Returns: 42
```

### dir\_length(path)

Count entries in a directory.

```javascript theme={null}
dir_length("{{Output}}/screenshots")  // Returns: 15
```

### file\_contains(path, pattern)

Check if file contains a pattern.

```javascript theme={null}
file_contains("{{Output}}/urls.txt", "admin")  // Returns: true or false
```

### regex\_extract(path, pattern)

Extract lines matching a regex from file.

```javascript theme={null}
regex_extract("{{Output}}/urls.txt", ".*api.*")  // Returns: ["https://api.example.com", ...]
```

### read\_file(path)

Read entire file contents.

```javascript theme={null}
read_file("{{Output}}/config.json")  // Returns: file contents as string
```

### read\_lines(path)

Read file as array of lines.

```javascript theme={null}
read_lines("{{Output}}/subdomains.txt")  // Returns: ["sub1.example.com", "sub2.example.com", ...]
```

### remove\_file(path)

Delete a file.

```javascript theme={null}
remove_file("{{Output}}/temp.txt")  // Returns: true or false
```

### remove\_folder(path)

Delete folder recursively.

```javascript theme={null}
remove_folder("{{Output}}/cache")  // Returns: true or false
```

### rm\_rf(path)

Delete file or folder recursively (like `rm -rf`).

```javascript theme={null}
rm_rf("{{Output}}/tmp")  // Returns: true or false
```

### remove\_all\_except(folder, keep\_file)

Remove everything under folder except the specified file.

```javascript theme={null}
remove_all_except("{{Output}}", "{{Output}}/keep.txt")  // Returns: true or false
```

### create\_folder(path)

Create folder recursively (like `mkdir -p`).

```javascript theme={null}
create_folder("{{Output}}/new-folder")  // Returns: true or false
```

### append\_file(dest, source)

Append source file content to destination file.

```javascript theme={null}
append_file("{{Output}}/all.txt", "{{Output}}/part.txt")  // Returns: true or false
```

### move\_file(source, dest)

Move file from source to destination.

```javascript theme={null}
move_file("{{Output}}/raw.txt", "{{Output}}/processed.txt")  // Returns: true or false
```

### glob(pattern)

List filenames matching glob pattern.

```javascript theme={null}
glob("{{Output}}/*.txt")  // Returns: ["file1.txt", "file2.txt", ...]
```

### grep\_string\_to\_file(dest, source, str)

Write lines containing string to destination file.

```javascript theme={null}
grep_string_to_file("{{Output}}/admin-urls.txt", "{{Output}}/urls.txt", "admin")  // Returns: true or false
```

### grep\_regex\_to\_file(dest, source, pattern)

Write lines matching regex to destination file.

```javascript theme={null}
grep_regex_to_file("{{Output}}/api-urls.txt", "{{Output}}/urls.txt", ".*api.*")  // Returns: true or false
```

### grep\_string(source, str)

Return lines containing string.

```javascript theme={null}
grep_string("{{Output}}/urls.txt", "admin")  // Returns: "https://example.com/admin\nhttps://example.com/admin/login"
```

### grep\_regex(source, pattern)

Return lines matching regex.

```javascript theme={null}
grep_regex("{{Output}}/urls.txt", ".*api.*")  // Returns: matching lines as string
```

### remove\_blank\_lines(path)

Remove blank lines from file in-place.

```javascript theme={null}
remove_blank_lines("{{Output}}/urls.txt")  // Returns: true or false
```

### chunk\_file(input, lines\_per\_chunk, output)

Split a file into chunks of N lines each, writing numbered output files.

```javascript theme={null}
chunk_file("{{Output}}/urls.txt", 1000, "{{Output}}/chunks/urls")
// Creates: urls-0.txt, urls-1.txt, ...
// Returns: true or false
```

### cut\_to\_file(input\_file, delim, field, output\_file)

Extract a specific field from each line using a delimiter and write results to a file.

```javascript theme={null}
cut_to_file("{{Output}}/data.csv", ",", 2, "{{Output}}/column2.txt")  // Returns: true or false
```

## String Functions

String manipulation operations.

### trim(str)

Remove leading/trailing whitespace.

```javascript theme={null}
trim("  hello world  ")  // Returns: "hello world"
```

### split(str, delim)

Split string by delimiter into array.

```javascript theme={null}
split("a,b,c", ",")  // Returns: ["a", "b", "c"]
```

### join(arr, delim)

Join array elements with delimiter.

```javascript theme={null}
join(["a", "b", "c"], "-")  // Returns: "a-b-c"
```

### replace(str, old, new)

Replace all occurrences of old with new.

```javascript theme={null}
replace("hello world", "world", "there")  // Returns: "hello there"
```

### contains(str, substr)

Check if string contains substring.

```javascript theme={null}
contains("hello world", "world")  // Returns: true
```

### starts\_with(str, prefix)

Check if string starts with prefix.

```javascript theme={null}
starts_with("hello", "hel")  // Returns: true
```

### ends\_with(str, suffix)

Check if string ends with suffix.

```javascript theme={null}
ends_with("hello.txt", ".txt")  // Returns: true
```

### to\_lower\_case(str)

Convert to lowercase.

```javascript theme={null}
to_lower_case("HELLO")  // Returns: "hello"
```

### to\_upper\_case(str)

Convert to uppercase.

```javascript theme={null}
to_upper_case("hello")  // Returns: "HELLO"
```

### match(str, pattern)

Check if string matches regex pattern.

```javascript theme={null}
match("test123", "[0-9]+")  // Returns: true
```

### regex\_match(pattern, str)

Check if string matches regex (pattern first).

```javascript theme={null}
regex_match("[0-9]+", "test123")  // Returns: true
```

### cut\_with\_delim(input, delim, field)

Extract field by delimiter (1-indexed, like `cut`).

```javascript theme={null}
cut_with_delim("a:b:c", ":", 2)  // Returns: "b"
```

### normalize\_path(input)

Replace special characters (/ | : etc.) with underscore.

```javascript theme={null}
normalize_path("test/path:file")  // Returns: "test_path_file"
```

### normal\_path(input)

Normalize to path-friendly format (same as `{{TargetSpace}}`).

```javascript theme={null}
normal_path("https://example.com/path")  // Returns: "example.com_path"
```

### clean\_sub(path, target?)

Clean and deduplicate subdomains in file, optionally filter by target domain.

```javascript theme={null}
clean_sub("{{Output}}/subdomains.txt", "example.com")  // Returns: true or false
```

### trim\_left(input, substring)

Trim a substring from the left/start of a string.

```javascript theme={null}
trim_left("https://example.com", "https://")  // Returns: "example.com"
```

### trim\_right(input, substring)

Trim a substring from the right/end of a string.

```javascript theme={null}
trim_right("example.com/", "/")  // Returns: "example.com"
```

### trim\_string(input, substring)

Trim a substring from both ends of a string.

```javascript theme={null}
trim_string("---hello---", "---")  // Returns: "hello"
```

### cut\_space(input, field)

Split string by whitespace and extract field (1-indexed).

```javascript theme={null}
cut_space("hello world foo", 2)  // Returns: "world"
```

### get\_target\_space(input)

Sanitize and truncate input for use as a workspace-safe target name (same as `{{TargetSpace}}`).

```javascript theme={null}
get_target_space("https://example.com/path")  // Returns: "example.com_path"
```

### pick\_valid(v1, v2, ..., v10)

Return the first non-empty value from the given arguments (up to 10).

```javascript theme={null}
pick_valid("", "", "fallback")  // Returns: "fallback"
pick_valid(target, "default.com")  // Returns: target if non-empty, else "default.com"
```

**Aliases:** `cut` is an alias for `cut_with_delim`. `bash` is an alias for `exec_cmd`.

## Type Conversion Functions

Convert between data types.

### parse\_int(str)

Parse string to integer.

```javascript theme={null}
parse_int("42")  // Returns: 42
```

### parse\_float(str)

Parse string to float.

```javascript theme={null}
parse_float("3.14")  // Returns: 3.14
```

### to\_string(val)

Convert value to string.

```javascript theme={null}
to_string(123)  // Returns: "123"
```

### to\_boolean(val)

Convert value to boolean.

```javascript theme={null}
to_boolean("true")  // Returns: true
to_boolean(1)       // Returns: true
```

## Type Detection Functions

Detect input types.

### get\_types(input)

Detect input type: file, folder, cidr, ip, url, domain, or string.

```javascript theme={null}
get_types("192.168.1.0/24")     // Returns: "cidr"
get_types("example.com")        // Returns: "domain"
get_types("https://example.com") // Returns: "url"
get_types("/etc/passwd")        // Returns: "file"
```

### is\_file(path)

Check if path is an existing file.

```javascript theme={null}
is_file("/tmp/data.txt")  // Returns: true or false
```

### is\_dir(path)

Check if path is an existing directory.

```javascript theme={null}
is_dir("/tmp/output")  // Returns: true or false
```

### is\_git(path)

Check if path is inside a git repository.

```javascript theme={null}
is_git("/path/to/project")  // Returns: true or false
```

### is\_url(input)

Check if input is a valid URL.

```javascript theme={null}
is_url("https://example.com")  // Returns: true
is_url("not-a-url")            // Returns: false
```

### is\_compress(path)

Check if path is a compressed archive file (.zip, .tar.gz, .tgz, etc.).

```javascript theme={null}
is_compress("archive.tar.gz")  // Returns: true
is_compress("file.txt")        // Returns: false
```

### detect\_language(path)

Detect the dominant programming language of a source folder (supports 26+ languages).

```javascript theme={null}
detect_language("/path/to/project")  // Returns: "javascript"
detect_language("{{Output}}/repo")   // Returns: "python"
```

## Utility Functions

General utility operations.

### len(val)

Get length of string or array.

```javascript theme={null}
len("hello")   // Returns: 5
len([1, 2, 3]) // Returns: 3
```

### is\_empty(val)

Check if value is empty.

```javascript theme={null}
is_empty("")      // Returns: true
is_empty("hello") // Returns: false
```

### is\_not\_empty(val)

Check if value is not empty.

```javascript theme={null}
is_not_empty("test")  // Returns: true
is_not_empty("")      // Returns: false
```

### printf(message)

Print message to stdout.

```javascript theme={null}
printf("Scan started for " + target)
```

### cat\_file(path)

Print file contents to stdout.

```javascript theme={null}
cat_file("{{Output}}/results.txt")
```

### exit(code)

Exit the scan with specified code.

```javascript theme={null}
exit(0)  // Success
exit(1)  // Error
```

### exec\_cmd(command)

Execute bash command and return output.

```javascript theme={null}
exec_cmd("whoami")  // Returns: "root"
exec_cmd("date")    // Returns: "Mon Jan 20 10:30:00 UTC 2025"
```

### sleep(seconds)

Pause execution for n seconds.

```javascript theme={null}
sleep(5)  // Pause for 5 seconds
```

### command\_exists(command)

Check if command exists in PATH.

```javascript theme={null}
command_exists("nmap")   // Returns: true or false
command_exists("nuclei") // Returns: true or false
```

## Logging Functions

Log messages with level prefixes.

### log\_debug(message)

Log debug message with `[DEBUG]` prefix.

```javascript theme={null}
log_debug("Processing target: " + target)
```

### log\_info(message)

Log info message with `[INFO]` prefix.

```javascript theme={null}
log_info("Scan completed successfully")
```

### log\_warn(message)

Log warning message with `[WARN]` prefix.

```javascript theme={null}
log_warn("Rate limit approaching")
```

### log\_error(message)

Log error message with `[ERROR]` prefix.

```javascript theme={null}
log_error("Failed to connect to target")
```

## Color Printing Functions

Print messages with colored output.

### print\_green(message)

Print message in green color.

```javascript theme={null}
print_green("Success!")
```

### print\_blue(message)

Print message in blue color.

```javascript theme={null}
print_blue("Processing {{Target}}")
```

### print\_yellow(message)

Print message in yellow color.

```javascript theme={null}
print_yellow("Warning: Rate limit hit")
```

### print\_red(message)

Print message in red color.

```javascript theme={null}
print_red("Error occurred")
```

## Runtime Variable Functions

Set and get variables at runtime.

### set\_var(name, value)

Set a runtime variable for later retrieval.

```javascript theme={null}
set_var("api_url", "https://api.example.com")
```

### get\_var(name)

Get a runtime variable value.

```javascript theme={null}
get_var("api_url")  // Returns: "https://api.example.com"
```

## HTTP Functions

HTTP requests and network operations.

### http\_request(url, method, headers, body)

Make HTTP request with full control.

```javascript theme={null}
http_request("https://api.example.com/data", "POST",
  {"Authorization": "Bearer token", "Content-Type": "application/json"},
  '{"key":"value"}')
// Returns: {statusCode: 200, body: "...", headers: {...}}
```

### http\_get(url)

HTTP GET request with structured response.

```javascript theme={null}
http_get("https://api.example.com/data")
// Returns: {statusCode: 200, body: "...", headers: {...}}
```

### http\_post(url, body)

HTTP POST request with structured response.

```javascript theme={null}
http_post("https://api.example.com/submit", '{"key":"value"}')
// Returns: {statusCode: 200, body: "...", headers: {...}}
```

### get\_ip(domain\_or\_url)

Resolve domain or URL to IP address.

```javascript theme={null}
get_ip("example.com")              // Returns: "93.184.216.34"
get_ip("https://example.com/path") // Returns: "93.184.216.34" (auto-parses URL)
```

## Generation Functions

Generate random values.

### random\_string(length)

Generate random alphanumeric string.

```javascript theme={null}
random_string(16)  // Returns: "aB3xY9kLm2nP7qRs"
```

### uuid()

Generate UUID v4.

```javascript theme={null}
uuid()  // Returns: "550e8400-e29b-41d4-a716-446655440000"
```

## Encoding Functions

Encode and decode data.

### base64\_encode(str)

Encode string to base64.

```javascript theme={null}
base64_encode("hello")  // Returns: "aGVsbG8="
```

### base64\_decode(str)

Decode base64 string.

```javascript theme={null}
base64_decode("aGVsbG8=")  // Returns: "hello"
```

## Data Query Functions

Query structured data.

### jq(jsonData, query)

Extract data using jq syntax.

```javascript theme={null}
jq('{"name":"test","version":"1.0"}', '.name')  // Returns: "test"
jq('{"items":[1,2,3]}', '.items[]')             // Returns: [1, 2, 3]
jq('{"a":{"b":"c"}}', '.a.b')                   // Returns: "c"
```

### jq\_from\_file(path, query)

Extract data using jq from JSON file.

```javascript theme={null}
jq_from_file("{{Output}}/data.json", ".results[].url")
```

## Notification Functions

Send notifications via various channels.

### notify\_telegram(message)

Send message to configured Telegram chat.

```javascript theme={null}
notify_telegram("Scan completed for {{Target}}")  // Returns: true or false
```

### send\_telegram\_file(path, caption?)

Send file to Telegram with optional caption.

```javascript theme={null}
send_telegram_file("{{Output}}/report.pdf", "Scan report for {{Target}}")  // Returns: true or false
```

### notify\_webhook(message)

Send message to all configured webhooks.

```javascript theme={null}
notify_webhook("Scan completed for {{Target}}")  // Returns: true or false
```

### send\_webhook\_event(eventType, data)

Send structured event to all webhooks.

```javascript theme={null}
send_webhook_event("scan_complete", {target: "{{Target}}", status: "success"})  // Returns: true or false
```

### notify\_telegram\_channel(channel, message)

Send message to a specific Telegram channel.

```javascript theme={null}
notify_telegram_channel("alerts", "Critical finding on {{Target}}")  // Returns: true or false
```

### send\_telegram\_file\_channel(channel, path, caption?)

Send file to a specific Telegram channel with optional caption.

```javascript theme={null}
send_telegram_file_channel("reports", "{{Output}}/report.pdf", "Report for {{Target}}")  // Returns: true or false
```

### notify\_message\_as\_file\_telegram(path)

Send file content as a text file to Telegram.

```javascript theme={null}
notify_message_as_file_telegram("{{Output}}/summary.txt")  // Returns: true or false
```

### notify\_message\_as\_file\_telegram\_channel(channel, path)

Send file content as a text file to a specific Telegram channel.

```javascript theme={null}
notify_message_as_file_telegram_channel("reports", "{{Output}}/summary.txt")  // Returns: true or false
```

## Event Generation Functions

Generate structured events for the event system.

### generate\_event(workspace, topic, source, data\_type, data)

Generate a structured event and send to server/webhooks.

```javascript theme={null}
// Simple string data
generate_event("{{Workspace}}", "discovery", "subdomain-scan", "domain", "api.example.com")

// Complex object data
generate_event("{{Workspace}}", "vulnerability", "nuclei", "finding", {
  url: "https://example.com/admin",
  severity: "critical",
  template: "CVE-2024-1234"
})
// Returns: true (always true - events are queued if server unavailable)
```

**Parameters:**

* `workspace` - Workspace name (required, use `{{Workspace}}`)
* `topic` - Event category (e.g., "discovery", "vulnerability")
* `source` - Origin of the event (e.g., "amass", "nuclei")
* `data_type` - Type of data (e.g., "domain", "url", "finding")
* `data` - The actual data payload (string or object)

**Event Payload Structure:**

```json theme={null}
{
  "workspace": "example.com",
  "topic": "discovery",
  "source": "amass",
  "data_type": "subdomain",
  "data": "api.example.com",
  "run_id": "abc123",
  "workflow_name": "recon",
  "timestamp": "2025-01-15T10:30:00Z"
}
```

### generate\_event\_from\_file(workspace, topic, source, data\_type, path)

Read a file and generate an event for each non-empty line.

```javascript theme={null}
generate_event_from_file("{{Workspace}}", "discovery", "amass", "subdomain", "{{Output}}/subdomains.txt")
// Returns: 42 (count of events generated)
```

**Parameters:**

* `workspace` - Workspace name (required)
* `topic` - Event category
* `source` - Origin of the event
* `data_type` - Type of data
* `path` - Path to file (one item per line)

## CDN/Storage Functions

Cloud storage operations (S3-compatible).

### cdn\_upload(localPath, remotePath)

Upload file to cloud storage.

```javascript theme={null}
cdn_upload("{{Output}}/report.zip", "scans/{{Target}}/report.zip")  // Returns: true or false
```

### cdn\_download(remotePath, localPath)

Download file from cloud storage.

```javascript theme={null}
cdn_download("wordlists/common.txt", "/tmp/common.txt")  // Returns: true or false
```

### cdn\_exists(remotePath)

Check if file exists in cloud storage.

```javascript theme={null}
cdn_exists("scans/{{Target}}/report.zip")  // Returns: true or false
```

### cdn\_delete(remotePath)

Delete file from cloud storage.

```javascript theme={null}
cdn_delete("scans/{{Target}}/old-report.zip")  // Returns: true or false
```

### cdn\_sync\_upload(localDir, remotePrefix)

Sync local directory to cloud storage (delta upload).

```javascript theme={null}
cdn_sync_upload("{{Output}}", "scans/{{Target}}/")
// Returns: {uploaded: 5, skipped: 10, errors: 0}
```

### cdn\_sync\_download(remotePrefix, localDir)

Sync cloud storage to local directory (delta download).

```javascript theme={null}
cdn_sync_download("base-setup/", "{{BaseFolder}}")
// Returns: {downloaded: 3, skipped: 7, errors: 0}
```

### cdn\_get\_presigned\_url(remotePath, expiryMins?)

Generate presigned URL for file access.

```javascript theme={null}
cdn_get_presigned_url("scans/target/report.zip", 60)
// Returns: "https://bucket.s3.amazonaws.com/scans/target/report.zip?X-Amz-..."
```

### cdn\_list(prefix?)

List files with metadata from cloud storage.

```javascript theme={null}
cdn_list("scans/")
// Returns: [{key: "scans/target1/report.zip", size: 1024, lastModified: "..."}, ...]
```

### cdn\_stat(remotePath)

Get file metadata from cloud storage.

```javascript theme={null}
cdn_stat("scans/target/report.zip")
// Returns: {key: "...", size: 1024, lastModified: "...", etag: "..."} or null
```

### cdn\_read(remotePath)

Read file content from cloud storage and return as string.

```javascript theme={null}
cdn_read("config/settings.yaml")  // Returns: file contents as string
```

### cdn\_ls\_tree(prefix?, depth?)

List cloud storage contents in a tree-like format.

```javascript theme={null}
cdn_ls_tree("scans/")       // Returns: tree-formatted string
cdn_ls_tree("scans/", 2)    // Returns: tree limited to depth 2
```

## Unix Command Wrappers

Wrappers around common Unix commands.

### sort\_unix(input, output?)

Sort file with `LC_ALL=C sort -u` (in-place if no output specified).

```javascript theme={null}
sort_unix("{{Output}}/urls.txt")                              // In-place
sort_unix("{{Output}}/urls.txt", "{{Output}}/urls-sorted.txt") // To new file
// Returns: true or false
```

### wget\_unix(url, output?)

Download file with wget.

```javascript theme={null}
wget_unix("https://example.com/file.txt", "/tmp/file.txt")  // Returns: true or false
```

### git\_clone(repo, dest?)

Clone git repository (shallow clone).

```javascript theme={null}
git_clone("https://github.com/user/repo", "/tmp/repo")  // Returns: true or false
```

### zip\_unix(source, dest)

Create zip archive using `zip -r`.

```javascript theme={null}
zip_unix("{{Output}}", "{{Output}}/archive.zip")  // Returns: true or false
```

### unzip\_unix(source, dest?)

Extract zip archive using `unzip`.

```javascript theme={null}
unzip_unix("/tmp/archive.zip", "/tmp/extracted")  // Returns: true or false
```

### tar\_unix(source, dest)

Create tar.gz archive using `tar -czf`.

```javascript theme={null}
tar_unix("{{Output}}", "{{Output}}/archive.tar.gz")  // Returns: true or false
```

### untar\_unix(source, dest?)

Extract tar.gz archive using `tar -xzf`.

```javascript theme={null}
untar_unix("/tmp/archive.tar.gz", "/tmp/extracted")  // Returns: true or false
```

### diff\_unix(file1, file2, output?)

Compare files with diff command.

```javascript theme={null}
diff_unix("old.txt", "new.txt")                    // Returns: diff output
diff_unix("old.txt", "new.txt", "changes.diff")   // Writes to file, returns: true or false
```

### sed\_string\_replace(sed\_syntax, source, dest)

String replacement with sed `s/old/new/g` syntax.

```javascript theme={null}
sed_string_replace("s/http/https/g", "{{Output}}/urls.txt", "{{Output}}/urls-fixed.txt")
// Returns: true or false
```

### wget(url, outputPath)

Download file using pure Go (no wget dependency). Supports segmented downloads.

```javascript theme={null}
wget("https://example.com/file.zip", "/tmp/file.zip")  // Returns: true or false
```

### git\_clone\_subfolder(git\_url, subfolder, dest)

Clone only a specific subfolder from a git repository.

```javascript theme={null}
git_clone_subfolder("https://github.com/user/repo.git", "tools/scanner", "/tmp/scanner")
// Returns: true or false
```

### sed\_regex\_replace(sed\_syntax, source, dest)

Regex replacement with sed `s/pattern/repl/g` syntax.

```javascript theme={null}
sed_regex_replace("s/[0-9]+/NUM/g", "{{Output}}/data.txt", "{{Output}}/data-clean.txt")
// Returns: true or false
```

## Archive Functions (Go)

Pure Go implementations for archive operations (no Unix dependencies).

### zip\_dir(source, dest)

Zip directory using Go's archive/zip.

```javascript theme={null}
zip_dir("{{Output}}", "{{Output}}/archive.zip")  // Returns: true or false
```

### unzip\_dir(source, dest)

Unzip archive using Go's archive/zip.

```javascript theme={null}
unzip_dir("/tmp/archive.zip", "/tmp/extracted")  // Returns: true or false
```

### extract\_to(source, dest)

Auto-detect archive format (.zip, .tar.gz, .tar.bz2, .tar.xz, .tgz) and extract. Removes destination directory first if it exists.

```javascript theme={null}
extract_to("{{Output}}/repo.tar.gz", "{{Output}}/repo")  // Returns: true or false
extract_to("{{Output}}/tools.zip", "{{Output}}/tools")    // Returns: true or false
```

## Diff Functions

File comparison and diff extraction.

### extract\_diff(file1, file2)

Get lines that are only in file2 (new content).

```javascript theme={null}
extract_diff("{{Output}}/old-subs.txt", "{{Output}}/new-subs.txt")
// Returns: "newsub1.example.com\nnewsub2.example.com"
```

## Output Functions

Save content and convert data formats.

### save\_content(content, path)

Save string content to file.

```javascript theme={null}
save_content("Hello World", "{{Output}}/greeting.txt")  // Returns: true or false
```

### jsonl\_to\_csv(source, dest)

Convert JSONL file to CSV.

```javascript theme={null}
jsonl_to_csv("{{Output}}/assets.jsonl", "{{Output}}/assets.csv")  // Returns: true or false
```

### csv\_to\_jsonl(source, dest)

Convert CSV file to JSONL.

```javascript theme={null}
csv_to_jsonl("{{Output}}/data.csv", "{{Output}}/data.jsonl")  // Returns: true or false
```

### jsonl\_unique(source, dest, fields)

Deduplicate JSONL by hashing selected fields.

```javascript theme={null}
jsonl_unique("{{Output}}/httpx.jsonl", "{{Output}}/httpx-unique.jsonl", ["status", "words", "lines"])
// Returns: true or false
```

### jsonl\_filter(source, dest, fields)

Filter JSONL to selected fields only.

```javascript theme={null}
jsonl_filter("{{Output}}/httpx.jsonl", "{{Output}}/httpx-filtered.jsonl", "host,status,hash.body_sha256")
// Returns: true or false
```

### jsonl\_rename\_key(source, dest, mappings)

Rename keys in JSONL records based on a mapping string.

```javascript theme={null}
jsonl_rename_key("{{Output}}/data.jsonl", "{{Output}}/renamed.jsonl", "old_key:new_key,host:domain")
// Returns: true or false
```

## URL Processing Functions

URL deduplication and filtering.

### interesting\_urls(src, dest, json\_field?)

Deduplicate URLs by hostname+path+params, filter static files and noise patterns.

```javascript theme={null}
// Plain text file
interesting_urls("{{Output}}/all-urls.txt", "{{Output}}/interesting.txt")

// JSONL file with URL in specific field
interesting_urls("{{Output}}/katana.jsonl", "{{Output}}/interesting.jsonl", "url")
// Returns: true or false
```

### get\_parent\_url(url)

Strip the last path component from a URL.

```javascript theme={null}
get_parent_url("https://example.com/api/v1/users")  // Returns: "https://example.com/api/v1"
```

### parse\_url(url, format)

Parse a URL and format output using directives (similar to unfurl).

```javascript theme={null}
parse_url("https://user:pass@example.com:8080/path?q=1", "%s://%d%p")
// Returns: "https://example.com/path"
```

### parse\_url\_file(input, format, output)

Apply `parse_url` formatting to each line of a file and write results.

```javascript theme={null}
parse_url_file("{{Output}}/urls.txt", "%d", "{{Output}}/domains.txt")  // Returns: true or false
```

### query\_replace(url, value, mode?)

Replace all query parameter values in a URL with a given value.

```javascript theme={null}
query_replace("https://example.com/search?q=test&page=1", "FUZZ")
// Returns: "https://example.com/search?q=FUZZ&page=FUZZ"
```

### path\_replace(url, value, position?)

Replace a path segment at a given position (or all segments) with a value.

```javascript theme={null}
path_replace("https://example.com/api/v1/users", "FUZZ", 2)
// Returns: "https://example.com/api/FUZZ/users"
```

## Markdown Functions

Markdown rendering and conversion.

### render\_markdown\_from\_file(path)

Render markdown with terminal styling.

```javascript theme={null}
render_markdown_from_file("{{Output}}/report.md")  // Returns: rendered markdown string
```

### print\_markdown\_from\_file(path)

Print markdown with syntax highlighting to stdout.

```javascript theme={null}
print_markdown_from_file("{{Output}}/summary.md")
```

### convert\_jsonl\_to\_markdown(input\_path, output\_path)

Convert JSONL to markdown table and write to file.

```javascript theme={null}
convert_jsonl_to_markdown("{{Output}}/assets.jsonl", "{{Output}}/assets.md")  // Returns: true or false
```

### convert\_csv\_to\_markdown(path)

Convert CSV to markdown table.

```javascript theme={null}
convert_csv_to_markdown("{{Output}}/data.csv")  // Returns: markdown table string
```

### render\_markdown\_report(template\_path, output\_path)

Render markdown template with `osm-func` blocks evaluated.

```javascript theme={null}
render_markdown_report("{{Templates}}/report.md", "{{Output}}/report.md")  // Returns: true or false
```

### generate\_security\_report(template\_path)

Generate security report from template to `{{Output}}/security-report.md` and register as artifact.

```javascript theme={null}
generate_security_report("{{MarkdownTemplates}}/security-report-template.md")  // Returns: true or false
```

## Database Functions

Database operations for assets, vulnerabilities, and statistics.

### Artifact Registration

#### register\_artifact(path, type?)

Register file as scan artifact.

```javascript theme={null}
register_artifact("{{Output}}/nuclei.json", "nuclei")  // Returns: true or false
```

#### store\_artifact(path)

Store file as run artifact for current workspace.

```javascript theme={null}
store_artifact("{{Output}}/report.md")  // Returns: true or false
```

### Database Updates

#### db\_update(table, key, field, value)

Update database field.

```javascript theme={null}
db_update("workspaces", "{{Workspace}}", "status", "completed")  // Returns: true or false
```

### Asset Import

#### db\_import\_asset(workspace, json)

Import asset from JSON (upsert - update or insert).

```javascript theme={null}
db_import_asset("{{Workspace}}", '{"asset_value":"sub.example.com","asset_type":"subdomain"}')
// Returns: true or false
```

#### db\_raw\_insert\_asset(workspace, json)

Insert asset from JSON (pure insert, returns ID).

```javascript theme={null}
db_raw_insert_asset("{{Workspace}}", '{"asset_value":"api.example.com"}')  // Returns: 123 (asset ID)
```

#### db\_import\_asset\_from\_file(workspace, file\_path)

Import assets from JSONL file (httpx format supported).

```javascript theme={null}
db_import_asset_from_file("{{Workspace}}", "{{Output}}/httpx.jsonl")  // Returns: 42 (count)
```

#### db\_quick\_import\_asset(workspace, asset\_value, asset\_type?)

Quick import a single asset by value with optional type.

```javascript theme={null}
db_quick_import_asset("{{Workspace}}", "sub.example.com", "subdomain")  // Returns: true or false
```

#### db\_partial\_import\_asset(workspace, asset\_type, asset\_value)

Import a single asset with explicit type.

```javascript theme={null}
db_partial_import_asset("{{Workspace}}", "subdomain", "api.example.com")  // Returns: true or false
```

#### db\_partial\_import\_asset\_file(workspace, asset\_type, file\_path)

Import assets from a plain text file (one per line) with explicit type.

```javascript theme={null}
db_partial_import_asset_file("{{Workspace}}", "subdomain", "{{Output}}/subs.txt")  // Returns: 150 (count)
```

#### db\_import\_dns\_asset(workspace, file\_path)

Import DNS record assets from file.

```javascript theme={null}
db_import_dns_asset("{{Workspace}}", "{{Output}}/dns-records.jsonl")  // Returns: 50 (count)
```

#### db\_import\_custom\_asset(workspace, file\_path, asset\_type?, source?)

Import custom assets from file with optional type and source.

```javascript theme={null}
db_import_custom_asset("{{Workspace}}", "{{Output}}/custom.jsonl", "ip", "masscan")
// Returns: {imported: 100, skipped: 5}
```

### Vulnerability Import

#### db\_import\_vuln(workspace, json)

Import single vulnerability from JSON (nuclei format).

```javascript theme={null}
db_import_vuln("{{Workspace}}", '{"template-id":"cve-2024-1234","info":{"name":"CVE","severity":"high"}}')
// Returns: true or false
```

#### db\_import\_vuln\_from\_file(workspace, file\_path)

Import vulnerabilities from JSONL file (nuclei format).

```javascript theme={null}
db_import_vuln_from_file("{{Workspace}}", "{{Output}}/nuclei.jsonl")  // Returns: 15 (count)
```

### Statistics Updates (Write)

These functions count lines in a file and update workspace statistics.

#### db\_total\_subdomains(path)

```javascript theme={null}
db_total_subdomains("{{Output}}/subdomains.txt")  // Returns: 150
```

#### db\_total\_urls(path)

```javascript theme={null}
db_total_urls("{{Output}}/urls.txt")  // Returns: 5000
```

#### db\_total\_assets(path)

```javascript theme={null}
db_total_assets("{{Output}}/assets.txt")  // Returns: 200
```

#### db\_total\_vulns(path)

```javascript theme={null}
db_total_vulns("{{Output}}/vulns.txt")  // Returns: 25
```

#### db\_vuln\_critical(path)

```javascript theme={null}
db_vuln_critical("{{Output}}/nuclei.json")  // Returns: 2
```

#### db\_vuln\_high(path)

```javascript theme={null}
db_vuln_high("{{Output}}/nuclei.json")  // Returns: 5
```

#### db\_vuln\_medium(path)

```javascript theme={null}
db_vuln_medium("{{Output}}/nuclei.json")  // Returns: 10
```

#### db\_vuln\_low(path)

```javascript theme={null}
db_vuln_low("{{Output}}/nuclei.json")  // Returns: 8
```

#### db\_total\_ips(path)

```javascript theme={null}
db_total_ips("{{Output}}/ips.txt")  // Returns: 50
```

#### db\_total\_links(path)

```javascript theme={null}
db_total_links("{{Output}}/links.txt")  // Returns: 1000
```

#### db\_total\_content(path)

```javascript theme={null}
db_total_content("{{Output}}/content.txt")  // Returns: 500
```

#### db\_total\_archive(path)

```javascript theme={null}
db_total_archive("{{Output}}/archive.txt")  // Returns: 100
```

### Statistics Queries (Read)

These functions read current workspace statistics (no arguments needed).

#### db\_select\_total\_subdomains()

```javascript theme={null}
db_select_total_subdomains()  // Returns: 150
```

#### db\_select\_total\_urls()

```javascript theme={null}
db_select_total_urls()  // Returns: 5000
```

#### db\_select\_total\_assets()

```javascript theme={null}
db_select_total_assets()  // Returns: 200
```

#### db\_select\_total\_vulns()

```javascript theme={null}
db_select_total_vulns()  // Returns: 25
```

#### db\_select\_vuln\_critical()

```javascript theme={null}
db_select_vuln_critical()  // Returns: 2
```

#### db\_select\_vuln\_high()

```javascript theme={null}
db_select_vuln_high()  // Returns: 5
```

#### db\_select\_vuln\_medium()

```javascript theme={null}
db_select_vuln_medium()  // Returns: 10
```

#### db\_select\_vuln\_low()

```javascript theme={null}
db_select_vuln_low()  // Returns: 8
```

### Data Selection

#### db\_select\_assets(workspace, format)

Select all assets from workspace.

```javascript theme={null}
db_select_assets("{{Workspace}}", "markdown")  // Returns: markdown table
db_select_assets("{{Workspace}}", "jsonl")     // Returns: JSONL string
```

#### db\_select\_assets\_filtered(workspace, status\_code, asset\_type, format)

Select assets with filters.

```javascript theme={null}
db_select_assets_filtered("{{Workspace}}", "200", "subdomain", "jsonl")
// Returns: filtered JSONL
```

#### db\_select\_vulnerabilities(workspace, format)

Select all vulnerabilities from workspace.

```javascript theme={null}
db_select_vulnerabilities("{{Workspace}}", "markdown")  // Returns: markdown table
```

#### db\_select\_vulnerabilities\_filtered(workspace, severity, asset\_value, format)

Select vulnerabilities with filters.

```javascript theme={null}
db_select_vulnerabilities_filtered("{{Workspace}}", "critical", "", "jsonl")
// Returns: filtered JSONL
```

#### db\_select(sql\_query, format)

Execute SELECT query with format.

```javascript theme={null}
db_select("SELECT * FROM assets WHERE workspace = '{{Workspace}}' LIMIT 10", "markdown")
```

#### db\_select\_to\_file(sql\_query, dest)

Execute SELECT and write markdown to file.

```javascript theme={null}
db_select_to_file("SELECT * FROM assets", "{{Output}}/assets.md")  // Returns: true or false
```

#### db\_select\_to\_jsonl(sql\_query, fields, dest)

Execute SELECT and write JSONL with specified fields.

```javascript theme={null}
db_select_to_jsonl("SELECT * FROM assets", "asset_value,status_code", "{{Output}}/assets.jsonl")
// Returns: true or false
```

### Diff Tracking

#### db\_asset\_diff(workspace)

Get asset diff as JSONL string (new/changed assets since last scan).

```javascript theme={null}
db_asset_diff("{{Workspace}}")  // Returns: JSONL string
```

#### db\_vuln\_diff(workspace)

Get vulnerability diff as JSONL string.

```javascript theme={null}
db_vuln_diff("{{Workspace}}")  // Returns: JSONL string
```

#### db\_asset\_diff\_to\_file(workspace, dest)

Write asset diff to JSONL file.

```javascript theme={null}
db_asset_diff_to_file("{{Workspace}}", "{{Output}}/asset-diff.jsonl")  // Returns: true or false
```

#### db\_vuln\_diff\_to\_file(workspace, dest)

Write vulnerability diff to JSONL file.

```javascript theme={null}
db_vuln_diff_to_file("{{Workspace}}", "{{Output}}/vuln-diff.jsonl")  // Returns: true or false
```

### Run Status

#### run\_status(workspace, format)

Query run records for a workspace.

```javascript theme={null}
run_status("{{Workspace}}", "markdown")  // Returns: markdown table of runs
run_status("{{Workspace}}", "jsonl")     // Returns: JSONL string
```

#### run\_status\_by\_uuid(uuid, format)

Query a specific run by UUID.

```javascript theme={null}
run_status_by_uuid("abc123-uuid", "markdown")  // Returns: markdown table
```

### Event Log Management

#### db\_reset\_event\_logs(workspace?, topic\_pattern?)

Reset (delete) event logs, optionally filtered by workspace and topic pattern.

```javascript theme={null}
db_reset_event_logs()                              // Reset all event logs
db_reset_event_logs("{{Workspace}}")               // Reset for specific workspace
db_reset_event_logs("{{Workspace}}", "discovery.*") // Reset matching topic pattern
// Returns: {reset: 42, total: 100}
```

### Runtime Export

#### runtime\_export()

Export scan and workspace state to `run-state.json`.

```javascript theme={null}
runtime_export()  // Returns: true or false
```

## Installer Functions

Download and install packages.

### go\_getter(url, dest)

Download files/repos using go-getter (supports git, http, s3, gcs, etc.).

```javascript theme={null}
go_getter("https://github.com/user/repo.git?ref=main", "{{Output}}/repo")
go_getter("s3::https://bucket.s3.amazonaws.com/file.zip", "/tmp/file.zip")
// Returns: true or false
```

### go\_getter\_with\_sshkey(ssh\_key\_path, git\_url, dest)

Clone git repo via SSH with specified key.

```javascript theme={null}
go_getter_with_sshkey("~/.ssh/id_rsa", "git@github.com:user/private-repo.git", "{{Output}}/repo")
// Returns: true or false
```

### nix\_install(package, dest?)

Install package via Nix package manager.

```javascript theme={null}
nix_install("nuclei", "{{Binaries}}")  // Returns: true or false
```

### filepath\_installer(local\_path, tool\_name, dest?)

Install a binary from a local file path by copying it to the destination.

```javascript theme={null}
filepath_installer("/tmp/downloads/nuclei", "nuclei", "{{Binaries}}")  // Returns: true or false
```

## Environment Functions

Environment variable operations.

### os\_getenv(name)

Get environment variable.

```javascript theme={null}
os_getenv("HOME")      // Returns: "/home/user"
os_getenv("API_KEY")   // Returns: "secret" or ""
```

### os\_setenv(name, value)

Set environment variable.

```javascript theme={null}
os_setenv("API_KEY", "new-secret")  // Returns: true or false
```

## SARIF Functions

Parse and import SARIF (Static Analysis Results Interchange Format) output from SAST tools.

### db\_import\_sarif(workspace, file\_path)

Import vulnerabilities from a SARIF file into the database. Supports Semgrep, Trivy, Kingfisher, Bearer, and other SARIF-producing tools.

```javascript theme={null}
db_import_sarif("{{Workspace}}", "{{Output}}/semgrep.sarif")
// Returns: {imported: 15, skipped: 2, errors: 0}
```

### convert\_sarif\_to\_markdown(input\_path, output\_path)

Convert SARIF file to readable markdown tables.

```javascript theme={null}
convert_sarif_to_markdown("{{Output}}/scan.sarif", "{{Output}}/findings.md")  // Returns: true or false
```

## Nmap/Port Functions

Port scanning and result processing.

### nmap\_to\_jsonl(input\_path, output\_path)

Convert nmap XML or gnmap output to JSONL format.

```javascript theme={null}
nmap_to_jsonl("{{Output}}/nmap.xml", "{{Output}}/ports.jsonl")  // Returns: true or false
```

### run\_nmap(target, flags?, output?)

Execute nmap scan and auto-convert results to JSONL.

```javascript theme={null}
run_nmap("192.168.1.0/24")                                    // Returns: path to output JSONL
run_nmap("10.0.0.1", "-sV -p 80,443", "{{Output}}/nmap")      // Returns: path to output JSONL
```

### db\_import\_port\_assets(workspace, file\_path, source?)

Import port scan JSONL into database as IP assets.

```javascript theme={null}
db_import_port_assets("{{Workspace}}", "{{Output}}/ports.jsonl", "nmap")
// Returns: {imported: 50, skipped: 0, errors: 0}
```

## Tmux Functions

Manage background processes via tmux sessions.

### tmux\_run(command, session\_name?)

Create a detached tmux session running a command. Auto-generates a `bosm-<random8>` name if not provided.

```javascript theme={null}
tmux_run("nuclei -l targets.txt -o results.txt")           // Returns: "bosm-a1b2c3d4" (session name)
tmux_run("long-scan.sh", "my-scan")                        // Returns: "my-scan"
```

### tmux\_capture(session\_name)

Capture the current pane output from a tmux session. Pass `"all"` to capture all sessions.

```javascript theme={null}
tmux_capture("my-scan")  // Returns: current output as string
tmux_capture("all")      // Returns: output from all sessions
```

### tmux\_send(session\_name, command)

Send keystrokes to a running tmux session.

```javascript theme={null}
tmux_send("my-scan", "q")       // Returns: true or false
tmux_send("my-scan", "C-c")     // Send Ctrl+C
```

### tmux\_kill(session\_name)

Kill a tmux session.

```javascript theme={null}
tmux_kill("my-scan")  // Returns: true or false
```

### tmux\_list()

List all tmux session names.

```javascript theme={null}
tmux_list()  // Returns: ["bosm-a1b2c3d4", "my-scan", ...]
```

## SSH/Sync Functions

Remote execution and file synchronization.

### ssh\_exec(host, command, user?, key\_path?, password?, port?)

Execute a command on a remote host via SSH (uses connection pooling).

```javascript theme={null}
ssh_exec("10.0.0.1", "whoami")                                          // Returns: "root"
ssh_exec("10.0.0.1", "ls /opt", "admin", "~/.ssh/id_rsa", "", 22)      // Returns: command output
```

### ssh\_rsync(host, src, dest, user?, key\_path?, password?, port?)

Copy files to/from a remote host via rsync over SSH.

```javascript theme={null}
ssh_rsync("10.0.0.1", "{{Output}}/results/", "/tmp/results/", "admin", "~/.ssh/id_rsa")
// Returns: true or false
```

### sync\_from\_master(src, dest)

Pull files from the master node. Falls back to local copy if not in distributed mode.

```javascript theme={null}
sync_from_master("{{BaseFolder}}/wordlists/", "{{Output}}/wordlists/")  // Returns: true or false
```

### sync\_from\_worker(identifier, ip, src, dest)

Sync files from a specific worker node.

```javascript theme={null}
sync_from_worker("wosm-abc123", "10.0.0.2", "/tmp/results/", "{{Output}}/worker-results/")
// Returns: true or false
```

### rsync\_to\_worker(identifier, ip, src, dest)

Push files to a specific worker node.

```javascript theme={null}
rsync_to_worker("wosm-abc123", "10.0.0.2", "{{Output}}/config/", "/tmp/config/")
// Returns: true or false
```

## Script Execution Functions

Execute Python and TypeScript code from workflows.

### exec\_python(code)

Run inline Python code. Prefers `uv` → `python3` → `python`.

```javascript theme={null}
exec_python('import json; print(json.dumps({"status": "ok"}))')  // Returns: '{"status": "ok"}'
```

### exec\_python\_file(path)

Run a Python file. Prefers `uv` → `python3` → `python`.

```javascript theme={null}
exec_python_file("{{Output}}/scripts/analyze.py")  // Returns: script stdout
```

### exec\_ts(code)

Run inline TypeScript code via `bun -e`.

```javascript theme={null}
exec_ts('console.log("Hello from TypeScript")')  // Returns: "Hello from TypeScript"
```

### exec\_ts\_file(path)

Run a TypeScript file via `bun run`.

```javascript theme={null}
exec_ts_file("{{Output}}/scripts/process.ts")  // Returns: script stdout
```

## LLM Functions

Invoke LLM models from workflows.

### llm\_invoke(message)

Send a simple message to the configured LLM and return the response.

```javascript theme={null}
llm_invoke("Summarize these findings: " + read_file("{{Output}}/results.txt"))
// Returns: LLM response as string
```

### llm\_invoke\_custom(message, body\_json)

Send a message with a custom POST body template. Use `{{message}}` as placeholder in the body.

```javascript theme={null}
llm_invoke_custom("Analyze this", '{"model": "gpt-4", "messages": [{"role": "user", "content": "{{message}}"}]}')
// Returns: LLM response as string
```

### llm\_conversations(msg1, msg2, ...)

Multi-turn LLM conversation. Messages use `"role:content"` format.

```javascript theme={null}
llm_conversations("system:You are a security analyst", "user:What is XSS?")
// Returns: LLM response as string
```

## Agent/Distributed Functions

Run ACP agents and distribute execution across nodes.

### run\_agent(message, agent\_name?)

Run an ACP agent and return its output. Defaults to `claude-code`.

```javascript theme={null}
run_agent("Analyze the scan results in {{Output}}")               // Returns: agent output
run_agent("Review this code", "codex")                            // Returns: agent output
```

### run\_on\_master(action, ...args)

Execute a function or command on the master node via Redis.

```javascript theme={null}
run_on_master("func", 'db_import_sarif("ws", "/path/to/file.sarif")')  // Returns: true or false
```

### run\_on\_worker(scope, action, ...args)

Execute a function or command on worker nodes.

```javascript theme={null}
run_on_worker("all", "func", 'log_info("hello from workers")')  // Returns: true or false
```

## Module Control Functions

Control module execution flow.

### skip(message?)

Abort remaining steps in the current module. The flow continues to the next module.

```javascript theme={null}
skip("No targets found, skipping module")  // Raises SkipModuleError
skip()                                     // Skip without message
```

### run\_module(module, target, params?)

Run an osmedeus module programmatically.

```javascript theme={null}
run_module("subdomain", "example.com")                      // Returns: execution output
run_module("portscan", "example.com", "threads=20")          // With custom params
```

### run\_flow(flow, target, params?)

Run an osmedeus flow programmatically.

```javascript theme={null}
run_flow("general", "example.com")                          // Returns: execution output
run_flow("recon", "example.com", "threads=10")               // With custom params
```

## Snapshot Functions

Workspace export and import as compressed ZIP archives.

### snapshot\_export(workspace, dest?)

Export a workspace as a ZIP archive. Returns the path to the created file.

```javascript theme={null}
snapshot_export("{{Workspace}}")                               // Returns: "/path/to/snapshot.zip"
snapshot_export("{{Workspace}}", "/tmp/backup.zip")            // Returns: "/tmp/backup.zip"
```

### snapshot\_import(source)

Import a workspace from a ZIP file or URL. Returns the workspace name.

```javascript theme={null}
snapshot_import("/tmp/snapshot.zip")                           // Returns: "example.com"
snapshot_import("https://example.com/snapshot.zip")            // Returns: "example.com"
```

## Authentication Functions

System authentication operations.

### sudo\_auth(password?, keepalive?)

Authenticate sudo credentials and optionally keep them alive for the session.

```javascript theme={null}
sudo_auth()                    // Prompt for password
sudo_auth("mypassword", true)  // Authenticate and keep credentials alive
// Returns: true or false
```

## Usage Examples

### Conditional Execution

```yaml theme={null}
- name: scan-if-hosts
  type: bash
  pre_condition: 'file_length("{{Output}}/hosts.txt") > 0 && file_exists("{{Output}}/hosts.txt")'
  command: nuclei -l {{Output}}/hosts.txt
```

### Chain Functions

```yaml theme={null}
- name: process
  type: function
  functions:
    - log_info("Starting processing")
    - save_content("processing", "{{Output}}/status.txt")
    - log_info("Processing complete")
```

### Export Results

```yaml theme={null}
- name: count
  type: function
  function: file_length("{{Output}}/results.txt")
  exports:
    result_count: "{{result}}"

- name: report
  type: function
  function: log_info("Found " + result_count + " results")
```

### Decision Based on Function

```yaml theme={null}
- name: check-size
  type: function
  function: file_length("{{Output}}/data.txt")
  exports:
    size: "{{result}}"
  decision:
    switch: "{{size}}"
    cases:
      "0": { goto: no-data }
    default: { goto: process-data }
```

### HTTP API Integration

```yaml theme={null}
- name: submit-results
  type: function
  function: |
    http_post("https://api.example.com/results",
      '{"target": "{{Target}}", "count": ' + result_count + '}')
```

### Event Generation

```yaml theme={null}
- name: emit-discoveries
  type: function
  functions:
    - generate_event_from_file("{{Workspace}}", "discovery", "amass", "subdomain", "{{Output}}/subdomains.txt")
    - log_info("Emitted subdomain discovery events")
```

### Database Reporting

```yaml theme={null}
- name: generate-report
  type: function
  functions:
    - save_content("# Vulnerability Report\n\n", "{{Output}}/report.md")
    - append_file("{{Output}}/report.md", db_select_vulnerabilities("{{Workspace}}", "markdown"))
    - log_info("Report generated at {{Output}}/report.md")
```

## CLI Testing

```bash theme={null}
# Test file functions
osmedeus func e 'file_exists("/etc/passwd")'
osmedeus func e 'file_length("/etc/hosts")'

# Test string functions
osmedeus func e 'trim("  hello  ")'
osmedeus func e 'split("a,b,c", ",")'

# Test with target
osmedeus func e 'log_info("Target: " + target)' -t example.com

# Test JSON
osmedeus func e 'jq("{\"a\":1}", ".a")'

# Test event generation
osmedeus func e 'generate_event("test-workspace", "test.topic", "cli", "test", "data")'

# Bulk processing
osmedeus func e 'log_info("Processing: " + target)' -T targets.txt -c 10

# List functions
osmedeus func list -s "file"
osmedeus func list -s "event" --example
```

## Next Steps

* [Functions Overview](overview) - How to use functions in workflows
* [Step Types](/workflows/step-types) - Function steps
* [Control Flow](/workflows/control-flow) - Conditions and decisions
