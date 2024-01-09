## Eureka Server

**Eureka Server**는 **Eureka client**들을 등록하고 이를 한번에 관리할 수 있다. **MSA 에서는 도메인 또는 IP 정보를 Eureka Server에 등록된 노드들의 정보를 가지온 뒤, Api Gateway 서버를 통해서 원하는 위치로 라우팅해준다.** 

**Dependency**

```groovy
implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-server'
	implementation 'org.springframework.boot:spring-boot-starter-security'
	implementation group: 'org.springframework.boot', name: 'spring-boot-starter-actuator', version: '2.7.5'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	testImplementation 'org.springframework.security:spring-security-test'
```

**버전 명시(build.gradle)**

```groovy
ext {
	set('springCloudVersion', "2021.0.5")
}
dependencyManagement {
	imports {
		mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
	}
}
```

**단일 구성**

보통의 환경에서는 유레카 서버를 단독으로 쓰지 않고 두 개 이상으로 설치하여 가용성을 높인다. 하나의 **유레카 서버가 문제가 발생해도** 다른 하나가 있기 때문에 문제 상황을 막을 수 있다. 또 로드밸런싱 기능을 통해 대용량 트래픽에도 과부화를 막을 수 있게 한다. 

```yaml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

--- or  ---
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
```

### <속성>

- *eureka.client.fetch-registry* : 유레카 Client는 유레카 Server에서 등록된 서비스들의 정보를 가져와서 로컬에 캐시한다. 30초 마다 가져와 업데이트 해준다. 단일 구성의 경우, 가져올 유레카 서버가 없기 때문에 설정을 false로 한다. (default : true)
- *eureka.client.resterWithEureka* : 유레카 서버 등록 여부
- 유레카 클라이언트는 자신의 정보를 유레카에 등록해야 하고 등록된 registry를 가져와야 한다. 하지만 유레카 서버는 자신을 등록할 필요도 없고 registry를 가져올 필요도 없기 때문에 false로 설정
- *eureka.server.enable-self-preservation* : 운영 환경에서는 **무조건 true**, 이 설정은 유레카 Client가 네트워크 장애로 연결이 끊겼을 때, 지정된 시간안에 하트비트가 들어오지 않으면 서비스를 바로 제거한다. 만약 **false로 설정하면 바로 서비스를 제거한다.**
- *eureka.server.expected-client-renewal-interval-seconds* : 기본 값은 **90**이다. 마지막 하트비트를 받고 기다리는 초를 의미
- *eureka.server.eviction-interval-timer-in-ms :* 기본 값은 60초이고 해당 시간이 지나면 종료된 클라이언트를 제거하도록 한다.

**다중 구성**

유레카 서버는 기본적으로 다중 구성을 사용한다. 각 유레카 서버는 `eureka.instance.hostname` 을 사용하여 구분한다. peer1 유레카 서버가 같이 사용할 유레카 서버인 peer2 를 설정하려면 `eureka.client.serviceUrl.defaultZone` 에 설정하면 된다. 

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: https://peer1/eureka/,http://peer2/eureka/,http://peer3/eureka/

---
spring:
  profiles: peer1
eureka:
  instance:
    hostname: peer1

---
spring:
  profiles: peer2
eureka:
  instance:
    hostname: peer2

---
spring:
  profiles: peer3
eureka:
  instance:
    hostname: peer3
```

- hostname은 `/etc/hosts` 에 등록된 도메인을 기준으로 합니다.
- 도메인이 아닌 Ip를 기준으로 하고 싶다면 `eureka.instance.preferIpAddress : true` 로 설정하면 된다.

**보안 구성**

만약 eureka server에 Spring Security를 구성한다면 ‘/eureka/**’ 의 csrf 검증을 무시하도록 설정해야 한다. security 설정을 하면 기본적으로 모든 요청에 CSRF 토큰을 전달하는데. 기본적으로 유레카 클라이언트들은 CSRF 토큰을 포함하지 않는다. 그러니 **ignoring** 설정한다. 

```java
@EnableWebSecurity
public class WebSecurityConfig {

    @Bean
    public SecurityFilterChain configure(HttpSecurity http) throws Exception{
        http.csrf().ignoringAntMatchers("/eureka/**");
        return http.build();
    }
}
```

## Eureka Client

**Dependency**

```groovy
implementation 'org.springframework.boot:spring-boot-starter-web'
implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client:3.1.4'
..
```

**기본 구성**

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
```

### <속성>

*eureka.client.healthcheck.enabled* : 기본 값은 true이고, 이 값에 따라 유레카 서버에게 하트비트를 보내는지 여부를 체크할 수 있다. actuator의 health check를 사용해서 확인한다. 

*eureka.instance.hostname* : hosts에 저장된 **eureka server**의 hostname과 일치하도록 설정한다. 

*eureka.instance.prefer-ip-address* : hosts에 eureka server의 호스트명이 등록되어 있지 않을 경우가 있을 수 있기 때문에 ip를 우선적으로 찾을 수 있도록 한다. 

**참조**

[Spring Cloud Netflix](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/#why-is-it-so-slow-to-register-a-service)

https://assu10.github.io/dev/2020/12/05/spring-cloud-eureka-configuration/

[](https://www.baeldung.com/eureka-self-preservation-renewal)