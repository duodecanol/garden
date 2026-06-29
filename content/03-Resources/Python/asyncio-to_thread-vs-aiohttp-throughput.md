---
type: reference
status: active
publish: true
date: 2026-02-15
tags:
  - type/reference
  - topic/dev
topics:
  - python
  - asyncio
  - concurrency
  - performance
  - docker
related:
  - "[[Technical Stack & Concurrency for Internal KB]]"
source: https://github.com/EffectivePythonExercises/asyncio-requests-throughput-exp
---

# asyncio.to_thread(requests) vs aiohttp — 처리량 병목 실험

> [!info] Source
> 직접 수행한 실험을 정리한 레포: [EffectivePythonExercises/asyncio-requests-throughput-exp](https://github.com/EffectivePythonExercises/asyncio-requests-throughput-exp)
> 원시 데이터(`results*/*.json`), 플롯, 실행 스크립트는 레포가 권위 있는 출처. 이 노트는 결론과 재사용 가능한 지식의 요약본.

## 핵심 질문

> "기존 동기 HTTP 라이브러리(`requests`)를 `asyncio.to_thread()`로 감싸면 네이티브 async(`aiohttp`)와 같은 동시성을 얻을 수 있는가?"

답은 **아니오**. `asyncio.to_thread`의 기본 `ThreadPoolExecutor`가 **암묵적 세마포어**로 작동해 동시 I/O를 풀 크기만큼으로 고정한다. 동시 요청이 풀 크기를 넘기는 순간 처리량이 포화되고 초과분은 전부 큐잉 지연으로 전환된다.

## 가장 중요한 한 줄

> [!important] 병목의 본질
> `asyncio.to_thread`로 N개 코루틴을 동시에 띄워도 실제 블로킹 I/O를 수행하는 스레드는 기본 풀 크기(1 CPU에서 5개)뿐이다. 나머지는 스레드풀 큐에서 대기한다. 처리량은 동시성과 무관하게 **`pool_size / delay`** 로 고정된다.
>
> ```
> N coroutines → to_thread() → [ThreadPool: 5 workers] → network I/O
>                               ^^^^^^^^^^^^^^^^^^^^^^^^ 여기가 병목
> ```
> 이는 `asyncio.Semaphore(5)`를 건 것과 동일하지만, 개발자가 **인지하지 못한 채** 발생한다는 점이 위험.

## 메커니즘

`asyncio.to_thread(func, ...)`는 내부적으로 `loop.run_in_executor(None, ...)`을 호출 → 첫 인자가 `None`이면 이벤트 루프의 **기본 executor**(lazy 생성되는 `ThreadPoolExecutor`)를 사용한다. `max_workers`를 명시하지 않으므로 기본 공식이 그대로 적용된다.

`ThreadPoolExecutor` 기본 워커 수 공식:

| Python 버전 | 공식 |
|---|---|
| 3.5–3.7 | `os.cpu_count() * 5` |
| 3.8–3.12 | `min(32, os.cpu_count() + 4)` |
| 3.13+ | `min(32, (os.process_cpu_count() or 1) + 4)` |

1 CPU 환경: `min(32, 1 + 4) = 5`.

추가 비용: `to_thread`는 매 호출마다 `contextvars.copy_context()`를 실행 — 50만 회 기준 약 4.69% 오버헤드 ([cpython#136157](https://github.com/python/cpython/issues/136157)).

## 실험 설계

- **환경**: Docker 컨테이너, 1 CPU / 1GB RAM. Mock 서버는 50ms 고정 지연 후 JSON 응답.
- **독립 변수**: 클라이언트(`requests_thread` vs `aiohttp`) × 동시 요청 수(100 / 1,000 / 4,000 / 9,999).
- **측정**: 총 소요시간, 1초 버킷 처리량, 요청별 지연 분포.
- **3회 반복**: ① py3.11 ② py3.13 ③ py3.13 + `cpuset` + `PYTHON_CPU_COUNT`.

## 결과 (3차, 풀 5 확정)

| 동시성 | requests_thread | aiohttp | 배속 |
|:---:|---:|---:|:---:|
| 100 | 1.04s (96 req/s) | 0.09s (1,094 req/s) | 11.4× |
| 1,000 | 10.33s (97 req/s) | 0.32s (3,145 req/s) | 32.5× |
| 4,000 | 41.46s (96 req/s) | 1.26s (3,176 req/s) | 32.9× |
| 9,999 | 103.41s (97 req/s) | 3.01s (3,322 req/s) | 34.4× |

검증된 사항:

1. **처리량 = `pool_size / delay`.** 풀 32 → ~620 req/s, 풀 5 → ~97 req/s. 비율 620/97 ≈ 6.4 = 32/5 로 정확히 일치 (오차 ~3%, RTT·컨텍스트 스위칭).
2. **aiohttp는 동시성에 비례 확장.** 풀 5 환경에서 최대 **34배** 우위.
3. **큐잉 지연이 tail latency를 결정.** P99 = `(총 요청 / 풀 크기) × 서버 지연`. 풀 5, N=9999: 이론 99,990ms vs 실측 102,198ms (오차 2%).
4. **계단(staircase) 패턴.** 지연-인덱스 상관계수 N=9999에서 1.000000, 50ms 밴드당 요청 수가 정확히 풀 크기(5)와 일치. aiohttp에서는 계단 패턴이 전혀 나타나지 않음(버스트 완료).

지연 분포 (P50 / P99):

| 동시성 | requests_thread | aiohttp |
|:---:|---:|---:|
| 100 | 544 / 1,037ms | 83 / 86ms |
| 9,999 | 51,623 / 102,198ms | 2,014 / 2,660ms |

부수 발견: Python 3.13에서 aiohttp가 빨라짐 (N=4000 기준 1.59s → 1.06s, ~49% 향상, 이벤트 루프 최적화 추정).

![throughput-comparison](https://obsidian-image-worker.metalhwal.workers.dev/journal/2026/06/29/1782750368669-throughput_timeline.png?16VhkqqXQb)

## 별첨 지식: Docker에서 Python `os.cpu_count()` 제어

1·2차 실험에서 `cpus: "1"` 만으로는 스레드풀이 **5가 아니라 32**로 잡혔다. 이게 가장 재사용 가치가 높은 발견.

> [!warning] CFS 대역폭 제한 ≠ CPU 친화도(affinity)
> Docker `cpus: "1"`은 CFS 대역폭(`cpu.max`)만 제한할 뿐 affinity를 바꾸지 않는다. Python의 `os.cpu_count()` / `os.process_cpu_count()`는 둘 다 `sched_getaffinity()` 기반이라 CFS 제한을 **감지하지 못한다** → 호스트 CPU 수(32)를 반환 → 풀이 32로 생성됨.

제어 방법 비교:

| 방법 | `cpu_count()` | `process_cpu_count()` | `sched_getaffinity` | 스레드풀 | 코드 수정 | 최소 Python |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `cpus: 1` (CFS만) | 32 | 32 | 32 | 32 | 불필요 | — |
| `PYTHON_CPU_COUNT=1` | **1** | **1** | 32 | **5** | 불필요 | 3.13 |
| `cpuset: "0"` | 32 | **1** | **1** | **5**† | 불필요 | 3.13† |
| `sitecustomize.py` 몽키패치 | **1** | **1** | 변화없음 | **5** | 이미지만 | 모든 버전 |
| `os.sched_setaffinity(0,{0})` | 32 | **1** | **1** | **5**† | 필요 | 3.13† |

† Python 3.12 이하는 `ThreadPoolExecutor`가 `os.cpu_count()`를 쓰므로 효과 없음 (여전히 32).

효과 **없는** 방법: `OMP_NUM_THREADS`(OpenMP 전용), 컨테이너 내부 cgroup 직접 수정(권한 부족), `PYTHONSTARTUP`(대화형만).

권장 구성 (Python 3.13+, 세 레이어를 모두 적용):

```yaml
services:
  benchmark:
    cpuset: "0"                # 커널 스케줄링: CPU 0에서만 실행 → sched_getaffinity={0}
    deploy:
      resources:
        limits:
          cpus: "1"            # 커널 대역폭: CFS, 실제 CPU 시간 제한
          memory: 1g
    environment:
      PYTHON_CPU_COUNT: "1"    # Python 런타임: os.cpu_count() 반환값 오버라이드
```

Python 3.12 이하에서는 `sitecustomize.py` 몽키패치가 유일한 코드 무수정 방법:

```dockerfile
RUN echo 'import os; os.cpu_count = lambda: 1' > \
    /usr/local/lib/python3.11/site-packages/sitecustomize.py
```

## 실무 가이드라인

| 시나리오 | 권장 |
|---|---|
| 동시 HTTP 요청 < 30개 | `to_thread` + `requests` (간단하고 충분) |
| 동시 HTTP 요청 100개 이상 | `aiohttp` 네이티브 async (확장성 필수) |
| 레거시 동기 라이브러리 통합 | `to_thread` — 단 **기본 풀 크기 인지 필수** |
| 최대 처리량 프로덕션 | `aiohttp` + `TCPConnector` 튜닝 |

`to_thread`를 쓸 때 풀 크기를 키우려면: `loop.set_default_executor(ThreadPoolExecutor(max_workers=N))`.

## 참고 자료

- [CPython `asyncio/threads.py`](https://github.com/python/cpython/blob/main/Lib/asyncio/threads.py) · [`concurrent/futures/thread.py`](https://github.com/python/cpython/blob/main/Lib/concurrent/futures/thread.py)
- [`os.process_cpu_count()` 도입 — cpython#109649](https://github.com/python/cpython/issues/109649) · [`PYTHON_CPU_COUNT` — #109595](https://github.com/python/cpython/issues/109595) · [`to_thread` contextvars 오버헤드 — #136157](https://github.com/python/cpython/issues/136157)
- [psf/requests — Session thread safety #2766](https://github.com/psf/requests/issues/2766) · [PoolManager LRU 경합 #1871](https://github.com/psf/requests/issues/1871)
- [aiohttp TCPConnector 문서](https://docs.aiohttp.org/en/stable/client_reference.html)
- [Cal Paterson — "Async Python is not faster"](https://calpaterson.com/async-python-is-not-faster.html)
