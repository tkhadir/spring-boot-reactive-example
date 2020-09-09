# spring-boot-reactive-custom-error-handler 

> ตัวอย่างการเขียน Spring-boot Reactive Custom Error Handler 

# 1. เพิ่ม Dependencies และ Plugins

pom.xml 
``` xml
...
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.3.2.RELEASE</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <scope>provided</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <executions>        
                <execution>            
                    <id>build-info</id>            
                    <goals>                
                        <goal>build-info</goal>            
                    </goals>        
                    <configuration>                
                        <additionalProperties>                    
                            <java.version>${java.version}</java.version>                                   
                        </additionalProperties>            
                    </configuration>        
                </execution>    
            </executions>
        </plugin>
    </plugins>
</build>
...
```

# 2. เขียน Main Class 

``` java
@SpringBootApplication
@ComponentScan(basePackages = {"me.jittagornp"})
public class AppStarter {

    public static void main(String[] args) {
        SpringApplication.run(AppStarter.class, args);
    }

}
```

# 3. เขียน Controller
``` java
@RestController
public class HomeController {


    @GetMapping({"", "/"})
    public Mono<String> hello() {
        throw new InvalidUsernamePasswordException();
    }

    @GetMapping("/serverError")
    public Mono<String> serverError() {
        throw new RuntimeException();
    }

}
```
- ลอง throw RuntimeException และ InvalidUsernamePasswordException ดู 
- เราสามารถ throw Exception ประเภทอื่น ๆ ตามที่เราต้องการได้ 

