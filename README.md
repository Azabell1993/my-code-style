# 코드 스타일 및 개발 습관

## 개요
본 문서는 저의 코드 스타일과 개발 습관을 설명하며, 다양한 상황에서 선호하는 구현 방식과 설계 철학을 정리한 것입니다.

## 1. **핵심 로직 중심의 설계**
저는 불필요한 복잡성을 배제하고 핵심 로직에 집중하는 스타일을 선호합니다. 코드의 구조는 단순하면서도 확장 가능하도록 설계됩니다.

### **예제: 프로세스 관리 시스템**
```c
#define MAX_PROCESSES 10

typedef struct {
    char name[256];
    int running; // (0: 중지, 1: 실행)
} Process;

static Process process_table[MAX_PROCESSES];
static int process_count = 0;

int kernel_create_process(const char *process_name) {
    if (process_count >= MAX_PROCESSES) {
        az_printf("\nError: Process table is full.");
        return 0;
    }
    strncpy(process_table[process_count].name, process_name, sizeof(process_table[process_count].name) - 1);
    process_table[process_count].running = 1;
    process_count++;
    az_printf("\nProcess created: %s", process_name);
    return 1;
}
```
💡 **핵심 포인트:**
- 최대한 간결한 구조 유지
- 불필요한 동적 할당 피하기 (배열 사용)
- `az_printf()`를 활용하여 디버깅 가능하도록 로그 출력

## 2. **가변성을 고려한 함수 설계**
저는 재사용성과 확장성을 고려하여 코드를 작성합니다. 함수는 단일 책임 원칙을 따르며, 여러 기능을 포함하지 않도록 신경 씁니다.

### **예제: 가변 인자 처리 출력 함수**
```c
void az_printf(const char *format, ...) {
    if (qt_print_function) {
        char buffer[1024];
        va_list args;
        va_start(args, format);
        vsnprintf(buffer, sizeof(buffer), format, args);
        va_end(args);
        qt_print_function(buffer);
    }
}
```
💡 **핵심 포인트:**
- 가변 인자를 활용하여 확장 가능하도록 구현
- Qt의 UI 출력과 연동 가능하도록 `qt_print_function`을 사용
- `buffer[1024]`를 활용하여 안전한 문자열 처리를 보장

## 3. **플랫폼 독립적인 시스템 정보 수집**
저는 크로스플랫폼을 고려한 개발을 선호하며, OS별로 다른 API를 사용하더라도 코드 스타일의 일관성을 유지합니다.

### **예제: CPU 정보 가져오기 (Windows/Linux/macOS 지원)**
```cpp
std::string EdgeClient::getCPUInfo() {
    std::ostringstream cpuInfo;
    #ifdef _WIN32
        SYSTEM_INFO sysInfo;
        GetSystemInfo(&sysInfo);
        cpuInfo << "Windows CPU, Cores: " << sysInfo.dwNumberOfProcessors;
    #elif __linux__
        std::ifstream cpuinfo("/proc/cpuinfo");
        std::string line;
        while (std::getline(cpuinfo, line)) {
            if (line.find("model name") != std::string::npos) {
                cpuInfo << line.substr(line.find(":") + 2);
                break;
            }
        }
    #elif __APPLE__
        char buffer[128];
        size_t bufferSize = sizeof(buffer);
        sysctlbyname("machdep.cpu.brand_string", &buffer, &bufferSize, NULL, 0);
        cpuInfo << buffer;
    #endif
    return cpuInfo.str();
}
```
💡 **핵심 포인트:**
- `#ifdef`를 사용하여 플랫폼별 코드를 분리
- OS별로 다른 시스템 호출을 활용하면서도 일관된 인터페이스 유지
- `std::ostringstream`을 사용하여 문자열 처리를 편리하게 구현

## 4. **스마트 포인터를 활용한 메모리 관리**
저는 수동적인 메모리 관리를 최소화하고 `std::shared_ptr` 및 `std::unique_ptr`을 적극 활용하여 메모리 누수를 방지합니다.

### **예제: 스마트 포인터 활용**
```cpp
std::shared_ptr<int> processData = std::make_shared<int>(rand() % 1000);
kernel_printf("Process Smart Pointer Created With Value: %d\n", *processData);
```
💡 **핵심 포인트:**
- `std::make_shared`를 사용하여 불필요한 `new` 호출 방지
- 스마트 포인터의 `use_count()`를 통해 참조 횟수 추적 가능
- `kernel_printf()`를 활용하여 시스템 로그에 기록

## 5. **직관적인 CLI 명령어 설계**
저는 CLI 개발 시 직관적인 명령어 설계를 지향하며, 사용자 경험을 고려하여 커맨드 프롬프트를 구성합니다.

