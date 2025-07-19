RPC（Remote Procedure Call）是一种计算机通信协议，允许一个程序（客户端）通过网络向另一个程序（服务器）请求服务，而无需了解底层网络技术的细节。RPC 模型通常包含以下几个主要模块：

1. 客户端（Client）
2. 服务器（Server）
3. 通信协议（Protocol）
4. 序列化/反序列化（Serialization/Deserialization）
![rpc](https://i-blog.csdnimg.cn/direct/1ce7dc0da984482787a9428554656755.png)
下面是一个简单的 RPC 框架的演示，使用 C++ 实现。这个演示不包括网络通信的细节，而是侧重于 RPC 框架的结构。

首先，我们需要一个协议来序列化和反序列化数据。这里我们使用 JSON，可以使用第三方库如 `nlohmann/json`。



~~~cpp
#include <iostream>
#include <nlohmann/json.hpp>
#include <string>
#include <functional>
#include <unordered_map>

using json = nlohmann::json;
using std::string;
using std::function;

// 序列化函数
template <typename T>
string serialize(const T& data) {
    return json(data).dump();
}

// 反序列化函数
template <typename T>
T deserialize(const string& data) {
    T obj;
    json j = json::parse(data);
    j.get_to(obj);
    return obj;
}

// 服务接口
class IService {
public:
    virtual ~IService() = default;
    virtual string Call(const string& method, const string& params) = 0;
};

// RPC 服务器
class RPCServer {
    std::unordered_map<string, function<string(string)>> handlers;

public:
    template <typename Func>
    void Register(const string& method, Func&& func) {
        handlers[method] = func;
    }

    string Handle(const string& method, const string& params) {
        if (handlers.find(method) != handlers.end()) {
            return handlers[method](params);
        } else {
            return "{}";
        }
    }
};

// RPC 客户端
class RPCClient {
    string Call(const string& host, const string& port, const string& method, const string& params) {
        // 这里应该有网络通信代码，发送请求并接收响应
        // 为了演示，我们假设直接调用服务器的 Handle 方法
        RPCServer server;
        server.Handle(method, params);
        return server.Handle(method, params);
    }
};

// 示例服务实现
class ExampleService : public IService {
public:
    string Call(const string& method, const string& params) override {
        if (method == "add") {
            int a = deserialize<int>(params);
            int b = deserialize<int>(params);
            return serialize(a + b);
        }
        return "{}";
    }
};

// 主函数
int main() {
    RPCServer server;
    ExampleService service;
    
    // 注册服务方法
    server.Register("add", [&service](const string& params) {
        return service.Call("add", params);
    });

    RPCClient client;
    string result = client.Call("localhost", "8080", "add", serialize(1) + "," + serialize(2));
    std::cout << "Result: " << result << std::endl;

    return 0;
}

~~~

