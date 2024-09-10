---
title: Gateway 서버에서 Rest API 호출을 GRPC 호출로 변환하기
date: 2024-09-05
categories: [Server]
tags: [Grpc, Gateway Server]
---

안녕하세요! 😊 요즘 일상 블로그도 안쓰기도 하고 기왕이면 최대한 의미있게 쓰고 싶다는 생각에
개발 블로그 쓰기 시작하는 데에도, 완성하는 데에도 오래 걸려서 정말 오랜만에 블로그를 쓰러 앉았는데요!

이전 장애인 취업 지원 플랫폼 개발에서, 모놀리식 구조를 MSA로 변경하기로 하면서
하나의 이슈 사항 중 하나가, 호기롭게 `grpc` 도입을 추진했지만
프론트 분들은 다들 안드로이드와 크틀린 자체가 처음이셨고, 이 때문에 `grpc` 학습까지 이루어진다면
저희 프로젝트에서의 러닝 커브가 너무 커질 것 같아 이 부분이 고민이었습니다

그래서 프론트에서 호출을 보낼 때는 `rest api` 로 보내는 걸로 방식을 변경하게 되면서 <br>
그럼 기존 로직을 `rest api` 형태로 각각의 서비스에서 감싸줄 것이냐, 혹은 `gateway`에서 이 과정을 변환해줄 것이냐
하는 제 고민이 생겼습니다..

> 1. 기존 로직을 `rest api` 형태로 각각의 서비스에서 감싸서 내보내기
> 2. `gateway`에서 `rest api` 호출을 `grpc` 로 변환해 호출을 넘겨주고 넘겨받기
{: .prompt-tip }

다음 두 가지의 선택지 중에서 어떤 선택지를 고를지 고민했던 과정들을 공유해볼까 합니다

일단 제가 맡은 Auth 쪽 파트를 Controller 로 감싸줬습니다

```java
    @Operation(summary = "로그인", description = """
            아이디, 비밀번호 기반으로 paseto 토큰 발급
            """)
    @PostMapping("/create")
    public ResponseEntity<String> createAuth(@RequestBody CreateAuthRequest request) throws InvalidProtocolBufferException {
        CreateTokenRequest tokenRequest = CreateTokenRequest.newBuilder()
                .setAuth(AuthData.newBuilder()
                        .setId(request.getId())
                        .setPw(request.getPw())
                        .setUserType(UserType.valueOf(request.getUserType().getName()))
                        .build())
                .build();

        CreateTokenResponse response = authGrpcClient.createAuth(tokenRequest);

        String jsonResponse = JsonFormat.printer().print(response);
        HttpHeaders headers = new HttpHeaders();
        headers.add(HttpHeaders.CONTENT_TYPE, "application/json");

        return new ResponseEntity<>(jsonResponse, headers, HttpStatus.OK);
    }
```
<span style="color:gray">다크모드가 더 눈에 안좋다길래 라이트로 바꿨는데.. 인텔리제이 라이트모드는 봐도봐도 눈에 안익네요</span>

