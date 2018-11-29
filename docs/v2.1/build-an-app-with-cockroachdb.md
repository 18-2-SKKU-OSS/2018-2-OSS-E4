---
title: Build an App with CockroachDB
summary: The tutorials in this section show you how to build a simple application with CockroachDB, using PostgreSQL-compatible client drivers and ORMs.
tags: golang, python, java
toc: false
twitter: false
---

이 섹션의 튜토리얼에서는 PostgreSQL과 호환되는 클라이언트 드라이버와 ORM을 사용하여 CockroachDB로 간단한 어플리케이션을 제작하는 방법을 보여줍니다.

{{site.data.alerts.callout_info}}여기에 포함된 드라이버와 ORM은 <strong>베타-레벨</strong> 지원을 요청할 수 있을 정도로 테스트되었습니다. 즉, 드라이버나 ORM의 고급 기능이나 불명확한 기능을 사용하는 어플리케이션의 경우 호환성이 없을 수 있습니다. 만약 문제가 발생할 경우 상세 설명과 함께 <a href="https://github.com/cockroachdb/cockroach/issues/new">이슈 열기</a> 를 하여 저희가 전체를 지원할 수 있도록 도와주시길 부탁드립니다.</a>{{site.data.alerts.end}}

앱 언어 | 사용되는 드라이버 | 사용되는 ORM
-------------|-----------------|-------------
Go | [pq](build-a-go-app-with-cockroachdb.html) | [GORM](build-a-go-app-with-cockroachdb-gorm.html)
Python | [psycopg2](build-a-python-app-with-cockroachdb.html) | [SQLAlchemy](build-a-python-app-with-cockroachdb-sqlalchemy.html)
Ruby | [pg](build-a-ruby-app-with-cockroachdb.html) | [ActiveRecord](build-a-ruby-app-with-cockroachdb-activerecord.html)
Java | [jdbc](build-a-java-app-with-cockroachdb.html) | [Hibernate](build-a-java-app-with-cockroachdb-hibernate.html)
Node.js | [pg](build-a-nodejs-app-with-cockroachdb.html) | [Sequelize](build-a-nodejs-app-with-cockroachdb-sequelize.html)
C++ | [libpqxx](build-a-c++-app-with-cockroachdb.html) | No ORMs tested
C# (.NET) | [Npgsql](build-a-csharp-app-with-cockroachdb.html) | No ORMs tested
Clojure | [java.jdbc](build-a-clojure-app-with-cockroachdb.html) | No ORMs tested
PHP | [php-pgsql](build-a-php-app-with-cockroachdb.html) | No ORMs tested
Rust | [postgres](build-a-rust-app-with-cockroachdb.html) | No ORMs tested
