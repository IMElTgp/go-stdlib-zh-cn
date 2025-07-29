# Go标准库net、sync、net/http、context中文翻译

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## 项目说明

本项目对Golang标准库关键模块的中文翻译，旨在帮助学习Go的并发与网络编程。**非官方翻译，仅供参考**。

**版本基准**：go1.22.2 linux/amd64  
**最新官方文档**：[pkg.go.dev](https://pkg.go.dev/)

### 翻译原则

1. 技术术语首次出现时保留英文（例：互斥锁`Mutex`）

2. Go特有名词不翻译（如`Goroutine`）

3. 代码/注释保持英文原文

## 目录结构

```text
go-stdlib-zh-cn/
├── LICENSE 
├── GO-LICENSE 
├── README.md 
├── .gitignore 
├── sync/ 
│ ├── README.md 
│ ├── mutex.md 
│ ├── rwmutex.md 
│ ├── waitgroup.md 
│ ├── once.md 
│ └── cond.md 
├── net/ 
│ ├── README.md 
│ ├── conn.md 
│ ├── dial.md 
│ └── listen.md 
├── net_http/ 
│ ├── README.md 
│ ├── server.md 
│ ├── client.md 
│ ├── handler.md 
│ ├── request.md 
│ ├── response.md 
│ └── transport.md 
└── context/ 
├── README.md 
├── context.md 
├── withcancel.md 
├── withdeadline.md 
├── withtimeout.md 
└── withvalue.md 
```

## 贡献指南

由于译者个人水平有限，欢迎通过Issue报告错误或提交PR改进翻译，请确保：

- **版本匹配**  

  确保翻译内容与**当前**官方文档一致（[查看当前版本](https://pkg.go.dev/)）

- **术语处理**  

  1. 优先沿用现有翻译术语（可通过搜索确认）

  2. 若需修改现有术语，**请说明修改理由**（如译者的翻译不准确）

  3. 新术语保持直译+英文标注

- **格式规范**  

  - 保留原始文档的代码块/注释（不翻译）

  - 函数/类型名保留英文原名（例：`sync.Mutex`不译）  

  - 中文标点使用全角符号（。？！等）

## 其他

感谢Go团队所作的优秀标准库及官方文档！
