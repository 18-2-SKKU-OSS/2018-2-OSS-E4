---
title: ALTER DATABASE
summary: Use the ALTER DATABASE statement to change an existing database.
toc: false
---

`ALTER DATABASE` [statement](sql-statements.html)은 스키마(개요) 변화를 데이터베이스에 적용합니다.

{{site.data.alerts.callout_info}}
테이블 잠금 또는 기타 사용자가 볼 수 있는 중단시간 없이 CockroachDB가 스키마 요소를 어떻게 변화시키는지 이해하려면, [Online Schema Changes in CockroachDB](https://www.cockroachlabs.com/blog/how-online-schema-changes-are-possible-in-cockroachdb/)를 참조하십시오.
{{site.data.alerts.end}}

`ALTER DATABASE` 사용에 대한 정보는 관련 부속 명령에 대한 문서를 참조하십시오.

부속명령 | 설명
-----------|------------
[`CONFIGURE ZONE`](configure-zone.html) | <span class="version-tag">v2.1에서 :</span> 데이터 베이스를 위한 [복제 영역 구성](configure-replication-zones.html) 
[`RENAME`](rename-database.html) | 데이터베이스 이름 변경 
