---
layout: post
title: "부서(Dept) 트리 구조 리팩토링: 이중 for문에서 계층형 트리로"
comments: true
excerpt: "상위-하위 관계가 불명확한 부서 데이터의 이중 for문 순회를 계층형 트리 구조와 복합 PK로 재설계하고, depth 갱신·하위부서 탐색·Navigation 트리 빌드까지 실제 코드와 함께 기록합니다."
date: 2026-02-27
categories: [Server]
tags: [Refactoring, JPA, QueryDSL, TreeStructure, CompositeKey, SpringBoot, DataModeling]
---

# 부서(Dept) 트리 구조 리팩토링: 이중 for문에서 계층형 트리로

## 들어가며

사내 포털의 부서(Dept) 관리 페이지를 리뉴얼하면서, 기존 부서 데이터 구조의 근본적인 문제를 발견하게 되었습니다.

부서 데이터는 본질적으로 **트리(계층) 구조**입니다. "코코네M > SYF > Web팀"처럼 상위-하위 관계가 존재하고, 이 관계를 기반으로 조직도를 렌더링하거나, 특정 부서의 하위 부서를 모두 조회하거나, 메달 지급 대상자를 찾는 등의 기능이 필요합니다.

하지만 기존 구조에서는 이 상위-하위 관계가 명시적으로 정의되어 있지 않았고, 조회 시 이중 for문으로 전체 데이터를 반복 순회하고 있었습니다. 특히 **depth(계층 깊이) 계산 오류**, **하위 부서 탐색의 비효율**, **부서-사원 간 참조 정합성 부재** 등의 문제가 리뉴얼 과정에서 여러 곳에서 드러났습니다.

이 글에서는 기존 구조의 문제를 분석하고, 트리 구조 도입 → depth 갱신 로직 설계 → 복합 PK로 정합성 확보 → 하위부서 재귀 탐색 → Navigation 트리 빌드까지, 실제 코드와 함께 전 과정을 공유합니다.

---

## 서비스 전체 구조 이해

### 부서 데이터가 사용되는 곳

사내 포털에서 부서(Dept) 데이터는 다양한 기능의 기반이 됩니다.

```
부서(Dept)를 사용하는 주요 기능
├── 조직도 렌더링        ── 부서 트리를 UI로 표시
├── 메달 지급 대상자 조회  ── 특정 부서 + 모든 하위 부서의 사원 목록
├── Navigation 메뉴      ── 권한 기반 계층형 메뉴 구성
├── 크루(Crew) 관리       ── 사원의 소속 부서 정보
└── 권한(Role) 관리       ── 부서 단위 권한 부여
```

핵심 테이블 관계는 이렇습니다.

```
Dept (부서)
  ├── DeptCrew (부서-사원 매핑, 복합 PK: deptId + crewSeq)
  │     └── Crew (사원)
  ├── Role (권한: 사원별 부서 단위 권한)
  └── Navigation (계층형 메뉴, 별도 트리)
```

### 문제의 범위

부서 트리 구조의 문제는 하나의 기능이 아니라 **시스템 전반**에 영향을 미치고 있었습니다.

- **조직도**: depth 계산 오류로 3depth 부서가 2depth로 표시
- **메달 지급**: 하위 부서 탐색이 느리고, 누락되는 케이스 존재
- **Navigation**: depth 컬럼이 없어 클라이언트에서 계층을 추정
- **부서-사원 매핑**: 정합성 제약이 없어 잘못된 데이터 삽입 가능

---

## 기존 구조 분석

### 기존 부서 테이블

기존 부서 테이블에는 **상위 부서를 가리키는 컬럼이 없었습니다**.

```sql
CREATE TABLE dept (
    dept_id     VARCHAR(50) PRIMARY KEY,   -- 부서 식별번호 (문자열)
    dept_name   VARCHAR(100) NOT NULL,
    sort_order  INT DEFAULT 0,
    use_yn      CHAR(1) DEFAULT 'Y'
);
```

**SYF**와 **Web팀**이 상위-하위 관계라는 정보가 테이블 어디에도 없었습니다.

### 기존 트리 빌드: 이중 for문

부서 간 상위-하위 관계가 DB에 없으니, 조회 로직에서 **모든 부서 쌍을 비교**해야 했습니다.

```java
// 기존 코드 (의사 코드)
public List<DeptTreeResponse> buildDeptTree() {
    List<Dept> allDepts = deptRepository.findAll();
    List<DeptTreeResponse> result = new ArrayList<>();

    for (Dept dept : allDepts) {
        boolean isTopLevel = true;
        for (Dept other : allDepts) {
            if (isChildOf(dept, other)) {
                isTopLevel = false;
                break;
            }
        }
        if (isTopLevel) {
            DeptTreeResponse node = toResponse(dept);
            node.setChildren(findChildren(dept, allDepts)); // 재귀적으로 다시 전체 순회
            result.add(node);
        }
    }
    return result;
}
```

| 문제점 | 설명 |
|---|---|
| **O(n^2) 이상의 시간 복잡도** | 이중 for문 + 재귀 호출 |
| **관계 추정의 불안정성** | 이름/정렬 순서 등으로 부모-자식을 추정 |
| **depth 계산 오류** | 동적 계산 시 누락/중복 발생 |
| **데이터 정합성 부재** | DB 레벨 관계 제약 없음 |

### 기존 하위부서 탐색: 반복적 조회

메달 지급 대상자를 찾으려면 **특정 부서 + 모든 하위 부서**의 사원을 조회해야 합니다. 기존 하위부서 탐색 로직은 이렇게 되어 있었습니다.

```java
/**
 * 해당 그룹의 전체 하위부서 리스트 가져오기
 */
private List<String> getSubAllDeptIdList(List<String> deptIdList) {
    List<String> allDeptIdList = new ArrayList<>(deptIdList);
    List<String> subDeptIdList = new ArrayList<>(this.getSubDeptIdList(deptIdList));

    while (subDeptIdList.size() != 0) {
        allDeptIdList.addAll(subDeptIdList);
        subDeptIdList = new ArrayList<>(this.getSubDeptIdList(subDeptIdList));
    }

    return allDeptIdList;
}
```

이 코드는 **레벨별로 한 단계씩 내려가며 하위 부서를 조회**합니다. depth가 5단계라면, **DB 조회가 최소 5번** 발생합니다.

`getSubDeptIdList`의 내부를 보면:

```java
private List<String> getSubDeptIdList(List<String> deptIdList) {
    List<String> allDeptIdList = new ArrayList<>();

    for (String medalDeptId : deptIdList) {
        Dept parentDept = deptRepository.findByDeptId(medalDeptId);
        Map<String, DeptInfo> allDeptMap = deptExecutor.getAllMap();
        List<Dept> allDepts = deptInfoMapper
            .toEntity(allDeptMap.get(parentDept.getDeptId()).getChildren());

        allDeptIdList.addAll(allDepts.stream()
                .map(Dept::getDeptId)
                .toList());
    }

    return allDeptIdList;
}
```

매 레벨마다:
1. `deptRepository.findByDeptId()` — 부서 엔티티 조회
2. `deptExecutor.getAllMap()` — 전체 부서 맵 조회
3. children 추출

이 과정이 **레벨 수 x 해당 레벨의 부서 수**만큼 반복됩니다. 캐싱(`getAllMap`)이 있어 DB 부하는 줄었지만, 구조적으로 비효율적이었습니다.

---

## 트리 저장 모델 이론 비교

parentId를 도입하기 전에, 계층형 데이터를 RDB에 저장하는 대표적인 네 가지 모델을 비교 검토했습니다. 트리를 RDB에 매핑하는 문제는 오래된 주제이고, 각 방법에는 명확한 트레이드오프가 존재합니다.

### 1. Adjacency List (인접 리스트)

**부모의 PK를 자식 행에 저장**하는 가장 단순한 방식입니다. 이번 리팩토링에서 선택한 방식이기도 합니다.

```sql
CREATE TABLE dept (
    dept_id        VARCHAR(50) PRIMARY KEY,
    dept_name      VARCHAR(100) NOT NULL,
    parent_dept_id VARCHAR(50) NULL,       -- 부모 부서 참조
    depth          INT NOT NULL DEFAULT 0,
    FOREIGN KEY (parent_dept_id) REFERENCES dept(dept_id)
);
```

```
장점: INSERT/UPDATE/DELETE가 단순 (자기 행만 수정)
단점: 하위 전체 조회 시 재귀 호출 또는 CTE 필요
```

### 2. Nested Set (중첩 집합)

각 노드에 **left, right 번호**를 부여해서, 특정 노드의 하위 전체를 **범위 조건 하나**로 가져올 수 있는 방식입니다.

```sql
CREATE TABLE dept (
    dept_id   VARCHAR(50) PRIMARY KEY,
    dept_name VARCHAR(100) NOT NULL,
    lft       INT NOT NULL,
    rgt       INT NOT NULL
);

-- 하위 전체 조회: 범위 조건 한 방
SELECT * FROM dept
WHERE lft >= 2 AND rgt <= 11;
```

```
장점: 하위 전체 조회가 O(1) SQL 한 줄
단점: INSERT/DELETE 시 lft, rgt 번호를 대량 UPDATE 해야 함
```

