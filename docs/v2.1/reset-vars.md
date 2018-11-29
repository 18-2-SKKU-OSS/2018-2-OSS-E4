---
title: RESET (session variable)
summary: The SET statement resets a session variable to its default value.
toc: true
---

`RESET` [statement](sql-statements.html)는 [session variable](set-vars.html)을 클라이언트 세션의 기본값으로 재설정합니다.


## 필수 권한

세션 설정을 재설정할 때 [privileges](privileges.html)이 필요하지 않습니다.

## 시놉시스

<section>{% include {{ page.version.version }}/sql/diagrams/reset_session.html %}</section>

## 매개변수

 Parameter | Description 
-----------|-------------
 `session_var` | [session variable](set-vars.html#supported-variables) 이름. 

## Example

{{site.data.alerts.callout_success}} 세션 변수를 재설정하기위해 <a href="set-vars.html#reset-a-variable-to-its-default-value"><code>SET .. TO DEFAULT</code></a>를 사용합니다.{{site.data.alerts.end}}

{% include copy-clipboard.html %}
~~~ sql
> SET default_transaction_isolation = SNAPSHOT;
~~~

{% include copy-clipboard.html %}
~~~ sql
> SHOW default_transaction_isolation;
~~~

~~~
+-------------------------------+
| default_transaction_isolation |
+-------------------------------+
| SNAPSHOT                      |
+-------------------------------+
(1 row)
~~~

{% include copy-clipboard.html %}
~~~ sql
> RESET default_transaction_isolation;
~~~

{% include copy-clipboard.html %}
~~~ sql
> SHOW default_transaction_isolation;
~~~

~~~
+-------------------------------+
| default_transaction_isolation |
+-------------------------------+
| SERIALIZABLE                  |
+-------------------------------+
(1 row)
~~~

## See also

- [`SET` (session variable)](set-vars.html)
- [`SHOW` (session variables)](show-vars.html)
