---
type: reference
status: active
publish: true
date: 2026-06-09
tags:
  - type/reference
  - topic/dev
  - topic/software-engineering
  - topic/programming-languages
  - status/active
topics:
  - Reflection
  - Dialect
  - Metaprogramming
  - Introspection
  - Macros
  - DSL
aliases:
  - 리플렉션
  - 메타프로그래밍
  - 프로그래밍 언어 방언
---

# Reflection · Dialect · Metaprogramming

코드가 데이터를 처리하는 단계를 넘어 **자기 자신 또는 언어 규칙과 상호작용**하는 언어 이론/메타프로그래밍 용어 정리.

## Reflection (리플렉션)
프로그램이 **런타임에 자기 자신의 구조와 동작을 검사·수정**하는 능력. (운전 중인 차가 스스로 보닛을 열어 엔진을 점검·튜닝하는 격.)

- 객체가 "나는 어떤 클래스인가?", "어떤 메서드를 갖는가?" 를 물어 컴파일 시점에 몰랐던 메서드도 동적으로 실행.
- 사례: ORM 이 DB row 를 객체로 변환할 때 객체의 청사진을 리플렉션으로 들여다봄. 프레임워크에서 매우 흔함.
- 대표 언어: Java, Python, C#, Ruby.

## Dialect (방언)
기존 언어의 **변형·확장**. 부모 언어의 문법·의미를 공유하되 새 기능/규칙/문법 변화를 더함. (미국식 vs 영국식 영어처럼.)

- **Lisp** — Clojure, Scheme, Common Lisp 이 모두 Lisp 방언.
- **TypeScript** — JavaScript 에 정적 타입을 더하고 다시 JS 로 컴파일하는 방언으로 통용.
- **C++** — 초기엔 "C with Classes", 사실상 C 의 방언으로 출발.

## 관련 개념 ("the like")

- **Introspection (인트로스펙션)** — 리플렉션의 **읽기 전용** 버전. 타입·속성을 검사하지만 *변경은 안 함*. 예: JS `typeof`, Python `isinstance()`.
- **Metaprogramming (메타프로그래밍)** — 리플렉션을 포함하는 **상위 우산 개념**. "코드를 작성·조작하는 코드를 쓰는 것."
- **Macros (매크로)** — **컴파일 타임**에 코드를 생성하는 규칙(Rust, C, Lisp). 런타임에 작동하는 리플렉션과 달리, 프로그램 실행 전 코드로 펼쳐짐(expand).
- **DSL (Domain-Specific Language)** — 한 가지 작업에 특화된 미니 언어. 예: SQL(데이터베이스용). Ruby 등에서 메타프로그래밍으로 "internal DSL" 을 만들어 코드를 자연어처럼 읽히게 하기도 함.

> [!note] 핵심 축
> 리플렉션/인트로스펙션 = **런타임** 자기 검사(수정 여부로 갈림). 매크로 = **컴파일 타임** 코드 생성. 둘 다 메타프로그래밍의 하위. 방언/DSL 은 언어 *설계* 축의 개념.

---
인덱스: [[03-Resources/Jargon/index|Jargon]]
