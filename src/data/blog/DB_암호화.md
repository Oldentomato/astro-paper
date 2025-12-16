---
  author: jowoosung
  pubDatetime: 2025-12-16T14:34:51.623Z
  modDatetime: 2025-12-16T14:34:52.208Z
  title: DB 암호화
  slug: db암호화
  featured: true
  draft: false
  tags: 
    - infra
    - secure
  description: DB 암호화하는 방법을 fastAPI를 이용하여 구성
---
## Table of contents. 
## DB 암호화
요즘처럼 보안에 민감한 시기에 db를 암호화하는 방법에 대해 심도있게 공부하려한다.  
이 글은 다음 블로그의 내용을 참고하여 작성되었다.  
[블로그 이동하기](https://littlemobs.com/blog/db-personal-info-encryption-introduction/?utm_source=chatgpt.com)  
아래의 칼럼들로 구성되어있는 테이블이 있다고 가정하겠다.  
- name: 사용자의 이름
- email: 사용자의 이메일 주소 
- password: 사용자 계정의 비밀번호
그냥 일반적으로 데이터를 암호화하여 저장만 한다면 간단하게 끝날 일이지만,  
다음과 같은 요구사항들이 추가된다면 고려해야할 부분이 늘어난다.  
- 암호화: name, email, password는 모두 bytes 형식으로 저장되어야 한다.  
- 정렬: name과 email은 ORDER BY 쿼리를 통해 정렬이 가능해야 한다.  
- 검색: name과 email은 LIKE 쿼리를 통해 검색이 가능해야 한다.  
- 보안: password는 어떤 방법으로도 복호화할 수 없게 만들어야 한다.  
- 고유성: email은 사용자 계정 로그인에 필요한 값이므로, unique해야 한다.  
위의 요구사항들을 모두 만족시키려면 다양한 방법들을 이용해야 한다. 그리고 이 방법들을 찾기 위해 고려해야할 내용은 크게 세가지이다.  
- 복호화 가능 여부  
- 암호화와 복호화의 주체  
- 랜덤성 여부  
이 다음부터 위 사항들을 고려한 방법들에 대해 설명하겠다.  

## 요구사항에 따른 암호화 방법  
### 복호화 가능여부와 랜덤성  
우선 주어진 칼럼들은 모두 암호화가 되어야하는데, 여기서 name과 email은 암호화/복호화가 되어야 한다.  
저장이 될때는 암호화로 저장되고 읽어올때는 복호화가 되어야하고, password는 복호화할 필요없이 사용자의 입력을 암호화하고 대조만 하면 되기 때문에 암호화만 걸어주면 된다.  
  
그리고 암호화를 할 때는 매번 다른 값으로 변환되게, 즉 랜덤성이 있어야한다.  
매번 같은 값으로 암호화가 된다면 여러 개의 암호화된 문자열을 두고 패턴을 찾게되고 결국엔 원본 문자열이 노출될 수가 있다.  
이렇게 매번 같은 값으로 암호화되는 것을 결정적 암호화(deterministic encryption)라고 한다.  
  
### 암호화와 복호화의 주체  
암호화와 복호화는 코드단, 즉 Application에서 수행되거나 DB내 사용자 정의함수로 인해 수행될 수 있다.  
결론만 말하자면 암호화/복호화 처리를 해야하는 부분은 DB에서 하는것이 효율적이다.  
데이터를 암호화하여 저장한다면 LIKE나 ORDER BY같은 쿼리를 사용할 수 없다.  
Application에서 수행된다면 모든 데이터를 조회하여 가져온 다음, 모두 복호화해야 정렬이나 초성검색이 가능하다.  
이러면 속도도 느리고 메모리도 부족해지기 때문에 매우 비효율적이기에 단순히 SELECT 쿼리만 사용해야한다.  
하지만 꼭 DB에서만 해야하는 법은 아니다. 아까처럼 SELECT 쿼리만 사용해도 되면서 비교 로직을 더 심도있게 작성하고 싶다면 Application단에서 수행해도 된다.  
  
## 암호화된 unique 값  
요구사항의 마지막 부분을 보면 "email은 사용자 계정 로그인에 필요한 값이므로, unique해야 한다." 라고 나와있다.  
데이터들은 deterministic encryption으로 암호화되어야 하지만, unique해지려면 랜덤성이 있으면 의미가 없어진다.  
그러면 어쩔수없이 deterministic encryption방식으로 암호화를 해야하기에 column을 하나 더 추가하여 저장하도록 한다.  
하지만 이 방식도 결국엔 의미가 없는 것이 여러 값들이 쌓여 패턴이 파악되면 암호화를 걸어도 의미가 없어지긴 한다.  
고로, 이 요구사항은 처음부터 모순관계라 완벽한 솔루션은 없다고 보면 된다.  
  
모든 칼럼의 내용에 대한 처리방법을 정리하자면 다음 표와 같다.  
| Column | Decryptable | Encryptor | Randomness |
| --- | --- | --- | --- |
| name | O | DB | O |
| email | O | DB | O |
| password | X | APP | O |
| email_hash | X | APP | X |

다음은 위 내용들을 토대로 파이썬코드로 작성해보겠다.  

## FastAPI코드로 구성하기  
### 디렉토리 구성  
디렉토리 구성은 다음과 같다.  
```text
project/
├── app/
│   ├── db/
│   │   ├── pool.py      # DB 연결 & 세션 설정
│   │   └── encryption.sql     # DB 암호화 함수 정의
│   │
│   ├── security/
│   │   ├── password.py        # 비밀번호 해시
│   │   └── email.py           # email_hash 생성
│   │
│   ├── repository/
│   │   └── user_repository.py # SQL 접근 계층
│   │
│   └── main.py
│
├── config/
│   └── settings.py            # 환경변수 관리
│
└── requirements.txt
```

## DB 접속코드
db 접속구성 코드를 pool.py에서 작성해준다.  
```python
import asyncpg
from config.settings import DATABASE_URL, DB_ENCRYPT_KEY

_pool: asyncpg.Pool | None = None


async def init_db_pool():
    global _pool
    _pool = await asyncpg.create_pool(DATABASE_URL)


async def get_connection(): #pool에서 커넥션을 받아옴
    if _pool is None:
        raise RuntimeError("DB pool is not initialized")

    conn = await _pool.acquire()
    return conn


async def release_connection(conn): #pool에게 커넥션을 반납
    await _pool.release(conn)


async def init_session(conn):
    # 트랜잭션 안에서만 유효
    await conn.execute(
            "SELECT set_config('app.encrypt_key', $1, true)",
            DB_ENCRYPT_KEY
        )
```
여러 사용자의 db접속을 구성하기 위해 async형태의 conn을 pool로 받고, 반환하는 방식으로 구성했다.  
여기서 init_session()함수는 db작업을 수행하기 전 실행해주는 함수로써, 특정 상수(암호화를 위한 키)를 등록해준다.  
set_config의 3번째 인자에 true라고 되어있는데, 이는 SET LOCAL을 허용한다는 뜻으로 false를 한다면 SET으로 설정된다.  
SET으로 하면 세션 전체라는 뜻이고 SET LOCAL은 현재 트랜잭션만 이라는 뜻이다. 세션 전체로 해놓으면 키가 다른요청에 남을 수 있고, 자동으로 폐기되지도 않는다. 하지만 SET LOCAL로 설정해놓으면 각 트랜잭션에만 등록이 되고, 자동으로 폐기가 되기에 훨씬 안전하다.  

## DB내 함수 선언  
DB내에 암호화/복호화를 하기 위해서는 pgcrypto라는 extension과 사용자 정의 함수가 필요하다. 해당 쿼리의 내용은 다음과 같다.  
```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE OR REPLACE FUNCTION encrypt_text(p_text TEXT)
RETURNS BYTEA AS $$
BEGIN
    RETURN pgp_sym_encrypt(
        p_text,
        current_setting('app.encrypt_key')
    );
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION decrypt_text(p_data BYTEA)
RETURNS TEXT AS $$
BEGIN
    RETURN pgp_sym_decrypt(
        p_data,
        current_setting('app.encrypt_key')
    );
END;
$$ LANGUAGE plpgsql;
```

## user_repository  
이제 db에 접근하여 유저 데이터 읽기/쓰기 코드를 작성해준다.  
```python
# user_repository.py
from app.db.pool import get_connection, release_connection, init_session
from app.security.email import make_email_hash
from app.security.password import hash_password, verify_password

class UserRepository:
    async def create_user(self, email: str, name: str, password: str):
        conn = await get_connection()

        try:
            async with conn.transaction():
                await init_session(conn)

                await conn.execute("""
                    INSERT INTO users (
                        email,
                        name,
                        email_hash,
                        password_hash
                    )
                    VALUES (
                        encrypt_text($1),
                        encrypt_text($2),
                        $3::bytea,
                        $4::bytea
                    )
                """,
                email,
                name,
                make_email_hash(email),
                hash_password(password)
                )

        except Exception:
            raise

        finally:
            await release_connection(conn)

    # =========================
    # READ (단건 조회 - 복호화)
    # =========================
    async def get_user_by_email(self, email: str):
        conn = await get_connection()

        try:
            async with conn.transaction():
                await init_session(conn)

                row = await conn.fetchrow("""
                    SELECT
                        id,
                        decrypt_text(email) AS email,
                        decrypt_text(name)  AS name,
                        created_at
                    FROM users
                    WHERE email_hash = $1
                """, make_email_hash(email))

                if not row:
                    return None

                return dict(row)

        finally:
            await release_connection(conn)


    # =========================
    # AUTH
    # =========================
    async def authenticate(self, email: str, password: str):
        conn = await get_connection()

        try:
            async with conn.transaction():
                await init_session(conn)

                row = await conn.fetchrow("""
                    SELECT password_hash
                    FROM users
                    WHERE email_hash = $1
                """, make_email_hash(email))

                if not row:
                    return False

                return verify_password(password, row["password_hash"])

        finally:
            await release_connection(conn)

```
쿼리문 내에 encrypt_text함수가 있는데 이게 아까전에 등록한 사용자 정의함수이다.  
꼭 서버 실행전에 db에 위 함수를 등록해야 한다.  
여기서 email과 password를 hash해주는 함수가 있는데, 이 코드는 다음과 같다.  

## hash 함수  
```python
# email.py
import hashlib

def make_email_hash(email: str) -> bytes:
    return hashlib.sha256(email.encode()).digest()
```

```python
# password.py
import bcrypt

def hash_password(password: str) -> str:
    return bcrypt.hashpw(
        password.encode(),
        bcrypt.gensalt()
    )

def verify_password(password: str, hashed: str) -> bool:
    return bcrypt.checkpw(
        password.encode(),
        hashed
    )

```
email의 hashlib.sha256은 랜덤성이 없는 hash암호화이기 때문에 비밀번호같이 노출에 민감한 데이터에는 절대로 쓰여선 안된다.  
자동으로 랜덤한 salt로 변환되는 bcrypt를 이용하여 비밀번호를 암호화해야 한다.  

## 엔드포인트  
이제 간단한 테스트를 위해 엔드포인트 라우터를 작성해준다.  
```python
from fastapi import APIRouter, HTTPException
from app.repository.user_repository import UserRepository

router = APIRouter()
repo = UserRepository()


@router.post("/users")
async def create_user(email: str, name: str, password: str):
    try:
        await repo.create_user(email, name, password)
        return {"status": "ok"}
    except Exception as e:
        print(e)
        raise HTTPException(status_code=400, detail="User creation failed")


@router.post("/login")
async def login(email: str, password: str):
    if not await repo.authenticate(email, password):
        raise HTTPException(status_code=401, detail="Invalid credentials")
    return {"status": "authenticated"}

```
그리고 메인함수는 아래와 같다.  

```python
from fastapi import FastAPI
from app.api.users import router as user_router
from app.db.pool import init_db_pool

app = FastAPI()

@app.on_event("startup")
async def startup():
    await init_db_pool()

app.include_router(user_router)

```
