# API 与 MCP 规范（核心能力）

更新时间：2026-03-06

## 1. 业务 API（最小集合）

基础前缀：`/api/v1`

1. `GET /projects`
- 返回可用项目列表。

2. `GET /files?project_id={id}&path={path}`
- 列出目录内容。
- 支持可选关键字过滤：`keyword`。

3. `POST /files/upload`
- 入参：`project_id`、`path`、文件二进制。
- 出参：文件元数据（name, size, updated_at）。

4. `GET /files/download?project_id={id}&path={path}`
- 下载文件流。

5. `DELETE /files?project_id={id}&path={path}`
- 删除文件（仅当前阶段简单删除）。

## 2. 返回结构建议

成功：
```json
{
  "ok": true,
  "data": {}
}
```

失败：
```json
{
  "ok": false,
  "error": {
    "code": "FILE_NOT_FOUND",
    "message": "file does not exist"
  }
}
```

## 3. MCP Tools（FastMCP）

建议最小工具集：

1. `list_projects()`
2. `list_files(project_id, path="/", keyword="")`
3. `upload_file(project_id, path, content_base64)`
4. `download_file(project_id, path)`
5. `delete_file(project_id, path)`

约束：
- MCP 不直接操作磁盘，只调用业务 API。
- 所有工具必须传 `project_id`。

## 4. 错误码（最小）

- `INVALID_PROJECT`
- `INVALID_PATH`
- `FILE_NOT_FOUND`
- `FILE_TOO_LARGE`
- `UNSUPPORTED_FILE_TYPE`
- `INTERNAL_ERROR`