노드 하나를 삽입하면 그 오른쪽에 있는 **모든 노드의 lft, rgt를 +2씩** 밀어야 합니다. 부서 수가 수백 건이라도, 한 번의 INSERT에 수십~수백 행이 UPDATE되는 구조이므로 쓰기 빈도가 높으면 부담이 큽니다.

### 3. Materialized Path (경로 구체화)

각 노드에 **루트부터 자신까지의 경로를 문자열로 저장**하는 방식입니다.

```sql
CREATE TABLE dept (
    dept_id   VARCHAR(50) PRIMARY KEY,
    dept_name VARCHAR(100) NOT NULL,
    path      VARCHAR(500) NOT NULL  -- 예: '/coconeM/SYF/Web'
);

-- 하위 전체 조회: LIKE 패턴 한 줄
SELECT * FROM dept
WHERE path LIKE '/coconeM/SYF/%';
```

```
장점: 하위 전체 조회가 LIKE 한 줄, 경로를 사람이 읽기 쉬움
단점: 부모가 바뀌면 해당 노드 + 모든 후손의 path를 일괄 UPDATE
      LIKE 'prefix%'는 인덱스를 탈 수 있지만, 경로가 길어지면 인덱스 효율 저하
```

### 4. Closure Table (폐쇄 테이블)

**모든 조상-후손 쌍**을 별도 테이블에 저장하는 방식입니다. 가장 정규화된 접근이지만, 공간 비용이 큽니다.

```sql
CREATE TABLE dept (
    dept_id   VARCHAR(50) PRIMARY KEY,
    dept_name VARCHAR(100) NOT NULL
);

CREATE TABLE dept_closure (
    ancestor_id   VARCHAR(50) NOT NULL,
    descendant_id VARCHAR(50) NOT NULL,
    depth         INT NOT NULL DEFAULT 0,
    PRIMARY KEY (ancestor_id, descendant_id),
    FOREIGN KEY (ancestor_id) REFERENCES dept(dept_id),
    FOREIGN KEY (descendant_id) REFERENCES dept(dept_id)
);

-- 하위 전체 조회: JOIN 한 번
SELECT d.* FROM dept d
JOIN dept_closure c ON d.dept_id = c.descendant_id
WHERE c.ancestor_id = 'SYF';
```

```
장점: 하위/상위 전체 조회가 모두 JOIN 한 번, 깊이 필터도 쉬움
단점: 노드 N개일 때 최대 O(N^2) 행이 필요, INSERT 시 조상 수만큼 행 삽입
```

### 네 모델의 비교 정리

| 연산 | Adjacency List | Nested Set | Materialized Path | Closure Table |
|---|---|---|---|---|
| INSERT | **O(1)** | O(n) lft/rgt 갱신 | O(1) | O(depth) 조상 수 |
| DELETE | **O(1)** | O(n) lft/rgt 갱신 | O(subtree) 후손 path | O(subtree) 후손 행 |
| 부모 변경 (reparent) | **O(1)** | O(n) | O(subtree) | O(subtree) |
| 하위 전체 조회 | O(depth) 재귀 | **O(1)** 범위 쿼리 | O(1) LIKE | **O(1)** JOIN |
| 공간 복잡도 | **O(n)** | **O(n)** | **O(n)** | O(n^2) 최악 |
| JPA 호환성 | **매우 좋음** | 보통 | 보통 | 별도 엔티티 필요 |

### 왜 Adjacency List를 선택했는가

우리 시스템의 특성을 정리하면 이렇습니다.

1. **부서 수가 수백 건 수준**: 전체 부서가 200~300개이므로, 하위 전체 조회의 재귀 비용이 문제가 되지 않습니다. depth가 최대 5단계이므로 BFS 쿼리도 5회면 충분합니다.

2. **변경 빈도가 매우 낮음**: 부서 구조는 연 1~2회 수준으로 변경됩니다. Nested Set이나 Materialized Path처럼 변경 비용이 큰 모델을 선택할 이유가 없었습니다.

3. **JPA/Hibernate와의 호환성**: Adjacency List는 `@ManyToOne` 자기 참조 한 줄이면 끝입니다. Nested Set의 lft/rgt 관리나 Closure Table의 별도 엔티티/리포지토리는 JPA 엔티티 설계를 복잡하게 만듭니다.

4. **팀의 이해도**: Adjacency List는 가장 직관적인 모델이므로, 팀 전체가 구조를 쉽게 이해하고 유지보수할 수 있습니다.

결론적으로, **읽기 성능이 극단적으로 중요하지 않고, 쓰기의 단순성과 유지보수성이 더 중요한 경우**에는 Adjacency List가 최적의 선택이었습니다. 만약 부서가 수만 건이거나, 하위 전체 조회가 초당 수백 번 호출되는 상황이었다면 Closure Table이나 CTE 재귀를 더 적극적으로 고려했을 것입니다.

---

## 1차 개선: parentId 도입 + 트리 빌드 리팩토링

### 테이블에 parentId 추가

가장 근본적인 개선은 **parentId 컬럼을 추가**하는 것이었습니다.

```sql
ALTER TABLE dept ADD COLUMN parent_dept_id VARCHAR(50) NULL;
ALTER TABLE dept ADD COLUMN depth INT NOT NULL DEFAULT 0;
```

### 트리 빌드: 단일 순회 makeChildStruct

parentId가 생기면서, 이중 for문 없이 트리를 구성할 수 있게 되었습니다. Navigation 메뉴에도 동일한 패턴을 적용했습니다.

```java
/**
 * tree 구조 기반 navigation childList 생성 로직
 */
@Transactional
public List<NavigationDto> makeChildStruct(List<NavigationDto> resultList) {
    List<NavigationDto> deptDtoList = new ArrayList<>();

    for (NavigationDto navigation : resultList) {
        if (navigation.getParentNavigationSeq() == null) {
            // 루트 노드: 결과 리스트에 직접 추가
            deptDtoList.add(navigation);
            continue;
        }
        // 부모를 찾아서 children에 추가
        resultList.stream()
            .filter(navigationDto -> Objects.equals(
                navigationDto.getNavigationSeq(),
                navigation.getParentNavigationSeq()))
            .forEach(navigationDto ->
                navigationDto.getChildren().add(navigation));
    }
    return deptDtoList;
}
```

이 로직의 핵심을 풀어보면:

```
입력: [A(root), B(parent=A), C(parent=A), D(parent=B)]  (flat list)

1회차: A → parentId == null → roots에 추가
2회차: B → parentId == A → A.children에 B 추가
3회차: C → parentId == A → A.children에 C 추가
4회차: D → parentId == B → B.children에 D 추가

결과:
A
├── B
│   └── D
└── C
```

**이 구현의 시간 복잡도 분석:**

현재 코드에서 각 비루트 노드마다 `resultList.stream().filter()`로 부모를 찾고 있으므로, 사실 **O(n^2)** 입니다. 하지만 이전의 이중 for문 + 재귀 구조보다는 단순하고, 부서 수가 수백 건 수준이므로 실무에서는 충분했습니다.

더 나아가 **O(n)으로 최적화**하려면 Map 기반으로 변경할 수 있습니다.

```java
// O(n) 버전: Map 기반 트리 빌드
public List<NavigationDto> makeChildStructOptimized(List<NavigationDto> resultList) {
    Map<Integer, NavigationDto> nodeMap = new LinkedHashMap<>();
    for (NavigationDto nav : resultList) {
        nodeMap.put(nav.getNavigationSeq(), nav);
    }

    List<NavigationDto> roots = new ArrayList<>();
    for (NavigationDto nav : resultList) {
        if (nav.getParentNavigationSeq() == null) {
            roots.add(nav);
        } else {
            NavigationDto parent = nodeMap.get(nav.getParentNavigationSeq());
            if (parent != null) {
                parent.getChildren().add(nav);
            }
        }
    }
    return roots;
}
```

Map 룩업이 O(1)이므로 전체 O(n)으로 내려갑니다. 같은 패턴을 Dept 트리에도 동일하게 적용할 수 있습니다.

---

## depth 갱신: 일회성 마이그레이션과 운영 로직

### 문제: depth 컬럼이 비어있다

parentId를 추가한 뒤, **기존 데이터의 depth를 일괄 계산**해야 했습니다. 부모-자식 관계는 있지만, depth 값이 null인 상태에서 시작해야 합니다.

### 일회성 depth 마이그레이션: updateDepth()

Navigation에 적용한 depth 갱신 로직입니다.

```java
/**
 * 일회성 API: depth column 추가 후 depth 업데이트
 * childList 내부 로직으로 변경함에 따라 depth column 추가
 */
@Transactional
public void updateDepth() {
    List<Navigation> navigations = navigationRepository.getAllNavigationList();

    while (!navigations.isEmpty()) {
        for (Navigation navigation : navigations) {
            if (navigation.getParentNavigationSeq() == null) {
                // 루트 노드: depth = 1
                navigation.updateDepth(1);
                navigations.remove(navigation);
                break;
            } else {
                Navigation parent = navigationRepository
                    .findByNavigationSeq(navigation.getParentNavigationSeq());
                if (parent.getDepth() != null) {
                    // 부모의 depth가 이미 설정되어 있으면 → depth = parent + 1
                    navigation.updateDepth(parent.getDepth() + 1);
                    navigations.remove(navigation);
                    break;
                }
                // 부모의 depth가 아직 null → 다음 반복에서 처리
            }
        }
    }
}
```

