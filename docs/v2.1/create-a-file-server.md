---
title: Create a File Server for Imports and Backups
summary: Learn how to create a simple file server for use with CockroachDB IMPORT and BACKUP
toc: true
---

[`IMPORT`](import.html) 프로세스 또는 [CockroachDB 엔터프라이즈 백업](backup.html)을 위한 파일을 저장할 위치가 필요하지만, 클라우드 스토리지 제공자에 대한 액세스 권한이 없는 경우(또는 단순히 사용할 수 없는 경우) 로컬 파일 서버를 실행할 수 있습니다. 이 파일 서버는 [HTTP 내보내기 스토리지 API](#http-export-storage-api)에 대한 지원을 활용하여 사용할 수 있습니다.

이는 특히 다음 경우에 유용합니다:

- CockroachDB가 아직 기본 지원을 받지 못하는 맞춤형 또는 독점 스토리지 공급자에 대한 호환성 계층 구현
- 사내 스토리지 사용

## HTTP 내보내기 스토리지 API

외부 파일을 읽거나 써야 하는 CockroachDB 작업([`IMPORT`](import.html) 와 [`BACKUP`](backup.html) 등)은 주소 앞에 `http://fileserver/mnt/cockroach-exports`와 같이 `http`를 붙여서 HTTP 내보내기 스토리지 API를 사용할 수 있습니다.

이 API는 `GET`, `PUT`, `DELETE` 메소드를 사용합니다. 이것은 전형적인 HTTP 요청처럼 작동합니다. 일부 경로에 대한 `PUT` 요청 후 후속 `GET` 요청은 `DELETE` 요청이 수신될 때까지 `PUT` 요청 바디에 보내진 내용을 반환해야 합니다.

## 예제

`GET`, `PUT`, `DELETE` 등의 방식을 지원하는 파일 서버 소프트웨어는 모두 사용할 수 있습니다. 
일반적인 코드 샘플은 다음과 같습니다:

- [PHP를 `IMPORT`와 함께 사용](#using-php-with-import)
- [Python을 `IMPORT`와 함께 사용](#using-python-with-import)
- [Ruby를 `IMPORT`와 함께 사용](#using-ruby-with-import)
- [Caddy를 파일 서버로 사용](#using-caddy-as-a-file-server)
- [nginx를 파일 서버로 사용](#using-nginx-as-a-file-server)

{{site.data.alerts.callout_info}}<code>cockroach </code>를 실행하는 컴퓨터는 파일 서버로 사용하지 않는 것을 권장합니다. cockroach를 실행하는 컴퓨터를 파일 서버로 사용하면 I/O 작업이 용량을 초과할 경우 성능에 부정적인 영향을 미칠 수 있습니다.{{site.data.alerts.end}}

### PHP를 `IMPORT`와 함께 사용

PHP 언어는 HTTP 서버를 내장하고 있으며, 아래 명령을 사용하여 로컬 파일을 제공할 수 있습니다. 로컬로 제공된 파일을 가져오는 방법에 대한 자세한 내용은 [`IMPORT`][import]문의 설명서를 참조하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cd /path/to/data
$ php -S 127.0.0.1:3000 # files available at e.g., 'http://localhost:3000/data.sql'
~~~

### Python을 `IMPORT`와 함께 사용

Python 언어는 HTTP 서버를 내장하고 있으며, 아래 명령을 사용하여 로컬 파일을 제공할 수 있습니다. 로컬로 제공된 파일을 가져오는 방법에 대한 자세한 내용은 [`IMPORT`][import]문의 설명서를 참조하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cd /path/to/data
$ python -m SimpleHTTPServer 3000 # files available at e.g., 'http://localhost:3000/data.sql'
~~~

If you use Python 3, try:

{% include copy-clipboard.html %}
~~~ shell
$ cd /path/to/data
$ python -m http.server 3000
~~~

### Ruby를 `IMPORT`와 함께 사용

Ruby 언어는 HTTP 서버를 내장하고 있으며, 아래 명령을 사용하여 로컬 파일을 제공할 수 있습니다. 로컬로 제공된 파일을 가져오는 방법에 대한 자세한 내용은 [`IMPORT`][import]문의 설명서를 참조하십시오.

{% include copy-clipboard.html %}
~~~ shell
$ cd /path/to/data
$ ruby -run -ehttpd . -p3000 # files available at e.g., 'http://localhost:3000/data.sql'
~~~

### Caddy를 파일 서버로 사용

1. [Caddy 웹 서버 다운로드](https://caddyserver.com/download):
다운로드 전 **Customize your build** 단계에서, **Plugins** 을 열고 `http.upload` 옵션이 체크되어 있는지 확인하십시오.

2. 제공하려는 파일이 들어 있는 디렉터리에 `caddy` 바이너리를 복사한 후 커맨드 라인 또는 [캐디 파일](https://caddyserver.com/docs/caddyfile)을 통해 이를 [업로드 지시문](https://caddyserver.com/docs/http.upload)과 함께 실행하십시오.

- 커맨드라인 예시 (TLS 없이):
    {% include copy-clipboard.html %}
    ~~~ shell
    $ caddy -root /mnt/cockroach-exports "upload / {" 'to "/mnt/cockroach-exports"' 'yes_without_tls' "}"
    ~~~
- `Caddyfile` example (using a key and cert):
    {% include copy-clipboard.html %}
    ~~~ shell
    tls key cert
    root "/mnt/cockroach-exports"
    upload / {
      to "/mnt/cockroach-exports"
    }
    ~~~

Caddy에 대해 더 자세한 설명이 알고 싶다면, [관련 문서](https://caddyserver.com/docs)를 참조하십시오.

### nginx를 파일 서버로 사용

1. `webdav` 모듈과 함께 (`-full`또는 이와 유사한 이름의 패키지에 포함되는 경우가 많다) `nginx` 설치 

2. `nginx.conf` 파일에서 `dav_methods PUT DELETE` 지시문을 추가한다. 예시:

    {% include copy-clipboard.html %}
    ~~~ nginx
    events {
        worker_connections  1024;
    }
    http {
      server {
        listen 20150;
        location / {
          dav_methods  PUT DELETE;
          root /mnt/cockroach-exports;
          sendfile           on;
          sendfile_max_chunk 1m;
        }
      }
    }
    ~~~

## 더 알아보기

- [`IMPORT`][import]
- [`BACKUP`](backup.html) (*Enterprise only*)
- [`RESTORE`](restore.html) (*Enterprise only*)

<!-- Reference Links -->

[import]: import.html
