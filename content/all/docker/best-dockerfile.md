---
title: Dockerfile 최적화하기
date: 2026-04-28
tags:
  - docker
  - Dockerfile
  - 멀티모듈
  - multi-stage-build
  - 캐싱
---
# 1. 빌드 위치 선정
먼저 빌드 위치에 따라 두 가지 선택지가 존재한다.
1. 최상위 경로에 Dockerfile이 1개 위치한 경우
2. 각 모듈의 루트 경로에 Dockerfile이 위치한 경우

빌드 컨텍스트가 루트를 바라보는 전자의 경우 중복되는 설정을 관리하기 편하기도 하고 의존성 문제도 적겠지만, 모듈 하나의 작은 변경에도 전체 빌드 컨텍스트가 재전송되어 효율이 떨어질것이다. 또한 서비스마다의 빌드 조건들을 전부 개별 설정해주어야해서 응집도가 떨어지고 복잡해질것이다.  

현재 진행 중인 프로젝트는 독립적으로 실행 가능한 4개의 서비스로 구성된 멀티모듈 구조다. 각 서비스는 상호 의존성이 없고 별도의 공통 모듈도 존재하지 않아, **빌드 컨텍스트를 각 모듈로 한정하여 독립성을 유지하는 것이 유리**하다. 또한 모든 모듈이 실행 가능하므로 각 모듈별로 빌드 시 **꼭 필요한 파일만 포함함으로써 이미지 크기를 최적화하고 빌드 속도를 높일 수 있다.**
> 다만, 베이스 이미지 업데이트나 보안 패치 같은 공통 수정 사항 발생 시 모든 Dockerfile을 일일이 수정해야 하는 관리상의 번거로움이 존재한다. 이러한 문제는 CI/CD 파이프라인에서 빌드 프로세스를 자동화하여 해결해봐야겠다.

# 2. 이미지 생성
## 1) 개선 전; *가장 기본적인 Dockerfile*
이전 게시물에서 가장 기본적인 Dockerfile을 작성했었다.
```Dockerfile
FROM gradle:8.12-jdk17
WORKDIR /app

COPY . .

RUN ./gradlew :module-name:bootJar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "build/libs/app.jar"]
```
**빌드 결과 분석**
![[스크린샷 2026-05-06 오전 10.47.05.png]]
하나의 모듈만 빌드하더라도 첫 빌드 시 **약 6분**동안 **2.6GB**라는 어마어마한 용량의 이미지가 생성된다.  
이미지 생성은 성공했지만 Gradle 베이스 이미지를 포함한 모든 라이브러리를 처음 내려받기 때문에 시간도 오래걸렸을 뿐더러 빌드 도구(Gradle + JDK)와 소스 코드, 실행 결과물이 한 이미지에 몽땅 포함되어 굉장히 무겁다.
## 2) 1차 개선; *빌드 단계의 분리*
> [!tip] `docker history [image-name]`
> 특정 도커 이미지의 레이어 구성과 빌드 시 실행된 명령(RUN, COPY, LABEL 등)을 확인하여 ==각 레이어의 크기, 생성 시간, 명령어를 상세히 파악==할 수 있어 이미지 최적화 시 유용하다.

현재 이미지 대부분의 용량을 차지하는 원인들은 다음과 같았다.
```
CREATED BY                                      SIZE 
...
WORKDIR /home/gradle                            4.1kB
WORKDIR /app                                    8.19kB
RUN |1 GRADLE_DOWNLOAD_SHA256=8d97a97984f6cb…   279kB
RUN |1 GRADLE_DOWNLOAD_SHA256=8d97a97984f6cb…   152MB   # Gradle 실행 파일 다운로드 및 압축 해제
RUN /bin/sh -c set -o errexit -o nounset    …   57.3kB
RUN /bin/sh -c set -o errexit -o nounset    …   220MB   # Gradle 환경변수 및 기본 라이브러리 세팅
RUN /bin/sh -c set -eux;     echo "Verifying…   12.3kB
RUN /bin/sh -c set -eux;     apt-get update;…   68.8MB
RUN /bin/sh -c set -eux;     ARCH="$(dpkg --…   279MB   # 베이스 이미지 JDK 환경 (Java 실행 & 컴파일)
RUN /bin/sh -c ./gradlew :sytk-booking:bootJ…   438MB   # Gradle 빌드 시에 프로젝트에 필요한 외부 라이브러리(Spring, Hibernate, Jackson 등) 전체 다운로드
EXPOSE [8080/tcp]                               0B
ENTRYPOINT ["java" "-jar" "build/libs/app.ja…   0B      # 실제 실행 파일(.jar)은 소량
COPY . . # buildkit                             328MB   # 프로젝트 전체 소스 파일
COPY --chmod=755 entrypoint.sh /__cacert_ent…   12.3kB
/bin/sh -c #(nop) ADD file:6df775300d76441aa…   87.6MB  # OS 베이스 리눅스 파일 시스템
...
```