**이 알고리즘의 동작 원리를 단계별로 보면:**

```
초기 상태: [A(p=null,d=null), B(p=A,d=null), C(p=B,d=null), D(p=A,d=null)]

1번째 while 루프:
  A → parent==null → depth=1, 리스트에서 제거 → break
  남은: [B(p=A,d=null), C(p=B,d=null), D(p=A,d=null)]

2번째 while 루프:
  B → parent=A, A.depth=1(!=null) → depth=2, 제거 → break
  남은: [C(p=B,d=null), D(p=A,d=null)]

3번째 while 루프:
  C → parent=B, B.depth=2(!=null) → depth=3, 제거 → break
  남은: [D(p=A,d=null)]

4번째 while 루프:
  D → parent=A, A.depth=1(!=null) → depth=2, 제거 → break
  남은: []

결과: A(1), B(2), D(2), C(3)
```

이 방식의 핵심은 **부모의 depth가 결정된 것만 처리하고, 아직 결정 안 된 것은 다음 루프로 미룬다**는 것입니다. 마치 위상 정렬(Topological Sort)처럼 의존 관계가 해결된 것부터 순서대로 처리합니다.

**주의점:**

```java
// for문 내에서 리스트를 remove하고 break하는 패턴
for (Navigation navigation : navigations) {
    // ...처리...
    navigations.remove(navigation);
    break;  // ← ConcurrentModificationException 방지를 위한 break
}
```

일반적으로 for-each 루프에서 리스트를 수정하면 `ConcurrentModificationException`이 발생합니다. 여기서는 **remove 직후 break로 루프를 빠져나가기 때문에** 예외가 발생하지 않습니다. 하지만 이 패턴은 의도를 파악하기 어렵고, 실수로 break를 빼먹으면 런타임 에러가 발생하므로 주의가 필요합니다.

Iterator를 사용하면 더 안전하게 구현할 수 있습니다.

```java
// 더 안전한 버전: Iterator 사용
while (!navigations.isEmpty()) {
    Iterator<Navigation> it = navigations.iterator();
    while (it.hasNext()) {
        Navigation navigation = it.next();
        if (navigation.getParentNavigationSeq() == null) {
            navigation.updateDepth(1);
            it.remove();  // Iterator.remove()는 안전
        } else {
            Navigation parent = navigationRepository
                .findByNavigationSeq(navigation.getParentNavigationSeq());
            if (parent.getDepth() != null) {
                navigation.updateDepth(parent.getDepth() + 1);
                it.remove();
            }
        }
    }
}
```

Iterator 버전은 한 번의 내부 루프에서 **처리 가능한 모든 노드를 한꺼번에 제거**하므로, while 루프의 반복 횟수가 depth 단계 수로 줄어듭니다 (기존은 노드 수만큼 반복).

### depth 갱신 알고리즘의 시간 복잡도 분석

이 알고리즘을 좀 더 이론적으로 분석해보겠습니다.

**원본 코드의 시간 복잡도 (remove + break 패턴)**

원본 코드는 while 루프 한 번에 노드를 **딱 하나씩** 제거합니다. 노드가 총 N개이면 while 루프가 정확히 N번 실행됩니다. 각 while 루프 안에서는 for-each로 리스트를 순회하다가 처리 가능한 노드를 만나면 break합니다.

- **최선의 경우**: 리스트가 이미 위상 정렬 순서대로 정렬되어 있으면, 매번 첫 번째 원소를 처리합니다. 하지만 `findByNavigationSeq` DB 조회가 매번 발생하므로, 총 **O(N)번의 DB 조회**가 필요합니다.
- **최악의 경우**: 리스트가 역순(잎 노드가 앞, 루트가 맨 뒤)이면, 첫 번째 while 루프에서 N-1개를 건너뛴 뒤 루트를 처리합니다. 두 번째 루프에서는 N-2개를 건너뛰고... 이렇게 되면 **O(N^2)번의 비교 + O(N)번의 DB 조회**가 발생합니다.

```
최악의 경우 시나리오:
리스트: [D(p=C), C(p=B), B(p=A), A(p=null)]

1번째 while: D 확인(부모 depth null) → C 확인(부모 depth null) → B 확인(부모 depth null) → A 처리, break
2번째 while: D 확인(부모 depth null) → C 확인(부모 depth null) → B 처리, break
3번째 while: D 확인(부모 depth null) → C 처리, break
4번째 while: D 처리, break

비교 횟수: 4 + 3 + 2 + 1 = 10 = N(N+1)/2 → O(N^2)
```

**Iterator 버전의 시간 복잡도**

Iterator 버전에서는 한 번의 while 루프에서 **그 시점에 처리 가능한 모든 노드를 한꺼번에 제거**합니다. 트리의 depth를 D라 하면:

- 1번째 while 루프: 루트 노드들(depth=1) 전부 처리
- 2번째 while 루프: depth=2인 노드들 전부 처리
- ...
- D번째 while 루프: 잎 노드들(depth=D) 전부 처리

while 루프가 **D번** 실행되고, 각 루프에서 남은 노드를 전부 순회하므로 총 비교 횟수는 **O(N x D)** 입니다.

| 버전 | while 루프 수 | 총 비교 횟수 | DB 조회 수 |
|---|---|---|---|
| 원본 (remove + break) | N | O(N^2) 최악 | N |
| Iterator 버전 | D (트리 depth) | O(N x D) | N |

부서 트리에서 D는 보통 3~5 수준이므로, Iterator 버전은 사실상 **O(N)** 에 가깝습니다.

### 위상 정렬(Topological Sort)과의 관계

이 depth 갱신 알고리즘은 본질적으로 **위상 정렬(Topological Sort)**의 한 형태입니다. 위상 정렬은 DAG(Directed Acyclic Graph)에서 간선의 방향을 거스르지 않도록 노드를 정렬하는 알고리즘입니다.

부서 트리에서의 의존 관계를 그래프로 표현하면 이렇습니다.

```
의존 관계 그래프:
A(루트) ← B ← C
A(루트) ← D

간선의 의미: "X ← Y" = Y의 depth를 결정하려면 X의 depth가 먼저 결정되어야 함

위상 정렬 결과: [A, B, D, C] 또는 [A, D, B, C]
                (A가 가장 먼저, C가 가장 나중)
```

우리 알고리즘은 **부모의 depth가 결정되었는지**를 기준으로 처리 가능 여부를 판단합니다. 이것은 위상 정렬에서 **진입 차수(in-degree)가 0인 노드를 먼저 처리**하는 것과 동일한 원리입니다.

### Kahn의 알고리즘과의 비교

위상 정렬의 대표적인 구현인 **Kahn의 알고리즘**은 진입 차수 기반으로 동작합니다. 우리 depth 갱신 알고리즘과 비교해보겠습니다.

```java
// Kahn의 알고리즘 스타일로 depth를 갱신하는 버전
public void updateDepthKahn(List<Navigation> navigations) {
    // 1. 각 노드의 부모 관계를 Map으로 구성
    Map<Integer, Navigation> nodeMap = new HashMap<>();
    Map<Integer, List<Navigation>> childrenMap = new HashMap<>();
    Queue<Navigation> queue = new LinkedList<>();

    for (Navigation nav : navigations) {
        nodeMap.put(nav.getNavigationSeq(), nav);
        childrenMap.putIfAbsent(nav.getNavigationSeq(), new ArrayList<>());
        if (nav.getParentNavigationSeq() != null) {
            childrenMap
                .computeIfAbsent(nav.getParentNavigationSeq(), k -> new ArrayList<>())
                .add(nav);
        }
    }

    // 2. 루트 노드(부모가 없는 노드)를 큐에 넣기
    for (Navigation nav : navigations) {
        if (nav.getParentNavigationSeq() == null) {
            nav.updateDepth(1);
            queue.offer(nav);
        }
    }

    // 3. BFS로 자식 노드의 depth를 결정
    while (!queue.isEmpty()) {
        Navigation current = queue.poll();
        List<Navigation> children = childrenMap.getOrDefault(
            current.getNavigationSeq(), Collections.emptyList());
        for (Navigation child : children) {
            child.updateDepth(current.getDepth() + 1);
            queue.offer(child);
        }
    }
}
```

| 비교 항목 | 우리 구현 (Iterator 버전) | Kahn 스타일 |
|---|---|---|
| 자료구조 | List + Iterator | Map + Queue (BFS) |
| DB 조회 | 매 노드마다 `findByNavigationSeq` | 초기에 전체 로드 후 메모리에서 처리 |
| 시간 복잡도 | O(N x D) | **O(N)** |
| 공간 복잡도 | O(N) | O(N) — Map, Queue 추가 |
| 구현 난이도 | 단순 | Map/Queue 구성 필요 |

