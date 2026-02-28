---
layout: post
title: ErrorResponse 핸들링과 문서화
comments: true
excerpt: ""
date: 2023-10-20
categories: [Server]
tags: [RestDocs , Swagger , SpringBoot, ErrorResponse]
---

안녕하세요, Web 팀 인턴 정다연 입니다. 🙇🏻‍♀️

사내 포털 작업 중 서버에 log 로만 error message 를 남길 뿐, 
Front 에 Error Message 를 정확하게 넘겨주지 않아 error 발생 시 
log 를 항상 확인해야 하는 번거로움이 있어 Error Response 관련 여러 테스트를 진행하게 되었습니다.

오늘은 Error Response 관련 테스트를 진행하면서 적용한 작업 및 문서화 경험들을 공유드릴까 합니다. 
<br>
다만 실제 적용이 아닌 테스팅만 진행한 터라 혹시 내용 상 오류 존재 시 편하게 안내 주시면 감사하겠습니다. 🙇🏻‍♀ 

## Spring Boot 예외 처리 흐름

---

들어가기에 앞서 컨트롤러에서 발생한 예외를 처리하기 위해 Spring이 따르는 흐름을 살펴보겠습니다.

<!-- 이미지 생략 -->

1. 먼저 Spring은 `@Controller` 혹은 `@ControllerAdvice` 가 붙은 클래스 내 에서 예외 처리기(`@ExceptionHandler`가 붙은 메서드) 를 검색합니다. (`ExceptionHandlerExceptionResolver` 참조)
2. 그런 다음 던져진 예외가 `@ResponseStatus` 처리되었거나 `ResponseStatusException`에서 파생되었는지 확인합니다.(`ResponseStatusExceptionResolver` 참조)
3. 그 후 Spring MVC의 기본 예외 핸들러를 통과합니다. (`DefaultHandlerExceptionResolver` 참조)
4.  마지막에 아무 것도 발견되지 않으면 그 뒤에 있는 전역 오류 처리기와 함께 오류 페이지 보기로 전달됩니다. 오류 처리기 자체에서 예외가 발생한 경우에는 이 단계는 실행되지 않습니다.
5.  전역 오류 처리기가 비활성화되는 등의 이유로 `error view` 가 발견되지 않거나 4단계를 건너뛰면 컨테이너에서 예외가 처리됩니다.

`DefaultHandlerExceptionResolver`가 하는 일은 `BasicAuthenticationEntryPoint`와 같이
`.response.sendError`를 호출하는 것입니다.<br>
이는 기본 컨테이너가 오류를 위임하는 `/error` handler와 마찬가지로 `BasicErrorController`에 의해 처리됩니다.


## Error Response , 어떻게 제어할까?

---

제어 한 Error Code 관련 Response 를 return 해주는 방식은 크게 두 가지로 나눌 수 있습니다. 

첫 번째 케이스로는 제어하는 모든 Error Response 를 HTTP Status 200 OK 값으로 전달 하는 방법입니다. 
다만 이는 실제로 success 한 값과 구분되어야 하므로 주로 isSuccess 와 같은 flag 값과 제어하고 싶은 code 값을
response body 값으로 함께 보내 제어해줍니다.
이 방식으로 error response 관리 시 백엔드에서 제어하고자 하는 error 값인지, 그렇지 않은 error 값인지 구분이 가능하다는 장점이 있습니다. 

- 실패 시 예시 response
```
{
  "success": false,
  "status": 1006,
  "code": "COMMON-ERR-500",
  "reason": "could not execute statement",
  "timestamp": "2023-10-20T06:12:32.109+00:00",
  "path": "/api/myProfile"
}
```
- 성공 시 response 값을 data 에 넣어 return 
```
{
  "success": true,
  "status": 200,
  "data": {},
  "timeStamp": "2023-10-20T06:12:32.109+00:00"
}
```

두 번째 케이스로는 HTTP Status 를 지정한 unique 한 code 값으로 내려주는 방법입니다. 
이는 첫 번째 케이스 보다 구현 방식이 간략하다는 장점을 가지고 있습니다.


```
{
  "status": 1006,
  "detail": "COMMON-ERR-500",
  "timestamp": "2023-10-20T06:21:51.563+00:00",
  "message": "could not execute statement",
  "path": "/api/myProfile"
}
```


