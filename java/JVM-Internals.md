# [번역] JVM Internals
이 기사에서는 JVM(Java Virtual Machine)의 내부 아키텍처에 대해 설명합니다. 다음 다이어그램은 Java Virtual Machine Specification Java SE 7 Edition을 준수하는 일반적인 JVM 주요 내부 구성 요소를 보여줍니다.

![JVM_Internal_Architecture_small](./images/JVM_Internal_Architecture_small.png)

이 다이어그램에 표시된 구성 요소는 아래에서 두 섹션으로 각각 설명됩니다. 첫 번째 섹션 은 각 thread에 대해 생성되는 구성 요소를 다루고 두 번째 섹션 은 thread와 독립적으로 생성되는 구성 요소를 다룹니다.

- threads
  - JVM System threads
  - Per thread
  - program Counter (PC)
  - Stack
  - Native Stack
  - Stack Restrictions
  - Frame
  - Local Variables Array
  - Operand Stack
- Dynamic Linking
- Shared Between threads
  - Heap
  - Memory Management
  - Non-Heap Memory
  - Just In Time (JIT) Compilation
  - Method Area
  - Class File Structure
  - Classloader
  - Faster Class Loading
  - Where Is The Method Area
  - Classloader Reference
  - Run Time Constant Pool
  - Exception Table
  - Symbol Table
  - Interned Strings (String Table)

## Thread
- thread는 프로그램에서 실행되는 thread입니다.  
- JVM은 애플리케이션이 동시에 실행 중인 여러 thread를 가질 수 있도록 합니다.  
- Hotspot JVM에는 Java thread와 기본 운영 체제 thread간에 직접 매핑이 있습니다.  
- thread-local storage, allocation buffers, synchronization objects, stacks and the program counter와 같은 Java thread에 대한 모든 상태를 준비한 후 기본 thread가 생성됩니다.  
- Java thread가 종료되면 기본 thread가 회수됩니다.  
- 따라서 운영 체제는 모든 thread를 예약하고 사용 가능한 모든 CPU에 디스패치해야 합니다.  
- 기본 thread가 초기화되면 Java thread에서 run() 메소드를 호출합니다.
- run() 메서드가 반환되면, 포착되지 않은 예외가 처리되고 native thread는 thread종료의 결과로 JVM을 종료해야 하는지 확인합니다(즉, 마지막 비 데몬 thread인지). 
- thread가 종료되면 기본 thread와 Java thread 모두에 대한 모든 리소스가 해제됩니다.

## JVM System Threads
jconsole 또는 디버거를 사용하는 경우 백그라운드에서 실행 중인 수많은 thread가 있는 것을 볼 수 있습니다. 이러한 백그라운드 thread는 public static void main(String[]) 호출의 일부로 생성되는 main thread 및 main thread에 의해 생성된 모든 thread와 함께 실행됩니다.  
Hotspot JVM의 주요 백그라운드 시스템 thread는 다음과 같습니다.

#### VM Thread
- 이 thread는 JVM이 안전 지점에 도달해야 하는 작업이 나타날 때까지 기다립니다. 이러한 작업이 별도의 thread에서 발생해야 하는 이유는 모두 JVM이 힙 수정이 발생할 수 없는 안전한 지점에 있어야 하기 때문입니다. 이 thread가 수행하는 작업 유형은 "세계 정지(stop-the-world)" 가비지 수집, thread stack 덤프, thread 일시 중단 및 편향된 잠금 취소입니다.

#### Periodic task thread
- 이 thread는 주기적인 작업의 실행을 예약하는데 사용되는 timer event(i.e. interrupts)를 담당합니다.

#### GC threads
- 이 thread는 JVM에서 발생하는 다양한 유형의 garbage collection activities를 지원합니다.

#### Signal dispatcher thread
- 이 thread는 JVM 프로세스로 전송된 신호를 수신하고 적절한 JVM 메소드를 호출하여 JVM 내부에서 처리합니다.

## Per Thread
각 실행 thread에는 다음 구성 요소가 있습니다.

### Program Counter(PC)
native가 아닌 경우 현재 명령어(또는 opcode)의 주소입니다. 현재 방법이 기본이면 PC는 정의되지 않습니다. 모든 CPU에는 PC가 있으며 일반적으로 PC는 각 명령어 이후에 증가하므로 실행할 다음 명령어의 주소를 보유합니다. JVM은 PC를 사용하여 명령을 실행하는 위치를 추적합니다. PC는 실제로 메서드 영역의 메모리 주소를 가리킵니다.

### Stack
각 thread에는 해당 thread에서 실행되는 각 메서드에 대한 frame을 보유하는 자체 stack이 있습니다. stack은 LIFO(Last In First Out) 데이터 구조이므로 현재 실행 중인 메서드가 stack의 맨 위에 있습니다. 모든 메서드 호출에 대해 새 frame이 생성되고 stack 맨 위에 추가(pushed)됩니다. 메서드가 정상적으로 반환되거나 메서드 호출 중에 잡히지 않은 예외가 throw 되면 frame이 제거(popped)됩니다. stack은 frame 개체를 푸시하고 팝하는 것을 제외하고는 직접 조작되지 않으므로 frame 개체가 힙에 할당될 수 있으며 메모리가 인접할 필요가 없습니다.