### **예제: 커맨드 실행 함수**
```cpp
void CmdWindow::handleCommand(const QString &command) {
    if (command == "help") {
        ui->textEdit->append("Available commands: create, list, kill, clear");
    } else if (command.startsWith("create ")) {
        QString process_name = command.mid(7);
        kernel_create_process(process_name.toStdString().c_str());
    } else {
        ui->textEdit->append("Unknown command. Type 'help' for a list of commands.");
    }
}
```
💡 **핵심 포인트:**
- 사용자가 직관적으로 입력할 수 있도록 `help` 명령어 제공
- `startsWith()`를 활용하여 명령어 구분
- `append()`를 사용하여 UI에 피드백 제공

## 6. **자작 스마트 포인터 설계**
저는 스마트 포인터를 활용하여 안전한 메모리 관리를 수행하며, 필요할 경우 직접 자작 스마트 포인터를 설계하여 사용할 수 있습니다.

### **예제: 스마트 포인터 구조체 구현**
```c
typedef struct SmartPtr {
    void *ptr;
    int *ref_count;
    pthread_mutex_t *mutex;
} SmartPtr;

SmartPtr create_smart_ptr(size_t size) {
    SmartPtr sp;
    sp.ptr = malloc(size);
    sp.ref_count = (int *)malloc(sizeof(int));
    *(sp.ref_count) = 1;
    sp.mutex = (pthread_mutex_t *)malloc(sizeof(pthread_mutex_t));
    pthread_mutex_init(sp.mutex, NULL);
    return sp;
}

void retain(SmartPtr *sp) {
    pthread_mutex_lock(sp->mutex);
    (*(sp->ref_count))++;
    pthread_mutex_unlock(sp->mutex);
}

void release(SmartPtr *sp) {
    int should_free = 0;
    pthread_mutex_lock(sp->mutex);
    (*(sp->ref_count))--;
    if (*(sp->ref_count) == 0) {
        should_free = 1;
    }
    pthread_mutex_unlock(sp->mutex);

    if (should_free) {
        free(sp->ptr);
        free(sp->ref_count);
        pthread_mutex_destroy(sp->mutex);
        free(sp->mutex);
    }
}
```
💡 **핵심 포인트:**
- `ref_count`를 활용한 참조 카운팅
- `pthread_mutex_t`를 사용한 스레드 안전한 관리
- `retain()`과 `release()`를 통해 자동 메모리 해제

---

## 7. **C 스타일 객체지향 (C-OOP) 설계**
C 언어에서 객체지향적인 개념을 적용하려면, 구조체와 함수 포인터를 활용하여 캡슐화, 상속, 다형성을 구현해야 합니다.   
자주 사용하는 방법은 아니지만, 필요에 의해 C에서도 OOP 패턴을 도입하여 재사용성과 유지보수성을 높이는 방식을 선호합니다.

### **7.1. 기본적인 구조체 기반 객체지향 설계**
이 예제는 가장 간단한 형태의 OOP 개념을 적용한 코드로, 구조체와 함수 포인터를 활용하여 메서드를 구현하는 방식을 보여줍니다.

```c
typedef struct _Hello {
    void (*HELLO)();
} HELLOWORLD;

void HELLO_PRINT() {
    puts("Hello World");
}

void HELLO_INIT(HELLOWORLD *this) {
    this->HELLO = HELLO_PRINT;
}

int main() {
    HELLOWORLD HELLO;
    HELLO_INIT(&HELLO);
    HELLO.HELLO();
}
```

💡 **핵심 포인트**
- 구조체 내부에 함수 포인터를 포함하여 객체의 메서드를 정의
- `HELLO_INIT()` 함수를 활용하여 생성자 역할을 수행
- 캡슐화 개념을 적용하여 코드의 모듈성을 높임

---

### **7.2. 함수 포인터를 활용한 연산 객체**
이 예제에서는 연산을 수행하는 객체를 만들어, 데이터를 저장하고 계산하는 기능을 포함하도록 설계되었습니다.

```c
typedef struct _Result_Sum {
    int x, y, result;
    int (*INPUT)(struct _Result_Sum*, int, int, int);
    int (*PUTS)(const struct _Result_Sum*);
} SUM;

int SUM_INPUT(struct _Result_Sum* this, int x, int y, int result) {
    return this->result = this->x + this->y;
}

int SUM_RESULT(const struct _Result_Sum* this) {
    return printf("%d + %d = %d\n", this->x, this->y, this->result);
}

void SUM_INIT(SUM* this) {
    this->INPUT = SUM_INPUT;
    this->PUTS = SUM_RESULT;
}

int main() {
    SUM new_sum = {10, 20, 0};
    SUM_INIT(&new_sum);
    new_sum.INPUT(&new_sum, new_sum.x, new_sum.y, new_sum.result);
    new_sum.PUTS(&new_sum);
}
```

