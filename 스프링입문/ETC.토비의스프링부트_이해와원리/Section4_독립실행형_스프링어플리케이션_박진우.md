# [Section 4] 독립 실행형 스프링 애플리케이션

---

# #1. 스프링 컨테이너 사용

![section4_1.png](imgs%2Fsection4_1.png)

- Front Controller가 Hello Controller 객체(Bean)를 직접 가지고 있으면서 관리하는 것이 아닌,
  **Spring Container**에게 관리 위임

```java
public class HellobootApplication {

	public static void main(String[] args) {

		//스프링 컨테이너 생성 : ApplicationContext
		GenericApplicationContext applicationContext = new GenericApplicationContext();
		applicationContext.registerBean(HelloController.class); // 어떤 클래스를 이용하여 Bean Object를 만들어 줄 것인가를 지정
		applicationContext.refresh(); // Container 초기화 -> Bean Object를 생성

		//서블릿 컨테이너 생성
		TomcatServletWebServerFactory serverFactory = new TomcatServletWebServerFactory();
		WebServer webServer = serverFactory.getWebServer(servletContext -> {
//			HelloController helloController = new HelloController();

			servletContext.addServlet("frontcontroller", new HttpServlet() { //어댑터 클래스
				@Override
				protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException { //서블릿의 실제 기능 구현
					//인증, 보안, 다국어 처리 등 공통 기능
					if (req.getRequestURI().equals("/hello") && req.getMethod().equals(HttpMethod.GET.name())){
						String name = req.getParameter("name");

						//Binding: 웹 요청을 가지고 처리하는 로직 코드에서 사용할 수 있도록 새로운 타입으로 변환하는 것
						HelloController helloController = applicationContext.getBean(HelloController.class);
						String ret = helloController.hello(name);

//						resp.setStatus(HttpStatus.OK.value()); //status(200)
						resp.setContentType(MediaType.TEXT_PLAIN_VALUE); //"Content-Type", "text/plain"
						resp.getWriter().println(ret) ;
					} else{
						resp.setStatus(HttpStatus.NOT_FOUND.value());
					}
				}
				//서블릿 컨테이너에서 어떤 서블릿을 사용해야할 지 정하는 Mapping을 설정해야 함.
			}).addMapping("/*"); //모든 요청을 해당 서블릿이 처리
		});
		webServer.start(); //Tomcat Start
	}
}
```

# #2. 의존 오브젝트 추가

> 위와 같이 작성했을 때, **Front Container**가 직접 `new` 키워드를 사용하여 Object를 만들어 쓰는것과 차이가 무엇일까?
→ **Spring Container**가 고정적으로 할 수 있는 일들을 이후에 적용 가능하도록 **기본 구조를 짜놓는다는 것**
>
- Spring Container는 Singleton Pattern과 유사하게 어떤 타입의 오브젝트를 한 번만 만들어두고, 이를 재사용하는 구조
- 역할에 따라 Object를 분리하고, 하나의 Object가 기능이 필요하면, 해당 기능을 가진 다른 Object에게 요청

## [실습]

- `SimpleHelloService`에 이름을 출력하는 간단한 로직을 가진 메서드 정의
- `HelloController`에서 해당 기능을 사용할 수 있도록 구성
- **Controller**가 해야 하는 핵심 Task : **유저의 요청 사항을 검증**

### “SimpleHelloService”

```java
public class SimpleHelloService {

    String sayHello(String name) {

        return "Hello " + name;
    }
}
```

### “HelloController”

```java
import java.util.Objects;

@RestController
public class HelloController {

    @GetMapping("/hello")
    public String hello(String name){

        // Object 생성
        SimpleHelloService helloService = new SimpleHelloService();

        // Null 여부 체크
        // null -> exception
        return helloService.sayHello(Objects.requireNonNull(name));
    }
}
```

### “결과”

![section4_2.png](imgs%2Fsection4_2.png)
![section4_3.png](imgs%2Fsection4_3.png)


# #3. Dependency Injection

## “의존관계란?”

![section4_4.png](imgs%2Fsection4_4.png)

- `HelloController`가 `SimpleHelloService`의 특정 기능을 사용하고자 함.
- 이때, `SimpleHelloService`의 객체 생성을 하고, 해당 객체 내부에 있는 기능을 사용함.
- 이러한 경우, **`HelloController`는 `SimpleHelloService`에 의존한다** 라고 표현

![section4_5.png](imgs%2Fsection4_5.png)