그리고 성능 비교를 위해, `gateway` 서버 쪽에 rest api 호출을 grpc 호출로 변환해주는 로직을 작성해주었습니다
다만, 이게 초기 작성 버전이 맞는지는 모르겠습니다.. 여러 번의 변환이 있었는데
local history 에 남아있지 않네요 😓
```java
@Slf4j
@Component
@RequiredArgsConstructor
public class GrpcRequestConverterFilter implements GatewayFilter, Ordered {

    // 서비스 인스턴스를 찾기 위한 로드 밸런서를 생성하는 데 사용합니다
    private final ReactiveLoadBalancer.Factory<ServiceInstance> loadBalancerFactory;

    @Override
    // 들어오는 요청을 처리합니다. 이때 ServerWebExchange exchange 는 요청 및 응답을 포함하는 개체이며, GatewayFilterChain chain 는 다음 필터를 호출하기 위한 체인 객체입니다. 
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 요청의 URL 경로(path)와 쿼리 매개변수(queryParams)를 가져옵니다
        String path = exchange.getRequest().getURI().getPath();
        Map<String, String> queryParams = exchange.getRequest().getQueryParams().toSingleValueMap();

        // chooseInstance 메서드를 사용하여 "AUTH-SERVICE"라는 서비스의 인스턴스를 선택합니다
        return chooseInstance("AUTH-SERVICE")
                .flatMap(instance -> {
                    if (instance == null) {
                        // 선택된 인스턴스가 없다면, 로그를 남기고 다음 필터로 요청을 전달합니다
                        log.error("No available instances for AUTH-SERVICE");
                        return chain.filter(exchange);
                    }

                    // 인스턴스 메타데이터에서 gRPC 포트를 가져옵니다. 포트가 없다면 로그를 남기고 다음 필터로 요청을 전달합니다.
                    // 사전 작업으로 gRPC 의 포트 번호를 eureka 에 등록할 때 메타데이터로 등록하게끔 해두었습니다
                    Map<String, String> metadata = instance.getMetadata();
                    String grpcPort = metadata.get("gRPC_port");

                    // 인스턴스 메타데이터에서 gRPC 포트를 가져옵니다. 포트가 없다면 로그를 남기고 다음 필터로 요청을 전달합니다.
                    if (grpcPort == null) {
                        log.error("No gRPC port found in service instance metadata");
                        return chain.filter(exchange);
                    }

                    // makeGrpcCall 메서드를 호출하여 gRPC 호출을 수행하고, 결과를 반환합니다
                    return makeGrpcCall(path, queryParams, instance.getHost(), Integer.parseInt(grpcPort))
                            .flatMap(responseBytes -> {
                                exchange.getResponse().getHeaders().setContentType(MediaType.APPLICATION_JSON);
                                return exchange.getResponse().writeWith(
                                        Mono.just(exchange.getResponse().bufferFactory().wrap(responseBytes))
                                );
                            })
                            .doFinally(signalType -> {
                            });
                })
                .switchIfEmpty(chain.filter(exchange))
                // 오류가 발생하면 로그를 남기고 다음 필터로 요청을 전달합니다
                .onErrorResume(e -> {
                    log.error("Exception occurred during gRPC call", e);
                    return chain.filter(exchange);
                });
    }

  /**
   * gRPC 서비스를 호출하는 역할을 하는 메서드
   */
    private Mono<byte[]> makeGrpcCall(String path, Map<String, String> queryParams, String host, int grpcPort) {
        // ManagedChannel을 사용하여 gRPC 서버와 통신할 채널을 생성합니다.
        ManagedChannel channel = ManagedChannelBuilder.forAddress(host, grpcPort)
                .usePlaintext()
                .build();
        AuthServiceGrpc.AuthServiceStub stub = AuthServiceGrpc.newStub(channel);

        // path에 /auth-service/create-auth 가 포함된다면 인증 생성 요청을 보냅니다
        if (path.contains("/auth-service/create-auth")) {
            // grpc proto 호출 객체로 변환
            AuthData authData = AuthData.newBuilder()
                    .setId(queryParams.get("id"))
                    .setPw(queryParams.get("pw"))
                    .build();
            CreateTokenRequest request = CreateTokenRequest.newBuilder()
                    .setAuth(authData)
                    .build();

            // grpc 서버에 비동기 통신
            return callCreateAuth(stub, request);

        } // path에 /auth-service/verify-auth 가 포함된다면 인증 확인 요청을 보냅니다
        else if (path.contains("/auth-service/verify-auth")) {
          // grpc proto 호출 객체로 변환
            VerifyTokenRequest request = VerifyTokenRequest.newBuilder()
                    .setToken(queryParams.get("token"))
                    .build();

            // grpc 서버에 비동기 통신
            return callVerifyAuth(stub, request);

        } else {
            return Mono.just(new byte[0]);
        }
    }

    // 인증 생성을 위해 gRPC 서버와 비동기 통신을 수행
    private Mono<byte[]> callCreateAuth(AuthServiceGrpc.AuthServiceStub stub, CreateTokenRequest request) {
        return Mono.create(sink -> {
            StreamObserver<CreateTokenResponse> responseObserver = new StreamObserver<>() {
                @Override
                // 응답이 도착했을 때 호출
                public void onNext(CreateTokenResponse response) {
                    try {
                        String jsonResponse = convertToJson(response);
                        byte[] responseBytes = response.toByteArray();
                        sink.success(jsonResponse.getBytes(StandardCharsets.UTF_8));
                    } catch (Exception e) {
                        sink.error(e);
                    }
                }

                @Override
                // 에러 발생 시 호출
                public void onError(Throwable t) {
                    sink.error(t);
                }

                @Override
                // 작업이 완료되었을 때 호출
                public void onCompleted() {
                }
            };

            stub.createAuth(request, responseObserver);
        });
    }

    // 인증 확인을 위해 gRPC 서버와 비동기 통신을 수행
    private Mono<byte[]> callVerifyAuth(AuthServiceGrpc.AuthServiceStub stub, VerifyTokenRequest request) {
        return Mono.create(sink -> {
            StreamObserver<VerifyTokenResponse> responseObserver = new StreamObserver<>() {
                @Override
                // 응답이 도착했을 때 호출
                public void onNext(VerifyTokenResponse response) {
                    try {
                        byte[] responseBytes = response.toByteArray();
                        sink.success(responseBytes);
                    } catch (Exception e) {
                        sink.error(e);
                    }
                }

                @Override
                // 에러 발생 시 호출
                public void onError(Throwable t) {
                    sink.error(t);
                }

                @Override
                // 작업이 완료되었을 때 호출
                public void onCompleted() {
                }
            };

            stub.verifyAuth(request, responseObserver);
        });
    }

    private String convertToJson(Message response) throws InvalidProtocolBufferException {
        return JsonFormat.printer().print(response);
    }

    // 주어진 serviceId에 대해 사용할 수 있는 서비스 인스턴스를 선택
    private Mono<ServiceInstance> chooseInstance(String serviceId) {
        // 로드 밸런서를 사용하여 서비스 인스턴스를 선택하고, 이를 비동기적으로 반환
        ReactorServiceInstanceLoadBalancer loadBalancer = (ReactorServiceInstanceLoadBalancer) loadBalancerFactory.getInstance(serviceId);
        return Mono.defer(() -> loadBalancer.choose()
                .map(response -> response.hasServer() ? response.getServer() : null));
    }

    @Override
    // 필터가 낮은 우선순위를 갖도록 (-2 설정)
    public int getOrder() {
        return -2;
    }
}
```
GatewayFilter와 Ordered 인터페이스를 구현한 클래스로 `GrpcRequestConverterFilter` 를 작성해주었습니다.
Spring Cloud Gateway의 필터로 동작하여 gRPC 요청 변환을 담당하도록 합니다.
> 코드의 상세 설명은 주석을 달아주었습니다
{: .prompt-danger }


