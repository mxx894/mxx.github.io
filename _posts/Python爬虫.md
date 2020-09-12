# Python爬虫

## Urllib库用法（python3）

> urllib.request 请求
>
> urllib.error 异常处理
>
> urllib.parase url解析
>
> urllib.robbotparaser robbot.txt解析（用的很少了）

### 1. urlopen用法

`urllib.request.urlopen(url,date=None,[timeout,]*cafile=None,capath=None,cadefault=False,context=None)`

```python
import urllib.request
#get method
response=urllib.request.urlopen('https://www.baidu.com')
print(response.read().decoce('utf-8'))
```

  ```python
import urllib.requestr
import urllib.parse
date=byte(urllib.prase.urlencode({'world':'hello'},encoding='utf8')
response=urllib.request.urlopen('https://httpbin.org/post',data=data)
print(request.read()) #提交date
  ```

### 2. response相应

```python
import urllib.request
response=urllib.request.urlopen('https://www.baidu.com')
print(type(response)) #类型
print(response.status)#状态
print(response.getheaders())#响应头
print(response.getheaders('Service'))#头参数
print(response.read).decode('utf-8')#相应内容
```



### 3. request 请求

request请求可以构造请参数包括求头(headers)

```python
from urllib import request,parse
url ='http://httpbin.org/post'
headers={
  'User-Agent':' Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_3) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.5 Safari/605.1.15'
  'Host':'httpbin.org'
}
dict={
  'name':'Germey'
}
data=bytes(parse.urlencode(dict),encoding='utf-8')
req=request.Request(url=url,data=data,headers=headers,method='POST')#构造urlopen的参数
response=requset.urlopen(req)
print(response.read().decoce('utf-8'))

#另外 可以利用req.add_header() 方式添加请求头


```