💡 **핵심 포인트**
- 구조체 내부에서 함수 포인터를 활용하여 메서드를 지정
- `SUM_INIT()`를 통해 객체 초기화 및 메서드 바인딩
- 캡슐화를 통해 `INPUT()`과 `PUTS()`를 메서드처럼 호출 가능

---

### **7.3. 구조체를 통한 상속 및 다형성 구현**
C 언어에서 구조체를 활용한 상속을 흉내내는 방식입니다. 자식 구조체가 부모 구조체를 포함하여 확장하는 형태를 가지며, 함수 포인터를 통해 다형성을 흉내낼 수 있습니다.

```c
typedef struct MEMBER {
    int id;
    char name[50];
    void (*PRINT)(struct MEMBER*);
} MEMBER;

typedef struct STUDENT {
    MEMBER base;
    int grade;
} STUDENT;

void MEMBER_PRINT(MEMBER* this) {
    printf("ID: %d, Name: %s\n", this->id, this->name);
}

void STUDENT_PRINT(MEMBER* this) {
    STUDENT* student = (STUDENT*)this;
    printf("ID: %d, Name: %s, Grade: %d\n", student->base.id, student->base.name, student->grade);
}

void MEMBER_INIT(MEMBER* this, int id, const char* name) {
    this->id = id;
    strncpy(this->name, name, sizeof(this->name) - 1);
    this->PRINT = MEMBER_PRINT;
}

void STUDENT_INIT(STUDENT* this, int id, const char* name, int grade) {
    MEMBER_INIT(&this->base, id, name);
    this->base.PRINT = STUDENT_PRINT;
    this->grade = grade;
}

int main() {
    MEMBER member;
    STUDENT student;

    MEMBER_INIT(&member, 1, "John Doe");
    STUDENT_INIT(&student, 2, "Alice", 90);

    member.PRINT(&member);
    student.base.PRINT((MEMBER*)&student);
}
```

💡 **핵심 포인트**
- `STUDENT` 구조체가 `MEMBER` 구조체를 포함하여 상속 개념을 구현
- `PRINT()` 함수 포인터를 다형성처럼 활용하여 동적 바인딩 지원
- `MEMBER_INIT()`와 `STUDENT_INIT()`를 활용하여 생성자 역할 수행

---

### **7.4. STL과 유사한 데이터 구조 구현**
C++의 `std::vector`와 유사한 `ArrayList`를 C에서 구조체와 함수 포인터로 구현한 예제입니다.

```c
typedef struct {
    int* buffer;
    size_t size;
    size_t capacity;
    void (*push_back)(struct ArrayList*, int);
    void (*pop_back)(struct ArrayList*);
    int (*at)(const struct ArrayList*, size_t);
} ArrayList;

void ArrayList_push_back(ArrayList* self, int value) {
    if (self->size == self->capacity) {
        size_t new_capacity = self->capacity * 2 + 1;
        self->buffer = realloc(self->buffer, new_capacity * sizeof(int));
        self->capacity = new_capacity;
    }
    self->buffer[self->size++] = value;
}

int ArrayList_at(const ArrayList* self, size_t index) {
    return index < self->size ? self->buffer[index] : -1;
}

void ArrayList_init(ArrayList* self) {
    self->buffer = NULL;
    self->size = 0;
    self->capacity = 0;
    self->push_back = ArrayList_push_back;
    self->at = ArrayList_at;
}

int main() {
    ArrayList list;
    ArrayList_init(&list);

    list.push_back(&list, 10);
    list.push_back(&list, 20);
    printf("Element at index 1: %d\n", list.at(&list, 1));

    free(list.buffer);
}
```

💡 **핵심 포인트**
- 동적 배열 구조를 활용하여 `push_back()`과 `at()`을 제공
- 함수 포인터를 사용하여 `ArrayList` 메서드처럼 사용 가능
- 동적 할당을 통해 유동적인 크기 조정 지원

---

### **7.5. 인라인 어셈블리 활용한 최적화**
C 코드에 인라인 어셈블리를 삽입하여 최적화하는 방식도 활용합니다. 이는 특정 프로세서 아키텍처에서 성능을 극대화하는 데 사용됩니다.

```c
#include <stdio.h>

int add(int x, int y) {
    int result;
    __asm__ ("addl %%ebx, %%eax"
             : "=a" (result)
             : "a" (x), "b" (y));
    return result;
}

int main() {
    int sum = add(10, 20);
    printf("Sum: %d\n", sum);
    return 0;
}
```

💡 **핵심 포인트**
- 어셈블리 코드를 직접 삽입하여 연산 성능 향상
- `__asm__` 문법을 사용하여 레지스터 기반 연산 수행
- 하드웨어 최적화를 고려한 설계


---
**작성자:** 박지우 (Azabell1993)