Kahn 스타일은 **DB 조회를 완전히 제거**할 수 있습니다. 처음에 전체 노드를 메모리에 올린 뒤, 부모-자식 관계를 Map으로 구성하고, BFS로 순회하면서 depth를 결정합니다. 노드 수가 수백 건이라 우리 구현도 충분했지만, 만약 수만 건의 노드를 처리해야 한다면 Kahn 스타일이 훨씬 효율적입니다.

### 순환 참조(Cycle) 발생 시의 위험

위상 정렬은 **DAG(비순환 방향 그래프)**에서만 유효합니다. 만약 데이터에 순환 참조가 존재하면 어떻게 될까요?

```
순환 참조 시나리오:
A(parent=C) → B(parent=A) → C(parent=B)

A의 depth를 결정하려면 C의 depth가 필요
C의 depth를 결정하려면 B의 depth가 필요
B의 depth를 결정하려면 A의 depth가 필요
→ 무한 루프!
```

현재 우리 구현에서 순환 참조가 발생하면:
- **원본 코드**: while 루프가 영원히 종료되지 않습니다. 어떤 노드도 처리할 수 없는 상태로 무한 반복합니다.
- **Iterator 버전**: 마찬가지로 while 루프 안에서 어떤 노드도 제거되지 않아 무한 루프에 빠집니다.

**방지 방법 1: DB 레벨 제약**

부서 테이블에 **자기 자신을 부모로 가리키는 것을 방지하는 CHECK 제약**을 추가할 수 있습니다.

```sql
ALTER TABLE dept ADD CONSTRAINT chk_no_self_parent
    CHECK (dept_id != parent_dept_id);
```

하지만 이것은 직접적인 자기 참조만 막을 뿐, A→B→C→A 같은 간접 순환은 막을 수 없습니다.

**방지 방법 2: 애플리케이션 레벨 검증**

부모 변경(reparent) 시, 새로운 부모가 자신의 후손인지 확인하는 로직을 추가합니다.

```java
/**
 * 순환 참조 검증: newParentId가 deptId의 후손인지 확인
 */
private void validateNoCycle(String deptId, String newParentId) {
    String current = newParentId;
    Set<String> visited = new HashSet<>();

    while (current != null) {
        if (current.equals(deptId)) {
            throw new IllegalArgumentException(
                "순환 참조가 발생합니다: " + deptId + "의 후손을 부모로 지정할 수 없습니다.");
        }
        if (!visited.add(current)) {
            throw new IllegalStateException(
                "이미 순환 참조가 존재합니다: " + current);
        }
        Dept parent = deptRepository.findByDeptId(current);
        current = parent != null ? parent.getParentDeptId() : null;
    }
}
```

**방지 방법 3: 마이그레이션 시 안전장치**

depth 갱신 마이그레이션에 **최대 반복 횟수 제한**을 추가합니다.

```java
int maxIterations = navigations.size();
int iteration = 0;

while (!navigations.isEmpty()) {
    if (iteration++ > maxIterations) {
        throw new IllegalStateException(
            "순환 참조 감지: 처리되지 않은 노드 " + navigations.size() + "개 남음. "
            + "남은 노드: " + navigations.stream()
                .map(n -> n.getNavigationSeq().toString())
                .collect(Collectors.joining(", ")));
    }
    // ... 기존 처리 로직 ...
}
```

정상적인 트리에서는 while 루프가 최대 N번(원본) 또는 D번(Iterator) 실행되므로, N번을 초과하면 순환 참조로 판단할 수 있습니다. 이 방식이 가장 간단하면서도 실용적인 안전장치입니다.

---

## 복합 PK 설계: DeptCrewId

### 왜 복합 PK가 필요했는가

부서와 사원의 매핑 테이블 `DeptCrew`에는 **한 사원이 한 부서에 한 번만 소속된다**는 비즈니스 규칙이 있었습니다. 이를 DB 레벨에서 보장하기 위해 **복합 기본키**를 도입했습니다.

```java
/**
 * Multi Key
 */
public class DeptCrewId implements Serializable {
    // 부서 식별번호
    private String deptId;
    // 사원 식별번호
    private int crewSeq;
}
```

이 복합 PK가 하는 일:

```sql
-- 논리적 의미: 하나의 (부서, 사원) 조합은 유일해야 한다
PRIMARY KEY (dept_id, crew_seq)
```

| 시나리오 | 단일 PK (AUTO_INCREMENT) | 복합 PK (deptId + crewSeq) |
|---|---|---|
| 같은 사원을 같은 부서에 중복 등록 | 가능 (ID만 다르면 됨) | **불가능** (PK 중복) |
| 이 사원이 이 부서에 있는가? 조회 | WHERE dept_id=? AND crew_seq=? (인덱스 필요) | **PK 자체가 인덱스** |
| 부서별 사원 조회 | 별도 인덱스 필요 | PK의 **왼쪽 접두사(deptId)로 자동 지원** |

### JPA @IdClass 복합 PK의 내부 동작

JPA에서 복합 PK를 사용할 때는 **단순히 두 필드를 PK로 선언하는 것 이상**의 메커니즘이 작동합니다. 내부 동작을 이해해야 의도하지 않은 버그를 방지할 수 있습니다.

**equals/hashCode 계약**

JPA 스펙(JSR 338)은 복합 PK 클래스에 `equals()`와 `hashCode()`를 **반드시 재정의**하도록 요구합니다. 그 이유는 Hibernate의 1차 캐시(Persistence Context)가 엔티티를 **PK 객체를 키로 하는 Map**으로 관리하기 때문입니다.

```java
// Hibernate의 1차 캐시 내부 구조 (개념적 표현)
Map<EntityKey, Object> entityCache;

// EntityKey는 (엔티티 타입 + PK 값)으로 구성
// PK 값의 동등성 비교에 equals/hashCode가 사용됨
```

만약 `DeptCrewId`에 `equals()`/`hashCode()`를 재정의하지 않으면, Object의 기본 구현(메모리 주소 비교)이 사용됩니다. 그러면 **같은 (deptId, crewSeq) 값을 가진 두 PK 객체가 서로 다른 것으로 판단**되어, 1차 캐시에서 엔티티를 찾지 못하는 문제가 발생합니다.

```java
public class DeptCrewId implements Serializable {
    private String deptId;
    private int crewSeq;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        DeptCrewId that = (DeptCrewId) o;
        return crewSeq == that.crewSeq
            && Objects.equals(deptId, that.deptId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(deptId, crewSeq);
    }
}
```

**@IdClass vs @EmbeddedId 비교**

JPA에서 복합 PK를 구현하는 방법은 두 가지입니다.

```java
// 방식 1: @IdClass — 우리가 선택한 방식
@Entity
@IdClass(DeptCrewId.class)
public class DeptCrew {
    @Id
    private String deptId;
    @Id
    private int crewSeq;
    // ... 다른 필드들
}

// 방식 2: @EmbeddedId
@Entity
public class DeptCrew {
    @EmbeddedId
    private DeptCrewId id;
    // ... 다른 필드들
}
```

| 비교 항목 | @IdClass | @EmbeddedId |
|---|---|---|
| 필드 접근 | `deptCrew.getDeptId()` 직접 접근 | `deptCrew.getId().getDeptId()` 중첩 접근 |
| JPQL 쿼리 | `SELECT d.deptId FROM DeptCrew d` | `SELECT d.id.deptId FROM DeptCrew d` |
| PK 클래스 어노테이션 | 없음 | `@Embeddable` 필요 |
| Spring Data JPA 호환 | 좋음 | 좋음 |
| 가독성 | PK 필드가 엔티티에 직접 노출 | PK가 하나의 객체로 캡슐화 |

`@IdClass`를 선택한 이유는 **기존 코드에서 `deptId`, `crewSeq`를 엔티티 필드로 직접 참조하는 곳이 많았기 때문**입니다. `@EmbeddedId`로 변경하면 `.getId().getDeptId()` 형태로 모든 참조를 수정해야 하므로, 마이그레이션 비용이 더 컸습니다.

**Hibernate 1차 캐시에서의 복합 PK 동작**

Hibernate의 `EntityManager.find()`를 호출하면, 내부적으로 다음과 같은 과정이 진행됩니다.

```
entityManager.find(DeptCrew.class, new DeptCrewId("SYF", 142))

1. DeptCrewId("SYF", 142) 객체 생성
2. 1차 캐시(PersistenceContext)에서 EntityKey로 조회
   - EntityKey = (DeptCrew.class, DeptCrewId("SYF", 142))
   - Map.get() 호출 → DeptCrewId.hashCode()로 버킷 찾기
   - 버킷 내에서 DeptCrewId.equals()로 정확한 엔티티 찾기
3. 캐시에 있으면 → 바로 반환 (DB 조회 없음)
4. 캐시에 없으면 → SELECT 쿼리 실행 → 캐시에 저장 → 반환
```

이 과정에서 `equals()`/`hashCode()`가 올바르게 구현되어 있지 않으면, 동일한 PK임에도 캐시 미스가 발생하여 **불필요한 DB 조회가 반복**됩니다. 단순히 중복 방지만의 문제가 아니라, **성능에도 직접적인 영향**을 미치는 것입니다.

