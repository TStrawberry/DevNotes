# Swift - 一次使用泛型和协议优化代码的实践
## 写在最开始
个人随笔,文笔实在有限,强烈建议读者直接查看[**demo**](https://github.com/tangtaotao/ServerAPIDemo)。
## 准备工作
Swift为开发者提供了强大的泛型和协议支持,本文将实现一个简单的网络层封装,看看这两种强大特性如何为代码带来提升。  
由于本文的重点集中在泛型和协议上上,因此其他的逻辑将使用三方库,demo中使用的三方库有:

- [Alamofire](https://github.com/Alamofire/Alamofire) - 网络请求
- [ObjectMapper](https://github.com/Hearst-DD/ObjectMapper) - 转模型
- [AlamofireObjectMapper](https://github.com/tristanhimmelman/AlamofireObjectMapper) - 对以上两者的封装

项目的UI结构很简单,根控制器的view正中心有两个UILabel用于显示网络请求的结果。  
项目中使用的服务器接口来自[**DarkSky**](https://darksky.net/dev/docs)提供的公开接口,用于获取天气。不过笔者将仅使用 Forecast Request 这一个接口进行演示。该接口的信息如下:  

```
https://api.darksky.net/forecast/[key]/[latitude],[longitude]

{
	"timezone": "America/New_York",
	"currently": {
		"summary": "Drizzle",
		...
	},
	...
}
```  
因此根据返回的数据格式建立模型如下:

```swift
/// Models.swift

import Foundation
import ObjectMapper

/// 简单起见,我们只将timezone和summary两个字段读取出来就好。
struct Weather: Mappable {
    
    var timezone    : String?
    var summary     : String?
    
    init?(map: Map) {
        guard let _ = map["timezone"].currentValue else { return nil }
        guard let _ = map["currently.summary"].currentValue else { return nil }
    }
    
    mutating func mapping(map: Map) {
        timezone    <- map["timezone"]
        summary     <- map["currently.summary"]
    }
}

```
## 构建基本的请求方法
使用 AlamofireObjectMapper 可以很容易的写出一个从网络请求到模型的接口,因为它其实都帮你做好了:  

```swift
func request(url: URLConvertible, parameters: Alamofire.Parameters? = nil, callBack:  @escaping (Weather?, Error?) -> Void) {
    Alamofire
        .request(url)
        .responseObject { (dataResponse: DataResponse<Weather>) in
            if let weather = dataResponse.result.value {
                callBack(weather, nil)
            } else if let error = dataResponse.error {
                callBack(nil, error)
            } else {
                fatalError("No weather and no error!!!")
            }
        }
}

```  
以此就可以为模版为更多的接口构建请求方法了,但是可能读者已经知道,这样做没有什么意义,有大量的代码重复,不一样的只是将Weather修改成其他的模型类型。所以接下来的一步就是要使用泛型将网络请求的逻辑进行抽象了。  
其实在代码中已经有一个泛型函数了,那就是 responseObject... 方法。它将网络请求的数据进行转成了模型,放在了 dataResponse 中。于此同时, 我们通过在代码中显式的指定 dataResponse 的类型为 DataResponse<Weather> 也就将模型的类型(也就是Weather)传递到了该函数中,以此才能正确的创建我们需要的类型。所以接下来, 泛型改造的第一步就是要将模型类型使用泛型传递。  

## 引入泛型参数
在本项目中, 能够转模型的要求其实很简单, 就是必须实现 ObjectMapper.Mappable 这个协议(当然不仅仅是 ObjectMapper.Mappable, 这里笔者只已 Mappable 为例, 关于 ObjectMapper 的更多用法请查看其主页), 因此对泛型类型进行协议约束就好。改造后的版本为:  

```swift
func request<T: Mappable>(url: URLConvertible, parameters: Alamofire.Parameters? = nil, callBack:  @escaping (Result<T>) -> Void) {
    Alamofire
        .request(url)
        .responseObject { (dataResponse: DataResponse<T>) in
            if let value = dataResponse.result.value {
                callBack(Result<T>.success(value))
            } else if let error = dataResponse.error {
                callBack(Result<T>.failure(error))
            } else {
                fatalError("No weather and no error!!!")
            }
        }
}
```
给request...方法增加一个泛型<T: Mappable>, 从而将模型的类型通过 T 来进行表示, 达到了抽离公共逻辑的目的。另外, 这个版本的实现还对结果的传递方式进行了处理, Result 定义在 Alamofire 中, 用于表示一个请求的两种结果, 即 .success(Value) 和 .failure(Error), 使用它来表示结果更加合适。  
当前的版本其实本无太大的问题,泛型支持,类型自动匹配。但是如果使用这个方法进行网络请求,会出现一个"小问题"。例如请求天气大概是这个样子的:  

```swift
let url = "https://api.darksky.net/forecast/7678bc967b9c7266d48c3ff5601d0735/30.660053,104.068482"

request(url: url) { (result: Result<Weather>) in
	
}
```
问题就在于:使用的时候必须明确的指定 callBack 闭包的参数 result 的类型。为什么笔者觉得这样是一个"问题",因为这里存在一个逻辑问题。  
考虑一种现实的场景: 团队开发中有两个成员A和B, A负责负责编写网络层, 并且根据各个接口的数据定义相应的模型供同事使用。如果B拿到的是以上的这种接口,即**请求接口前必须知道返回值类型才能正常使用, 而不是根据接口得到返回值的信息。**  虽然这样可以正常的使用,但是A也免不了要使用一些注释或者fatalError("xxx")类似的形式来告知使用者返回值类型必须是什么。  
但是实际上,在这里result的类型可以说就是类型推断的起点, 要解决这个问题,**就意味着必须在其他地方引入泛型参数来进行类型推断了。**  

## 协议化请求参数
在进行进一步的构造之前, 需要先做一些其他的工作。就是将请求的某些固定参数封装起来。在真的的开发环境中, 一个请求需要配置的固定参数可能有很多(url, 请求方法, 请求头, 甚至转模型的keyPath)。简单起见, 这里只已 url 为例, 引入协议 APIProtocol :

```swift
protocol APIProtocol {
    associatedtype T
    // 简单起见, 只封装了 url 为例子。
    var APIInfo: (url:URLConvertible, modeType: T.Type) { get }
}
```
因此现在的请求函数应该是:

```swift
func request<API: APIProtocol>(api: API, parameters: Alamofire.Parameters? = nil, callBack:  @escaping (Result<T>) -> Void) where API.T: Mappable {
    Alamofire
        .request(api.APIInfo.url)
        .responseObject { (dataResponse: DataResponse<API.T>) in
            if let value = dataResponse.result.value {
                callBack(Result<API.T>.success(value))
            } else if let error = dataResponse.error {
                callBack(Result<API.T>.failure(error))
            } else {
                fatalError("No weather and no error!!!")
            }
        }
}
```
其实引入协议不是笔者一时兴起, 刚刚上面已经提到, 需要在其他地方引入泛型参数进行类型推断, 而 Protocol 就是泛型很好的载体。引入协议不仅可以将相对固定不变的参数集中管理, 还可以引入泛型参数。因此接下来就需要实现一个了 APIProtocol 协议的类型, 来试试这个能不能奏效。  

```swift
struct API {
    
    struct UserAPI<T>: APIProtocol {
        private let url: URLConvertible
        private let modeType: T.Type
        
        var APIInfo: (url:URLConvertible, modeType: T.Type) {
            return (self.url, self.modeType)
        }
        
        init(url: URLConvertible, modeType: T.Type) {
            self.url = url
            self.modeType = modeType
        }
    }
    
    /// 获取天气信息
    static let getWeather = UserAPI(url: "https://api.darksky.net/forecast/7678bc967b9c7266d48c3ff5601d0735/30.660053,104.068482", modeType: Weather.self)   
}

```
因此现在调用接口获取天气就是这样的:

```swift
// 现在不用明确指定 result 的类型为 Result<Weather> 了, 因为类型已经自动推断出来了。
request(api: API.getWeather) { (result) in

}
```
现在是不是清爽多了, 笔者觉得是的😄。  
 
## 写在最后
非常感谢看到这里, 老实说这是笔者第一篇纯实践性的文章。文笔有限, 诚然文章逻辑混乱, 很多逻辑没有解释, 其实笔者自己也不知道作何解释, 实在抱歉, 可能读者直接看代码来得更直接, demo中笔者还提供了模型集合的处理方案以及结合RxSwift的方案。 如果读者发现任何错误的地方, 感谢指教。
