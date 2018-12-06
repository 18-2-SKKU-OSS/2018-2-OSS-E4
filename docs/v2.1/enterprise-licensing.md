---
title: Enterprise Licensing
summary: Request and set trial and enterprise license keys for CockroachDB
toc: true
---

CockroachDB는 코어 및 [엔터프라이즈 특징](https://www.cockroachlabs.com/pricing/)를 모두 포함하는 단일 바이너리를 배포합니다. 라이센스 키 없이 핵심 기능을 사용할 수 있습니다. 그러나 엔터프라이즈 기능을 사용하려면 평가판 또는 엔터프라이즈 라이센스 키가 필요합니다.

이 페이지는 CockroachDB에 대한 평가판 및 엔터프라이즈 라이센스 키를 얻고 설정하는 방법을 보여줍니다.


## 라이센스 종류

타입 | 특징
-------------|------------
**평가판 라이센스** | 시험 사용권은 30일 동안 무료로 CockroachDB 엔터프라이즈 기능을 사용해 볼 수 있습니다.
**엔터프라이즈 라이센스** | 유료 엔터프라이즈 라이센스는 CockroachDB 엔터프라이즈 기능을 더 긴 기간(1년 이상) 동안 사용할 수 있도록 해줍니다.

## 평가판 또는 엔터프라이즈 라이센스 키 가져오기

평가판 라이센스 키를 받기 위해 [the registration form](https://www.cockroachlabs.com/get-cockroachdb/) 를 작성하면 몇 분 이내에 e-메일을 통해 평가판 라이센스 키를 받을 수 있습니다.
엔터프라이즈 라이센스로 업그레이드하려면, <a href="mailto:sales@cockroachlabs.com">Sales에 컨택</a> 하십시오.

## 평가판 또는 엔터프라이즈 라이센스 키 설정

CockroachDB `root` 사용자는, CockroachDB 설정에 따라 [built-in SQL shell](use-the-built-in-sql-client.html) 불안정하거나 안전한 모드로 여십시오. 다음 예에서는 CockroachDB가 안전하지 않은 모드로 실행되고 있다고 가정하십시오. 그런 다음 `SET CLUSTER SETTING` 령을 사용하여 조직의 이름과 라이센스 키를 설정하십시오:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --insecure
~~~

{% include copy-clipboard.html %}
~~~ sql
>  SET CLUSTER SETTING cluster.organization = 'Acme Company';
~~~

{% include copy-clipboard.html %}
~~~ sql
>  SET CLUSTER SETTING enterprise.license = 'xxxxxxxxxxxx';
~~~

## 라이센스 키 확인

라이센스 키를 확인하려면, open the [built-in SQL shell](use-the-built-in-sql-client.html) 을 열고 `SHOW CLUSTER SETTING` 명령을 사용하여 조직 이름 및 라이센스 키를 확인하십시오:

{% include copy-clipboard.html %}
~~~ sql
>  SHOW CLUSTER SETTING cluster.organization;
~~~
~~~
+----------------------+
| cluster.organization |
+----------------------+
| Acme Company         |
+----------------------+
(1 row)
~~~

{% include copy-clipboard.html %}
~~~ sql
>  SHOW CLUSTER SETTING enterprise.license;
~~~
~~~
+--------------------------------------------------------------------+
|                         enterprise.license                         |
+--------------------------------------------------------------------+
| xxxxxxxxxxxx                                                       |
+--------------------------------------------------------------------+
(1 row)
~~~

라이센스 설정은 명령이 실행되는 노드의 cockroach.log에도 기록됩니다:

{% include copy-clipboard.html %}
~~~ sql
$ cat cockroach.log | grep license
~~~
~~~
I171116 18:11:48.279604 1514 sql/event_log.go:102  [client=[::1]:56357,user=root,n1] Event: "set_cluster_setting", target: 0, info: {SettingName:enterprise.license Value:xxxxxxxxxxxx User:root}
~~~

## 만료된 라이센스 갱신

라이센스가 만료되면 엔터프라이즈 기능이 작동을 멈추지만 프로덕션 설정은 영향을 받지 않습니다. 예를 들어 라이센스를 갱신할 때까지 백업 및 복원 기능은 작동하지 않지만 중단 없이 CockroachDB의 다른 모든 기능을 계속 사용할 수 있습니다.

만료된 라이센스를 갱신하려면, <a href="mailto:sales@cockroachlabs.com">Sales에 컨택</a> 하고, ㅣ그리고 새로운 라이센스를 [설정](enterprise-licensing.html#set-the-trial-or-enterprise-license-key) .

## 참고 문헌

- [`SET CLUSTER SETTING`](set-cluster-setting.html)
- [`SHOW CLUSTER SETTING`](show-cluster-setting.html)
- [Enterprise Trial –– Get Started](get-started-with-enterprise-trial.html)
