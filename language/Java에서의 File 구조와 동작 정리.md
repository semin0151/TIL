## 🧩 1. File의 설계 철학

> java.io.File은 “파일의 내용(content)”이 아니라 “파일 자체(metadata)”를 다루는 클래스

즉,

- 파일의 존재, 이름, 경로, 권한 등 “파일 시스템 정보”를 다루며
- 실제 데이터 읽기/쓰기는 `FileInputStream`, `FileOutputStream`(Stream 계열)이 담당합니다.

---

## ⚙️ 2. File과 Stream의 역할 구분

| 구분 | 주요 클래스 | 역할 | 실제 I/O 발생 | 블로킹 여부 |
| --- | --- | --- | --- | --- |
| **파일 조작(메타데이터)** | `File`, `Files`, `Path` | 파일 이름, 경로, 삭제, 이동, 권한 관리 | ❌ (metadata only) | ❌ |
| **데이터 입출력** | `FileInputStream`, `FileOutputStream` | 파일의 실제 내용 읽기/쓰기 | ✅ (read/write syscall) | ✅ 가능 |

📘 즉,

`File`은 **OS 파일 시스템 테이블(inode, dentry)** 조작,

`Stream`은 **디스크 데이터 블록 I/O** 담당.

---

## 🧠 3. File.delete()의 실제 동작 과정

> File.delete() → OS의 unlink(path) syscall 호출
> 

### 내부 단계

1. **디렉터리 엔트리 제거** (파일 이름 → inode 연결 해제)
2. **inode 참조 카운트 감소**
3. **참조가 0이면 커널이 디스크 블록 해제** (비동기 처리)

### 결과

- `delete()` 호출 시점에는 파일 “이름”만 사라짐
- 파일 내용(디스크 블록)은 커널이 나중에 정리
- 다른 프로세스가 여전히 fd를 열고 있으면 실제 삭제는 지연됨

💡 즉, 삭제는 빠르지만 **디스크 공간 회수는 지연될 수 있음.**

---

## ⚙️ 4. File 재생성 시 동작

> 동일한 경로로 새 파일을 write 하면,
> 
> 
> **같은 이름이지만 완전히 다른 inode(메타데이터)** 를 가진 새 파일이 생성됩니다.
> 

```scss
[삭제 전]
  "data.txt" → inode #42

File.delete()  → unlink()
FileOutputStream("data.txt") → open(O_CREAT)
[삭제 후]
  "data.txt" → inode #77
```

즉,

경로는 같아도 파일은 완전히 새 객체 (새 inode, 새 fd).

이전 파일은 완전히 독립적인 존재였음.

---

## ⚙️ 5. File.renameTo(), File.exists() 등도 메타데이터 조작

| 메서드 | 내부 syscall | 의미 | 디스크 I/O |
| --- | --- | --- | --- |
| `renameTo()` | `rename()` | inode 연결 변경 (데이터 이동 없음) | ❌ |
| `exists()` | `stat()` | 파일 상태 조회 | ❌ |
| `length()` | `stat()` | inode의 size 필드 조회 | ❌ |
| `mkdir()` | `mkdir()` | 디렉터리 엔트리 생성 | ❌ |
| `setReadable()` | `chmod()` | 권한 비트 변경 | ❌ |
| `delete()` | `unlink()` | 디렉터리 엔트리 제거 | ❌ |

💡 이들은 모두 **커널 메모리 내 파일시스템 구조(VFS layer)** 조작이라,

**스레드 블로킹 없이 즉시 반환**됩니다.

---

## 🧱 6. File과 Stream의 계층 구조 (요약 다이어그램)

```
java.io.File         → 파일 이름 / 경로 / 존재 여부 (메타데이터)
java.io.FileInputStream  → 파일 내용 읽기 (read(fd))
java.io.FileOutputStream → 파일 내용 쓰기 (write(fd))
java.nio.file.Files      → 고급 파일 조작 (복사, 이동, 권한 등)
```

```
┌─────────────────────────────┐
│ Application (Java)          │
│  ├─ File (metadata)         │
│  ├─ FileInputStream (read)  │
│  ├─ FileOutputStream (write)│
│  └─ Files / Path (NIO2)     │
└────────────┬────────────────┘
             │ syscalls
┌────────────▼────────────┐
│ OS Kernel (VFS Layer)   │
│  ├─ stat() / unlink()   │
│  ├─ read() / write()    │
│  └─ rename() / chmod()  │
└────────────┬────────────┘
             │
┌────────────▼────────────┐
│ File System (ext4, APFS)│
│  ├─ inode / dentry      │
│  └─ data blocks         │
└─────────────────────────┘
```

---

## ⚙️ 7. I/O 발생 여부 정리

| 작업 | I/O 발생 | 이유 |
| --- | --- | --- |
| `File.delete()` | ❌ | inode unlink (metadata 조작) |
| `File.renameTo()` | ❌ | 디렉터리 엔트리만 수정 |
| `File.length()` | ❌ | inode 메타데이터 조회 |
| `FileInputStream.read()` | ✅ | 파일 데이터 읽기 |
| `FileOutputStream.write()` | ✅ | 파일 데이터 쓰기 |
| `FileOutputStream.fd.sync()` | ✅ | 커널 Page Cache → 디스크 flush |

---

## ✅ 8. 결론 요약

| 구분 | 책임 | 실제 동작 | 디스크 접근 |
| --- | --- | --- | --- |
| **File** | 파일 존재·이름·권한·삭제 등 관리 | OS 파일시스템 메타데이터 조작 (`stat`, `unlink`, `rename`) | ❌ 없음 |
| **FileInputStream / FileOutputStream** | 파일 내용 읽기/쓰기 | `read(fd)`, `write(fd)` syscall | ✅ 있음 |
| **BufferedOutputStream** | I/O 성능 최적화 | JVM 버퍼 → OS Page Cache | ✅ 있음 |
| **FileDescriptor.sync()** | 디스크 기록 보장 | `fsync(fd)` 호출 | ✅ 디스크 flush |

> 💬 즉,
> 
> - **File 계열은 "파일의 존재"를 다루는 레벨 (메타데이터 조작)**
> - **Stream 계열은 "파일의 내용"을 다루는 레벨 (데이터 I/O)**
> 
> File.delete()나 renameTo()는 커널 내부에서 이름만 수정하며,
> 
> 실제 디스크 블록 해제나 기록은 커널이 나중에 처리합니다.
>
