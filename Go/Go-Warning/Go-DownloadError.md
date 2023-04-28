Golang 下载包时报错
```
 ~/Code…orld/Go/vanmmall $ go mod download
go: github.com/gocolly/colly/v2@v2.1.0: Get "https://proxy.golang.org/github.com/gocolly/colly/v2/@v/v2.1.0.mod": dial tcp 172.217.27.145:443: i/o timeout
```

```
# 启用 Go Modules 功能
go env -w GO111MODULE=on

# 配置 GOPROXY 环境变量，以下三选一

# 1. 七牛 CDN
go env -w  GOPROXY=https://goproxy.cn,direct

# 2. 阿里云
go env -w GOPROXY=https://mirrors.aliyun.com/goproxy/,direct

# 3. 官方
go env -w  GOPROXY=https://goproxy.io,direct
```

go mod download 下载成功

```
go mod download
```
