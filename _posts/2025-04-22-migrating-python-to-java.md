---
layout: post
title: GitHub Copilot으로 Python 앱을 Spring Boot 기반으로 마이그레이션 하기
categories: [ghcp]
---

이번 포스팅은 [2025년 4월 26일(토)에 진행될 GitHub Copilot BootCamp](https://event-us.kr/msftkrdevrel/event/101808?utm_source=eventus&utm_medium=organic&utm_campaign=channel-event) 중 자바 세션의 진행 과정을 정리하였습니다.

본 이벤트에서 진행된 전체 시나리오 및 소스코드는 [여기](https://github.com/devrel-kr/github-copilot-bootcamp-2025/)에서 확인하시기 바랍니다.

## 0. 사전 준비사항

개발 환경은 Visual Studio Code Insider 버전과 Devcontainer를 사용하였으며, 사전에 Python 기반 백엔드 앱과 Javascript 기반 프론트엔드 앱이 구동되어 있어야 합니다. 공유된 소스코드를 참고하여 다른 언어들에 대해서도 테스트를 진행한 뒤 앱을 구동시켜도 되고, 자바 과정만 빠르게 확인하고 싶다면 [여기](https://github.com/devrel-kr/github-copilot-bootcamp-2025/tree/main/complete)에서 완성된 소스와 구동 명령어를 이용하여 위의 두 앱을 구동시켜 줍니다.

### Python 기반 백엔드 앱 구동
다음 명령어를 사용하여 앱을 구동합니다.
```bash
cd python 
python -m venv .venv 
source .venv/bin/activate 
pip install fastapi uvicorn 
uvicorn main:app --reload 
```
다음 URL을 접속하여 앱이 잘 구동되었는지 확인해 봅니다. 
```
http://localhost:8000/docs
```
정상적으로 구동되었다면 Rest API 목록이 보일 것입니다.

![migrating-python-to-java 1](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/1.png)

### Javascript 기반 프론트엔드 앱 구동
다음 명령어를 사용하여 앱을 구동합니다.
```bash
cd javascript 
node --version 
npm install -g yarn 
yarn install 
yarn dev 
```
다음 URL을 접속하여 앱이 잘 구동되었는지 확인해 봅니다. 
```
http://localhost:3000
```

정상적으로 구동되었다면 프론트엔드 앱이 보일 것입니다.

![migrating-python-to-java 2](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/2.png)
 
**Note:** 만약 3000번 포트는 리슨하고 있는데 앱이 구동되지 않는다면, `complete/javascript/vite.config.js` 파일의 서버 설정에 다음과 같이 호스트 정보를 추가해 줍니다.
```
host: '0.0.0.0', 
```

### 프론트엔드 앱 사용해 보기
앱이 정상적으로 구동되었으면, 테스트를 위해 데이터를 입력합니다.

![migrating-python-to-java 3](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/3.png)

## 1. 마이그레이션 순서 리뷰
이제 마이그레이션을 할 준비가 되었습니다. Python 앱으로부터 Spring Boot 앱으로의 전체적인 마이그레이션 과정은 다음과 같습니다.

- [Python 앱 로직 확인](#2-python-앱-로직-확인)
  - [SQLite 데이터베이스 확인](#SQLite-데이터베이스-확인)
- [Spring Boot 앱으로 마이그레이션](#3-spring-boot-앱으로-마이그레이션)
  - [Spring Boot 프로젝트 생성](#spring-boot-프로젝트-생성)
  - [빌드 및 앱 구동](#빌드-및-앱-구동)
  - [Code Instruction 생성](#code-instruction-생성)
  - [REST API 추가 및 확인](#rest-api-추가-및-확인)
  - [Python에 작성된 REST API 추가](#python에-작성된-rest-api-추가)
  - [Database 설정 구성](#database-설정-구성)
- [백엔드 앱 전환 및 확인](#4-백엔드-앱-전환-및-확인)
  - [확인](#확인)

## 2. Python 앱 로직 확인

Python 앱에 어떤 기능들이 포함되어 있는지 확인하기 위해 `main.py` 파일을 열고 GitHub Copilot에 다음과 같이 질의합니다. 해당 기능을 확인하여 대략의 Python 로직을 이해하고 파일을 생성합니다.

```
이 파일의 소스 내용을 설명해 주고 다이어그램을 생성해줘
```

다음 위치에 다이어그램 파일이 생성된 것을 확인합니다.

```
../python/diagram.md
```
파일을 열고 다이어그램 내용을 확인해 봅니다. GitHub은 기본적으로 Mermaid 다이어그램을 지원합니다. 클래스 다이어그램, 데이터베이스 스키마 다이어그램, API 엔드포인트 다이어그램,애플리케이션 흐름도 등이 생성된 것을 확인할 수 있습니다. VS Code용 Mermaid Preview 익스텐션을 설치하여 다음과 같이 UI를 통해 확인할 수도 있습니다.

* 클래스 다이어그램 

  ![migrating-python-to-java 4](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/4.png)
* 데이터베이스 스키마 다이어그램

  ![migrating-python-to-java 5](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/5.png)
* API 엔드포인트 다이어그램 
  
  ![migrating-python-to-java 6](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/6.png)
* 애플리케이션 흐름도 
  
  ![migrating-python-to-java 7](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/7.png)



### SQLite 데이터베이스 확인

Python 앱에서 사용한 SQLite 데이터베이스를 열고 데이터를 확인해 봅니다. 저는 `SQLite Viewer`를 이용해서 데이터를 확인했습니다.

![migrating-python-to-java 8](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/8.png)

## 3. Spring Boot 앱으로 마이그레이션

이제 Spring Boot 앱으로 마이그레이션 할 준비가 되었습니다. 이제부터 GitHub Copilot을 이용하여 마이그레이션을 진행하겠습니다. 이후 질의과정은 예시일 뿐이므로, 본인의 상황에 맞게 변경하여 질의하며 마이그레이션을 진행해도 됩니다.

### Spring Boot 프로젝트 생성

Spring Boot Initializr를 이용하면 Spring Boot 앱을 쉽게 생성할 수 있습니다. Spring Initializr 익스텐션을 사용하여 VS Code에서 초기 프로젝트를 생성해 보도록 합니다. 설정 값은 다음과 같습니다.

* 프로젝트 선택: Create a Gradle Project
* Spring Boot Version: 3.4.4
* Project language: Java
* Group Id: com.example
* Artifact Id: demo
* Package Name: com.example.demo
* Packaging type: Jar
* Java version: 21
* Dependencies:
  * Spring Web
  * Spring Boot Actuator
  * Lombok

초기 Spring Boot 앱 프로젝트가 생성되었으면 우측 하단에의 "Open new project"를 클릭하여 방금 생성한 프로젝트를 오픈합니다.

![migrating-python-to-java 9](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/9.png)

만약 창이 사라져 방금 생성한 프로젝트를 오픈할 수 없다면, 다음과 같이 터미널에 명령어를 입력하여 프로젝트를 오픈합니다.
```bash
cd java
code demo
```

### 빌드 및 앱 구동

최초 생성된 앱이 에러가 없는지 빌드 후 앱 구동을 통해 확인해 봅니다.

```
./gradlew clean build bootRun
```

터미널에 다음과 같은 로그가 출력되면 앱이 정상적으로 구동된 것입니다.

![migrating-python-to-java 10](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/10.png)

### Code Instruction 생성
소스가 자동 생성될 때 지켜야 할 Code Convention을 지정합니다. `.github/code-instructions.md` 파일을 생성하고, 본인의 개발 스타일 및 프로젝트 규정 등을 반영하여 규칙을 정의합니다. 

![migrating-python-to-java 11](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/11.png)


### REST API 추가 및 확인

디펜던시가 에러 없이 잘 추가되었는지 확인하기 위해 RestController를 추가해 봅니다. 기본 호출할 REST API로 /hello 를 생성해 보겠습니다. GitHub Copilot을 Agent 모드로 변경하고, 모델을 "Claude 3.7 Sonnet"으로 변경한 후 다음과 같이 프롬프트를 입력합니다.

```
/hello REST API를 추가해주고, Swagger 설정도 추가해줘
```
소스 및 패키지가 이전에 지정한 코딩 규칙에 맞춰서 잘 생성된 것을 볼 수 있습니다.

![migrating-python-to-java 12](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/12.png)

다시 한 번 앱을 빌드하고 실행시켜 봅니다.

코드에 오류가 없으면 다음 URL을 이용하여 앱을 호출해 봅니다.

```
http://localhost:8080/hello
```
![migrating-python-to-java 13](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/13.png)

이번에는 Swagger UI를 접속해 봅니다.

```
http://localhost:8080/swagger-ui.html
```
![migrating-python-to-java 14](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/14.png)

앱이 정상적으로 호출되었으면 마이그레이션 준비가 완료된 것입니다.

### Python에 작성된 REST API 추가

Python 앱은 12개의 REST API로 이루어진 백엔드 서비스로 구성되어 있습니다. 이제 좀 전에 생성한 다이어그램 파일을 활용하여 이 REST API를 자바로 변경해 보겠습니다.
데이터베이스 스키마 다이어그램의 이미지를 캡쳐하여 GitHub Copilot에 붙여넣고, 다음과 같이 프롬프트를 입력합니다. 이 기능은 2025년 4월 현재 GitHub Copilot의 신기능인 `Vision` 기능을 사용해 보는 것입니다. 이 기능에 대해 좀 더 자세히 알고 싶다면 이 [링크](https://github.blog/changelog/2025-02-06-next-edit-suggestions-agent-mode-and-prompts-files-for-github-copilot-in-vs-code-january-release-v0-24/#vision-public-preview)를 클릭하세요.

```
이 이미지를 참고해서 클래스를 생성해줘.
```
![migrating-python-to-java 15](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/15.png)

다시 `python/diagram.md` 파일을 열고 다음과 같이 GitHub Copilot에 요청합니다.
```
이 다이어그램 파일을 참고해서 동일한 API 주소를 갖는 함수들을 현재 프로젝트에 추가해줘.
```
![migrating-python-to-java 16](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/16.png)

소스가 생성되였으면, 앱을 빌드하고 Swagger UI에 접속해서 12개의 REST API가 추가되었는지를 확인합니다. 만일 빌드 시 에러가 발생하거나, 원하는 API 목록이 출력되지 않는다면 다음 프롬프트를 사용해서 정상적인 결과값이 나올때까지 수정을 반복해서 이슈를 해결합니다.

```
빌드 시 오류가 발생해. 오류를 해결해줘 
```

반복 작업을 통해 에러가 수정되고, 원하는 API들이 다음과 같이 추가되었습니다.

![migrating-python-to-java 17](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/17.png)

### Database 설정 구성
데이터베이스에 대한 설정이 `application.properties` 파일에 아직 적용되지 않았습니다. 다음 프롬프트를 입력하여 데이터베이스 설정을 적용합니다.
```
데이터베이스 설정 정보가 없는 것 같아. sqlite 사용하도록 설정해줘
```

GitHub Copilot을 포함한 AI 도구들은 동일한 프롬프트를 사용하더라도 아직까지는 일정한 응답을 제공하지 않는 경우가 많습니다. 저도 여러번의 테스트를 거쳤지만, 어떤 경우에는 별다른 에러 없이 소스를 생성해 주었고, 어떤 경우는 수없이 수정을 해야 했습니다. 이 중 기억나는 한 가지는 데이터베이스의 날짜에 대한 데이터 타입으로 인해 지속적으로 오류를 고쳐야 하는 경우가 있었습니다. 

#### 컬럼명 확인
코딩 규칙에 Camel 표기법을 따르도록 정의되어 있는데, 이 경우 JPA는 자동으로 다음과 같이 매핑됩니다. 클래스의 컬럼명이 userId라면 데이터베이스에서는 user_id로 저장되는데, 이전에 Python 에서 생성된 데이터베이스의 컬럼명과 불일치하기 때문에 에러가 발생합니다. 따라서 클래스 컬럼명과 데이터베이스 컬럼명이 동일하도록 설정 파일을 수정해 주어야 합니다. GitHub Copilot에게 다음과 같이 명령을 전달합니다.

```
모델 클래스의 컬럼 이름 그대로 DB에서 사용하도록 하는 JPA 네이밍 컨벤션 규칙 추가해줘. 
```

GitHub Copilot이 `application.properties` 파일을 다음과 같이 수정하는 것을 확인할 수 있습니다.
```
# JPA 네이밍 전략 - 물리적 네이밍 전략: 모델 그대로 컬럼명 사용
spring.jpa.hibernate.naming.physical-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
spring.jpa.hibernate.naming.implicit-strategy=org.hibernate.boot.model.naming.ImplicitNamingStrategyLegacyJpaImpl
```

#### 데이터 타입
서로 다른 언어(Python, Java)로 개발된 백엔드 앱에서 Rest API를 테스트 하는 경우 다음과 같이 저장됩니다. Python은 TEXT 타입으로 `2025-04-23 23:58:59`와 같이 저장됩니다. Java의 경우 Date 형태로 생성되는 경우가 많았고, ``와 같이 데이터가 생성되는 경우가 있습니다. 이 경우 타입을 확인하여 동일한 형식으로 저장되도록 사전에 확인할 필요가 있습니다.

![migrating-python-to-java 17](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2025-04-22-migrating-python-to-java/17.png)

#### Database URL 변경

이제 기존 Python에서 사용하던 데이타베이스를 참조하도록 환경을 변경해 봅니다. 해당 파일은 다음 위치에 존재합니다.

```
/workspaces/github-copilot-bootcamp-2025/complete/python/sns.db
```

Spring Boot 프로젝트의 다음 파일을 오픈해서 파일 주소를 변경합니다.

* project_root: /workspaces/github-copilot-bootcamp-2025/java/demo
* 파일 명: <project_root>/src/main/resources/application.properties

변경할 내용은 다음과 같습니다.

```
spring.datasource.url=jdbc:sqlite:sns.db
```

> **NOTE**: 만약 Python 앱에서 쓰던 `sns.db`를 그대로 활용하고 싶다면, `python/sns.db` 파일을 `java/demo/sns.db`로 복사합니다.

다시 앱을 빌드해서 데이타베이스가 제대로 참조되었는지 확인합니다.

## 4. 백엔드 앱 전환 및 확인

### 확인

이제 모든 코드 생성이 완료되었습니다. 마지막으로 Javascript로 작성된 프론트엔드 앱에서 기존 URL과 동일하게 호출할 수 있도록 Spring Boot 앱의 포트를 변경합니다. 

변경 전 [여기](01-python.md#서비스-종료)를 참고하여 Python 앱을 구동 중지합니다. 
```
lsof -i:8000 | xargs kill -9
```


포트 변경을 위한 설정을 변경하기 위해 GitHub Copilot에 다음과 같이 프롬프트를 입력합니다.

```
앱 서비스 포트를 8000으로 변경해줘
```

이제 앱을 다시 빌드하고 구동시켜서 다음 URL로 접속이 되는지를 확인합니다.

```
https://localhost:8000/swagger-ui.html
```

앱이 정상적으로 구동되었으면 Node JS 프론트엔드 앱을 호출하여 앱에 이상이 없는지 확인합니다.

```
http://localhost:3000
```

마이그레이션이 성공적으로 완료되었습니다.