[이전 게시물](write-dockerfile)의 `COPY` 명령어 소개에서 *'어디서부터 어디까지를 컨테이너로 복사해와야할지'* 에 대해 의문점이 있었다.  
`FROM gradle:8.12-jdk17` 명령어로 Gradle 하나만 가져와서 사용하려 해도, Docker는 이를 실행하기 위해 필요한 운영체제(OS), 자바 설치 파일(JDK), Gradle 프로그램 본체를 모두 레이어에 쌓아서 가져오며 Gradle 빌드 시엔 프로젝트에 필요한 외부 라이브러리들도 전부 내려받는다.  

사실상 어플리케이션 실행 필요한 결과물은 JAR 파일 1개뿐이지만 **빌드에 사용된 JDK, Gradle 캐시, 소스 코드와 외부 라이브러리들이 최종 이미지에 모두 잔재해 있는 것이 용량을 차지하는 주된 원인**이었다. 이렇게되면 용량도 크게 차지하지만 보안에도 아주 취약하다.

이를 해결하기 위해 빌드 단계와 실행 단계를 분리하는 [멀티 스테이지 빌드](https://docs.docker.com/build/building/multi-stage/#name-your-build-stages)방법을 사용하여 **실행 시점에는 JRE(or 경량 JDK)와 빌드된 `.jar` 파일만 남도록** 한다.  

### *Multi-stage Build*
==Dockerfile을 빌드 단계와 실행 단계인 두 스테이지로 분리==한다.
1. **빌드 단계(Stage 1)**
	- 첫 번째 `FROM`문으로 시작하며, 빌드 환경을 구축한다.
	- 여기서 대용량의 gradle 환경, JDK, 외부 라이브러리들은 빌드를 위해 내려받아지지만 최종 이미지엔 포함되지 않을 것이다.
2. **실행 단계(Stage 2)**
	- 두 번째 `FROM`문으로 시작하며, 최종 배포용 이미지를 구축한다.
	- 베이스 이미지를 `gradle:8-jdk17` 같이 os가 포함된 이미지가 아닌 컴파일 도구가 빠져있어 가볍고 보안상 안전한 JRE 전용 이미지로 사용한다.
		- Java 17 이상부터는 JDK와 JRE의 경계가 모호해지면서 Docker Hub 공식 이미지 중 `jre` 태그가 없는 경우도 있다. 이땐 `-slim` 계열의 이미지를 선택하면 비슷한 경량화 효과를 얻을 수 있다.
	- 이전 단계(`builder`)의 파일 시스템에서 명시적으로 필요한 파일(ex: `.jar`)만 선택해서 최종 이미지로 복사해오고, 나머지는 최종 이미지의 **레이어 구성에서 제외**된다.
		- +) 빌드 컨텍스트(캐시)에는 남아있어 [다음 빌드 시 속도를 높이는 데 재사용](#3-2차-개선-캐싱-효율-극대화)할 수 있다.
```Dockerfile
# 1단계: 빌드 스테이지 (이미지 이름을 builder로 지정)
FROM gradle:8.12-jdk17 AS builder

WORKDIR /app

# 소스 코드 복사
COPY . .

# 특정 모듈 빌드
RUN ./gradlew :module-name:bootJar --no-daemon

# 2단계: 실행 스테이지 (최종 이미지)
# 빌드 도구가 없는 JRE 전용 이미지 사용 or slim 버전을 선택
FROM eclipse-temurin:17-jre-jammy

WORKDIR /app

# builder 스테이지에서 생성된 jar 파일만 추출하여 컨테이너로 복사
COPY --from=builder /app/module-name/build/libs/*.jar app.jar

# 실행할 포트 문서화
EXPOSE 8080

# 컨테이너 실행 명령
ENTRYPOINT ["java", "-jar", "app.jar"]
```
**변경사항**
1. `AS builder`, `--from=builder`: 두 번째 스테이지 시작 시 실행되는 `FROM`명령은 이전 명령에서 생성된 모든 상태를 초기화하므로 첫 번째 스테이지에 이미지 ID를 부여하여 결과물을 가져오도록 한다.
2. `--no-daemon`: Gradle은 속도 향상을 위해 백그라운드에 데몬을 띄워놓는다. 하지만 Docker 컨테이너 내부는 일회성 빌드 환경이므로 메모리만 소모하고, 빌드가 끝나도 프로세스가 남아서 컨테이너가 안 닫히는 문제 발생할 수 있으므로 백그라운드 데몬을 끄자.

> [!question] 멀티 스테이지가 나오기 전(Docker 17.05 이전)에는 어떤 방법을 사용했을까?
> 빌드 도구와 라이브러리를 이미지에서 빼고 싶어했던건 여전했기 때문에 이전에는 빌드용 컨테이너에서 빌드를 수행하고 생성된 `.jar` 파일을 **호스트 PC로** 복사한 후, 다시 실행용 컨테이너로 복사하는 쉘 스크립트를 직접 짜는 Builder Pattern을 사용하며 운영했다. 하지만 빌드 환경을 구축하기 위한 별도의 스크립트 파일을 만들어 관리해야하는 CI/CD 과정이 다소 복잡하여 현재는 Dockerfile 내에서 두 환경을 통합하여 운영하는 멀티 스테이지 방법이 많이 사용된다.

**빌드 결과 비교**
![[스크린샷 2026-05-06 오후 5.05.10.png]]
개선 전과 같은 상태에서 빌드한 결과 **약 3분 40초**가 소요되었고 **549.55MB**의 이미지를 생성했다.
- 빌드 소요 시간 : 약 6분 -> 약 3.5분 (**약 1.7배 단축**)
- 이미지 용량 : 2.6GB-> 549.55MB (**약 79% 절감**)

약 1/5 수준의 용량 경량화에 성공했을 뿐만 아니라 빌드에 걸리는 시간 또한 1.7배나 단축할 수 있었다. 서버에서 이미지를 `pull` 받을 때 네트워크 전송량이 1/5로 줄어들고 불필요한 파일 복사 과정이 생략되어 시간 측면에서도 빌드 효율을 크게 올릴 수 있었다.
## 3) 2차 개선; *캐싱 효율 극대화*
직접 빌드를 해보면 두 번째 이상의 빌드부터는 빌드 시간이 확연히 줄어드는 것을 확인할 수 있다.
- 개선 전 첫 빌드는 **약 6분**이 소요되었지만 재빌드 시 **3.7초**가 소요됐다.
- 멀티 스테이지 빌드에서 첫 빌드가 **약 3.5분**이었지만 재빌드 시 **3.2초**가 소요됐다.

도커 빌드 시간이 첫 번째와 두번째 이후에서 차이가 나는 핵심 이유는 **레이어 캐싱(Layer Caching)** 이다.  
도커는 빌드 명령을 내리면 먼저 이전 빌드에서 생성된 레이어가 있는지 확인한다. 그리고 기존 레이어가 이미 존재하는 경우 만약 파일 내용이나 명령어 문구가 수정되지 않았다면, 실제 명령된 작업을 수행하는 대신 저장되어있던 기존 레이어를 즉시 불러오는 캐싱 전략을 사용한다.

만약 모듈의 파일 변화가 간단한 소스 수정이어도 해당 모듈의 전체 환경을 처음부터 다시 빌드해야할까?
[이전 글](write-dockerfile)에서 이러한 의문점도 있었다.
> **`COPY`**
> 1. 설정 파일이나 패키지들은 자주 바뀌지 않으니 저장해두었다가 재사용하고 변경된 파일만 다시 복사해오면 안될까?
> 2. 전체 소스말고 필요한 소스 부분만 가져오는 것이 효율적이기도 하고 보안에도 더 유리하지 않나?
> 
> **`RUN`**
> 마찬가지로 내용이 변경되었을 때만 처음부터 실행하도록 하고, 패키지나 라이브러리들은 이전에 받아둔 결과물을 재사용할 수 있지 않을까?

### *의존성 캐싱(라이브러리 캐싱)*
`Dockerfile`의 명령어는 위에서부터 순서대로 실행되어, 수정된 부분부터 하위의 모든 레이어가 새로 빌드된다.
```Dockerfile
# 소스 코드 복사
COPY . .

# 특정 모듈 빌드
RUN ./gradlew :module-name:bootJar --no-daemon
```
지금처럼 소스코드 복사가 모듈 빌드보다 선행된다면, 코드 한 줄 고칠때마다 `COPY . .` 부터 재실행되어 모듈 내에 있는 수백 mb의 라이브러리를 다시 받고 빌드 연산을 처음부터 다시 진행한다. 따라서 ==변경이 많지 않는 설정 파일이나 패키지들은 변경이 잦은 소스 부분과 분리하여 설정 파일 먼저 따로 복사==해오는 방법이 효율적이다.  

또한 빌드 컨텍스트 내의 `build/`, `.gradle/`, 빌드 결과물(`target`, `build`), 각종 로그 파일처럼 프로덕션과 관련 없는 불필요한 파일 및 폴더들을 도커 데몬 전송되는 데이터에서 제외한다면 빌드 속도도 향상되고 보안에도 유리할 것이다.

(최종)
```Dockerfile
# 1
FROM gradle:8.12-jdk17 AS builder
WORKDIR /app

# 빌드 캐시 활용을 위해 루트의 설정 파일들 및 해당 모듈의 build.gradle 먼저 복사
COPY gradlew settings.gradle ./
COPY gradle ./gradle
COPY build.gradle ./   # build.gradle은 의존성을 추가할 때마다 수정되기 때문에 비교적 하위에 위치시킴
COPY module-name/build.gradle ./module-name/
  
# 소스 복사 전 의존성만 먼저 다운로드(캐싱 레이어; 가장 중요)
RUN ./gradlew :module-name:dependencies --no-daemon

# .dockerignore 사용
COPY . .

# 해당 모듈 빌드 진행 + jar 파일 생성 (메모리 절약 위해 테스트 스킵, gradle 데몬 비활성)
RUN ./gradlew :module-name:bootJar -x test --no-daemon

# 2
FROM eclipse-temurin:17-jre-jammy
WORKDIR /app

# 빌드 스테이지에서 생성된 jar 파일만 복사
COPY --from=builder /app/module-name/build/libs/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```
**변경사항**
- `:module-name:dependencies`
	- `:module-name`: 경로 지정을 위해 사용한다.
	- `:build`나 `:bootJar` 명령어를 사용한다면 현재 소스가 없는 상태이기 때문에 컴파일 에러가 발생한다.
	- `:dependencies`는 Gradle이 기본적으로 제공하는 태스크로, `bootJar`처럼 `컴파일 -> 테스트 -> 패키징`을 하지 않고 단순히 의존성 리스트만 출력한다. 따라서 소스 코드가 없어도 `build.gradle` 파일만 있으면 에러가 나지 않고 빌드 성공한다.
- `COPY module-name ./module-name`
	- 해당 모듈의 소스만 명시하여 복사해오는 경우
	- 프로젝트 구조가 변경 혹은 확장되거나 모듈 이름이 수정되거나 파일 위치가 바뀌면 Dockerfile을 매번 수정해야하기 때문에 `COPY . .` + [`.dockerignore`](https://github.com/ghsyn/sytk/tree/main/sytk-booking) 방식을 선택했다.
# 3. 최종 비교
### 1. 개선 전
![[스크린샷 2026-05-06 오전 10.47.05.png]]
### 2. 멀티 스테이지 빌드 후
![[스크린샷 2026-05-06 오후 5.05.10.png]]
### 3. 레이어 캐싱 후
![[스크린샷 2026-05-06 오후 11.59.12.png]]

| 시도                   | 이미지 용량       | 첫 빌드 소요 시간      | 재빌드 소요 시간    |
| -------------------- | ------------ | --------------- | ------------ |
| 개선 전                 | 2.6GB        | 358.7s (약 6분)   | 3.7s         |
| 1회 개선 후(멀티 스테이지 빌드)  | 549.55MB     | 208.0s (약 3.5분) | 3.2s         |
| 2회 개선 후(레이어 캐싱)      | 549.55MB     | 201.6s (약 3.4분) | 2.9s         |
| **개선 전 vs 최종 결과 비교** | **약 79% 절감** | **약 1.7배 단축**   | **약 22% 단축** |

*+) 레이어 캐싱의 효과는 사실 코드 변경이 많지 않을 때 의존성 다운로드를 건너뛰어 재빌드 시간이 줄어드는데에 가치가 있다.*

# 마치며
실제 이미지를 만들어보고 비교해보니 무거운 이미지와 긴 빌드 시간이 수치로만 보는 것보다 크게 와닿았다.  
기술적 지식도 중요하지만, 만들어지는 이미지가 배포 파이프라인에서 어떤 비용을 발생시키는지 고민해보는 과정 자체가 큰 배움이었다.  
이제 `docker-compose`를 작성해서 한 번에 여러 모듈 이미지 생성도 하고 CI/CD 파이프라인에서 빌드 프로세스도 자동화하고 해야겠다.

---
**관련 링크**
- [Dockerfile 주요 명령어 정리](write-dockerfile)
- [Docker Docs - # Multi-stage builds](https://docs.docker.com/build/building/multi-stage/#name-your-build-stages)
- [Docker Docs - # Building best practices](https://docs.docker.com/build/building/best-practices/)