그리고 이부분을 `RouteConfig` 쪽에 붙여주었습니다
```java
@Configuration
@AllArgsConstructor
public class RouteConfig {

    private final GrpcRequestConverterFilter grpcRequestConverterFilter;

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
                .route("auth-service", r -> r.path("/auth-service/**")
                        .filters(f -> {
                            f.filter(grpcRequestConverterFilter);
                            return f;
                        })
                        .uri("lb://AUTH-SERVICE"))
                .route("member-service", r -> r.path("/member-service/**")
                        .filters(f -> {
                            f.filter(grpcRequestConverterFilter);
                            return f;
                        })
                        .uri("lb://MEMBER-SERVICE"))
                .build();
    }
}

```
그리고 두 방식을 경주를 시켜보았습니다 😃 누가누가 더 빠르나 🐇🐢

<img width="868" alt="" src="https://github.com/user-attachments/assets/e7ba7942-f8fc-4cfd-8621-507f585e6e2b">
  <p align="center" style="color:rgb(128,128,128)">
   ☝🏻 gateway에서 변환해주는 방식
  </p>
<img width="876" alt="" src="https://github.com/user-attachments/assets/174bc8df-29eb-4c80-ade4-e08863e7d928">
  <p align="center" style="color:rgb(128,128,128)">
   ☝🏻 controller로 감싼 방식
  </p>