- 하지만 위와 같이 만약 `SimpleHelloService`가 아닌 `ComplexHelloService`의 기능이 필요하다고 할 때, 내 코드를 직접 수정해줘야 하는 불편함이 존재

![section4_6.png](imgs%2Fsection4_6.png)

- 따라서, `Controller`를 `Service` `Interface`에만 의존하도록 설정하고,
  `HelloService` `Interface`를 구현한 클래스들을 만들어둠으로써, 변경이 일어나더라도 유연한 대처가 가능하도록 설정

![section4_7.png](imgs%2Fsection4_7.png)

- 하지만, `HelloService` `Interface`와 의존 관계가 맺어져 있다고 하더라도, 어느 구현체의 Object를 이용하는 것인가에 대해 알 수가 없음.
- 따라서, 어떻게든 구현체와의 의존 관계를 맺어주어야 원하는 기능을 원활하게 사용할 수 있음.
- 이를 **DI(Dependency Injection)** 이라고 한다.

> **Assembler 란?**
→ `HelloController`가 사용하는 Object를 `new` 키워드로 직접 만들지 않고, 외부에서 만들어 주입을 해주는 것

> **Spring Container가 곧 Assembler**

# #4. 의존 오브젝트 DI 적용

### “HelloService”

```java
public interface HelloService {

    String sayHello(String name);
}
```

### “SimpleHelloService”

```java
public class SimpleHelloService implements HelloService {

    @Override
    public String sayHello(String name) {

        return "Hello " + name;
    }
}
```

### “HelloController”

```java
@RestController // 모든 메소드들에 @ResponseBody 어노테이션이 붙은 것과 같음
public class HelloController {

    // 재할당이 불가능하도록 설정
    private final HelloService helloService;

    // 생성자 주입
    public HelloController(HelloService helloService) {
        this.helloService = helloService;
    }

    @GetMapping("/hello")
    public String hello(String name){

        // Object 생성
        SimpleHelloService helloService = new SimpleHelloService();

        // Null 여부 체크
        // null -> exception
        return helloService.sayHello(Objects.requireNonNull(name));
    }
}
```

### “HellobootApplication”

```java
//스프링 컨테이너 생성 : ApplicationContext
		GenericApplicationContext applicationContext = new GenericApplicationContext();

		/** 빈의 생명주기는 Spring Container가 알아서 잘 관리해주기 때문에 등록의 순서는 의미가 없음 */
		applicationContext.registerBean(HelloController.class); // 어떤 클래스를 이용하여 Bean Object를 만들어 줄 것인가를 지정

//		applicationContext.registerBean(HelloService.class); // 인터페이스를 지정하면 안됨
		applicationContext.registerBean(SimpleHelloService.class); // 인터페이스를 구현하는 구현체를 특정해주어야 함

		applicationContext.refresh(); // Container 초기화 -> Bean Object를 생성
```

# #5. DispatcherServelet으로 전환

```java
public class HellobootApplication {

	public static void main(String[] args) {

		//스프링 컨테이너 생성 : ApplicationContext
//		GenericApplicationContext applicationContext = new GenericApplicationContext();

		// DispatcherServlet에 활용하기 위해 GenericWebApplicationContext으로 선언
		GenericWebApplicationContext applicationContext = new GenericWebApplicationContext();

		/** 빈의 생명주기는 Spring Container가 알아서 잘 관리해주기 때문에 등록의 순서는 의미가 없음 */
		applicationContext.registerBean(HelloController.class); // 어떤 클래스를 이용하여 Bean Object를 만들어 줄 것인가를 지정

//		applicationContext.registerBean(HelloService.class); // 인터페이스를 지정하면 안됨
		applicationContext.registerBean(SimpleHelloService.class); // 인터페이스를 구현하는 구현체를 특정해주어야 함

		applicationContext.refresh(); // Container 초기화 -> Bean Object를 생성

		//서블릿 컨테이너 생성
		TomcatServletWebServerFactory serverFactory = new TomcatServletWebServerFactory();
		WebServer webServer = serverFactory.getWebServer(servletContext -> {
//			HelloController helloController = new HelloController();

			servletContext.addServlet("dispatcherServlet",
					new DispatcherServlet(applicationContext)
			).addMapping("/*"); //모든 요청을 해당 서블릿이 처리
		});
		webServer.start(); //Tomcat Start
	}
}
```

- `DispatcherServlet` 활용을 위한 `GenericWebApplicationContext` 객체 선언
- 선언 후 `DispatcherServlet` 객체 생성자의 파라미터로 주입

