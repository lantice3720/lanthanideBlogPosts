---
title: Remark42를 통한 댓글 시스템 구축
description: 만들어둔 블로그에 댓글 시스템을 추가해 보자.
keywords:
  - 댓글
  - Remark42
tags:
  - Remark42
  - Docker
  - nginx
categories:
  - Blog
date: 2024-07-14T20:50:10+09:00
draft: false
aliases: 
weight: 10
build:
  list: always
  publishResources: true
  render: always
---
[저번 포스팅](/posts/github-pages를-사용해-정적-웹-페이지-블로그-만들기/)에서 만든 블로그를 흡족하게 바라보고 있자니 뭔가 아쉬운 느낌이 들었다. 이를테면, 댓글 기능이 없는 것이다. 이에 다양한 댓글 서비스를 찾아보았고, 대략의 후보군을 만들었다.
- Disqus
- Giscus
- Commento
- Waline
- Cactus
- Remark42

이 중 Disqus는 광고가 노출되는 등의 불편함이 있어 제외, Giscus는 GIthub 계정이 필수라 제외... 를 반복하다보니 Remark42가 가장 알맞는 듯 했다. [^호스팅]

## Remark42 소개
Remark42는 [공식 홈페이지](https://remark42.com/)에 따르면, 개인정보 보호에 집중한 가벼운 댓글 엔진이다. 실제로, GCP의 프리티어에서도 잘 돌아가는 모습을 지금 이 페이지의 하단에서 확인해볼 수 있다. 무엇보다 다양한 소셜 로그인을 지원하며, 마크다운 문법은 물론 답글이나 익명 댓글도 가능하니 있을 기능은 전부 있는 셈이다.

## Docker 설치
시작하기 전에, 이 글은 GCP의 e2-micro 머신에서 돌아가는 Ubuntu 20.05.06에서 테스트됐음을 밝힌다.

우선 편리하게도 Remark42는 docker를 사용한 설치를 지원한다. [Github 레포](https://github.com/umputun/remark42/blob/master/docker-compose.yml)에서 제공하는 `docker-compose.yml`을 복사해서 `docker-compose pull`로 이미지 다운받고 `docker-compose up -d`로 컨테이너 생성, 실행하면 된다. 이 때 주석을 한 번은 다 읽어보길 추천하며, 필자의 파일은 이렇다.

```yaml
# 경고 메시지 보기 싫어서 버전업했다. 
version: "2.1"

services:
  remark:
    image: umputun/remark42:latest
    container_name: "remark42"
    hostname: "remark42"
    restart: always

    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"

    # 원래 통짜 주석처리에 80포트로 바인딩되어 있는데, nginx나 certbot 등의 문제로 8080 포트를 사용한다.
    ports:
      - "8080:8080"
    #  - "443:8443"

    environment:
	  # 클라이언트가 액세스할 API URL
      - REMARK_URL=https://remark42.lanthanide.kr
      # nginx가 CORS를 컨트롤하기 때문에 자체적인 CORS를 껐다.
      - PROXY_CORS=true
      # 아마 댓글 관리에 사용되는 듯 하다
      - SECRET=********
      - DEBUG=true
      # Google과 Github의 OAuth를 활성화하고 받은 인증 정보이다. Anonymous는 현재 설정해 두지 않았다.
      - AUTH_GOOGLE_CID=********
      - AUTH_GOOGLE_CSEC=********
      - AUTH_GITHUB_CID=********
      - AUTH_GITHUB_CSEC=********
    volumes:
      - ./var:/srv/var
```

`curl http://127.0.0.1:8080/ping`을 입력했을 때 pong 메시지가 나오면 성공이다. 실패한다면 `docker conatiner ls -a`를 입력해 remark42가 healthy 상태인지 확인하자. 

## nginx 구축
사실 nginx 없이 바로 Docker를 노출시켜도 된다. 하지만 필자의 경우 ssl 적용을 위해 nginx를 구축해주기로 했다. 아마 다른 이유일 것 같지만, 댓글 로딩이 안 되는 문제가 있는 점도 한 몫 했다.

우선 certbot으로 인증서를 발급받도록 하자. `certbot --nginx -d remark42.lanthanide.kr`과 같이 입력 후 안내하는 대로 과정을 거치면 어느새 인증서가 뿅 생겨난다.

다음으로 `apt-get install nginx`를 통해 nginx를 설치하면 이제 systemctl을 통해 nginx를 reload, restart할 수 있다. nginx의 컨피그 변경 시 해주면 된다. 

이제 `etc/nginx/sites-available`에서 `default` 파일을 수정할 텐데, nginx를 다루는 법을 잘 아는 방문자라면 conf.d 폴더 내부에 `domain.conf` 파일을 만드는 등의 방법이 더 좋다고 할 수도 있겠으나... 필자는 뭔가 `default`파일을 수정하는 게 맘이 편하다.

```
# CORS를 여러 사이트에 허용시키기 위한 구문.
# 잘 감이 안 온다면, switch구문으로 $allowed_origin 값을 ""이나 $http_origin으로 설정한다 생각해도 좋다.
map "$http_origin" $allowed_origin {
    default "";
    "https://blog.lanthanide.kr" $http_origin;
    "http://localhost:1313" $http_origin;
}

server {
    # https 접속
    listen 443 ssl http2;
    server_name remark42.lanthanide.kr;

    # ssl 인증서를 설정한다. certbot이 만들어준 경로 그대로 입력하면 된다.
    ssl_certificate /etc/letsencrypt/live/remark42.lanthanide.kr/fullchain.pem; 
    ssl_certificate_key /etc/letsencrypt/live/remark42.lanthanide.kr/privkey.pem; 

    charset utf-8;

    root /var/www/html;
    index index.html;

   
    location / {
        # CORS 설정 부분
        add_header 'Access-Control-Allow-Origin' "$allowed_origin";
        add_header 'Access-Control-Allow-Methods' 'GET, PUT, POST, DELETE, OPTIONS';
        add_header Access-Control-Allow-Headers "Content-Type, Authorization, X-Requested-With";
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Methods' 'GET, PUT, POST, DELETE, OPTIONS';
            add_header 'Access-Control-Max-Age'        86400;
            add_header 'Access-Control-Allow-Credentials' 'true';
            add_header 'Access-Control-Allow-Headers' 'Authorization,DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Content';
            add_header 'Content-Length' 0;
            add_header 'Content-Type' 'text/plain; charset=UTF-8';
            return 204;
        }
        # 리버스 프록시 역할을 하도록 한다.
        proxy_redirect     off;
        proxy_set_header   Host             $host;
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;

        proxy_pass http://localhost:8080;
    }
}
# http로 접속 시 https로 리다이렉트
server {
    listen        80;
    server_name   remark42.lanthanide.kr;
    return 301 https://remark42.lanthanide.kr$request_uri;
}
```

이제 nginx를 reload 후 restart하면 백엔드는 끝이다. 위 curl처럼 로컬 브라우저에서 핑을 찍어보자.

## 블로그에 적용
감사하게도 stacks theme은 자체적으로 remark42 댓글을 지원한다. 덕분에 무척이나 편하게 작업했고, 이것은 간단하게 컨피그만 올리겠다.[^주의]
```yml
comments:
        enabled: true
        provider: remark42
                
        remark42:
            host: https://remark42.lanthanide.kr
            site: remark
            locale: ko
```

stacks theme을 사용하지 않는데다 본인의 테마가 remark42를 지원하지 않는 방문자라도 괜찮다. 아마 공식 문서에서 이미 봤겠지만, remark42는 매우 쉽게 댓글창을 페이지에 추가할 수 있다. 

```html
<script> 
var remark_config = {
	host: 'REMARK_URL',
	site_id: 'YOUR_SITE_ID'
}
!function(e, n) {
    for (var o = 0; o < e.length; o++) {
        var r = n.createElement('script'),
        c = '.js',
        d = n.head || n.body;
        'noModule' in r ? (r.type = 'module', c = '.mjs') : r.async = !0, r.defer = !0, r.src = remark_config.host + '/web/' + e[o] + c, d.appendChild(r)
    }
}(remark_config.components || ['embed'], document);
</script>
```

이것을 페이지에 하드코딩해버리자. 추가로 staks theme에서는 `window.REMARK42.changeTheme()`을 사용해 remark42의 테마를 변경하니, 참고하면 좋을 듯 하다.

## References
https://www.juannicolas.eu/how-to-set-up-nginx-cors-multiple-origins/

https://www.alesordo.com/blog/how-to-configure-remark42-for-comments

https://mulder21c.io/changed-comment-system-to-remark42/

[^호스팅]:사실 처음에는 Giscus를 사용하려 했다. 다름이 아니라, Github Disscusion을 사용하기 때문에 서버를 따로 열지 않아도 되기 때문이었다. 하지만 언급했듯 익명 작성이 불가능한데, [관련 이슈](https://github.com/giscus/giscus/issues/65#issuecomment-1705718319)를 보면 직접 개발할 수 있을 것으로도 보인다. 관심이 있는 방문자라면, 파이팅이다.
[^주의]:site는 `docker-compose.yml`에서 설정할 수 있다. remark는 기본값으로 필자는 그대로 사용했지만, 하나의 서버에서 여러 사이트의 댓글을 관리하고자 한다면 service.remark.environment에 `SITE=foo`를 설정해주자.