postman으로 동일하게 50번의 호출과 평균을 낸 결과 정답은 controller 로 감싼 방법이 더 빨랐습니다..
<br>이때부터 고민이 되기 시작했습니다 각각의 방식에는 장단점이 존재해보였어요

일단 controller로 감싼 방식의 장점은 명확하고 너무 강력했습니다 <b>"더 빠르다"</b> <br>
아무래도 MSA는 서버를 하나는 더 거쳐 호출이 보내지기 때문에, 이미 느려진다는 리스크를 갖고 있어 빠른게 최우선이었습니다

다만 gateway에서의 변환을 사용한다면 모든 서비스에서 controller로 감싸거나 크게 방식을 변환할 필요가 없었습니다
팀원분들도 학습을 위해 grpc를 사용하고 싶어 하셨고요
그리고 물론 gateway 단에서 이긴 하지만, grpc의 장점을 살려 비동기 처리가 가능하고, 
gateway에서 정말 중간다리 역할을 해주면서 에러나 끝난 시점의 동작도 제어가 가능하다는 점이 매력으로 다가왔습니다

<b>그래서 고민합니다.. gateway에서 시간을 단축할 순 없을까? 🤔</b>

시간 단축을 위해 시간이 느려질 수 있는 몇 가지 지점을 찾고 해결 가능한 걸 해결해보기로 하면서
이에 대한 몇 가지 방안을 생각해냅니다


>1. <b>gRPC 채널 재사용</b>: 현재 코드에서는 매번 `makeGrpcCall` 메서드가 호출될 때마다 새로운 `gRPC 채널`을 생성하고 있습니다.
>2. <b>불필요한 Mono 생성 제거</b>: `Mono` 객체를 생성하는 작업이 많아보였습니다. 가능하면 `Mono의 체인 형태`로 비동기 로직을 간결하게 작성할 수 있지 않을까 싶습니다.
>3. <b>gRPC Stub 재사용</b>: `gRPC Stub`도 매번 생성하고 있어 이 또한 채널과 마찬가지로 성능이 느려지는 원인으로 보입니다
>4. [부가] JSON 변환 비용 최적화: JSON 변환이 응답을 받을 때마다 일어나고 있었는데, 이것도 캐시된 프린터 객체를 사용하는게 낫지 않을까 싶습니다.
{: .prompt-warning }