# #7.  스프링 컨테이너로 통합

```java
public class HellobootApplication {

	public static void main(String[] args) {

		//스프링 컨테이너 생성 : ApplicationContext
//		GenericApplicationContext applicationContext = new GenericApplicationContext();

		// DispatcherServlet에 활용하기 위해 GenericWebApplicationContext으로 선언
		GenericWebApplicationContext applicationContext = new GenericWebApplicationContext(){

			// Context 초기화 메서드
			@Override
			protected void onRefresh() {
				super.onRefresh();

				//서블릿 컨테이너 생성
				TomcatServletWebServerFactory serverFactory = new TomcatServletWebServerFactory();
				WebServer webServer = serverFactory.getWebServer(servletContext -> {
//			HelloController helloController = new HelloController();

					servletContext.addServlet("dispatcherServlet",
							new DispatcherServlet(this) // 자기 자신을 참조
					).addMapping("/*"); //모든 요청을 해당 서블릿이 처리
				});
				webServer.start(); //Tomcat Start
			}
		};

		/** 빈의 생명주기는 Spring Container가 알아서 잘 관리해주기 때문에 등록의 순서는 의미가 없음 */
		applicationContext.registerBean(HelloController.class); // 어떤 클래스를 이용하여 Bean Object를 만들어 줄 것인가를 지정

//		applicationContext.registerBean(HelloService.class); // 인터페이스를 지정하면 안됨
		applicationContext.registerBean(SimpleHelloService.class); // 인터페이스를 구현하는 구현체를 특정해주어야 함

		applicationContext.refresh(); // Container 초기화 -> Bean Object를 생성
	}
}
```

# #8. 자바코드 구성 정보 사용 (@Configuration)

```java
@Configuration // Bean Object를 생성하는 Factory Method가 있는 클래스임을 지정 (컴포넌트 스캔 시 확인)
public class HellobootApplication {

	/** Java Factory Method 작성 */
	// Factory Method : Bean 객체 생성 메서드
	// @Bean : Bean Object를 생성하기 위한 메서드임을 지정
	@Bean
	public HelloController helloController(HelloService helloService) {

		return new HelloController(helloService);
	}

	@Bean
	public HelloService helloService() {

		return new SimpleHelloService();
	}
	public static void main(String[] args) {

		//스프링 컨테이너 생성 : ApplicationContext
//		GenericApplicationContext applicationContext = new GenericApplicationContext();

		// DispatcherServlet에 활용하기 위해 GenericWebApplicationContext으로 선언
//		GenericWebApplicationContext applicationContext = new GenericWebApplicationContext(){

		// @Configuration 어노테이션이 붙은 자바 코드를 이용하여 구성하므로 AnnotationConfigWebApplicationContext 으로 선언
		AnnotationConfigWebApplicationContext applicationContext = new AnnotationConfigWebApplicationContext(){

			// Context 초기화 메서드
			@Override
			protected void onRefresh() {
				super.onRefresh();

				//서블릿 컨테이너 생성
				ServletWebServerFactory serverFactory = new TomcatServletWebServerFactory();
				WebServer webServer = serverFactory.getWebServer(servletContext -> {
					servletContext.addServlet("dispatcherServlet",
							new DispatcherServlet(this) // 자기 자신을 참조
					).addMapping("/*"); //모든 요청을 해당 서블릿이 처리
				});
				webServer.start(); //Tomcat Start
			}
		};

		applicationContext.register(HellobootApplication.class); // 구성 정보가 들어있는 Configuration Class를 파라미터로 등록
		applicationContext.refresh(); // Container 초기화 -> Bean Object를 생성
	}
}
```

# #9. Component Scan

```java
@Configuration // Bean Object를 생성하는 Factory Method가 있는 클래스임을 지정 (컴포넌트 스캔 시 확인)
@ComponentScan // 현재 패키지부터 그 하위 모든 패키지 내 클래스 중 @Component가 붙은 클래스들을 Scan
public class HellobootApplication {
	public static void main(String[] args) {

		// @Configuration 어노테이션이 붙은 자바 코드를 이용하여 구성하므로 AnnotationConfigWebApplicationContext 으로 선언
		AnnotationConfigWebApplicationContext applicationContext = new AnnotationConfigWebApplicationContext(){

			// Context 초기화 메서드
			@Override
			protected void onRefresh() {
				super.onRefresh();

				//서블릿 컨테이너 생성
				ServletWebServerFactory serverFactory = new TomcatServletWebServerFactory();
				WebServer webServer = serverFactory.getWebServer(servletContext -> {
					servletContext.addServlet("dispatcherServlet",
							new DispatcherServlet(this) // 자기 자신을 참조
					).addMapping("/*"); //모든 요청을 해당 서블릿이 처리
				});
				webServer.start(); //Tomcat Start
			}
		};

		applicationContext.register(HellobootApplication.class); // 구성 정보가 들어있는 Configuration Class를 파라미터로 등록
		applicationContext.refresh(); // Container 초기화 -> Bean Object를 생성
	}
}
```