보통의 규모가 큰 서비스에서는 첫 번째와 같은 방식을, error code 세분화가 크게 중요하지 않은 서비스에서는 
두 번째와 같은 방식을 많이 차용하는 것으로 알고 있습니다.
서비스의 규모와 제어하고자 하는 Error Code 의 정도에 따라 보다 맞는 방식을 적용해 사용하면 될 것 같습니다.


다음으로 Spring Boot 를 기준으로 각각 Error Code 로 발생한 Exception 을 Handling 해 Response 를 return 하는 코드를 작성해보겠습니다.
이렇게 custom 한 Error Code 를 response 값으로 내려주고 싶습니다.


<center>
<!-- 이미지 생략 -->
</center>
Spring Boot 는 전역 예외 처리를 적용할 수 있는 `@ControllerAdvice`와 `@RestControllerAdvice` 에노테이션을 제공하고 있습니다.
공식 문서에 따르면 `RestControllerAdvice`는 `@ControllerAdvice`에 `@ResponseBody`가 포함된 개념입니다.
즉 `@RestControllerAdvice`로 선언하면 리턴 값을 응답 값의 body 형태로 전달합니다.


> A convenient base class for @ControllerAdvice classes that wish to provide centralized exception handling across all @RequestMapping methods through @ExceptionHandler methods.
This base class provides an @ExceptionHandler method for handling internal Spring MVC exceptions. This method returns a ResponseEntity for writing to the response with a message converter, in contrast to DefaultHandlerExceptionResolver which returns a ModelAndView.

`ResponseEntityExceptionHandler`는 Spring MVC 에서 발생할 수 있는 예외들을 미리 Handling 해놓은 클래스로,
`@ExceptionHandler` 메서드를 통해 모든 `@RequestMapping` 메서드에 걸쳐 중앙 집중식 예외 처리를 제공하려는 `@ControllerAdvice` 클래스를 위한 기본 클래스입니다. 

이 기본 클래스는 내부 Spring MVC 예외를 처리하기 위해 `@ExceptionHandler` 메서드를 제공합니다. 
이 메서드는 ModelAndView 를 반환하는 `DefaultHandlerExceptionResolver`와 달리 메시지 변환기를 사용하여 응답에 쓰기 위한 ResponseEntity 를 반환합니다.


따라서 다음 클래스를 extends 하는 방식을 통해 Exception 발생 시 제어한 ErrorCode 를 클라이언트에 원하는 Response 값으로 전달할 수 있습니다.

```
@RestController
@ControllerAdvice
public class CustomizedResponseEntityExceptionHandler extends ResponseEntityExceptionHandler {

    // 제어한 ApiException 관련 response Handling
    @ExceptionHandler(ApiException.class)
    public ResponseEntity<ErrorResponse> codeExceptionHandler(
            ApiException e, HttpServletRequest request) {
        ErrorCode errorReason = e.getErrorReason();
        ErrorResponse errorResponse =
                new ErrorResponse(errorReason, request.getRequestURL().toString());
        return ResponseEntity.status(HttpStatus.valueOf(200))
                .body(errorResponse);
    }

   // 제어한 AuthException 관련 response Handling
    @ExceptionHandler(AuthException.class)
    public final ResponseEntity<ErrorResponse> handleAuthException(AuthException ex, HttpServletRequest request) {
        ErrorCode errorReason = ex.getErrorReason();
        ErrorResponse errorResponse =
                new ErrorResponse(errorReason, request.getRequestURL().toString());
        return ResponseEntity.status(HttpStatus.valueOf(200))
                .body(errorResponse);
    }
    
    // 그 외 발생하는 Exception 관련 response Handling
    @ExceptionHandler(Exception.class)
    public final ResponseEntity<ErrorResponse> handleAllExceptions(Exception ex, WebRequest request) {
        ErrorResponse exceptionResponse =
                new ErrorResponse(false, 500,
                        INTER_SERVER_ERROR.getErrorCode(),
                        ex.getMessage(),
                        new Date(), request.getDescription(false));

        return ResponseEntity.status(HttpStatus.valueOf(200))
                .body(exceptionResponse);
    }
    
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(MethodArgumentNotValidException ex,
                                                                  HttpHeaders headers,
                                                                  HttpStatus status,
                                                                  WebRequest request) {
        ErrorResponse exceptionResponse = new ErrorResponse(false,
                INTER_SERVER_ERROR.getStatus(),
                INTER_SERVER_ERROR.getErrorCode(),
                "Validation Failed",
                new Date(),
                ex.getBindingResult().toString());

        return new ResponseEntity(exceptionResponse, HttpStatus.BAD_REQUEST);
    }
    
    ...
    
}    
    
```
이로써 우리가 제어하고 싶던 ErrorCode Response 를 핸들링 할 수 있게 되었습니다.

