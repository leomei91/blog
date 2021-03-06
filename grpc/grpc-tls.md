# 4.4 TLS 证书认证

项目地址：https://github.com/EDDYCJY/go-grpc-example

## 前言

在前面的章节里，我们介绍了 gRPC 的四种 API 使用方式。是不是很简单呢 😀

此时存在一个安全问题，先前的例子中 gRPC Client/Server 都是明文传输的，会不会有被窃听的风险呢？

从结论上来讲，是有的。在明文通讯的情况下，你的请求就是裸奔的，有可能被第三方恶意篡改或者伪造为“非法”的数据

## 抓个包

![image](https://image.eddycjy.com/15e68df2ba9aa7cace3e26e35c79f200.jpg)

![image](https://image.eddycjy.com/ebebd3ea7d306ad2fcd311f1d8b46cc0.jpg)

嗯，明文传输无误。这是有问题的，接下将改造我们的 gRPC，以便于解决这个问题 😤

## 证书生成

### 私钥

```
openssl ecparam -genkey -name secp384r1 -out server.key
```

### 自签公钥

```
openssl req -new -x509 -sha256 -key server.key -out server.pem -days 3650
```

#### 填写信息

```
Country Name (2 letter code) []:
State or Province Name (full name) []:
Locality Name (eg, city) []:
Organization Name (eg, company) []:
Organizational Unit Name (eg, section) []:
Common Name (eg, fully qualified host name) []:go-grpc-example
Email Address []:
```

### 生成完毕

生成证书结束后，将证书相关文件放到 conf/ 下，目录结构：

```
$ tree go-grpc-example 
go-grpc-example
├── client
├── conf
│   ├── server.key
│   └── server.pem
├── proto
└── server
    ├── simple_server
    └── stream_server
```

由于本文偏向 gRPC，详解可参见 [《制作证书》](https://segmentfault.com/a/1190000013408485#articleHeader3)。后续番外可能会展开细节描述 👌

## 为什么之前不需要证书 

在 simple_server 中，为什么“啥事都没干”就能在不需要证书的情况下运行呢？

### Server

```
grpc.NewServer()
```

在服务端显然没有传入任何 DialOptions

### Client

```
conn, err := grpc.Dial(":"+PORT, grpc.WithInsecure())
```

在客户端留意到 `grpc.WithInsecure()` 方法

```
func WithInsecure() DialOption {
	return newFuncDialOption(func(o *dialOptions) {
		o.insecure = true
	})
}
```

在方法内可以看到 `WithInsecure` 返回一个 `DialOption`，并且它最终会通过读取设置的值来禁用安全传输

那么它“最终”又是在哪里处理的呢，我们把视线移到 `grpc.Dial()` 方法内

```
func DialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
    ...
    
    for _, opt := range opts {
		opt.apply(&cc.dopts)
	}
    ...
    
    if !cc.dopts.insecure {
		if cc.dopts.copts.TransportCredentials == nil {
			return nil, errNoTransportSecurity
		}
	} else {
		if cc.dopts.copts.TransportCredentials != nil {
			return nil, errCredentialsConflict
		}
		for _, cd := range cc.dopts.copts.PerRPCCredentials {
			if cd.RequireTransportSecurity() {
				return nil, errTransportCredentialsMissing
			}
		}
	}
	...
	
	creds := cc.dopts.copts.TransportCredentials
	if creds != nil && creds.Info().ServerName != "" {
		cc.authority = creds.Info().ServerName
	} else if cc.dopts.insecure && cc.dopts.authority != "" {
		cc.authority = cc.dopts.authority
	} else {
		// Use endpoint from "scheme://authority/endpoint" as the default
		// authority for ClientConn.
		cc.authority = cc.parsedTarget.Endpoint
	}
	...
}
```

## gRPC

接下来我们将正式开始编码，在 gRPC Client/Server 上实现 TLS 证书认证的支持 🤔

### TLS Server

```
package main

import (
	"context"
	"log"
	"net"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"

	pb "github.com/EDDYCJY/go-grpc-example/proto"
)

...

const PORT = "9001"

func main() {
	c, err := credentials.NewServerTLSFromFile("../../conf/server.pem", "../../conf/server.key")
	if err != nil {
		log.Fatalf("credentials.NewServerTLSFromFile err: %v", err)
	}

	server := grpc.NewServer(grpc.Creds(c))
	pb.RegisterSearchServiceServer(server, &SearchService{})

	lis, err := net.Listen("tcp", ":"+PORT)
	if err != nil {
		log.Fatalf("net.Listen err: %v", err)
	}

	server.Serve(lis)
}
```

- credentials.NewServerTLSFromFile：根据服务端输入的证书文件和密钥构造 TLS 凭证

```
func NewServerTLSFromFile(certFile, keyFile string) (TransportCredentials, error) {
	cert, err := tls.LoadX509KeyPair(certFile, keyFile)
	if err != nil {
		return nil, err
	}
	return NewTLS(&tls.Config{Certificates: []tls.Certificate{cert}}), nil
}
```

- grpc.Creds()：返回一个 ServerOption，用于设置服务器连接的凭据。用于 `grpc.NewServer(opt ...ServerOption)` 为 gRPC Server 设置连接选项

```
func Creds(c credentials.TransportCredentials) ServerOption {
	return func(o *options) {
		o.creds = c
	}
}
```

经过以上两个简单步骤，gRPC Server 就建立起需证书认证的服务啦 🤔

### TLS Client

```
package main

import (
	"context"
	"log"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"

	pb "github.com/EDDYCJY/go-grpc-example/proto"
)

const PORT = "9001"

func main() {
	c, err := credentials.NewClientTLSFromFile("../../conf/server.pem", "go-grpc-example")
	if err != nil {
		log.Fatalf("credentials.NewClientTLSFromFile err: %v", err)
	}

	conn, err := grpc.Dial(":"+PORT, grpc.WithTransportCredentials(c))
	if err != nil {
		log.Fatalf("grpc.Dial err: %v", err)
	}
	defer conn.Close()

	client := pb.NewSearchServiceClient(conn)
	resp, err := client.Search(context.Background(), &pb.SearchRequest{
		Request: "gRPC",
	})
	if err != nil {
		log.Fatalf("client.Search err: %v", err)
	}

	log.Printf("resp: %s", resp.GetResponse())
}
```

- credentials.NewClientTLSFromFile()：根据客户端输入的证书文件和密钥构造 TLS 凭证。serverNameOverride 为服务名称

```
func NewClientTLSFromFile(certFile, serverNameOverride string) (TransportCredentials, error) {
	b, err := ioutil.ReadFile(certFile)
	if err != nil {
		return nil, err
	}
	cp := x509.NewCertPool()
	if !cp.AppendCertsFromPEM(b) {
		return nil, fmt.Errorf("credentials: failed to append certificates")
	}
	return NewTLS(&tls.Config{ServerName: serverNameOverride, RootCAs: cp}), nil
}
```

- grpc.WithTransportCredentials()：返回一个配置连接的 DialOption 选项。用于 `grpc.Dial(target string, opts ...DialOption)` 设置连接选项

```
func WithTransportCredentials(creds credentials.TransportCredentials) DialOption {
	return newFuncDialOption(func(o *dialOptions) {
		o.copts.TransportCredentials = creds
	})
}
```

## 验证

### 请求

重新启动 server.go 和执行 client.go，得到响应结果

```
$ go run client.go
2018/09/30 20:00:21 resp: gRPC Server
```

### 抓个包

![image](https://image.eddycjy.com/c8ad6edf1f7d084883b847b3eee29dd2.jpg)

成功。

## 总结

在本章节我们实现了 gRPC TLS Client/Servert，你以为大功告成了吗？我不 😤

## 问题

你仔细再看看，Client 是基于 Server 端的证书和服务名称来建立请求的。这样的话，你就需要将 Server 的证书通过各种手段给到 Client 端，否则是无法完成这项任务的

问题也就来了，你无法保证你的“各种手段”是安全的，毕竟现在的网络环境是很危险的，万一被...

我们将在下一章节解决这个问题，保证其可靠性 🙂

## 参考
### 本系列示例代码
- [go-grpc-example](https://github.com/EDDYCJY/go-grpc-example)