- `@ComponentScan` 어노테이션으로 활용
- 현재 패키지 기준 현재~하위 패키지의 내부에 `@Component` 어노테이션이 붙은 클래스들을 `Bean`으로 등록

    ```java
    @RestController
    @Component
    public class HelloController
    ```

- @Component 어노테이션은 메타 어노테이션이 붙어도 동일한 효과를 가져올 수 있다는 특징이 있음
    - 메타 어노테이션 : 해당 어노테이션이 상위에 붙은 다른 어노테이션

    ```java
    @Retention(RetentionPolicy.RUNTIME) // 생명주기 : RUNTIME동안 살아있음
    @Target(ElementType.TYPE) // Class나 Interface에 적용될 수 있도록 Type으로 지정
    @Component
    public @interface MyAnnotation {
    
    }
    ```

    ```java
    @RestController
    @MyAnnotation
    public class HelloController
    ```

- `@Service`, `@Controller`,  `@RestController`어노테이션은 모두 메타 어노테이션임
  따라서, 해당 어노테이션을 활용함으로써 네이밍 하는 역할로 사용할 수 있음.

## “@Controller와 @RestController의 차이”

![section4_8.png](imgs%2Fsection4_8.png)

- @Controller 어노테이션은 @Component의 메타 어노테이션으로 사용

![section4_9.png](imgs%2Fsection4_9.png)

- `@RestController`는 `@Controller`의 메타 어노테이션
- `@Controller` 어노테이션에 `@ResponseBody` 어노테이션을 추가 적용
    - `@ResponseBody` : API로서 기능을 하는 컨트롤러 메서드는 Body에 응답값을 넣어 표현

# #10. Bean 생명주기 메소드

현재 main의 onRefresh() 메서드를 통해 Bean을 주입해주고 있는 `TomcatServletWebServerFactory`와 `DispatcherServlet` 을 밖으로 빼내어 주입하는 Factory Method를 정의한다.

```java
/**
	 * ServletWebServerFactory와 DispatcherServlet를 Bean으로 등록하는 팩토리 메서드
	 */
	@Bean
	public ServletWebServerFactory servletWebServerFactory() {

		return new TomcatServletWebServerFactory();
	}

	@Bean
	public DispatcherServlet dispatcherServlet() {
		return new DispatcherServlet();
	}
```

```java
AnnotationConfigWebApplicationContext applicationContext = new AnnotationConfigWebApplicationContext(){

	// Context 초기화 메서드
	@Override
	protected void onRefresh() {
		super.onRefresh();

		// 서블릿 컨테이너 생성
		// 현재 클래스가 Configuration이므로 현재 객체에서 Bean 참조
		ServletWebServerFactory serverFactory = this.getBean(ServletWebServerFactory.class);
		DispatcherServlet dispatcherServlet = this.getBean(DispatcherServlet.class);

		dispatcherServlet.setApplicationContext(this); // applicationContext 지정

		WebServer webServer = serverFactory.getWebServer(servletContext -> {
			servletContext.addServlet("dispatcherServlet", dispatcherServlet
			).addMapping("/*"); //모든 요청을 해당 서블릿이 처리
		});
		webServer.start(); //Tomcat Start
	}
};
```

그런데 여기서, applicationContext를 set해주는 `dispatcherServlet.setApplicationContext(this)`가 없으면 어떻게 될까?

applicationContext를 주입하지 않으면 문제가 생기지 않을까?

하지만 놀랍게도 해당 코드는 이상없이 잘 실행된다!

그 이유는 무엇일까?

바로 **Spring Container가 Bean을 관리하면서
‘`DispatcherServlet`은 `applicationContext`가 필요하다’라고 판단하여 자동으로 주입해주기 때문이다!**

이것이 어떻게 가능할까?

`DispatcherServlet`은 `ApplicationContextAware`라는 interface를 구현하고 있다.

이를 살펴보면,