**Serializable 인터페이스가 필요한 이유**

`DeptCrewId`가 `Serializable`을 구현하는 것은 JPA 스펙의 요구사항입니다. 그 이유는 크게 두 가지입니다.

1. **2차 캐시(L2 Cache)**: Hibernate의 2차 캐시는 엔티티를 세션 간에 공유합니다. 이때 PK 객체가 캐시 키로 직렬화/역직렬화될 수 있으므로 `Serializable`이 필요합니다.

2. **세션 복제(Session Replication)**: 클러스터 환경에서 HTTP 세션에 엔티티를 저장하면, 세션 복제 시 PK 객체도 함께 직렬화됩니다. `Serializable`이 없으면 `NotSerializableException`이 발생합니다.

3. **JPA 스펙 준수**: JPA 2.2 스펙 2.4절은 PK 클래스가 반드시 `Serializable`이어야 한다고 명시하고 있습니다. Hibernate는 이를 강제하지 않는 경우도 있지만, 스펙을 따르는 것이 이식성과 안정성 측면에서 바람직합니다.

```java
// serialVersionUID를 명시적으로 선언하면 직렬화 호환성을 제어할 수 있음
public class DeptCrewId implements Serializable {
    private static final long serialVersionUID = 1L;

    private String deptId;
    private int crewSeq;
    // ...
}
```

`serialVersionUID`를 명시하지 않으면 JVM이 클래스 구조를 기반으로 자동 생성하는데, 필드 추가/제거 시 값이 바뀌어 역직렬화가 실패할 수 있습니다. 운영 환경에서는 **명시적으로 선언**하는 것이 안전합니다.

### 복합 PK의 인덱스 효과

`(deptId, crewSeq)` 복합 PK를 설정하면, DB가 자동으로 이 두 컬럼의 **복합 인덱스**를 생성합니다.

```sql
-- 이 쿼리들이 모두 PK 인덱스를 활용
SELECT * FROM dept_crew WHERE dept_id = 'SYF';                    -- 왼쪽 접두사 매칭
SELECT * FROM dept_crew WHERE dept_id = 'SYF' AND crew_seq = 142; -- 정확 매칭
SELECT * FROM dept_crew WHERE dept_id IN ('SYF', 'WEB');          -- IN 절
```

실제 코드에서 이 인덱스가 활용되는 곳:

```java
// 메달 지급 대상자 조회: 여러 부서의 사원을 한 번에 조회
List<DeptCrew> deptCrewList = deptCrewRepository
    .findDeptCrewsByDeptIdIsInAndCrewSeqIsNotAndIsLeaderFalse(
        deptIdList,     // 하위 부서 ID 전체 목록
        loginCrewSeq    // 조회자 본인 제외
    );
```

이 쿼리는 `deptId IN (...)` 조건을 사용하므로, **복합 PK의 왼쪽 컬럼(deptId) 인덱스를 타서 효율적으로 조회**됩니다.

### DeptCrewId에서 deptId가 String인 이유

`deptId`가 `Long`이 아닌 `String`인 것은 기존 시스템의 설계 때문입니다. 부서 ID가 **SYF**, **WEB** 같은 **의미 있는 문자열 코드**로 관리되고 있었고, 이 규칙을 변경하는 것은 기존 데이터와의 호환성 문제가 있어 유지했습니다.

문자열 PK의 단점(인덱스 크기 증가, 비교 연산 비용)이 있지만, 부서 수가 수백 건 수준이므로 실질적인 성능 영향은 미미합니다. 오히려 **사람이 읽기 쉬운 ID**라는 장점이 운영에서 더 유용했습니다.

---

## 하위부서 재귀 탐색: getSubAllDeptIdList

### 개선된 전체 하위부서 탐색

메달 지급 대상자를 찾을 때, **이 부서 + 모든 하위 부서**의 사원이 필요합니다. 이 탐색은 **반복적 BFS(Breadth-First Search)** 패턴으로 구현되어 있습니다.

```java
/**
 * 해당 그룹의 전체 하위부서 리스트 가져오기
 * BFS처럼 한 레벨씩 내려가며 하위부서를 수집
 */
private List<String> getSubAllDeptIdList(List<String> deptIdList) {
    // 시작 부서 ID들을 결과에 포함
    List<String> allDeptIdList = new ArrayList<>(deptIdList);
    // 현재 레벨의 하위부서 조회
    List<String> subDeptIdList = new ArrayList<>(this.getSubDeptIdList(deptIdList));

    // 더 이상 하위부서가 없을 때까지 반복
    while (subDeptIdList.size() != 0) {
        allDeptIdList.addAll(subDeptIdList);
        subDeptIdList = new ArrayList<>(this.getSubDeptIdList(subDeptIdList));
    }

    return allDeptIdList;
}
```

**BFS 방식의 동작을 그림으로 보면:**

```
시작: [코코네M]

1단계 getSubDeptIdList([코코네M]):
  → [SYF, 경영지원]
  allDeptIdList = [코코네M, SYF, 경영지원]

2단계 getSubDeptIdList([SYF, 경영지원]):
  → [Web팀, App팀, 인사팀]
  allDeptIdList = [코코네M, SYF, 경영지원, Web팀, App팀, 인사팀]

3단계 getSubDeptIdList([Web팀, App팀, 인사팀]):
  → [] (더 이상 하위부서 없음)
  while 종료

최종: [코코네M, SYF, 경영지원, Web팀, App팀, 인사팀]
```

이 전체 부서 ID 목록으로 `DeptCrew` 테이블에서 사원을 조회합니다.

```java
// 수집된 모든 부서 ID로 사원 한 번에 조회
List<DeptCrew> deptCrewList = deptCrewRepository
    .findDeptCrewsByDeptIdIsInAndCrewSeqIsNotAndIsLeaderFalse(
        deptIdList,      // [코코네M, SYF, 경영지원, Web팀, App팀, 인사팀]
        loginCrewSeq     // 본인 제외
    );

// 사원 시퀀스만 추출
List<Integer> crewSeqList = deptCrewList.stream()
    .map(DeptCrew::getCrewSeq)
    .collect(Collectors.toList());
```

### 대안: CTE(Common Table Expression) 재귀 쿼리

현재 BFS 방식은 depth 단계 수만큼 DB 쿼리가 발생합니다. MySQL 8.0 이상에서는 **WITH RECURSIVE** 문법을 사용하여, 하위부서 전체를 **단일 쿼리**로 가져올 수 있습니다.

```sql
-- MySQL 8.0+ WITH RECURSIVE를 사용한 하위부서 전체 조회
WITH RECURSIVE sub_depts AS (
    -- 앵커 멤버: 시작 부서
    SELECT dept_id, parent_dept_id, dept_name, depth
    FROM dept
    WHERE dept_id = 'COCONEM'

    UNION ALL

    -- 재귀 멤버: 현재 레벨의 자식들
    SELECT d.dept_id, d.parent_dept_id, d.dept_name, d.depth
    FROM dept d
    INNER JOIN sub_depts sd ON d.parent_dept_id = sd.dept_id
)
SELECT dept_id FROM sub_depts;
```

이 쿼리의 실행 과정을 풀어보면 이렇습니다.

```
1단계 (앵커): dept_id = 'COCONEM' → {코코네M}
2단계 (재귀): parent_dept_id = 'COCONEM' → {SYF, 경영지원}
3단계 (재귀): parent_dept_id IN ('SYF', '경영지원') → {Web팀, App팀, 인사팀}
4단계 (재귀): parent_dept_id IN ('Web팀', 'App팀', '인사팀') → {} (비어있음, 종료)

결과: {코코네M, SYF, 경영지원, Web팀, App팀, 인사팀}
```

**JPA/Spring Data에서 CTE를 사용하려면** 네이티브 쿼리를 작성해야 합니다.

```java
@Query(value = """
    WITH RECURSIVE sub_depts AS (
        SELECT dept_id, parent_dept_id
        FROM dept
        WHERE dept_id IN :deptIds

        UNION ALL

        SELECT d.dept_id, d.parent_dept_id
        FROM dept d
        INNER JOIN sub_depts sd ON d.parent_dept_id = sd.dept_id
    )
    SELECT dept_id FROM sub_depts
    """, nativeQuery = true)
List<String> findAllSubDeptIds(@Param("deptIds") List<String> deptIds);
```

### 애플리케이션 레벨 BFS vs DB 레벨 CTE의 트레이드오프

두 방식의 차이를 구체적으로 비교해보겠습니다.

| 비교 항목 | 애플리케이션 BFS | DB CTE 재귀 |
|---|---|---|
| 쿼리 수 | depth 단계 수 (D번) | **1번** |
| 네트워크 라운드트립 | D번 | **1번** |
| DB 부하 | 각 쿼리가 가벼움 | 단일 쿼리지만 DB 내부에서 재귀 |
| 코드 가독성 | Java 코드로 명확 | 네이티브 SQL, JPA 추상화 깨짐 |
| DB 종속성 | 없음 | MySQL 8.0+ 필요 |
| 테스트 용이성 | 단위 테스트 쉬움 | 통합 테스트 필요 |
| 캐싱 가능성 | 레벨별 결과 캐싱 가능 | 전체 결과만 캐싱 가능 |