### Native Stack
모든 JVM이 기본 메소드를 지원하는 것은 아니지만 일반적으로 thread 별 기본 메소드 stack을 생성하는 JVM이 지원됩니다. JVM이 JNI(Java Native Invocation)용 C 연결 모델을 사용하여 구현된 경우 기본 tack 은 C stack이 됩니다. 이 경우 기본 stack에서 인수 및 반환 값의 순서는 일반적인 C 프로그램과 동일합니다. 기본 메소드는 일반적으로 JVM 구현에 따라 JVM을 다시 호출하고 Java 메소드를 호출할 수 있습니다. 이러한 기본 Java 호출은 Stack(일반 Java Stack)에서 발생합니다. thread는 기본 stack을 떠나 Stack(일반 Java Stack)에 새 frame을 생성합니다.

### Stack Restrictions
Stack은 동적 또는 고정 크기일 수 있습니다. thread에 허용된 것보다 큰 stack이 필요한 경우 StackOverflowError 가 발생합니다. thread에 새 frame이 필요하고 이를 할당할 메모리가 충분하지 않으면 OutOfMemoryError가 발생합니다.

### Frame
모든 메서드 호출에 대해 새 frame이 생성되고 stack 맨 위에 추가(pushed)됩니다. 메서드가 정상적으로 반환되거나 메서드 호출 중에 잡히지 않은 예외가 throw 되면 frame이 제거(popped)됩니다. 예외 처리에 대한 자세한 내용은 아래의 예외 테이블 섹션을 참조하십시오.

각 프레임에는 다음이 포함됩니다.

- 지역 변수 배열(Local variable array)
- 반환 값(Return value)
- 피연산자 스택(Operand stack)
- 현재 메소드의 클래스에 대한 런타임 상수 풀에 대한 참조(Reference to runtime constant pool for class of the current method)

### Local Variables Array
local variables array은 this에 대한 참조, 모든 메서드 파라미터 및 기타 로컬로 정의된 변수를 포함하여 메서드 실행 중에 사용되는 모든 변수를 포함한다. class methods(i.e. static methods)의 경우, 방법 파라미터들은 0으로부터 시작하지만, 예를 들어 방법 제로 슬롯은 이를 위해 예약된다.

지역 변수는 다음과 같을 수 있습니다.
- boolean
- byte
- char
- long
- short
- int
- float
- double
- reference
- returnAddress

모든 유형은 두 개의 연속 슬롯을 사용하는 long 및 double을 제외하고 local variables array에서 단일 슬롯을 사용합니다. 이러한 유형은 너비가 두 배(32비트 대신 64비트)이기 때문입니다.

### Operand Stack
피연산자 stack은 범용 레지스터가 native CPU에서 사용되는 것과 유사한 방식으로 바이트 코드 명령어 실행 중에 사용됩니다. 대부분의 JVM 바이트 코드는 값을 생성하거나 소비하는 작업을 pushing, popping, duplicating, swapping, or executing operations 하여 피연산자 stack을 조작하는 데 시간을 보냅니다. 따라서 지역 변수 배열과 피연산자 stack 간에 값을 이동하는 명령은 바이트 코드에서 매우 자주 발생합니다. 예를 들어, 간단한 변수 초기화는 피연산자 stack과 상호 작용하는 2개의 바이트 코드를 생성합니다.

```java
int i;
```
다음 바이트 코드로 컴파일됩니다.
```java
0:	iconst_0	// 피연산자 스택의 맨 위로 0을 푸시합니다.
1:	istore_1	// 피연산자 스택의 맨 위에서 값을 팝하고 로컬 변수 1로 저장
```
local variables array, operand stack and run time constant pool간의 상호 작용을 설명하는 자세한 내용 은 아래의 클래스 파일 구조 섹션을 참조하십시오 .

### Dynamic Linking
각 프레임에는 런타임 상수 풀에 대한 참조가 포함됩니다. 참조는 해당 프레임에 대해 실행 중인 메서드의 클래스에 대한 상수 풀을 가리킵니다. 이 참조는 동적 연결을 지원하는 데 도움이 됩니다.

C/C++ 코드는 일반적으로 개체 파일로 컴파일된 다음 여러 개체 파일이 함께 연결되어 실행 파일이나 dll과 같은 사용 가능한 아티팩트를 생성합니다. 연결 단계에서 각 개체 파일의 기호 참조는 최종 실행 파일과 관련된 실제 메모리 주소로 대체됩니다. Java에서 이 연결 단계는 런타임에 동적으로 수행됩니다.