하지만 일반적으로 예외는 서버로 보낸 요청을 controller 가 받은 후 로직을 처리하는 과정에서 발생합니다. 
<br> 즉, 일단 요청이 컨트롤러에 도달한 후 예외 발생 및 처리를 하는 과정으로 작동합니다.

<center>
<!-- 이미지 생략 -->
</center>

하지만 Spring Boot Security는 요청이 controller 에 도달하기 전에 `filterChain`에서 예외를 발생시킵니다. 
즉, Spring Boot Security 관련 Exception 은 컨트롤러에서 발생하는 예외를 처리하는 `@ControllerAdvice`로 제어가 불가합니다. 

따라서 Spring Boot Security 관련 Exception 도 같은 Response 방식으로 통일해주기 위한 작업을 추가적으로 진행하겠습니다.

Filter에서 발생하는 Exception 중 `ExpiredJwtException`, `JwtException`, `IllegalArgumentException`과 같이 JWT 관련 Exception 은 이미 처리 되어 있어서,
해당 블로그에서는 추가로 작성하였던 인증, 인가 관련 Exception 별도 처리에 대해 서술하겠습니다.

`ExceptionTranslationFilter`란 `FilterSecurityInterceptor` 실행 중 발생할 수 있는 인증 예외인 `AuthenticationException`과 권한 예외인 `AccessDeniedException`을 처리하는 필터입니다.

> Handles any AccessDeniedException and AuthenticationException thrown within the filter chain.
This filter is necessary because it provides the bridge between Java exceptions and HTTP responses.
> It is solely concerned with maintaining the user interface.
> This filter does not do any actual security enforcement.


<!-- 이미지 생략 -->

우선적으로 `AuthenticationException`를 handling 해보겠습니다.

먼저, `AuthenticationEntryPoint`를 implements 합니다.
`AuthenticationEntryPoint`는 클라이언트로부터 자격 증명을 요청하는 HTTP 응답을 보내는데 사용됩니다.

```
@Component("customizedAuthenticationEntryPoint")
public class CustomizedAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Autowired
    @Qualifier("handlerExceptionResolver")
    private HandlerExceptionResolver resolver;

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) {
        resolver.resolveException(request, response, null, authException);
    }
}
```

이제 `AccessDeniedHandler`를 implements 해 같은 방식으로 `AccessDeniedException`도 handling 해주겠습니다.
`AccessDeniedHandler`는 접근 불가능한 url에 대한 에러를 처리 하는데 사용됩니다.
```
@Slf4j
@RequiredArgsConstructor
@Component("customizedAccessDeniedHandler")
public class CustomizedAccessDeniedHandler implements AccessDeniedHandler {

    @Autowired
    @Qualifier("handlerExceptionResolver")
    private HandlerExceptionResolver resolver;

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) {
        resolver.resolveException(request, response, null, accessDeniedException);
    }

}
```

`AuthenticationEntryPoint`는 빈으로 등록한 authEntryPoint 를,
`AccessDeniedHandler`는 빈으로 등록한 accessDeniedHandler 를 주입 후,
`exceptionHandling()`을 `SecurityConfig`의 필터 체인에 추가 해줍니다.

