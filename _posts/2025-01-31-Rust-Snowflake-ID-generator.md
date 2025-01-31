---
layout: post
title: "Rust로 Lock-Free 하게 Snowflake ID 생성기 만들기"
description: "두 개의 atomic 변수로 시작한 Snowflake ID 생성기가 어떻게 하나의 CAS 연산으로 진화했는지, 그 과정에서 마주친 문제를 파헤쳐봅니다."
keywords: "rust, snowflake, lock-free, mutex, atomic, cas, concurrency, thread-safety, memory-ordering"
author: "Jungwoo Song"
show_in_post_list: true
include_in_header: false
include_in_footer: false
robots: index, follow
---

# Rust로 Lock-Free 하게 Snowflake ID 생성기 만들기

## 들어가며
"아니, 이게 왜 안 되지?"

두 개의 atomic 변수를 사용해 구현한 Snowflake ID 생성기가 동시성 테스트에서 실패했습니다. 코드는 얼핏 보기에 멀티스레드 환경에서도 안전해 보였지만, 어딘가 놓친 것이 분명했죠.

이 글에서는 Twitter의 Snowflake ID 생성기를 Rust로 구현하면서, 두 개의 atomic 변수에서 하나의 CAS(Compare-and-Swap) 연산으로 발전시킨 과정을 공유하고자 합니다. 그 과정에서 마주친 race condition과 메모리 오더링 문제들, 그리고 이를 해결하기 위한 고민들을 함께 이야기해보겠습니다.

## Snowflake ID란?
분산 시스템에서 유일한 ID를 생성하는 것은 생각보다 까다로운 문제입니다. RDBMS의 auto increment에 의존하면 단일 DB 서버에 병목이 생기고 수평적 확장이 어려워집니다. UUID는 저장 공간을 많이 차지하고 인덱스 효율성이 떨어집니다.

Twitter는 이 문제를 해결하기 위해 Snowflake ID를 고안했습니다. 64비트 정수 하나로 이루어진 이 ID는 다음과 같은 구조를 가집니다:

| Sign Bit | Timestamp | Datacenter ID | Worker ID | Sequence |
|----------|-----------|---------------|-----------|----------|
| 1 bit    | 41 bits   | 5 bits       | 5 bits    | 12 bits  |
| 0        | 시간      | 데이터센터 ID | 워커 ID   | 순번     |


- 타임스탬프 (41비트): 밀리초 단위, 약 139년을 표현 가능
- 데이터센터 ID (5비트): 32개의 데이터센터 식별
- 머신 ID (5비트): 각 데이터센터당 32대의 서버 식별
- 시퀀스 번호 (12비트): 같은 밀리초 내에서 4096개의 고유 ID 생성 가능

이 구조는 64비트의 작은 사이즈로 ID의 순서를 보장하면서도 분산 시스템에서 서로 다른노드가 ID를 생성하더라도 항상 유일한 ID를 생성할 수 있도록 합니다.

## 두 개의 atomic 변수로 시작한 구현
기존 구현은 두 개의 atomic 변수를 사용해 구현했습니다:

```rust
#[derive(Debug)]
pub struct UniqueIdGenerator {
	epoch: SystemTime,
	pub datacenter_id: i32,
	pub machine_id: i32,
	timestamp: AtomicI64,
	sequence_num: AtomicI16,
}

impl UniqueIdGenerator {
	...
    pub fn generate(&self) -> i64 {
		self.sequence_num.store((self.sequence_num.load(Ordering::Relaxed) + 1) % 4096, Ordering::Relaxed);
		let mut now_millis = current_time_in_milli(self.epoch);

		if self.timestamp.load(Ordering::Relaxed) == now_millis {
			if self.sequence_num.load(Ordering::Relaxed) == 0 {
				now_millis = race_next_milli(self.timestamp.load(Ordering::Relaxed), self.epoch);
				self.timestamp.store(now_millis, Ordering::Relaxed);
			}
		} else {
			self.timestamp.store(now_millis, Ordering::Relaxed);
			self.sequence_num.store(0, Ordering::Relaxed);
		}

		self.get_snowflake()
    }
}
```