**성능 비교 (부서 200개, depth 5단계 기준)**

```
[애플리케이션 BFS]
  쿼리 5회 x 각 ~1ms = ~5ms + 네트워크 오버헤드 ~5ms = ~10ms

[DB CTE 재귀]
  쿼리 1회 x ~3ms = ~3ms + 네트워크 오버헤드 ~1ms = ~4ms
```

CTE가 약 2.5배 빠르지만, 절대적인 차이는 6ms 수준입니다. 부서 수가 수백 건이고 depth가 5단계인 현재 상황에서는 **체감할 수 있는 차이가 아닙니다**.

**CTE를 선택해야 하는 경우**는 다음과 같습니다.

- 부서(노드)가 **수만 건 이상**이고 depth가 10단계 이상
- 하위부서 탐색이 **초당 수백 회 이상** 호출
- 네트워크 지연이 큰 환경(DB가 다른 리전에 있는 경우)

현재 시스템에서는 애플리케이션 BFS가 충분하므로 유지했지만, 향후 부서 수가 급증하거나 조직도 API의 호출 빈도가 높아지면 CTE 전환을 고려할 수 있습니다.

---

## 메달 지급 로직에서의 부서 트리 활용

부서 트리 구조가 가장 복잡하게 활용되는 곳이 **메달 지급 기능**입니다. 전체 흐름을 코드와 함께 분석합니다.

### 메달 지급 대상자 조회 전체 흐름

```java
@Transactional
public Page<CrewDto> getMedalCrewList(CrewParamDto crewParamDto, Pageable pageable) {
    // Step 1. 로그인 사용자의 메달 지급 권한이 있는 부서 ID 추출
    List<String> medalDeptIdList = this.getLoginCrewsMedalDeptIdList();

    // Step 2. 해당 부서의 모든 하위부서까지 포함
    List<String> deptIdList = this.getSubAllDeptIdList(medalDeptIdList);

    // Step 3. DeptCrew 테이블에서 대상 사원 조회 (본인 제외, 리더 제외)
    int loginCrewSeq = LoginManager.getUserDetails().getCrewSeq();
    List<DeptCrew> deptCrewList = deptCrewRepository
        .findDeptCrewsByDeptIdIsInAndCrewSeqIsNotAndIsLeaderFalse(
            deptIdList, loginCrewSeq);

    // Step 4. crewSeq로 상세 정보 조회
    List<Integer> crewSeqList = deptCrewList.stream()
        .map(DeptCrew::getCrewSeq).collect(Collectors.toList());
    crewParamDto.setCrewSeqList(crewSeqList);

    // Step 5. 코드 값을 한글로 변환 (캐싱된 참조 데이터 사용)
    HashMap<String, Map<String, String>> cacheCodeMap = getCacheCodeMap();
    List<CrewDto> medalCrewList = crewQueryRepository
        .findMedalCrewList(crewParamDto, pageable);
    medalCrewList.forEach(crew -> changeCodeToKr(crew, cacheCodeMap));

    return new PageImpl<>(medalCrewList, pageable, totalCount);
}
```

### QueryDSL로 트리 조회 최적화

메달 지급 대상자 목록은 단순 조회가 아닙니다. **부서 필터, 검색어, 페이징**이 조합되는 동적 조건이 필요합니다. 이 경우 Spring Data JPA의 메서드 이름 기반 쿼리로는 한계가 있고, **QueryDSL**이 빛을 발합니다.

**동적 조건이 필요한 이유**

메달 지급 대상자 조회 화면에서는 다음 조건들이 **선택적으로** 조합됩니다.

```
검색 조건 조합 예시:
1. 부서만 필터:        WHERE dept_id IN (...)
2. 부서 + 이름 검색:   WHERE dept_id IN (...) AND crew_name LIKE '%홍%'
3. 부서 + 직급 필터:   WHERE dept_id IN (...) AND duty_id = 'MANAGER'
4. 전체 + 페이징:      WHERE dept_id IN (...) LIMIT 20 OFFSET 40
5. 전체 조건 조합:     WHERE dept_id IN (...) AND crew_name LIKE '%홍%'
                             AND duty_id = 'MANAGER' LIMIT 20 OFFSET 40
```

이것을 Spring Data JPA 메서드 이름으로 표현하면, 조합마다 별도 메서드가 필요합니다. 조건이 5개면 이론상 2^5 = 32개의 메서드가 필요해집니다.

**BooleanExpression 패턴**

QueryDSL에서는 `BooleanExpression`을 조건별로 분리하고, null이면 무시하는 패턴으로 깔끔하게 해결합니다.

```java
public class CrewQueryRepository {

    private final JPAQueryFactory queryFactory;

    public List<CrewDto> findMedalCrewList(CrewParamDto param, Pageable pageable) {
        return queryFactory
            .select(Projections.constructor(CrewDto.class,
                crew.crewSeq,
                crew.crewName,
                crew.deptId,
                crew.dutyId,
                crew.branchKey
            ))
            .from(crew)
            .where(
                crewSeqIn(param.getCrewSeqList()),
                crewNameContains(param.getSearchKeyword()),
                dutyIdEq(param.getDutyId()),
                branchKeyEq(param.getBranchKey())
            )
            .offset(pageable.getOffset())
            .limit(pageable.getPageSize())
            .orderBy(crew.crewName.asc())
            .fetch();
    }

    // 각 조건을 BooleanExpression으로 분리
    private BooleanExpression crewSeqIn(List<Integer> crewSeqList) {
        return crewSeqList != null && !crewSeqList.isEmpty()
            ? crew.crewSeq.in(crewSeqList)
            : null;  // null을 반환하면 where절에서 무시됨
    }

    private BooleanExpression crewNameContains(String keyword) {
        return StringUtils.hasText(keyword)
            ? crew.crewName.contains(keyword)
            : null;
    }

    private BooleanExpression dutyIdEq(String dutyId) {
        return StringUtils.hasText(dutyId)
            ? crew.dutyId.eq(dutyId)
            : null;
    }

    private BooleanExpression branchKeyEq(String branchKey) {
        return StringUtils.hasText(branchKey)
            ? crew.branchKey.eq(branchKey)
            : null;
    }
}
```

QueryDSL의 `where()` 메서드는 **null인 조건을 자동으로 무시**합니다. 따라서 `crewNameContains(null)`이 null을 반환하면, 해당 조건은 SQL에 포함되지 않습니다. 이 패턴 덕분에 조건 조합마다 별도 메서드를 만들 필요가 없습니다.

**카운트 쿼리 분리의 중요성**

페이징 처리에서는 **데이터 조회 쿼리**와 **총 건수(count) 쿼리**가 별도로 필요합니다. Spring Data JPA의 `Page`를 사용하면 카운트 쿼리가 자동 생성되지만, 이 자동 생성 쿼리가 비효율적인 경우가 있습니다.

```java
// 카운트 쿼리를 별도로 분리
public long countMedalCrewList(CrewParamDto param) {
    return queryFactory
        .select(crew.count())
        .from(crew)
        .where(
            crewSeqIn(param.getCrewSeqList()),
            crewNameContains(param.getSearchKeyword()),
            dutyIdEq(param.getDutyId()),
            branchKeyEq(param.getBranchKey())
        )
        .fetchOne();
}
```

카운트 쿼리를 분리하는 것이 중요한 이유는 다음과 같습니다.

1. **불필요한 JOIN 제거**: 데이터 조회 시에는 여러 테이블을 JOIN해서 상세 정보를 가져오지만, 카운트 쿼리에서는 건수만 필요하므로 JOIN을 줄일 수 있습니다.

2. **불필요한 ORDER BY 제거**: 카운트 쿼리에서 정렬은 의미 없습니다. Spring Data JPA의 자동 생성 쿼리는 ORDER BY를 포함할 수 있는데, 이를 제거하면 성능이 향상됩니다.

3. **캐싱된 참조 데이터와의 분리**: 앞서 설명한 `getCacheCodeMap()` 패턴처럼, JOIN 대신 애플리케이션에서 매핑하면 카운트 쿼리가 단순해집니다. 7개 테이블을 JOIN하는 카운트 쿼리는 사원 테이블만 카운트하는 것보다 훨씬 느립니다.

```
[자동 생성 카운트 쿼리]
SELECT COUNT(*)
FROM crew c
JOIN dept d ON c.dept_id = d.dept_id           -- 불필요
JOIN duty dt ON c.duty_id = dt.duty_id         -- 불필요
JOIN branch b ON c.branch_key = b.branch_key   -- 불필요
WHERE c.crew_seq IN (...)

[분리된 카운트 쿼리]
SELECT COUNT(*)
FROM crew c
WHERE c.crew_seq IN (...)
```

JOIN 3개를 제거하는 것만으로도, 사원 수가 많을 때 **카운트 쿼리 성능이 수 배 향상**될 수 있습니다.

### 참조 데이터 캐싱: Join 회피 패턴

Step 5에서 사용되는 `getCacheCodeMap()`은 **JOIN 쿼리의 성능 문제를 우회**하기 위한 캐싱 패턴입니다.