```
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    @Autowired
    @Qualifier("customizedAuthenticationEntryPoint")
    AuthenticationEntryPoint authEntryPoint;
    
    @Autowired
    @Qualifier("customizedAccessDeniedHandler")
    AccessDeniedHandler accessDeniedHandler;
    
     @Bean
    @Order(2)
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http.httpBasic().disable()
                .csrf().disable()
                .formLogin().disable()
                .cors().and()
                ...
               .addFilterBefore(new JwtAuthenticationFilter(jwtTokenProvider), UsernamePasswordAuthenticationFilter.class)
               .exceptionHandling()
               .authenticationEntryPoint(authEntryPoint)
               .accessDeniedHandler(accessDeniedHandler)
               .build();
               
  ...
   
```

위에서 작성 한 `CustomizedResponseEntityExceptionHandler`에 `AccessDeniedException`와 `AuthenticationException`을 Handling 하는 메서드를 추가적으로 작성해줍니다.

```
    @ExceptionHandler(AuthenticationException.class)
    public ResponseEntity<ErrorResponse> handleAuthenticationException(Exception e) {

        ErrorResponse errorResponse = new ErrorResponse(
                false,
                HttpStatus.UNAUTHORIZED.value(),
                HttpStatus.UNAUTHORIZED.toString(),
                "Authentication failed");
        return ResponseEntity.status(HttpStatus.valueOf(200))
                .body(errorResponse);
    }
    
        @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDeniedException(Exception e) {

        ErrorResponse errorResponse = new ErrorResponse(
                false,
                HttpStatus.FORBIDDEN.value(),
                HttpStatus.FORBIDDEN.toString(),
                "Access Denied");
        return ResponseEntity.status(HttpStatus.valueOf(200))
                .body(errorResponse);
    }    
```

번외로 다음과 같이 `CustomizedAccessDeniedHandler`, `CustomizedAuthenticationEntryPoint` 내부에
직접 response 값을 handling 해 작성 후
SecurityConfiig 에 ObjectMapper 와 함께 빈으로 등록 후 filterChain 에 추가하는 방법도 존재합니다.

```
@Component
@RequiredArgsConstructor
public class CustomizedAccessDeniedHandler implements AccessDeniedHandler {

    private final ObjectMapper objectMapper;
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {

        String responseContent = objectMapper.writeValueAsString(new ErrorResponse(false,403, "Access Denied", null));
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding("UTF-8");
        response.getWriter().write(responseContent);

   }   
}
```

```
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

	private final ObjectMapper objectMapper; 

	@Bean 
    public AccessDeniedHandler accessDeniedHandler(){
        return new CustomizedAccessDeniedHandler(objectMapper);
    }
    ...
    
    @Bean
    @Order(2)
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http.httpBasic().disable()
                .csrf().disable()
                .formLogin().disable()
                .cors().and()
                ...
               .addFilterBefore(new JwtAuthenticationFilter(jwtTokenProvider), UsernamePasswordAuthenticationFilter.class)
               .exceptionHandling()
               .accessDeniedHandler(accessDeniedHandler())
               .build();
    
```


다음과 같은 과정을 통해 전역적으로 ErrorCode 및 관련 Response 값이 처리된 것을 확인할 수 있습니다.



<!-- 이미지 생략 -->

<!-- 이미지 생략 -->

## 문서화 방법 : Swagger vs RestDocs

---

이렇게 제어한 Error Response 를 문서화 하려 합니다.
API 를 문서화 하기 위한 대표적인 방안으로 Swagger 와 RestDocs 를 꼽을 수 있습니다.

Swagger 는 Postman 와 같이 테스팅이 가능하다는 장점이 있지만, 프로덕션 코드에 추가되기에 코드 침투적입니다. 
따라서 새로운 코드 작성 시 Swagger 를 위한 코드도 추가로 작성해주어야 한다는 단점을 가지고 있습니다. 

<https://swagger.io/>


RestDocs 는 Test 코드가 강제되어 문서의 신뢰성이 높다는 특징을 가지고 있습니다. 
다만 문서를 커스터마이징 하려면 AsciiDoc 문법을 알아야 수월하고, Swagger 와 달리 문서 상에서 즉석으로 API 테스트가 불가합니다. 

<https://docs.spring.io/spring-restdocs/docs/current/reference/htmlsingle/>