# 4. เขียน Error Model 
``` java 
@Getter
@Setter
@Builder
@ToString
@NoArgsConstructor
@AllArgsConstructor
public class ErrorResponse {

    @JsonProperty("error")
    private String error;

    @JsonProperty("error_status")
    private int errorStatus;

    @JsonProperty("error_description")
    private String errorDescription;

    @JsonProperty("error_at")
    private LocalDateTime errorAt;

    @JsonProperty("error_trace_id")
    private String errorTraceId;

    @JsonProperty("error_uri")
    private String errorUri;

    @JsonProperty("error_on")
    private String errorOn;

    @JsonProperty("error_fields")
    private List<Field> errorFields;

    @JsonProperty("error_data")
    private Map<String, Object> errorData;

    @JsonProperty("state")
    private String state;

    ...
}
```
- Design ตามนี้ [https://developer.pamarin.com/document/error/](https://developer.pamarin.com/document/error/) 

# 5. เขียน ExceptionHandler 
ตัวจัดการ Error แต่ละประเภท   

### ประกาศ interface 
```java
public interface ErrorResponseExceptionHandler<E extends Throwable> {

    Class<E> getTypeClass();

    Mono<ErrorResponse> handle(final ServerWebExchange exchange, final E e);

}
```
### เขียน Adapter 

เพื่อใช้เป็นตัวกลางในการแปลง Exception

```java
public abstract class ErrorResponseExceptionHandlerAdapter<E extends Throwable> implements ErrorResponseExceptionHandler<E> {

    protected abstract Mono<ErrorResponse> buildError(ServerWebExchange exchange, E e);

    private String getErrorTraceId(final ServerWebExchange exchange) {
        return UUID.randomUUID().toString()
                .replace("-", "")
                .substring(0, 8)
                .toUpperCase();
    }

    private Mono<ErrorResponse> additional(final ErrorResponse err, final ServerWebExchange exchange, final E e) {
        return Mono.fromCallable(() -> {
            ServerHttpRequest httpReq = exchange.getRequest();
            ServerHttpResponse httpResp = exchange.getResponse();
            err.setState(httpReq.getQueryParams().getFirst("state"));
            err.setErrorAt(now());
            if(!hasText(err.getErrorTraceId())){
                err.setErrorTraceId(getErrorTraceId(exchange));
            }
            err.setErrorOn("0");
            httpResp.setStatusCode(
                    err.getErrorStatus() == 0
                            ? HttpStatus.INTERNAL_SERVER_ERROR
                            : HttpStatus.valueOf(err.getErrorStatus())
            );
            err.setErrorUri("https://developer.pamarin.com/document/error/");
            return err;
        });
    }

    @Override
    public Mono<ErrorResponse> handle(final ServerWebExchange exchange, final E e) {
        return buildError(exchange, e)
                .flatMap(err -> additional(err, exchange, e));
    }
}
```
### implementation Error แต่ละประเภท

ตัวจัดการ Exception  
```java
@Component
public class ErrorResponseRootExceptionHandler extends ErrorResponseExceptionHandlerAdapter<Exception> {

    @Override
    public Class<Exception> getTypeClass() {
        return Exception.class;
    }

    @Override
    protected Mono<ErrorResponse> buildError(final ServerWebExchange exchange, final Exception e) {
        return Mono.fromCallable(() -> {
            return ErrorResponse.serverError();
        });
    }
}
```
ตัวจัดการ InvalidUsernamePasswordException 
```java
@Component
public class ErrorResponseInvalidUsernamePasswordExceptionHandler extends ErrorResponseExceptionHandlerAdapter<InvalidUsernamePasswordException> {

    @Override
    public Class<InvalidUsernamePasswordException> getTypeClass() {
        return InvalidUsernamePasswordException.class;
    }

    @Override
    protected Mono<ErrorResponse> buildError(final ServerWebExchange exchange, final InvalidUsernamePasswordException e) {
        return Mono.fromCallable(() -> {
            return ErrorResponse.builder()
                    .error("invalid_username_password")
                    .errorDescription("invalid username or password")
                    .errorStatus(HttpStatus.BAD_REQUEST.value())
                    .build();
        });
    }
}
```
- สามารถเพิ่ม class ตัวจัดการ Exception ใหม่ได้เรื่อย ๆ 

# 6. เขียน Resolver 
สำหรับ resolve error แต่ละประเภท   
  
ประกาศ interface
```java 
public interface ErrorResponseExceptionHandlerResolver {

    Mono<ErrorResponseExceptionHandler> resolve(final Throwable e);

}
```

implement interface  
```java 
@Slf4j
@Component
public class DefaultErrorResponseExceptionHandlerResolver implements ErrorResponseExceptionHandlerResolver {

    private final Map<Class, ErrorResponseExceptionHandler> registry;

    private final ErrorResponseRootErrorHandler rootErrorHandler;

    private final ErrorResponseRootExceptionHandler rootExceptionHandler;

    @Autowired
    public DefaultErrorResponseExceptionHandlerResolver(
            final List<ErrorResponseExceptionHandler> handlers,
            final ErrorResponseRootErrorHandler rootErrorHandler,
            final ErrorResponseRootExceptionHandler rootExceptionHandler
    ) {
        this.registry = handlers.stream()
                .filter(this::ignoreHandler)
                .collect(toMap(ErrorResponseExceptionHandler::getTypeClass, handler -> handler));
        this.rootErrorHandler = rootErrorHandler;
        this.rootExceptionHandler = rootExceptionHandler;
    }

    private boolean ignoreHandler(final ErrorResponseExceptionHandler handler) {
        return !(handler.getTypeClass() == Exception.class
                || handler.getTypeClass() == Error.class);
    }

    @Override
    public Mono<ErrorResponseExceptionHandler> resolve(final Throwable e) {
        ErrorResponseExceptionHandler handler = registry.get(e.getClass());
        if (handler == null) {
            if (e instanceof Error) {
                handler = rootErrorHandler;
            } else {
                handler = rootExceptionHandler;
            }
        }
        log.debug("handler => {}", handler.getClass().getName());
        return Mono.just(handler);
    }

}
```

# 7. เขียน WebExceptionHandler  
เป็นตัวจัดการ Global Exception ทุกประเภท ซึ่ง WebFlux จะโยน Exception เข้ามาที่นี่ 
```java
@Slf4j
@Component
@Order(-2)
@RequiredArgsConstructor
public class ServerWebExceptionHandler implements WebExceptionHandler {

    private final ObjectMapper objectMapper;

    private final ErrorResponseExceptionHandlerResolver resolver;

    @Override
    public Mono<Void> handle(final ServerWebExchange exchange, final Throwable e) {
        log.warn("error => ", e);
        return resolver.resolve(e)
                .flatMap(handler -> (Mono<ErrorResponse>)handler.handle(exchange, e))
                .flatMap(err -> {
                    return jsonResponse(
                            exchange,
                            err
                    );
                });
    }

    private Mono<Void> jsonResponse(final ServerWebExchange exchange, final ErrorResponse err) {
        final ServerHttpResponse response = exchange.getResponse();
        final HttpHeaders headers = response.getHeaders();
        response.setStatusCode(HttpStatus.valueOf(err.getErrorStatus()));
        try {
            headers.put(HttpHeaders.CONTENT_TYPE, Collections.singletonList(MediaType.APPLICATION_JSON_VALUE));
        } catch (UnsupportedOperationException e) {

        }
        return Mono.create((final MonoSink<String> callback) -> {
            try {
                final String json = objectMapper.writeValueAsString(err);
                callback.success(json);
            } catch (final Exception e) {
                callback.error(e);
            }
        })
                .flatMap(json -> {
                    final DataBuffer buffer = response.bufferFactory().wrap(json.getBytes(Charset.forName("utf-8")));
                    return response.writeWith(Mono.just(buffer))
                            .doOnError(e -> DataBufferUtils.release(buffer));
                });
    }
}
```

# 8. Build
cd ไปที่ root ของ project จากนั้น  
``` shell 
$ mvn clean package
```

# 9. Run 
``` shell 
$ mvn spring-boot:run
```

# 10. เข้าใช้งาน

เปิด browser แล้วเข้า [http://localhost:8080](http://localhost:8080)

# ผลลัพธ์ที่ได้
```json
{
    "error": "invalid_username_password",
    "error_status": 400,
    "error_description": "invalid username or password",
    "error_at": "2020-09-09T22:14:02.377062",
    "error_trace_id": "59C1D7E4",
    "error_uri": "https://developer.pamarin.com/document/error/",
    "error_on": "0",
    "error_fields": [ ],
    "error_data": { },
    "state": null
}
```

ลองทดสอบอีกตัวอย่าง [http://localhost:8080/serverError](http://localhost:8080/serverError)

```json
{
    "error": "server_error",
    "error_status": 500,
    "error_description": "System error",
    "error_at": "2020-09-09T22:35:23.914991",
    "error_trace_id": "5056E04D",
    "error_uri": "https://developer.pamarin.com/document/error/",
    "error_on": "0",
    "error_fields": [ ],
    "error_data": { },
    "state": null
}
```

ลองทดสอบอีกตัวอย่าง [http://localhost:8080/unknown](http://localhost:8080/unknown)

```json
{
    "error": "not_found",
    "error_status": 404,
    "error_description": "not found",
    "error_at": "2020-09-09T22:39:38.638358",
    "error_trace_id": "C318CF63",
    "error_uri": "https://developer.pamarin.com/document/error/",
    "error_on": "0",
    "error_fields": [ ],
    "error_data": { },
    "state": null
}
```
