---
  author: 조우성
  pubDatetime: 2026-05-23T14:20:50.017Z
  modDatetime: 2026-05-23T14:20:50.298Z
  title: fastapi 긴 태스크 최적화
  slug: fastapi-긴-태스크-최적화
  featured: true
  draft: false
  tags:
    - python
    - infra
  description: 서버에서 큰 백그라운드 작업에 의한 영향 간섭 최적화
---
## Table of contents

# FastAPI에서 `asyncio.create_task()` vs `ThreadPoolExecutor`

FastAPI에서 SSE나 `httpx.AsyncClient()` 기반 스트리밍을 유지하면서 백그라운드 작업을 수행해야 하는 경우가 생겼다. 현재 구조는 다음과 같다.  

![업로드 이미지](https://github.com/Oldentomato/astro-paper/blob/main/src/data/images/1779547152771-new_vm.png?raw=true)  


| 작업 | 특성 |
|------|------|
| BigQuery 호출 | 네트워크 I/O 중심, sync SDK |
| GCS 업로드/다운로드 | 네트워크 I/O 중심, sync SDK |
| pandas 처리 | CPU + 메모리 (현재는 소규모) |
| agent orchestration | sync/blocking 가능성 |
| sync 기반 라이브러리 | event loop blocking 위험 |

---

## asyncio의 기본 구조

FastAPI의 async endpoint는 보통 **하나의 event loop** 위에서 동작한다. 구조를 단순화하면 다음과 같다.

```text
FastAPI Process
 └─ Event Loop
     ├─ request coroutine
     ├─ SSE coroutine
     ├─ websocket coroutine
     └─ background coroutine
```

**핵심:** 모든 async task는 **같은 event loop를 공유**한다. 따라서 loop 안에서 blocking이 발생하면 SSE·다른 API 요청·스트리밍까지 함께 지연될 수 있다.

이 저장소에서는 두 경로를 분리한다.

| 경로 | 방식 | blocking 여부 |
|------|------|----------------|
| `POST /stream` | `httpx.AsyncClient.stream` + `StreamingResponse` | non-blocking (pure async) |
| `POST /jobs` | `run_in_executor` + sync BQ/GCS/pandas | blocking 작업은 thread/process pool로 격리 |

---

## `asyncio.create_task()`란?

`asyncio.create_task()`는 **새 OS thread나 process를 만드는 것이 아니다.**  
현재 event loop에 coroutine을 등록해 **동시에 진행**하게 하는 것이다.

```python
import asyncio

async def background():
    await asyncio.sleep(5)
    print("done")

async def main():
    asyncio.create_task(background())
    return "ok"
```

이 경우:

- 새로운 OS thread 생성 **없음**
- 새로운 process 생성 **없음**
- **같은 event loop**에서 스케줄링됨

pure async I/O(`await` 기반)만 있다면 가장 가볍고 적합한 방식이다.

---

## `create_task`의 문제점

task 내부에 **blocking 코드**가 들어가면 event loop 전체가 멈출 수 있다.

```python
async def background():
    import time
    time.sleep(10)  # blocking — loop 정지
```

```python
async def background():
    df = pandas_heavy_work()  # GIL + CPU — loop 지연
```

발생 가능한 영향:

- SSE heartbeat·chunk 전송 지연
- 다른 요청 latency 증가
- websocket timeout
- upstream SSE 프록시 지연

즉, **같은 loop 안에서 blocking이 실행되면 async 시스템 전체에 영향**이 간다.

> **주의:** `google-cloud-bigquery`, `google-cloud-storage` 등 공식 클라이언트는 대부분 **동기(sync)** API다. `async def` 안에서 그대로 호출하면 `create_task`로 감싸도 loop를 막는다.

---

## `ThreadPoolExecutor`란?

`ThreadPoolExecutor`는 blocking 작업을 **별도 thread**에서 실행한다.

```text
Event Loop
 ├─ SSE (httpx async stream)
 ├─ API request
 └─ ThreadPoolExecutor
     ├─ thread 1  ← BQ job
     ├─ thread 2  ← GCS upload
     └─ thread 3  ← pandas transform
```

- event loop는 계속 살아 있음
- blocking 작업만 thread로 분리
- `loop.run_in_executor(executor, fn, *args)`로 결과를 기다리거나 fire-and-forget 가능

### 현재 workload 특성

| 작업 | 특성 |
|------|------|
| BigQuery | network wait 비중 큼 |
| GCS | network wait 비중 큼 |
| pandas | 단순 transform, 소규모 DataFrame |
| agent | sync/blocking 가능 |

전체적으로 **CPU-heavy보다 I/O mixed workload**에 가깝다.  
목표는 **SSE·async 스트리밍 안정성** — event loop를 blocking하지 않는 것이다.

---

## 왜 `ProcessPoolExecutor` 대신 `ThreadPoolExecutor`인가

### 1. BQ/GCS는 대부분 I/O 대기

BigQuery·GCS 호출 시간의 상당 부분은 **네트워크 대기**다. CPU를 계속 점유하는 작업이 아니므로 thread 기반 처리에 적합하다.

### 2. pandas 작업이 무겁지 않음

현재 pandas 사용은 단순 transform·소규모 DataFrame 수준이라 process 분리까지는 필요하지 않다.

### 3. SSE 안정성 확보가 목적

`ThreadPoolExecutor`만으로도 SSE, websocket, `httpx` async stream을 event loop에서 분리해 보호할 수 있다.

### 4. `ProcessPoolExecutor` 대비 구현 단순

process 기반은 다음 부담이 있다.

- pickle·직렬화 필요
- 메모리·startup 비용 증가
- 공유 상태(예: in-memory job status) 접근 복잡

thread는 메모리 공유·기존 객체 재사용이 쉽고 구현이 단순하다.

### 선택적 process pool

pandas가 커지거나 CPU-bound가 지배적이면 `ProcessPoolExecutor`를 고려할 수 있다.  
이 저장소는 환경 변수 `USE_PROCESS_POOL=true`로 전환 가능하다 (`app/core/executor.py`).

---

## FastAPI 적용 예시

### 1. Executor 생성 (앱 lifespan)

```python
# app/core/executor.py
from concurrent.futures import Executor, ProcessPoolExecutor, ThreadPoolExecutor

def create_executor(settings: Settings) -> Executor:
    if settings.use_process_pool:
        return ProcessPoolExecutor(max_workers=settings.background_workers)
    return ThreadPoolExecutor(
        max_workers=settings.background_workers,
        thread_name_prefix="bg-job",
    )
```

```python
# app/main.py
@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    app.state.executor = create_executor(settings)
    yield
    app.state.executor.shutdown(wait=True, cancel_futures=False)
```

앱 종료 시 pool을 정리해 thread/process leak을 방지한다.

### 2. blocking 작업 정의 (sync 함수)

thread/process pool 안에서는 **sync 함수**로 작성한다. `await`가 필요 없다.

```python
# app/services/background_job.py
def run_background_job(job_id: str, params: dict[str, Any]) -> None:
    # sync BigQuery / GCS / pandas
    ...
```

### 3. endpoint에서 실행

```python
# app/routers/jobs.py
@router.post("/jobs", response_model=JobResponse)
async def create_job(req: JobRequest, request: Request) -> JobResponse:
    job_id = str(uuid.uuid4())
    loop = asyncio.get_running_loop()
    executor = request.app.state.executor
    loop.run_in_executor(executor, run_background_job, job_id, req.model_dump())
    return JobResponse(status="ok", job_id=job_id)
```

동작 요약:

- API는 즉시 `{"status": "ok", "job_id": "..."}` 반환
- 실제 BQ/GCS/pandas는 pool thread(또는 process)에서 실행
- event loop는 SSE·다른 async 요청 처리 가능

### 4. SSE는 pure async 유지

```python
# app/services/sse_proxy.py
async with httpx.AsyncClient(timeout=timeout) as client:
    async with client.stream(...) as response:
        async for chunk in response.aiter_bytes():
            yield chunk
```

SSE 경로에는 **sync blocking 호출을 넣지 않는다.**

### 환경 변수

```env
BACKGROUND_WORKERS=4
USE_PROCESS_POOL=false
```

---

## worker란 무엇인가

**worker**는 문맥에 따라 의미가 다르다. 혼동하지 않도록 구분한다.

### 1. uvicorn worker

```bash
uvicorn app.main:app --workers 2
```

여기서 worker = **FastAPI 서버 프로세스** 개수.

```text
Process 1  
  └─ event loop  (독립)
Process 2  
  └─ event loop  (독립)
```

- CPU 멀티코어 활용·프로세스 격리에 유리
- in-memory job 상태·SSE 연결은 **프로세스 간 공유되지 않음**
- long-lived SSE는 sticky session 없이 `--workers 2` 이상이면 주의

### 2. `ThreadPoolExecutor` worker

```python
ThreadPoolExecutor(max_workers=4)
```

여기서 worker = **동시에 실행 가능한 background 작업 수**(thread 개수).

- 한 프로세스·한 event loop 안에서의 **백그라운드 동시성 상한**
- uvicorn worker 수와는 **별개**로 튜닝한다

---

## `max_workers` 산정

workload가 I/O mixed이면 CPU 코어 수에 엄격히 맞출 필요는 없다.

| workload | `max_workers` 추천 |
|----------|-------------------|
| CPU-heavy | 코어 수 근처 |
| I/O-heavy | 코어 수보다 크게 가능 (단, 외부 API 한도 주의) |
| mixed (BQ/GCS + 가벼운 pandas) | **4~8**부터 시작 |

### 이 프로젝트 기준

기본값 `BACKGROUND_WORKERS=4` (`app/config.py`).

이유:

- SSE 안정성 확보에 충분한 여유
- BQ/GCS 대기 시간 동안 다른 job 실행 가능
- thread 과다·외부 API 동시 호출 폭증 위험 완화
- 구현·운영 단순

### 너무 크게 잡으면 안 되는 이유

- thread마다 메모리·스택 비용
- context switching 증가
- GIL 경쟁 (CPU-bound 구간)
- BigQuery/GCS **quota·rate limit** 압박

**작게 시작 → latency·queue·에러율 모니터링 → 필요 시 증가**가 안전하다.

---

## 다른 선택지와 비교

| 방식 | 장점 | 단점 | 적합한 경우 |
|------|------|------|-------------|
| `asyncio.create_task()` | 가장 가벼움 | blocking에 취약 | pure async I/O만 |
| `BackgroundTasks` | FastAPI 내장, 간단 | **같은 event loop**에서 실행 — 무거운 sync 작업에 부적합 | 응답 후 가벼운 정리 |
| `run_in_executor` + thread | loop 보호, 구현 단순 | GIL, thread 비용 | BQ/GCS/sync SDK, 가벼운 pandas |
| `run_in_executor` + process | CPU-bound 격리 | pickle·복잡도·메모리 | 대용량 pandas, heavy CPU |
| Celery / Cloud Tasks 등 | 확장·재시도·큐 | 인프라 복잡 | 장시간·대량·분산 job |

이 저장소는 **단일 프로세스 + SSE 동시 유지**가 목표이므로, 외부 큐 없이 `ThreadPoolExecutor` + `run_in_executor`가 현실적인 균형점이다.

---

## 최종 정리

### `asyncio.create_task()`

| | |
|---|---|
| **장점** | 매우 가벼움, pure async에 최적 |
| **단점** | blocking 코드에 취약, loop 전체 영향 |
| **적합** | `await`만 있는 coroutine, SSE 내부 async 스트림 |

### `ThreadPoolExecutor`

| | |
|---|---|
| **장점** | blocking 격리, event loop 보호, 구현 단순 |
| **단점** | thread 비용, GIL, 동시성 상한 |
| **적합** | BigQuery, GCS, sync SDK, 가벼운 pandas |

### 결론

현재 workload는 BQ/GCS 중심이고 pandas는 무겁지 않으며, **SSE 안정성이 중요**하다.  
따라서 blocking 작업은 **`ThreadPoolExecutor` + `run_in_executor`로 event loop 밖 thread에 분리**하는 방식이 가장 현실적이고 단순한 선택이다.

SSE·upstream 프록시는 **async만** 사용하고, 무거운 GCP/pandas 작업은 **`POST /jobs`**처럼 pool로 격리하는 **이중 경로**를 유지하는 것이 권장 패턴이다.


