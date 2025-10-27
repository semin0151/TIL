## 🔹 List 계열

| 구현체 | 내부 구조 | 특징 | 시간 복잡도 | 순서 보장 | 스레드 세이프 |
| --- | --- | --- | --- | --- | --- |
| **ArrayList** | 동적 배열 (Resizable Array) | 가장 일반적. Random access 빠름. 중간 삽입/삭제는 느림. | 조회 O(1)삽입/삭제 O(n) | ✅ (인덱스 순서) | ❌ |
| **LinkedList** | 이중 연결 리스트 (Double Linked List) | 삽입/삭제가 빠름. 인덱스 접근은 느림. | 조회 O(n)삽입/삭제 O(1) | ✅ (삽입 순서) | ❌ |

---

## 🔹 Set 계열

| 구현체 | 내부 구조 | 특징 | 시간 복잡도 | 순서 보장 | 스레드 세이프 |
| --- | --- | --- | --- | --- | --- |
| **HashSet** | HashMap 기반 (Key만 사용) | 중복 불가. 순서 없음. | 삽입/삭제/탐색 O(1) | ❌ | ❌ |
| **LinkedHashSet** | HashSet + LinkedList | 입력 순서 유지. | 삽입/삭제/탐색 O(1) | ✅ (입력 순서) | ❌ |
| **TreeSet** | Red-Black Tree (이진 검색 트리) | 정렬된 순서로 저장 (Comparable/Comparator 필요). | 삽입/삭제/탐색 O(log n) | ✅ (정렬 순서) | ❌ |

---

## 🔹 Map 계열

| 구현체 | 내부 구조 | 특징 | 시간 복잡도 | 순서 보장 | 스레드 세이프 |
| --- | --- | --- | --- | --- | --- |
| **HashMap** | 해시 테이블 + 버킷 내 Red-Black Tree | 가장 일반적. Key 중복 불가, 순서 없음. | 조회/삽입/삭제 O(1) | ❌ | ❌ |
| **LinkedHashMap** | HashMap + 이중 연결 리스트 | 입력 순서 또는 Access 순서 유지 가능. | 조회/삽입/삭제 O(1) | ✅ (입력/접근 순서) | ❌ |
| **TreeMap** | Red-Black Tree | Key 기준 정렬 저장 (Comparable/Comparator). | 조회/삽입/삭제 O(log n) | ✅ (정렬 순서) | ❌ |

---

✅ **요약**

- 순서 유지가 필요한 경우 → `LinkedList`, `LinkedHashSet`, `LinkedHashMap`
- 정렬된 결과가 필요한 경우 → `TreeSet`, `TreeMap`
- 가장 빠른 일반용 컬렉션 → `ArrayList`, `HashSet`, `HashMap`