![section4_10.png](imgs%2Fsection4_10.png)

`applicationContext`를 멤버 변수로 저장해주는 set 메서드를 가지고 있는 것을 볼 수 있다.

`ApplicationContextAware`는 Bean을 컨테이너가 관리하는 중에 컨테이너가 관리하는 Object를 Bean에 자동으로 주입해주는 **Life Cycle 메서드**임을 알 수 있다.

따라서, `ApplicationContextAware`가 구현이 되어있다면, Spring Container는 인터페이스의 set 메서드를 통해 자동으로 주입해주는 기능을 하는 것이다.

이를 구현해보도록 하자!

```java
@RestController
@MyAnnotation
public class HelloController implements ApplicationContextAware{

    // 재할당이 불가능하도록 설정
    private final HelloService helloService;

    private ApplicationContext applicationContext;

    // 생성자 주입
    public HelloController(HelloService helloService, ApplicationContext applicationContext) {
        this.helloService = helloService;
    }

    @GetMapping("/hello")
    public String hello(String name) {

        // Object 생성
        SimpleHelloService helloService = new SimpleHelloService();

        // Null 여부 체크
        // null -> exception
        return helloService.sayHello(Objects.requireNonNull(name));
    }

    // Spring Container는 본인(ApplicationContext)도 관리 대상 Bean으로 인식
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println(applicationContext);

        this.applicationContext = applicationContext;
    }
}
```

여기서 나아가, `HelloController` Bean을 만들 때, 생성자의 파라미터를 활용하여 주입받았는데 이를 활용할 수 있다.

```java
@RestController
@MyAnnotation
public class HelloController{

    // 재할당이 불가능하도록 설정
    private final HelloService helloService;

    private ApplicationContext applicationContext;

    // 생성자 주입
    public HelloController(HelloService helloService, ApplicationContext applicationContext) {
        this.helloService = helloService;
        this.applicationContext = applicationContext;
    }

    @GetMapping("/hello")
    public String hello(String name) {

        // Object 생성
        SimpleHelloService helloService = new SimpleHelloService();

        // Null 여부 체크
        // null -> exception
        return helloService.sayHello(Objects.requireNonNull(name));
    }
}
```

# #11. SpringBootApplication

Bean을 주입하기 위한 DispatcherServlet을 등록하는 메서드를 main에서 분리하는 작업을 수행하자.

```java
public class MySpringApplication {

    // Configuration Class를 parameter로 받아 처리할 수 있도록 메서드 분리
    public static void run(Class<?> applicationClass, String... args) {
        // @Configuration 어노테이션이 붙은 자바 코드를 이용하여 구성하므로 AnnotationConfigWebApplicationContext 으로 선언
        AnnotationConfigWebApplicationContext applicationContext = new AnnotationConfigWebApplicationContext(){

            // Context 초기화 메서드
            @Override
            protected void onRefresh() {
                super.onRefresh();

                // 서블릿 컨테이너 생성
                // 현재 클래스가 Configuration이므로 현재 객체에서 Bean 참조
                ServletWebServerFactory serverFactory = this.getBean(ServletWebServerFactory.class);
                DispatcherServlet dispatcherServlet = this.getBean(DispatcherServlet.class);

                dispatcherServlet.setApplicationContext(this); // applicationCOntext 지정

                WebServer webServer = serverFactory.getWebServer(servletContext -> {
                    servletContext.addServlet("dispatcherServlet", dispatcherServlet
                    ).addMapping("/*"); //모든 요청을 해당 서블릿이 처리
                });
                webServer.start(); //Tomcat Start
            }
        };

        applicationContext.register(applicationClass); // 구성 정보가 들어있는 Configuration Class를 파라미터로 등록
        applicationContext.refresh(); // Container 초기화 -> Bean Object를 생성
    }
}
```

이렇게 따로 메서드를 구분해두고, 파라미터로 Configuration Class를 받아 처리한다면, 적절히 가변성을 갖춘 컨테이너 등록 메서드를 만들 수 있다.

이후 main 메서드에서 이를 구현해보면…

![section4_11.png](imgs%2Fsection4_11.png)

어딘가 많이 익숙한 친구가 보인다…

우리가 Spring Boot 프로젝트를 생성하면 가장 먼저 볼수있는 친구와 똑같이 생긴 메서드가 나와버렸다!!

이렇게 우리가 편리하게 만들어 사용하던 Spring Boot 프로젝트의 기본 구성 틀을 이해하는 시간을 가져보았다.