| 비고 | RestDocs                                     | Swagger                                                                                     |
| --- |---------------------------------------------|----------------------------------------------------------------------------------------------|
| 장점 | 프로덕트 코드에 영향이 없다.<br/> 테스트 코드 성공 시 문서가 작성된다. | API 테스팅 기능을 제공한다. <br/> 적용이 상대적으로 쉽다.                                                        |
 | 단점 | 적용이 번거롭다. <br/> API 테스팅이 불가하다.              | 프로덕트 코드에 Swagger 적용을 위한 어노테이션을 추가해야 한다. <br/> 프로덕트 코드와 동기화가 안될 수도 있거나 UI 상에서 자체적 오류가 있을 수 있다. |


각각의 장단점에 따라 상황에 맞는 문서화 방법을 골라 사용하면 좋을 것 같습니다.
혹은 `restdocs-api-spec` 라이브러리를 사용해 두 문서화 방식을 조합 하는 방식을 통해 사용하는 방법도 있습니다. 
해당 라이브러리 사용 시 RestDocs 와 같이 테스트 코드 통과 하게 되면 OpenApi 스펙을 얻어 이를 통해 Swagger-UI 문서를 띄워 테스트할 수 있는 환경을 제공 받을 수 있습니다.


## Swagger 에서의 Error Response 문서화

---

제어한 Error Code 의 모든 경우의 수가 아닌, Controller 별로 가능한 경우의 수 별로 제어해 문서화 하고 싶습니다.
`@ApiResponse`를 사용하면 HTTP 상태 코드 별 반환할 구체적인 응답을 설명할 수 있지만, Enum 으로 기껏 작성한 ErrorCode 를 이중으로 작성해야 하며
ErrorCode 값을 변경 시 사용한 곳에 찾아가 해당 값을 모두 변경 해주어야 하는 번거로움이 있습니다.

```
@ApiResponses(value = {
        @ApiResponse(code = 200, message = "OK", response = CustomerResponse.class),
        @ApiResponse(code = 400, message = "Invalid ID supplied"),
        @ApiResponse(code = 404, message = "Customer not found"),
        @ApiResponse(code = 500, message = "Internal server error", response = ErrorResponse.class)})
```


 다음과 같은 단점을 보완하고자 custom annotation 을 생성하여 controller 별 exception 발생 가능한 error code 를 작성해주었습니다. 

```
@Target({ANNOTATION_TYPE, METHOD, TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface ErrorResponseGroup {
    ErrorCode[] value();
}
```

```
@Target({METHOD, TYPE, ANNOTATION_TYPE, FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface ErrorResponses {
    ErrorResponseGroup[] groups() default {};
}
```
이때 `@Target`은 해당 애노테이션의 적용 대상을 제어합니다.<br>
또한 애노테이션에는 런타임 보존 기능(`@Retention(RetentionPolicy.RUNTIME)`)이 있어야 합니다. 그렇지 않으면 swagger 에는 보이지 않습니다.
`@Retention` 애노테이션은 애노테이션의 라이프 사이클 즉, 애노테이션이 언제까지 유효할지를 제어해줍니다.

이렇게 생성한 custom annotation 을 controller annotation 으로 작성해주면 생성한 api 별 가능한 error code 경우의 수를 제어할 수 있습니다.
다음은 그 예시입니다. 

```
@ErrorResponses(groups = {@ErrorResponseGroup({NO_RESERVE_EXIST, ALREADY_EXTEND_ERROR})})
```


사내 포털 는 현재 Springfox 를 기준으로 프로젝트가 구성되어 있습니다.

Springfox 라이브러리를 기준으로 Swagger 상에 에러코드를 제어해 문서화 하는 방식에 대한 문서가 구글링 해도 자세히 서술되지 않는 경우가 많아 구현에 있어 어려움을 겪은 바가 있어 해당 부분에 포커스해 작성하겠습니다.

Springfox 는 `OperationOperationBuilderPlugin`라는 interface 를 제공하고 있습니다.


> Implement this method to override the Operation using the OperationBuilder available in the context
Params:
context – - context that can be used to override the parameter attributes
See Also:
springfox.documentation.service.Operation, springfox.documentation.builders.OperationBuilder

context에서 사용 가능한 `OperationBuilder`를 사용하여 Operation을 재정의하려면 이 메서드를 implements 합니다.
따라서 해당 interface 를 implements 해 원하는 response 값을 문서화 하도록 변경해보겠습니다.

