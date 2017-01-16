#ORM 개념과 사용
#### BoB 5기 PJHS팀 김현우


## ORM 이란
우리 팀에서 사용하고 있는 SQLite (라즈베리파이)나 MySQL(패킷 분석 서버)은 흔히 볼 수 있는 관계형 데이터베이스 입니다. 관계형 데이터베이스는 데이터 베이스 테이블과의 관계성을 가진 데이터베이스를 말합니다. 

ORM은 Object-Relational Mapping의 약자로, 관계형 데이터베이스와 객체간의 매핑을 지원해주는 Framework (TOOL)을 지칭하는 말입니다. 

이 문서에서는 ORM에 대한 정보, 예제를 다루며 그 필요성에 대해 이야기해 보도록 하겠습니다.


기존에 우리가 `result.db` 와의 커넥션을 맺고, 쿼리에 대한 결과를 볼 때 사용한 코드입니다.
```python
def get_info_by_no_orm():
	cur = g.db.execute('select id, username from maillist')
	maillist = [dict(id=row[0], username=row[1]) for row in cur.fetchall()]
	return maillist
```

g.db.execute 에 들어간 조회 쿼리 컬럼을 하나하나 dict로 묶어준 후, 반환합니다. 


이번엔 ORM Framework 중 하나인 `SQLAlchemy`를 이용한 코드입니다.

```python
def get_info_by_orm():
	result = maillist.query.all()
	return result
```

`maillist`라는 테이블을 클래스로 정의하고, `query`를 통해 테이블에 대한 쿼리 객체를 준비하고 `all()`로 테이블에 있는 모든 row를 가져옵니다.

이번엔 위에서 정의한 각각의 `get_info` 함수를 이용해 view에서 template 에 넘기는 코드입니다.

```python
@app.route('/')
def index():
	result = get_info_by_no_orm()
	if len(result) <= 0:
		return redirect(url_for('error'))
	else ~
```

Raw SQL 문으로 가져온 데이터에 대한 Valid Check를 직접 해주어야 합니다.

하지만 ORM을 사용하면

```python
@app.route('/')
def index():
	result = maillist.query.first_or_404()
	return render_template('index.html', data=result)
```

SQLAlchemy 자체에서 Flask 의 View 관리도 해주기 때문에, 코드가 훨씬 더 간결해 집니다. 

(코드가 간결해진다는 것은 그만큼 오타의 비중을 줄일 수 있다는 뜻입니다.)


## ORM 사용법

보통 웹 개발을 할 때는 많은 개발자들이 MVC패턴을 사용합니다. (여기서 말하는 패턴은 코드의 디자인 패턴을 말합니다. 코드의 설계라고 봐도 무방합니다.)

_MVC패턴은 `Model`, `View`, `Controller` 의 구성으로, 
`Model`에서 `Controller`가 데이터를 가져오면 `View`에서 그 데이터를 보여주는 방식입니다. 흔하게 사용하는 방식이니 알아두시면 좋습니다._

**Flask 는 Micro Web Framework인 만큼, 다양한 방식의 코드가 나올 수 있습니다. 제 방식이 꼭 좋다는 말은 아니니, 많은 사람들의 코드를 보고, 가장 좋다고 느끼는 코드를 사용하는 것이 바람직합니다. Github 애용합시다.**

전체적인 app.py를 살펴보고, 이 파일에서의 MVC를 나눈 다음 한줄 한줄 살펴보도록 하겠습니다 :)

#### app.py
```python
from flask import Flask, redirect, url_for
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.SQLALCHEMY_DATABASE_URI = 'sqlite:///app.db'
db = SQLAlchemy(app)


class GuestBook(db.Model):
	_tablename_ = 'guestbook'
	id = db.Column(db.Integer, primary_key=True)
	content = db.Column(db.String(30), unique=True, nullable=True)


@app.route('/')
def index():
	return "Welcome to Kcrong's GuestBook !!"


@app.route('/show')
def show():
	return GuestBook.query.all()	

	
@app.route('/add/<string:content>')
def add(content=None):
	if content is None:
		return redirect(url_for('index'))
	
	g = GuestBook()
	g.content = content
	db.session.add(g)
	db.session.commit()

	
def main():
	app.run(host='0.0.0.0', port=5000, debug=True)	


if __name__ == '__main__':
	main()

```


전체적인 초반 코드입니다. 
```python
class GuestBook(db.Model): ~
```
데이터베이스에서 `GuestBook`이라는 테이블을 정의하며, 컬럼도 `id`와 `content` 컬럼을 자동으로 만들어줍니다. 

이 부분이 바로 `MVC`에서 `Model` 부분입니다.

```python
@app.route('/show')
def show(): 
	return GuestBook.query.all()
```
이 부분이 `GuestBook` 테이블에서 쿼리를 날리고, 데이터를 접속 유저에게 보여주는 부분입니다. `MVC`에서 `Controller + View` 부분입니다.

위 프로그램을 Raw Query 로 작성한다면 꽤나 골치아픈 일이 많아집니다. 우선 테이블을 정의할 때, 기존 DB파일에 데이터가 있는지 확인해주고, 없으면 SQL파일을 실행해야 한다던지, 행여 테이블 구조라도 바뀌게 된다면 `Alter` 문을 사용해서 테이블 구조를 하나하나 명시해주어야 합니다. 하지만 위 코드 처럼 `db.Model`을 상속받는 class 를 정의하고 사용한다면, 테이블 구조 변경과 같은 Migrate 이슈는 자동으로 ORM에서 해결해줍니다.

## ORM 좀 '더' 써보기
위 코드에서 테이블을 정의할 때, 단순히 테이블 명과 컬럼만을 정의하였습니다.
그리고 새 row 를 만들 때 
```python
g = GuestBook()
g.content = ~
```
이런 식으로 객체를 선언한 후, 객체에 대한 멤버 변수를 하나하나 정의해 주어야 합니다.

python 에서 class 를 정의할 때, `생성자` 라는 것을 이용하면 위 코드를 조금 간결하게 할 수 있습니다.

```python
class GuestBook(db.Model):
	_tablename_ = 'guestbook'
	id = db.Column(db.Integer, primary_key=True)
	content = db.Column(db.String(30), unique=True, nullable=True)

	def __init__(self, content):
		self.content = content

```
기존의 코드에 생성자를 추가해보았습니다. 이제 이 코드를 