```java
/**
 * 쿼리 성능 이슈로 인해 join Table 들 캐싱 후 crew data key 값으로 매칭 처리
 */
public HashMap<String, Map<String, String>> getCacheCodeMap() {
    Map<String, String> activeStatusMap = hrExecutor.findAllActiveStatuses().stream()
        .collect(Collectors.toMap(ActiveStatus::getActiveStatusKey, ActiveStatus::getKr));
    Map<String, String> branchMap = hrExecutor.findAllBranches().stream()
        .collect(Collectors.toMap(Branch::getBranchKey, Branch::getKr));
    Map<String, String> dutyMap = hrExecutor.findAllDuties().stream()
        .collect(Collectors.toMap(Duty::getDutyId, Duty::getDutyName));
    Map<String, String> jobKindMap = hrExecutor.findAllJobKinds().stream()
        .collect(Collectors.toMap(JobKind::getJobKindKey, JobKind::getKr));
    // ... (nationalityMap, workTypeMap 등)

    HashMap<String, Map<String, String>> resultMap = new HashMap<>();
    resultMap.put("activeStatusMap", activeStatusMap);
    resultMap.put("branchMap", branchMap);
    resultMap.put("dutyMap", dutyMap);
    resultMap.put("jobKindMap", jobKindMap);
    // ...
    return resultMap;
}
```

이 패턴이 필요한 이유:

```
[JOIN 방식] 사원 테이블에 7개 참조 테이블을 JOIN
  → 페이징 COUNT 쿼리에서도 7개 JOIN 발생
  → 느림 (특히 사원 수가 많을 때)

[캐싱 방식] 7개 참조 테이블을 각각 Map으로 캐싱 → 애플리케이션에서 매핑
  → COUNT 쿼리는 사원 테이블만 조회
  → 빠름
```

```java
// 캐싱된 Map으로 코드 → 한글 변환
public void changeCodeToKr(CrewDto crewDto,
                           HashMap<String, Map<String, String>> cacheCodeMap) {
    crewDto.setJobPart(cacheCodeMap.get("jobPartMap")
        .getOrDefault(toLowerCase(crewDto.getJobPart()), crewDto.getJobPart()));
    crewDto.setJobKind(cacheCodeMap.get("jobKindMap")
        .getOrDefault(toLowerCase(crewDto.getJobKind()), crewDto.getJobKind()));
    crewDto.setDutyId(cacheCodeMap.get("dutyMap")
        .getOrDefault(crewDto.getDutyId(), crewDto.getDutyId()));
    // ...
}
```

`getOrDefault`를 사용해, 매핑이 없는 경우 **원래 코드 값을 그대로 유지**합니다. 그리고 `toLowerCase()`로 대소문자를 통일하는데, 이때 null 체크가 중요합니다.

```java
// null-safe toLowerCase
private String toLowerCase(String str) {
    if (StringUtils.isNotBlank(str)) {
        return str.toLowerCase();
    }
    return null;
}
```

메서드 체이닝(`str.toLowerCase()`)을 직접 사용하면 `str`이 null일 때 `NullPointerException`이 발생하므로, 별도 메서드로 분리해 안전하게 처리합니다.

이 참조 데이터 캐싱 패턴은 ShortURL 최적화의 `@Cacheable` 패턴과 사고방식이 같습니다: **변하지 않는 데이터는 메모리에 올려두고, DB 접근을 최소화**하는 것입니다.

---

## 동시성 문제: 트리 구조 변경 시의 정합성

부서 트리 구조는 읽기 위주의 데이터이지만, **부서 이동(reparent)**, **부서 신설/폐지** 같은 변경이 발생할 때 정합성 문제가 생길 수 있습니다. 이 섹션에서는 트리 구조 변경 시 고려해야 할 동시성 문제를 다룹니다.

### 부서 이동(reparent) 시 depth 일괄 갱신의 원자성

부서를 다른 부모 아래로 이동하면, 해당 부서와 **모든 후손 부서의 depth**를 갱신해야 합니다.

```
이동 전:
코코네M (depth=1)
├── SYF (depth=2)
│   └── Web팀 (depth=3)
└── 경영지원 (depth=2)

작업: Web팀을 경영지원 아래로 이동

이동 후:
코코네M (depth=1)
├── SYF (depth=2)
└── 경영지원 (depth=2)
    └── Web팀 (depth=3)  ← parentId, depth 모두 변경 필요
```

이 경우에는 Web팀만 이동하면 되고 depth도 3으로 동일하지만, 더 복잡한 시나리오를 보겠습니다.

```
이동 전:
코코네M (depth=1)
├── SYF (depth=2)
│   ├── Web팀 (depth=3)
│   │   └── 프론트엔드 (depth=4)
│   └── App팀 (depth=3)
└── 경영지원 (depth=2)

작업: SYF를 경영지원 아래로 이동

이동 후:
코코네M (depth=1)
└── 경영지원 (depth=2)
    └── SYF (depth=3)          ← depth 2→3
        ├── Web팀 (depth=4)    ← depth 3→4
        │   └── 프론트엔드 (depth=5)  ← depth 4→5
        └── App팀 (depth=4)    ← depth 3→4
```

SYF를 이동하면 SYF 자신 + 모든 후손(Web팀, 프론트엔드, App팀)의 depth를 **일괄적으로 +1** 해야 합니다. 이 갱신이 **하나의 트랜잭션 안에서 원자적으로** 실행되어야 합니다.

```java
@Transactional
public void reparentDept(String deptId, String newParentId) {
    // 1. 순환 참조 검증
    validateNoCycle(deptId, newParentId);

    // 2. 새 부모의 depth 조회
    Dept newParent = deptRepository.findByDeptId(newParentId);
    Dept target = deptRepository.findByDeptId(deptId);
    int depthDiff = (newParent.getDepth() + 1) - target.getDepth();

    // 3. 대상 부서의 parentId 변경
    target.updateParentDeptId(newParentId);

    // 4. 대상 + 모든 후손의 depth 일괄 갱신
    List<String> allSubDeptIds = getSubAllDeptIdList(List.of(deptId));
    for (String subDeptId : allSubDeptIds) {
        Dept subDept = deptRepository.findByDeptId(subDeptId);
        subDept.updateDepth(subDept.getDepth() + depthDiff);
    }
}
```

만약 4단계에서 일부 후손만 갱신된 상태에서 예외가 발생하면, **`@Transactional`이 롤백**해주므로 데이터 불일치가 방지됩니다. 이것이 트리 구조 변경에서 트랜잭션이 중요한 이유입니다.

### 동시에 두 사용자가 부서 구조를 수정할 때

실무에서 부서 구조 변경은 연 1~2회 수준이므로 동시 수정이 발생할 확률은 낮습니다. 하지만 가능성이 있는 시나리오를 분석해보겠습니다.

```
시나리오: 관리자 A와 관리자 B가 동시에 부서 구조를 수정

시점 1: 관리자 A가 SYF를 경영지원 아래로 이동 시작
시점 2: 관리자 B가 Web팀을 SYF 밖으로 이동 시작

관리자 A의 작업:
  - SYF + 모든 하위부서의 depth 갱신 (Web팀 포함)

관리자 B의 작업:
  - Web팀의 parentId를 다른 부서로 변경
  - Web팀의 depth 갱신

문제: 관리자 A가 Web팀의 depth를 갱신하는 시점에,
      관리자 B가 이미 Web팀의 parentId를 변경했다면?
      → Web팀의 depth가 잘못된 값으로 설정될 수 있음
```

이 문제를 해결하는 대표적인 두 가지 접근법이 있습니다.

### 낙관적 락(Optimistic Lock)

**변경 충돌이 드물다**는 전제하에, 충돌이 발생하면 재시도하는 방식입니다. JPA의 `@Version` 어노테이션으로 구현합니다.

```java
@Entity
public class Dept {
    @Id
    private String deptId;

    private String parentDeptId;
    private int depth;

    @Version
    private Long version;  // 낙관적 락을 위한 버전 컬럼
    // ...
}
```

`@Version`이 붙은 필드는 UPDATE 시 자동으로 **WHERE version = ?** 조건이 추가됩니다. 두 트랜잭션이 같은 행을 동시에 수정하면, 먼저 커밋한 쪽이 version을 증가시키고, 나중에 커밋하는 쪽은 **version 불일치로 `OptimisticLockException`**이 발생합니다.

```sql
-- 관리자 A의 UPDATE (먼저 실행)
UPDATE dept SET depth = 3, version = 2 WHERE dept_id = 'SYF' AND version = 1;
-- 영향받은 행: 1 (성공)

-- 관리자 B의 UPDATE (나중에 실행)
UPDATE dept SET parent_dept_id = 'NEW', depth = 2, version = 2
WHERE dept_id = 'SYF' AND version = 1;
-- 영향받은 행: 0 → OptimisticLockException 발생!
```

```java
// 낙관적 락 재시도 패턴
@Retryable(value = OptimisticLockException.class, maxAttempts = 3)
@Transactional
public void reparentDeptWithRetry(String deptId, String newParentId) {
    reparentDept(deptId, newParentId);
}
```

### 비관적 락(Pessimistic Lock)