이 구현은 얼핏 보기에는 괜찮아 보이지만, 실제로는 몇 가지 심각한 문제점을 가지고 있었습니다:

### 1. 복잡한 메모리 오더링 요구사항

- 두 개의 `Atomic` 변수(`timestamp`와 `sequence_num`)가 서로 의존적인 관계를 가집니다
- 현재 사용된 `Ordering::Relaxed`는 이러한 의존성을 제대로 처리하지 못합니다
- 변수 간의 순서 보장이 필요한 경우 더 강력한 메모리 오더링(예: `Acquire`/`Release`)이 필요할 수 있습니다

### 2. 잠재적인 Race Condition

- `timestamp`와 `sequence_num` 업데이트가 각자 이뤄지다보니 잠재적인 Race Condition이 발생할 수 있습니다.

이러한 문제들로 인해 멀티스레드 환경에서 동일한 ID가 생성되는 심각한 버그가 발생할 수 있었습니다. 이는 Snowflake ID의 가장 기본적인 요구사항인 "유일성"을 위반하는 결과를 초래합니다.

## 해결 방안 탐색

이러한 문제들을 해결하기 위해 두 가지 접근 방식을 고려해보았습니다:

### 1. Bucket 기반 시퀀스 할당

첫 번째 접근 방식은 각 타임스탬프마다 미리 시퀀스 번호들을 버킷에 할당하는 것이었습니다:

```rust
struct SequenceBucket {
    timestamp: i64,
    sequences: Vec<i16>,
    current_index: AtomicUsize,
}
```

이 방식은 다음과 같은 장점이 있습니다:
- 시퀀스 번호의 선할당으로 경쟁 조건 감소
- 예측 가능한 시퀀스 번호 할당

하지만 다음과 같은 단점도 존재했습니다:
- 추가적인 메모리 사용
- 버킷 관리에 따른 오버헤드
- 여전히 버킷 전환 시점에서의 동기화 문제 존재

### 2. 단일 Atomic 변수와 CAS

두 번째 접근 방식은 타임스탬프와 시퀀스 번호를 하나의 atomic 변수에 담아 단일 CAS(Compare-and-Swap) 연산으로 처리하는 것이었습니다:

```rust
#[derive(Debug)]
pub struct UniqueIdGenerator {
	epoch: SystemTime,
	pub datacenter_id: i32,
	pub machine_id: i32,
    /// 64비트 정수 하나로 타임스탬프, 시퀀스 번호를 담습니다.
    id: AtomicI64,
}
```

이 방식의 장점은 다음과 같습니다:
- 하나의 원자적 연산으로 모든 상태 변경을 처리
- 메모리 사용량 최소화
- 더 단순한 구현과 명확한 동시성 보장

두 접근 방식을 비교 검토한 결과, 구현의 단순성과 확실한 동시성 보장을 위해 두 번째 방식을 선택했습니다. 이는 "단순할수록 좋다(Simple is better)"라는 원칙에도 부합했습니다.

```rust
impl UniqueIdGenerator {
    ...
    pub fn generate(&self) -> i64 {
        const MAX_SEQUENCE: i64 = 4095;

        loop {
            let now = current_time_in_milli(self.epoch);
            let current = self.id.load(Ordering::Relaxed);
            let last_ts = current >> 12;
            let last_seq = current & 0xFFF;

            if last_ts > now {
                spin_loop();
                continue;
            }

            let (new_ts, new_seq) = if last_ts == now {
                let next_seq = (last_seq.wrapping_add(1) & MAX_SEQUENCE) as i16;
                if next_seq == 0 {
                    let next_ts = self.wait_next_milli(now);
                    (next_ts, 0)
                } else {
                    (now, next_seq)
                }
            } else {
                (now, 0)
            };

            let new_id = new_ts << 12 | new_seq as i64;
            match self
                .id
                .compare_exchange(current, new_id, Ordering::Relaxed, Ordering::Relaxed)
            {
                Ok(_) => {
                    return (new_ts << 22)
                        | ((self.datacenter_id as i64) << 17)
                        | ((self.machine_id as i64) << 12)
                        | new_seq as i64;
                }
                Err(_) => continue,
            }
        }
    }
}
```