Java 클래스가 컴파일될 때 변수 및 메서드에 대한 모든 참조는 클래스의 상수 풀에 기호 참조로 저장됩니다. 기호 참조는 실제로 물리적 메모리 위치를 가리키는 참조가 아니라 논리적 참조입니다. JVM 구현은 기호 참조를 해결하는 시기를 선택할 수 있습니다. 이는 로드된 후 클래스 파일이 검증될 때 발생할 수 있으며, 대신에 이는 기호 참조가 처음으로 사용될 때 지연 또는 지연 해결이라고 하는 경우에 발생할 수 있습니다. 그러나 JVM은 각 참조가 처음 사용될 때 해결이 발생한 것처럼 동작해야 하며 이 시점에서 해결 오류를 발생시켜야 합니다. 바인딩은 직접 참조로 대체되는 기호 참조로 식별되는 필드, 메서드 또는 클래스의 프로세스입니다. 이것은 기호 참조가 완전히 대체되기 때문에 한 번만 발생합니다. 기호 참조가 아직 확인되지 않은 클래스를 참조하는 경우 이 클래스가 로드됩니다. 각 직접 참조는 변수 또는 메서드의 런타임 위치와 연결된 저장소 구조에 대한 오프셋으로 저장됩니다.

## Shared Between Threads
### Heap
힙은 런타임에 클래스 인스턴스와 배열을 할당하는 데 사용됩니다. 프레임이 생성된 후 크기가 변경되도록 설계되지 않았기 때문에 배열과 객체는 스택에 저장할 수 없습니다. 프레임은 힙의 개체 또는 배열을 가리키는 참조만 저장합니다. local variables array(in each frame)의 기본 변수 및 참조와 달리 객체는 항상 힙에 저장되므로 메서드가 종료될 때 제거되지 않습니다. 대신 개체는 가비지 수집기에 의해서만 제거됩니다.


- Young Generation
  - 종종 에덴과 생존자 사이에서 분리됨
- Old Generation (also called Tenured Generation)
- Permanent Generation

### Memory Management
Objects 및 Arrays는 명시적으로 할당 해제되지 않으며 대신 Garbage collector가 자동으로 회수합니다.

일반적으로 다음과 같이 작동합니다.
1. 새로운 객체와 배열이 young generation 에 생성됩니다.
2. Minor garbage collection은 young generation 에서 작동합니다. 아직 살아 있는 객체는 eden space에서 survivor space로 이동합니다.
3. 일반적으로 응용 프로그램 thread를 일시 중지하는 주요 garbage collection은 객체를 generation 간에 이동합니다. 아직 살아 있는 개체는 young generation에서 old(tenured) generation로 이동됩니다.
4. The permanent generation는 old generation가 수집될 때마다 수집됩니다. 둘 중 하나가 가득 차면 둘 다 수집됩니다.

### Non-Heap Memory
논리적으로 JVM 메커니즘의 일부로 간주되는 객체는 heap 에서 생성되지 않습니다.

The non-heap memory includes:

- **Permanent Generation** that contains
  - the method area
  - interned strings
- JIT 컴파일러에 의해 네이티브 코드로 컴파일된 메소드의 컴파일 및 저장에 사용되는 **Code Cache**

### Just In Time (JIT) Compilation
Java byte code는 해석되지만 JVM 호스트 CPU에서 네이티브 코드를 직접 실행하는 것만큼 빠르지는 않습니다. 성능을 향상시키기 위해 Oracle Hotspot VM은 정기적으로 실행되는 바이트 코드의 "hot" 영역을 찾아 native code로 컴파일합니다. 그런 다음 native code는 heap이 아닌 메모리의 code cache에 저장됩니다. 이러한 방식으로 Hotspot VM은 코드를 컴파일하는 데 걸리는 추가 시간과 해석된 코드를 실행하는 데 걸리는 추가 시간을 절충하는 가장 적절한 방법을 선택하려고 합니다.


### Method Area
Method Area 는 다음과 같은 클래스별 정보를 저장합니다.

- 클래스 로더 참조(Classloader Reference)
- 런타임 상수 풀(Run Time Constant Pool)
  - 숫자 상수(Numeric constants)
  - 필드 참조(Field references)
  - 방법 참조(Method References)
  - 속성(Attributes)
- 필드 데이터(Field data)
  - Per field
    - 이름(Name)
    - 유형(Type)
    - 수정자(Modifiers)
    - 속성(Attributes)
- 메소드 데이터(Method data)
  - Per method
    - 이름(Name)
    - 반환유형(Return Type)
    - 매개변수 유형(순서대로)(Parameter Types (in order))
    - 수정자(Modifiers)
    - 속성(Attributes)
- 메소드 코드(Method code)
  - Per method
    - 바이트코드(Bytecodes)
    - 피연산자 스택 크기(Operand stack size)
    - 지역 변수 크기(Local variable size)
    - 지역 변수 테이블(Local variable table)
    - 예외 테이블(Exception table)
      - 예외 처리기 별(Per exception handler)
        - 시작점(Start point)
        - 종점(End point)
        - 핸들러 코드의 PC 오프셋(PC offset for handler code)
        - catch되는 예외 클래스에 대한 상수 풀 인덱스(Constant pool index for exception class being caught)



### References
- [JVM Internals](https://blog.jamesdbloom.com/JVMInternals.html)