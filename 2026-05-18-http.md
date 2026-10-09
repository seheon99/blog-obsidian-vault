---
type: write-up
title: HTTP/0.9부터 HTTP/1.1까지
description: 유럽의 입자물리 연구소에서 만들어진 HTTP
topics:
  - Web
createdAt: 2026-05-18T16:00:00
---
## HTTP/0.9 (1991년)

### 1990년 유럽의 입자물리 연구소

HTTP/0.9는 1990년 유럽의 국제 입자물리 연구소인 CERN에 만들어졌다[^1]. CERN은 전 세계 100여개의 국가에서 17,000명의 과학자들이 모여 일하는 거대한 공동 연구소였다. 하지만 너무 많은 사람이 모이다보니 논문, 데이터, 정보가 수많은 컴퓨터에 흩어져 있었다.

그래서 영국의 엔지니어인 팀 버너스리는 흩어져있는 정보를 관리하는 시스템을 1989년 3월 처음 제안했고, 벨기에의 시스템 엔지니어 로베르 카이오와 함께 1990년 11월에 제안을 완성했다. 그 제안은 WorldWideWeb 이라고 부르는 hypertext 프로젝트에 대한 제안이었고, hypertext 문서들의 web을 브라우저로 볼 수 있게 하자는 프로젝트였다.

1990년이 끝나갈 때 쯤, 팀 버너스는 첫 웹 서버를 만들고 CERN에서 처음 브라우징을 시연했다. NeXT 컴퓨터에서 문서를 서빙할 수 있도록 코딩했고, 컴퓨터 전원을 꺼버리는 실수를 방지하기 위해 빨간색 글씨로 "This machine is a server. DO NOT POWER IT DOWN!!" 이라고 써놨다고 한다.

![[http-first-server-note.png]]

### 스펙 읽어보기

HTTP/0.9[^2]를 읽으며 신기한 부분을 찾았다. 요청은 아스키문자로 이루어진 한 줄로 보내고, 응답은 무조건 아스키문자의 HTML로만 해야했다. 클라이언트의 상태를 추가해서 요청을 보낼 수도 없고, 서버에서 예외가 발생해도, 에러가 발생해도 HTML 로만 응답해야 한다. 이렇게나 간단한 형태로 압축된 프로토콜이지만 요청을 보낼 때는 꼭 `GET` 을 붙여야 했다.

**요청**
```
GET /hypertext/WWW/TheP.html
```
**응답**
```html
<html>
<body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
</body>
</html>
```

> [!QUESTION] HTTP/0.9에는 사용할 수 있는 메서드가 `GET` 하나 뿐이었다. 그럼 생략할 수도 있지 않았을까?
> 첫 문서를 작성할 때부터 확장성을 고려해 메서드의 자리를 만들어 둔 것 같다. 일종의 Open Closed Principle 인가? 최대한 핵심만 남긴 첫 스펙 안에 메서드의 자리를 남겨두고, 메서드는 `GET` 하나만 남겨두는 것이 HTTP의 MVP를 보는 것 같기도 하다.

### HTTP/1.0 과 HTTP/1.1 (1996년)

웹은 사용자가 명령어를 외우지 않아도 링크를 따라 분산된 정보에 접근할 수 있다는 장점이 있었다. 프로토콜이 아주 간단했기 때문에 서버를 직접 구현하기도 쉬웠다. 그리고 1993년 4월, CERN은 WorldWideWeb 프로젝트의 소스코드를 무료로 공개했다. 이로 인해 1993년 말에는 500개 이상의 웹 서버가 생기게 되었다.

HTTP는 아주 간단한 스팩만 갖고있었고, 5년간 여러가지 확장 기능들을 붙이면서 발전했다. 이를 표준화하려는 시도가 있었고, 1995년 초 HTTP Working Group을 만들었다. HTTP Working Group은 HTTP의 표준화를 위해 지금까지의 관행을 문서화했다. 이 문서가 RFC 1945(HTTP/1.0) 이고, 표준화는 RFC 2068(HTTP/1.1)에서 시작했다. RFC 1945와 RFC 2068은 같은 표준화 시도의 산출물이고, 1995년부터 동시에 진행되어 HTTP/1.0은 1996년 5월, HTTP/1.1은 1997년 1월에 완성되었다.

**요청**
```
GET /index.html HTTP/1.1
Host: http.dev
```
**응답**
```
HTTP/1.0 200 OK
Content-Type: text/plain
Content-Length: 24

Hello, this is HTTP/1.0!
```

[^1]: https://home.cern/science/computing/the-birth-of-the-web/short-history-web/

[^2]: https://www.w3.org/Protocols/HTTP/AsImplemented.html
