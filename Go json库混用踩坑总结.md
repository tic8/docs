# Go 项目 json 标准库与 json-iterator/go 混用踩坑总结

## 背景

在对接 OpenC API 时，团队为了提升 JSON 序列化/反序列化性能，部分场景将 Go 标准库 `encoding/json` 替换成了 `github.com/json-iterator/go`（下称 json-iterator）。上线后遇到以下报错：

```
json unmarshal error: json: cannot unmarshal object into Go struct field rawAPIResponse.data of type uint8
```

经过排查，发现根本原因是 **json 标准库和 json-iterator 混用导致类型不兼容**，特此记录。

---

## 具体表现

* 结构体中 Data 字段定义为标准库的 `json.RawMessage`：

  ```go
  import "encoding/json"
  type rawAPIResponse struct {
      Code string          `json:"code"`
      Msg  string          `json:"msg"`
      Data json.RawMessage `json:"data"`
  }
  ```

* 但部分代码却用 json-iterator 的 `Unmarshal` 解析数据：

  ```go
  import json "github.com/json-iterator/go"
  json.Unmarshal(resp.Body(), &raw)
  ```

* 最终导致反序列化报错，提示类型不兼容。

---

## 排查过程

1. 检查响应数据格式，服务端返回正常；
2. 检查结构体定义，字段名与类型均无误；
3. 追溯到项目存在标准库和 json-iterator 混用，定位到类型断裂根因。

---

## 解决方案

### 1. 全局只用一种 json 库

* 只用 `encoding/json`（标准库）**或** 只用 `github.com/json-iterator/go`。
* 保证结构体类型、`RawMessage` 类型、所有 Marshal/Unmarshal 调用来源一致。

### 2. 推荐写法

统一引入 json-iterator：

````
```go
import json "github.com/json-iterator/go"

type rawAPIResponse struct {
    Code string          `json:"code"`
    Msg  string          `json:"msg"`
    Data json.RawMessage `json:"data"`
}

// 反序列化
err := json.Unmarshal(resp.Body(), &raw)
```
````

### 3. 检查所有引用

* 搜索项目中所有 `import "encoding/json"`，确保不会和 json-iterator 混用。
* 结构体类型和解析函数必须来源一致。

---

## 经验总结

* Go 标准库和 json-iterator 的 `RawMessage` 名称相同，底层却完全不同，**一旦混用，必出玄学 bug**；
* 出现 `cannot unmarshal object into Go struct field ... of type uint8` 这类莫名其妙的解析错误，**优先排查 json 包混用问题**；
* 团队协作时，建议通过 `import json "github.com/json-iterator/go"` 的方式全局统一，防止引入标准库造成隐患。

---

## 建议

* 代码审查时，遇到所有与 JSON 相关的 `import`、结构体、Marshal/Unmarshal 操作都需检查类型和包来源是否一致。
* 团队 wiki 固化此类最佳实践，减少新成员入坑概率。

---

> 遇到 JSON 反序列化类型异常，优先排查 json 包混用，统一全局用一种 json 包，所有 bug 自动消失！