## 최종 결과: 더 안전하고 효율적인 구현

새로운 구현은 이전 버전의 모든 문제점을 해결하면서도 여러 가지 이점을 가져왔습니다:

1. **향상된 원자성**
   - 하나의 CAS 연산으로 모든 상태 변경을 처리
   - Race condition의 가능성을 원천적으로 제거

2. **단순화된 메모리 오더링**
   - 두 변수 간의 복잡한 의존성 제거
   - `Ordering::Relaxed`만으로도 충분한 안전성 보장

3. **성능 최적화**
   - 불필요한 메모리 배리어 제거
   - 컴파일러의 최적화 여지 확대
   - 캐시 친화적인 단일 변수 접근

이러한 개선사항들은 Snowflake ID 생성기의 신뢰성과 성능을 크게 향상시켰습니다.

## 결론

"아니, 이게 왜 안 되지?"로 시작했던 문제가 이제는 명쾌하게 해결되었습니다. 두 개의 atomic 변수를 사용한 직관적인 접근 방식으로 시작했던 구현이, 하나의 CAS 연산을 사용하는 우아한 해결책으로 진화했습니다.

이 과정에서 race condition과 메모리 오더링이라는 까다로운 문제들을 마주쳤지만, 단일 atomic 변수로의 전환은 이 모든 문제를 깔끔하게 해결해주었습니다. 특히 하나의 CAS 연산으로 모든 상태 변경을 처리하면서 race condition의 가능성을 원천적으로 제거할 수 있었고, 메모리 오더링에 대한 복잡한 고려사항도 크게 단순화할 수 있었습니다.

---

## 부록: 성능 비교

이 글에서 다룬 Lock-Free 구현이 실제로 얼마나 효율적인지 궁금하실 것 같아 벤치마크 결과를 공유합니다. 비교를 위해 가장 단순한 접근 방식인 Mutex 기반 구현도 함께 테스트했습니다.

### Mutex 기반 구현

```rust
pub struct MutexUniqueIdGenerator {
    epoch: SystemTime,
    datacenter_id: i32,
    machine_id: i32,
    timestamp: Mutex<i64>,
    sequence: Mutex<i64>,
}

impl MutexIdGenerator {
    pub fn generate(&self) -> i64 {
        loop {
            if let (Ok(mut sequence), Ok(mut timestamp)) =
                (self.sequence.try_lock(), self.timestamp.try_lock())
            {
                let now = SystemTime::now()
                    .duration_since(self.epoch)
                    .unwrap()
                    .as_millis() as i64;

                if *timestamp == now {
                    *sequence = (*sequence + 1) & 4095;
                    if *sequence == 0 {
                        *timestamp = self.wait_next_milli(now);
                    }
                } else {
                    *timestamp = now;
                    *sequence = 0;
                }

                return (*timestamp << 22)
                    | ((self.datacenter_id as i64) << 17)
                    | ((self.machine_id as i64) << 12)
                    | *sequence;
            }

            spin_loop();
        }
    }
}
```

### 벤치마크 환경
- CPU: Apple M2 Pro (8코어)
- Memory: 16GB
- OS: macOS Sonoma 14.4.1
- Rust: 1.83.0

### 벤치마크 결과

![Benchmark Result](/img/2025-01-31-snowflake-id-generator/snoflake-bench-result-violin.svg)

벤치마크 결과는 Lock-Free 구현이 Mutex 기반 구현보다 약 32배 더 빠르다는 것을 보여줍니다.

또 Violin Plot의 형태를 통해 다음과 같은 결론을 도출할 수 있습니다:
- Lock-Free(Atomic) 구현:
  - 매우 좁고 뾰족한 형태 → 대부분의 실행이 비슷한 시간에 집중됨
  - 성능이 매우 일관적
- Mutex 구현:
  - 넓고 완만한 형태 → 실행 시간이 넓게 분포
  - 성능 예측이 어려움