따라서 이러한 방안을 해결하는 방안으로 코드를 리팩토링 하게 됩니다
> 이 또한 코드 상세 설명은 주석을 달아주겠습니다
{: .prompt-danger }

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class GrpcRequestConverterFilter implements GatewayFilter, Ordered {

    // 서비스 인스턴스를 찾기 위한 상수 또는 설정값들을 포함하는 객체 (-> chooseInstance 부분 constants 로 따로 관리)
    private final ConverterConstants converterConstants;
    // gRPC 통신을 위한 ManagedChannel 객체를 캐싱하는 맵 (키 : 인스턴스의 호스트와 포트를 결합한 문자열)
    private final Map<String, ManagedChannel> channelMap = new ConcurrentHashMap<>();
    // gRPC 호출을 위한 AuthServiceGrpc.AuthServiceStub 객체를 캐싱하는 맵
    private final Map<String, AuthServiceGrpc.AuthServiceStub> stubMap = new ConcurrentHashMap<>();

    // JSON 변환기 캐싱
    private static final JsonFormat.Printer JSON_PRINTER = JsonFormat.printer();

    @PreDestroy
    // Spring 애플리케이션이 종료되기 전에 호출하여 gRPC 채널이 남아 있는 경우 이를 모두 종료하여 리소스를 정리
    public void shutdown() {
        channelMap.values().forEach(channel -> {
            if (channel != null && !channel.isShutdown()) {
                channel.shutdown();
            }
        });
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();
        Map<String, String> queryParams = exchange.getRequest().getQueryParams().toSingleValueMap();

        return converterConstants.chooseInstance("AUTH-SERVICE")
                .flatMap(instance -> {
                    if (instance == null) {
                        log.error("No available instances for AUTH-SERVICE");
                        return chain.filter(exchange);
                    }

                    // 인스턴스의 호스트와 포트를 결합한 문자열을 생성해 인스턴스를 식별하기 위한 키로 사용
                    String instanceKey = instance.getHost() + ":" + instance.getPort();
                    
                    // 주어진 키가 stubMap에 없으면 새로운 gRPC stub을 생성하고, 있으면 기존 stub을 재사용
                    AuthServiceGrpc.AuthServiceStub stub = stubMap.computeIfAbsent(instanceKey, key -> {
                        ManagedChannel channel = ManagedChannelBuilder.forAddress(instance.getHost(), Integer.parseInt(instance.getMetadata().get("gRPC_port")))
                                .usePlaintext()
                                .build();
                        channelMap.put(instanceKey, channel);
                        return AuthServiceGrpc.newStub(channel);
                    });

                    return makeGrpcCall(path, queryParams, stub)
                            .flatMap(responseBytes -> {
                                exchange.getResponse().getHeaders().setContentType(MediaType.APPLICATION_JSON);
                                return exchange.getResponse().writeWith(
                                        Mono.just(exchange.getResponse().bufferFactory().wrap(responseBytes))
                                );
                            });
                })
                .switchIfEmpty(chain.filter(exchange))
                .onErrorResume(e -> {
                    log.error("Exception occurred during gRPC call", e);
                    return chain.filter(exchange);
                });
    }

    private Mono<byte[]> makeGrpcCall(String path, Map<String, String> queryParams, AuthServiceGrpc.AuthServiceStub stub) {
        return Mono.fromCallable(() -> {
            if (path.contains("/auth-service/create-auth")) {
                AuthData authData = AuthData.newBuilder()
                        .setId(queryParams.get("id"))
                        .setPw(queryParams.get("pw"))
                        .build();
                CreateTokenRequest request = CreateTokenRequest.newBuilder()
                        .setAuth(authData)
                        .build();

                return callCreateAuth(request, stub).block(); // 비동기 블록 대체

            } else if (path.contains("/auth-service/verify-auth")) {
                VerifyTokenRequest request = VerifyTokenRequest.newBuilder()
                        .setToken(queryParams.get("token"))
                        .build();

                return callVerifyAuth(request, stub).block(); // 비동기 블록 대체
            }
            return new byte[0];
        });
    }

    private Mono<byte[]> callCreateAuth(CreateTokenRequest request, AuthServiceGrpc.AuthServiceStub stub) {
        return Mono.fromCallable(() -> {
            //  gRPC 응답을 처리하는 StreamObserver의 구현체, SimpleStreamObserver 생성
            SimpleStreamObserver<CreateTokenResponse> observer = new SimpleStreamObserver<>();
            stub.createAuth(request, observer);
            return observer.getResponse(); // 응답 데이터 바로 반환
        });
    }

    private Mono<byte[]> callVerifyAuth(VerifyTokenRequest request, AuthServiceGrpc.AuthServiceStub stub) {
        return Mono.fromCallable(() -> {
            SimpleStreamObserver<VerifyTokenResponse> observer = new SimpleStreamObserver<>();
            stub.verifyAuth(request, observer);
            return observer.getResponse(); // 응답 데이터 바로 반환
        });
    }

    @Override
    public int getOrder() {
        return -2;
    }

    private static class SimpleStreamObserver<T extends Message> implements StreamObserver<T> { 
        
        private final CompletableFuture<byte[]> future = new CompletableFuture<>();

      /**
       * [중간 변경 사항]
       * 비동기 응답을 SimpleStreamObserver에서 JSON 변환과 MonoSink로 데이터를 전달하며 StreamObserver 내부에서 직접적으로 변환 작업을 수행하는 대신, 
       * 가능한 한 최소한의 작업만을 수행하고 나머지는 다른 비동기 작업으로 위임
       * 기존 코드 : sink.success(jsonResponse.getBytes(StandardCharsets.UTF_8));
       */
      public byte[] getResponse() {
            try {
                return future.get(); // 결과값을 동기식으로 반환
            } catch (InterruptedException | ExecutionException e) {
                throw new RuntimeException(e);
            }
        }

        @Override
        public void onNext(T response) {
            try {
                String jsonResponse = JSON_PRINTER.print(response);
                future.complete(jsonResponse.getBytes(StandardCharsets.UTF_8));
            } catch (Exception e) {
                future.completeExceptionally(e);
            }
        }

        @Override
        public void onError(Throwable t) {
            future.completeExceptionally(t);
        }

        @Override
        public void onCompleted() {
        }
    }
}
```
>1. <b>gRPC 채널 재사용</b>: `channelMap`을 사용하여 `ManagedChannel`을 캐싱함으로써, `stubMap`을 생성할 때 `ManagedChannel`을 매번 새로 생성하는 대신 `channelMap`에 저장된 채널을 재사용합니다
>2. <b>불필요한 Mono 생성 제거</b>: `Mono.defer`를 사용하여 비동기 작업을 지연 생성하고, `Mono.create` 대신 `Mono.fromCallable` 사용하여 불필요한 `Mono.create` 사용을 줄입니다. 
>3. <b>gRPC Stub 재사용</b>: `channelMap`과 유사하게 `stubMap`을 사용하여 `AuthServiceGrpc.AuthServiceStub`을 캐싱하고 재사용합니다. 
>4. [부가] JSON 변환 비용 최적화: `JsonFormat.printer()`를 클래스 레벨에서 `JSON_PRINTER`로 한 번만 생성하여 재사용합니다.
{: .prompt-info }

<img width="871" alt="" src="https://github.com/user-attachments/assets/4e959f3c-86ab-4cde-be5e-d57c10e1f9d3">


그래서 결국 더 빠른 gateway 에서의 변환을 만나볼 수 있었습니다 🥹 <br>
정말 토끼와 거북이의 경주 같네요 

다만 큰 문제는, <b>대체적으로 캐싱 전략으로 줄어든 시간이기에 초기 호출에서는 여전히 느릴 수 있습니다</b>

다만 이를 위해 사전에 초기화를 진행하게 되면 나중에 인스턴스나 grpc stub이 많아지면 메모리 사용량도 증가하고,
관리할 채널과 stub도 늘어나면서 네트워크 연결 수도 늘어나고 관리 오버헤드도 늘어나고 초기화에도 지연될 거고..
여러 문제로 인해 이건 LRU 같은 캐싱 전략을 사용할지, 최대 채널 수를 제한할지.. 동적으로 채널을 할당할지.. 등등 <br>
어떻게 하면 좋을지 고민해보고 나중에 다시 2탄으로 연재하거나 할 것 같습니다 🤦🏻‍♀️


이번 코드 작업은 정말 너무너무 재밌었습니다!
작업도 이렇게 해보면 되지 않을까? 하면서 순조롭고 빠르게 진행되었고
그래서인지 블로그도 앉은 자리에서 단숨에 써버렸네요!

아직 path로 받아오는 것도 body로 받아올 수 있게끔 고쳐야 하고 변경해야할 부분이 많지만, <br>
그래도 재밌는 개발 경험을 얻을 수 있어서 뜻깊었습니다 🔥☘️ <br>
그럼 다음에 또 재밌는 글로 찾아뵐 수 있으면 좋겠네요 🙃
