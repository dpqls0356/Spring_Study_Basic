# CH01. 스프링 부트 시작하기

------------

## [Hello Controller]
### "HelloController Code"
``` JAVA
@RestController
public class HelloController{

    @GetMapping("/hello")
    public String hello(String name){
        //param으로 들어간 name이 URL에 파라미터 형식으로 포함
        //localhost:8080/hello?name=name
        return "Hello " + name;
    }
}
```

### "HelloController API Test"
> ~/helloboot  % http -v ":8080/hello?name=Spring"

>GET /hello?name=Spring HTTP/1.1 \
Accept: */* \
Accept-Encoding: gzip, deflate \
Connection: keep-alive \
Host: localhost:8080 \
User-Agent: HTTPie/3.2.1



>HTTP/1.1 200 \
Connection: keep-alive \
Content-Length: 12 \
Content-Type: text/plain;charset=UTF-8 \
Date: Mon, 23 Jan 2023 09:26:40 GMT \
Keep-Alive: timeout=60

------------

## [HTTP 요청과 응답]
### "HTTP: HyperText Transfer Protocol"
> Web Client와 Web Container 사이에 요청(Request)과 응답(Response)가 쌍을 맺어 수행
>> #### 요청이 없는 응답은 없다.

> #### “HTTP”
>> 웹 요청은 어떤 방식으로 전달해야 하고 응답은 어떻게 받아야 하는지 정의한 표준 기술

### "Request"
- Request Line
    - Method: GET, POST, DELETE …
    - Path
    - HTTP Version

    ```
    GET /hello?name=Spring HTTP/1.1
    ```

- Headers: 요청 처리 방식, 응답 생성 타입 등 정보 확인
- Message Body

### "Response"
- Status Line
    - HTTP Version
    - Status Code
    - Status Text

    ```java
    HTTP/1.1 200
    ```

- Headers: Message Body의 타입
- Message Body