**변경 전에 행을 잠그는 방식**입니다. 다른 트랜잭션은 락이 해제될 때까지 대기합니다.

```java
// 비관적 락: SELECT ... FOR UPDATE
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT d FROM Dept d WHERE d.deptId = :deptId")
Dept findByDeptIdForUpdate(@Param("deptId") String deptId);
```

```sql
-- 실제 실행되는 SQL
SELECT * FROM dept WHERE dept_id = 'SYF' FOR UPDATE;
-- 이 행이 잠기므로, 다른 트랜잭션은 이 행을 읽거나 수정할 수 없음
```

### 어떤 락을 선택할 것인가

| 비교 항목 | 낙관적 락 | 비관적 락 |
|---|---|---|
| 충돌 빈도가 낮을 때 | **적합** (대부분 충돌 없이 통과) | 불필요한 락 오버헤드 |
| 충돌 빈도가 높을 때 | 재시도 비용 증가 | **적합** (대기 후 순차 처리) |
| 읽기 성능 | 영향 없음 | 락으로 인한 읽기 대기 가능 |
| 데드락 위험 | 없음 | **있음** (여러 행을 잠글 때) |
| 구현 복잡도 | @Version + 재시도 로직 | @Lock 한 줄 |

우리 시스템에서는 **낙관적 락**이 적합합니다. 이유는 다음과 같습니다.

1. **부서 구조 변경은 연 1~2회**: 충돌 확률이 극히 낮으므로, 재시도 비용이 거의 발생하지 않습니다.
2. **읽기가 압도적으로 많음**: 조직도 조회, 메달 대상자 조회 등 읽기 요청이 대부분이므로, 비관적 락이 읽기 성능을 저하시킬 수 있습니다.
3. **데드락 위험 회피**: 부서 이동 시 여러 행을 동시에 수정하므로, 비관적 락은 데드락 위험이 있습니다.

다만, 현재 시스템에서는 부서 구조 변경이 워낙 드물기 때문에 **별도의 락을 도입하지 않고**, 관리자 페이지에서 **한 번에 한 명만 부서 구조를 수정할 수 있도록 UI 레벨에서 제어**하는 방식을 사용하고 있습니다. 기술적으로 완벽한 해결책은 아니지만, 변경 빈도와 팀 규모를 고려하면 충분히 실용적인 선택이었습니다.

---

## 최종 결과

### 구조 개선 요약

| 영역 | 기존 | 최종 |
|---|---|---|
| 부서 관계 | 추정 (이중 for문) | parentId FK 명시 |
| 트리 빌드 | O(n^2) 이중 for문 | O(n) 단일 순회 (makeChildStruct) |
| depth 관리 | 없음 (동적 계산) | DB 컬럼 + 일회성 마이그레이션 |
| 부서-사원 매핑 | 단일 PK | 복합 PK (deptId + crewSeq) |
| 하위부서 탐색 | 비체계적 순회 | BFS 레벨별 탐색 |
| 참조 데이터 | 매번 7개 JOIN | 캐싱 Map + 애플리케이션 매핑 |

### 복합 PK가 지켜주는 것들

| 시나리오 | 단일 PK | 복합 PK (deptId + crewSeq) |
|---|---|---|
| 같은 부서에 같은 사원 중복 등록 | **가능** (버그) | **불가능** (PK 위반) |
| 부서별 사원 조회 | 별도 인덱스 필요 | PK 인덱스 자동 활용 |
| 특정 (부서, 사원) 존재 확인 | WHERE + AND | PK 조회 O(1) |

---

## 개선 과정에서 배운 것들

### 1. 데이터 모델이 코드의 복잡도를 결정한다

기존의 이중 for문은 **코드가 나쁜 것**이 아니라, **데이터 모델에 관계 정보가 없었기 때문에 코드가 그렇게 될 수밖에 없었던 것**입니다. parentId 하나가 추가되었을 뿐인데, 조회 로직의 복잡도가 O(n^2)에서 O(n)으로 바뀌었습니다.

코드를 리팩토링하기 전에, **데이터 모델이 비즈니스 관계를 제대로 표현하고 있는지**를 먼저 확인해야 한다는 것을 배웠습니다.

### 2. 복합 PK는 편의성 vs 정합성의 트레이드오프

단일 PK(`AUTO_INCREMENT`)는 JPA에서 다루기 편합니다. `@GeneratedValue`만 붙이면 됩니다. 하지만 복합 PK를 도입하면 `@IdClass`와 `Serializable` 구현이 필요하고, `equals()`/`hashCode()`도 신경 써야 합니다.

그럼에도 복합 PK를 선택한 이유는, **같은 부서에 같은 사원이 중복 등록될 수 없다**는 규칙을 DB가 강제해주기 때문입니다. 애플리케이션 코드의 `if`문은 누군가 빼먹을 수 있지만, PK 제약은 **절대 우회할 수 없는 마지노선**입니다.

### 3. 트리 구조는 저장보다 변경과 탐색이 어렵다

트리를 DB에 저장하는 것 자체는 간단합니다. 하지만:
- depth를 일괄 갱신하는 마이그레이션 (updateDepth)
- 모든 하위부서를 빠짐없이 탐색하는 로직 (getSubAllDeptIdList)
- 트리를 flat list에서 다시 구성하는 빌드 (makeChildStruct)

이 세 가지가 진짜 어려운 부분이었습니다. 트리 구조를 설계할 때는 **읽기, 쓰기, 탐색 패턴을 모두 함께 고려**해야 한다는 것을 깨달았습니다.

### 4. 같은 캐싱 철학이 반복된다

ShortURL 최적화에서 `@Cacheable`로 CountryIp를 캐싱한 것과, 여기서 `getCacheCodeMap()`으로 참조 데이터를 캐싱한 것은 **같은 원칙**입니다: 변하지 않는 데이터는 메모리에 올려두고, 매번 DB를 치지 않는다.

이 패턴이 반복된다는 것은, **성능 최적화의 가장 기본적인 원칙이 불필요한 I/O 제거**라는 것을 의미합니다.

### 5. 기존 코드에는 이유가 있다

기존 이중 for문 코드를 처음 봤을 때는 왜 이렇게 짰을까 하는 의문이 들었지만, parentId가 없는 테이블 구조에서는 **그것이 유일한 방법**이었습니다. 코드의 문제가 아니라 데이터 설계의 문제였고, 코드는 그 제약 안에서 최선을 다한 것이었습니다.

리팩토링을 할 때 기존 코드를 비판하기보다는, **이 코드가 이렇게 된 맥락(제약)을 먼저 이해하는 자세**가 더 나은 해결책으로 이끌어준다는 것을 느꼈습니다.

### 6. 트리 모델 선택은 도메인 특성에 달려있다

Adjacency List, Nested Set, Materialized Path, Closure Table 네 가지 모델을 비교해보면서, **만능인 모델은 없다**는 것을 확인했습니다. 부서 수가 수백 건이고 변경이 드문 우리 시스템에서는 가장 단순한 Adjacency List가 최적이었지만, 수만 건의 카테고리를 관리하는 커머스 시스템이었다면 Closure Table이나 Nested Set이 더 나은 선택이었을 것입니다.

기술 선택에서 중요한 것은 **이론적으로 가장 우수한 모델**이 아니라, **우리 도메인의 읽기/쓰기 패턴에 가장 잘 맞는 모델**을 고르는 것이라는 점을 다시 한 번 체감했습니다.

### 7. 동시성은 발생 빈도로 판단한다

락(Lock)의 필요성을 판단할 때, **이론적으로 가능한 문제**와 **실무에서 실제로 발생하는 문제**를 구분하는 것이 중요합니다. 부서 구조 변경이 연 1~2회인 시스템에 비관적 락을 도입하면, 99.99%의 읽기 요청이 불필요한 락 체크 비용을 지불하게 됩니다.

하지만 동시성 문제를 **아예 무시**하는 것은 위험합니다. 우리는 **UI 레벨의 접근 제어**라는 현실적 타협과, 향후 필요 시 **낙관적 락을 도입할 수 있는 설계**(`@Version` 컬럼 추가 여지)를 함께 확보해두었습니다. 완벽한 동시성 제어보다는, **현재 상황에 적합한 수준의 제어 + 확장 가능성**을 갖추는 것이 실용적이라고 생각합니다.

---

## 마치며

이 작업은 단순히 느린 코드를 빠르게 고친 것이 아니라, **데이터 모델의 표현력을 높여서 코드와 DB 양쪽의 복잡도를 동시에 줄인** 경험이었습니다.

결과적으로:
- 이중 for문 → 단일 순회 `makeChildStruct` (성능)
- 관계 추정 → 명시적 parentId FK (정확성)
- depth 없음 → depth 컬럼 + 마이그레이션 (계층 표현)
- 단일 PK → 복합 PK `DeptCrewId` (정합성)
- 매번 JOIN → 참조 데이터 캐싱 `getCacheCodeMap` (효율)

다섯 가지가 연쇄적으로 개선되었고, 이 모든 것의 출발점은 **parentId 컬럼 하나를 추가하는 것**이었습니다. 작은 모델 변경이 시스템 전체의 품질을 바꿀 수 있다는 것을 체감한 경험이었습니다.