```
@Component
@Slf4j
@Order(SwaggerPluginSupport.SWAGGER_PLUGIN_ORDER)
public class OperationBuilderPluginImpl implements OperationBuilderPlugin {

    @Override
    public void apply(OperationContext context) throws
            UnsupportedOperationException, ClassCastException, IllegalArgumentException {
        // context 에서 custom annotation 이 있는지 찾아 가져옵니다
        Optional<ErrorResponses> methodAnnotation = context.findAnnotation(ErrorResponses.class);
        
        // 가져온 annotation 을 기반으로 이를 response 에 추가한 response set type 을 가져옵니다.
        Set<Response> responses = new HashSet<>(this.addErrorCodes(context, methodAnnotation));
        
        // custom 한 response 을 context operation 에 build 해줍니다. 
        context.operationBuilder().responses(responses);
    }
}
```
다음과 같이 기존 `apply()` method 를 @Override 하는 방식을 통해 우리가 원하는 ErrorCode 를 response 값에 같이 문서화해줄 수 있을 것 같습니다.

이제 기존 response 값에 errorCode 관련 response 추가해주는 로직을 작성해보겠습니다.

```
    @SuppressWarnings({"CyclomaticComplexity", "NPathComplexity"})
    private Set<Response> addErrorCodes(OperationContext context, Optional<ErrorResponses> methodAnnotation){
        Set<Response> responses = new HashSet<>();
        
        // custom 한 annotation 존재 시 
        methodAnnotation.ifPresent(errorResponses -> Arrays.stream(errorResponses.groups()).forEach(
                code -> { 
                // @errorResponses group 의 value 값으로 해당 Errorcode 배열 추출
                    ErrorCode[] errorCodes = code.value();
                    
                    // errorCodes response 값으로 변환 후 add
                    for(ErrorCode errorCode : errorCodes){
                        responses.add(this.addErrorCodeToContext(errorCode, context));
                    }

                }
        ));
        return responses;
    }
```

```
    private Response addErrorCodeToContext(ErrorCode errorCode, OperationContext context){
        // 기존 OperationContext 기반으로 responseContext 객체 생성
        ResponseContext responseContext = new ResponseContext(
                context.getDocumentationContext(),
                context);
        
        // response 값에 추가할 error 관련 response 객체 생성       
       ErrorResponse errorResponse =
                new ErrorResponse(errorCode, responseContext.getOperationContext().requestMappingPattern());
                
        // 생성한 response 객체 examples 에 add
        List<Example> examples = new ArrayList<>();
        examples.add(new ExampleBuilder()
                .mediaType("*/*")
                .description(errorResponse.getReason())
                .summary(errorResponse.getReason())
                .id(errorResponse.getCode())
                .value(errorResponse).build());

        // 추가한 examples 와 함께 반환할 response build 후 반환
        Response apiResponse = new ResponseBuilder()
                .code(errorCode.getErrorCode())
                .description(errorCode.getCodeMessage())
                .examples(examples)
                .build();

        return apiResponse;
    }
```


이제 Swagger 상에 우리가 제어한 ErrorCode 가 Response 값으로 추가된 것을 확인할 수 있습니다.

<center>
<!-- 이미지 생략 -->
</center>

Springdoc-openapi 는 swagger-ui 상에 제공할 값을 커스터마이징 할 수 있는 `customize()` 메소드를 지원합니다.
따라서 다음 메소드를 활용해 swagger 를 보다 간편하고 유연하게 커스터마이징이 가능합니다. 

---

지금까지 Spring Boot 에서 Error Response 에 관한 제어와 Swagger 상의 문서화에 대해 간략하게 알아봤습니다.
Error Response 관련 테스팅을 진행 하면서 Spring Boot 에서의 예외 처리 흐름에 대해서도 알아볼 수 있어 뜻깊은 시간이었습니다.

천고마비와 독서의 계절 🍁 이제 드디어 가을인 건지 날이 부쩍 추워졌는데 다들 감기 조심 하시고 
<br> 남은 올 한 해 잘 마무리 하시길 바랍니다! 

지금까지 읽어주셔서 감사합니다 🙇🏻‍♀️ 


