# [번역] JVM Internals
이 기사에서는 JVM(Java Virtual Machine)의 내부 아키텍처에 대해 설명합니다. 다음 다이어그램은 Java Virtual Machine Specification Java SE 7 Edition을 준수하는 일반적인 JVM 주요 내부 구성 요소를 보여줍니다.

![JVM_Internal_Architecture_small](./images/JVM_Internal_Architecture_small.png)

이 다이어그램에 표시된 구성 요소는 아래에서 두 섹션으로 각각 설명됩니다. 첫 번째 섹션 은 각 thread에 대해 생성되는 구성 요소를 다루고 두 번째 섹션 은 thread와 독립적으로 생성되는 구성 요소를 다룹니다.

- Threads
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
- Thread는 프로그램에서 실행되는 thread입니다.  
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
- 이 thread는 JVM이 안전 지점에 도달해야 하는 작업이 나타날 때까지 기다립니다. 이러한 작업이 별도의 thread에서 발생해야 하는 이유는 모두 JVM이 heap 수정이 발생할 수 없는 안전한 지점에 있어야 하기 때문입니다. 이 thread가 수행하는 작업 유형은 "세계 정지(stop-the-world)" 가비지 수집, thread stack 덤프, thread 일시 중단 및 편향된 잠금 취소입니다.

#### Periodic task thread
- 이 thread는 주기적인 작업의 실행을 예약하는데 사용되는 timer event(i.e. interrupts)를 담당합니다.

#### GC threads
- 이 thread는 JVM에서 발생하는 다양한 유형의 garbage collection activities를 지원합니다.

#### Signal dispatcher thread
- 이 thread는 JVM 프로세스로 전송된 신호를 수신하고 적절한 JVM 메소드를 호출하여 JVM 내부에서 처리합니다.

## Per Thread
각 실행 thread에는 다음 구성 요소가 있습니다.

### Program Counter(PC)
native가 아닌 경우 현재 명령어(또는 opcode)의 주소입니다. 현재 method이 기본이면 PC는 정의되지 않습니다. 모든 CPU에는 PC가 있으며 일반적으로 PC는 각 명령어 이후에 증가하므로 실행할 다음 명령어의 주소를 보유합니다. JVM은 PC를 사용하여 명령을 실행하는 위치를 추적합니다. PC는 실제로 메서드 영역의 메모리 주소를 가리킵니다.

### Stack
각 thread에는 해당 thread에서 실행되는 각 메서드에 대한 frame을 보유하는 자체 stack이 있습니다. stack은 LIFO(Last In First Out) 데이터 구조이므로 현재 실행 중인 메서드가 stack의 맨 위에 있습니다. 모든 메서드 호출에 대해 새 frame이 생성되고 stack 맨 위에 추가(pushed)됩니다. 메서드가 정상적으로 반환되거나 메서드 호출 중에 잡히지 않은 예외가 throw 되면 frame이 제거(popped)됩니다. stack은 frame 개체를 푸시하고 팝하는 것을 제외하고는 직접 조작되지 않으므로 frame 개체가 heap에 할당될 수 있으며 메모리가 인접할 필요가 없습니다.

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
local variables array은 this에 대한 참조, 모든 메서드 파라미터 및 기타 로컬로 정의된 변수를 포함하여 메서드 실행 중에 사용되는 모든 변수를 포함한다. class methods(i.e. static methods)의 경우, method 파라미터들은 0으로부터 시작하지만, 예를 들어 method 제로 슬롯은 `this`를 위해 예약된다.

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

Java 클래스가 컴파일될 때 변수 및 메서드에 대한 모든 참조는 클래스의 상수 풀에 기호 참조로 저장됩니다. 기호 참조는 실제로 물리적 메모리 위치를 가리키는 참조가 아니라 논리적 참조입니다. JVM 구현은 기호 참조를 해결하는 시기를 선택할 수 있습니다. `this`는 로드된 후 클래스 파일이 검증될 때 발생할 수 있으며, 대신에 이는 기호 참조가 처음으로 사용될 때 지연 또는 지연 해결이라고 하는 경우에 발생할 수 있습니다. 그러나 JVM은 각 참조가 처음 사용될 때 해결이 발생한 것처럼 동작해야 하며 이 시점에서 해결 오류를 발생시켜야 합니다. 바인딩은 직접 참조로 대체되는 기호 참조로 식별되는 필드, 메서드 또는 클래스의 프로세스입니다. 이것은 기호 참조가 완전히 대체되기 때문에 한 번만 발생합니다. 기호 참조가 아직 확인되지 않은 클래스를 참조하는 경우 this class가 로드됩니다. 각 직접 참조는 변수 또는 메서드의 런타임 위치와 연결된 저장소 구조에 대한 오프셋으로 저장됩니다.

## Shared Between Threads
### Heap
heap은 런타임에 클래스 인스턴스와 배열을 할당하는 데 사용됩니다. 프레임이 생성된 후 크기가 변경되도록 설계되지 않았기 때문에 배열과 객체는 스택에 저장할 수 없습니다. 프레임은 heap의 개체 또는 배열을 가리키는 참조만 저장합니다. local variables array(in each frame)의 기본 변수 및 참조와 달리 객체는 항상 heap에 저장되므로 메서드가 종료될 때 제거되지 않습니다. 대신 개체는 Garbage collector에 의해서만 제거됩니다.

Garbage collection을 지원하기 위해 heap은 세 섹션으로 나뉩니다.
- Young Generation
  - 종종 Eden and Survivor 사이에서 분리됨
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
Java byte code는 해석되지만 JVM 호스트 CPU에서 네이티브 코드를 직접 실행하는 것만큼 빠르지는 않습니다. 성능을 향상시키기 위해 Oracle Hotspot VM은 정기적으로 실행되는 바이트 코드의 "hot" 영역을 찾아 native code로 컴파일합니다. 그런 다음 native code는 heap이 아닌 메모리의 code cache에 저장됩니다. 이러한 방식으로 Hotspot VM은 코드를 컴파일하는 데 걸리는 추가 시간과 해석된 코드를 실행하는 데 걸리는 추가 시간을 절충하는 가장 적절한 method을 선택하려고 합니다.


### Method Area
Method Area 는 다음과 같은 클래스별 정보를 저장합니다.

- 클래스 로더 참조(Classloader Reference)
- 런타임 상수 풀(Run Time Constant Pool)
  - 숫자 상수(Numeric constants)
  - 필드 참조(Field references)
  - method 참조(Method References)
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

모든 thread는 동일한 method area을 공유하므로 method area 데이터에 대한 액세스 및 동적 연결 프로세스는 thread로 부터 안전해야 합니다. 두 thread가 아직 로드되지 않은 클래스의 필드 또는 메서드에 액세스하려고 시도하는 경우 한 번만 로드되어야 하며 두 thread 모두 로드될 때까지 실행을 계속해서는 안 됩니다.

### Class File Structure 
컴파일된 클래스 파일은 다음 구조로 구성됩니다.

```java
ClassFile {
    u4			magic;
    u2			minor_version;
    u2			major_version;
    u2			constant_pool_count;
    cp_info		contant_pool[constant_pool_count – 1];
    u2			access_flags;
    u2			this_class;
    u2			super_class;
    u2			interfaces_count;
    u2			interfaces[interfaces_count];
    u2			fields_count;
    field_info		fields[fields_count];
    u2			methods_count;
    method_info		methods[methods_count];
    u2			attributes_count;
    attribute_info	attributes[attributes_count];
}
```

| Value                                     |Desc.|
|-------------------------------------------|---------------------------------------------------------------------------------|
| `magic`, `minor_version`, `major_version` | 클래스 버전 및 이 클래스가 컴파일된 JDK 버전에 대한 정보를 지정합니다.                                      |
| `constant_pool`                           | symbol table과 유사하지만 여기에는 더 많은 데이터가 포함되어 있으며 아래에 자세히 설명되어 있습니다.                        |
| `access_flags`                            | 이 클래스에 대한 modifiers 목록을 제공합니다.                                                  |
| `this_class`                              | 이 클래스의 완전한 이름을 제공하는 `constant_pool`에 대한 index, 즉 `org/jamesdbloom/foo/Bar`          |
| `super_class`                             | `java/lang/Object`와 같은 수퍼 클래스에 대한 symbolic references를 제공하는 constant_pool에 대한 인덱스 |
| `interfaces`                              | 구현된 모든 인터페이스에 대한 symbolic references를 제공하는 `constant_pool`에 대한 인덱스 배열.            |
| `fields`                                  |각 필드에 대한 완전한 설명을 제공하는 `constant_pool`에 대한 인덱스 배열.|
| `methods`                                 |각 메서드 서명에 대한 완전한 설명을 제공하는 `constant_pool`에 대한 인덱스 배열, 메서드가 추상 또는 기본이 아닌 경우 바이트코드도 존재합니다.|
| `attributes`                              |`RetentionPolicy.CLASS` 또는 `RetentionPolicy.RUNTIME`이 있는 주석을 포함하여 클래스에 대한 추가 정보를 제공하는 다른 값의 배열|

`javap` 명령을 사용하여 컴파일된 Java 클래스의 바이트 코드를 볼 수 있습니다.

다음의 간단한 클래스를 컴파일하면:
```java
package org.jvminternals;

public class SimpleClass {

  public void sayHello() {
    System.out.println("Hello");
  }

}
```

그런 다음 실행하면 다음 출력을 얻습니다:

```shell
$ javap -v -p -s -sysinfo -constants classes/org/jvminternals/SimpleClass.class
```

```java
public class org.jvminternals.SimpleClass
  SourceFile: "SimpleClass.java"
  minor version: 0
  major version: 51
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#17         //  java/lang/Object."<init>":()V
   #2 = Fieldref           #18.#19        //  java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #20            //  "Hello"
   #4 = Methodref          #21.#22        //  java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #23            //  org/jvminternals/SimpleClass
   #6 = Class              #24            //  java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lorg/jvminternals/SimpleClass;
  #14 = Utf8               sayHello
  #15 = Utf8               SourceFile
  #16 = Utf8               SimpleClass.java
  #17 = NameAndType        #7:#8          //  "<init>":()V
  #18 = Class              #25            //  java/lang/System
  #19 = NameAndType        #26:#27        //  out:Ljava/io/PrintStream;
  #20 = Utf8               Hello
  #21 = Class              #28            //  java/io/PrintStream
  #22 = NameAndType        #29:#30        //  println:(Ljava/lang/String;)V
  #23 = Utf8               org/jvminternals/SimpleClass
  #24 = Utf8               java/lang/Object
  #25 = Utf8               java/lang/System
  #26 = Utf8               out
  #27 = Utf8               Ljava/io/PrintStream;
  #28 = Utf8               java/io/PrintStream
  #29 = Utf8               println
  #30 = Utf8               (Ljava/lang/String;)V
{
  public org.jvminternals.SimpleClass();
    Signature: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
        0: aload_0
        1: invokespecial #1    // Method java/lang/Object."<init>":()V
        4: return
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
          0      5      0    this   Lorg/jvminternals/SimpleClass;

  public void sayHello();
    Signature: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
        0: getstatic      #2    // Field java/lang/System.out:Ljava/io/PrintStream;
        3: ldc            #3    // String "Hello"
        5: invokevirtual  #4    // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        8: return
      LineNumberTable:
        line 6: 0
        line 7: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
          0      9      0    this   Lorg/jvminternals/SimpleClass;
}
```

- **Constant Pool** - 이것은 symbol table이 일반적으로 제공하는 것과 동일한 정보를 제공하며 아래에 더 자세히 설명되어 있습니다.
- **Methods** - 각각은 4개의 영역을 포함합니다:
  - signature and access flags
  - byte code
  - LineNumberTable - 이것은 디버거에 정보를 제공하여 어떤 라인이 바이트 코드 명령어에 해당하는지 나타냅니다. 예를 들어 Java 코드의 라인 6은 sayHello 메소드의 바이트 코드 0에 해당하고 라인 7은 바이트 코드 8에 해당합니다.
  - LocalVariableTable - 이것은 frame에 제공된 모든 지역 변수를 나열합니다. 두 예에서 유일한 지역 변수는 이것입니다.

이 클래스 파일에는 다음 바이트 코드 피연산자가 사용됩니다.

| Value             | Desc.                                                                                                                                                                                                                                                                                                                                                                                                                         |
|-------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `aload_0`         | 이 opcode는 `aload_<n>` 형식의 opcode 그룹 중 하나입니다. 그들은 모두 객체 참조를 피연산자 스택에 로드합니다. `<n>`은 액세스 중인 로컬 변수 배열의 위치를 참조하지만 0, 1, 2 또는 3일 수 있습니다. 객체 참조 `iload_<n>`, `lload_<n>`, `float_<n>` 및 `dload_<n>`이 아닌 값을 로드하기 위한 다른 유사한 opcode가 있습니다. 여기서 i는 `int`, l은 `long`, f는 `float` 및 `double`용인 d입니다. 인덱스가 3보다 높은 지역 변수는 `iload`, `lload`, `float`, `dload` 및` aload`를 사용하여 로드할 수 있습니다. 이러한 opcode는 모두 로드할 로컬 변수의 인덱스를 지정하는 단일 피연산자를 사용합니다. |
| `ldc`             | 이 opcode는 런타임 상수 풀에서 피연산자 스택으로 상수를 푸시하는 데 사용됩니다.                                                                                                                                                                                                                                                                                                                                                                              |
| `getstatic`       | 이 opcode는 런타임 상수 풀에 나열된 정적 필드의 정적 값을 피연산자 스택으로 푸시하는 데 사용됩니다.                                                                                                                                                                                                                                                                                                                                                                  |
| `invokespecial`, `invokevirtual`| 이러한 opcode는 `invokedynamic`, `invokeinterface`, `invokespecial`, `invokestatic`, `invokevirtual` 메소드를 호출하는 opcode 그룹에 있습니다. 이 클래스 파일에서 `invokespecial`과 `invokevirutal`은 둘 다 사용됩니다. 이들 간의 차이점은 `invokevirutal`이 객체의 클래스를 기반으로 메소드를 호출한다는 것입니다. `invokespecial` 명령어는 현재 클래스의 슈퍼클래스의 private 메서드와 메서드뿐만 아니라 인스턴스 초기화 메서드를 호출하는 데 사용됩니다.                                                                                         |
|`return`| 이 opcode는 opcode `ireturn`, `lreturn`, `freturn`, `dreturn`, `areturn` 및 `return` 그룹에 있습니다. 이러한 각 opcode는 i가 `int`용이고 l이 `long`용이고 f가 `float`용이고 d가 `double`용이고 a가 객체 참조용인 다른 유형을 반환하는 유형이 지정된 반환문입니다. 선행 유형 문자 `return`이 없는 opcode는 `void`만 반환합니다.                                                                                                                                                                           |

일반적인 바이트 코드에서 대부분의 피연산자는 다음과 같이 지역 변수, operand stack 및 run time constant pool과 상호 작용합니다.

생성자에는 두 개의 명령어가 있습니다. 먼저 `this`는 피연산자 스택에 푸시되고, 다음으로 슈퍼 클래스에 대한 생성자가 호출되어 `this` 값을 소비하여 피연산자 스택에서 꺼냅니다.

![bytecode_explanation_SimpleClass](./images/bytecode_explanation_SimpleClass.png)

- sayHello() 메서드는 위에서 자세히 설명한 것처럼 런타임 상수 풀을 사용하여 실제 참조에 대한 기호 참조를 해결해야 하므로 더 복잡합니다.
- 첫 번째 피연산자 `getstatic`은 시스템 클래스의 정적 필드에 대한 참조를 피연산자 스택으로 푸시하는 데 사용됩니다.
- 다음 피연산자 ldc는 문자열 "Hello"를 피연산자 스택에 푸시합니다.
- 마지막 피연산자 `invokevirtual`은 `System.out`의 `println` 메서드를 호출하여 피연산자 스택에서 `"Hello"`를 인수로 팝하고 현재 스레드에 대한 새 프레임을 만듭니다.

![bytecode_explanation_sayHello](./images/bytecode_explanation_sayHello.png)

### Classloader
JVM은 부트스트랩 클래스 로더를 사용하여 초기 클래스를 로드하여 시작합니다. 그런 다음 `public static void main(String[])`이 호출되기 전에 클래스가 연결되고 초기화됩니다. 이 메서드를 실행하면 필요에 따라 추가 클래스와 인터페이스의 `loading`, `linking` 및 `initalization` 차례로 실행됩니다.

**Loading**은 특정 이름을 가진 클래스 또는 인터페이스 유형을 나타내는 클래스 파일을 찾아 바이트 배열로 읽는 과정입니다.
다음으로 바이트가 구문 분석되어 클래스 개체를 나타내고 올바른 주 버전 및 부 버전이 있는지 확인합니다.
직접 수퍼클래스로 명명된 모든 클래스 또는 인터페이스도 로드됩니다. 이 작업이 완료되면 이진 표현에서 클래스 또는 인터페이스 개체가 생성됩니다.

**Linking**은 클래스 또는 인터페이스를 사용하여 유형과 해당 유형의 직접적인 수퍼클래스 및 수퍼인터페이스를 확인하고 준비하는 프로세스입니다. linking은 확인, 준비 및 선택적으로 해결의 세 단계로 구성됩니다.

- **Verifying**은 클래스 또는 인터페이스 표현이 구조적으로 정확하고 Java 프로그래밍 언어 및 JVM의 의미론적 요구사항을 준수하는지 확인하는 프로세스입니다.예를 들어 다음 검사가 수행됩니다.
  1. 일관되고 올바른 형식의 기호 테이블
  2. final methods / classes가 재정의되지 않음
  3. 메소드는 액세스 제어 키워드를 준수합니다
  4. 메소드에 올바른 수와 유형의 매개변수가 있습니다
  5. 바이트코드는 스택을 잘못 조작하지 않습니다
  6. 변수는 읽기 전에 초기화됩니다
  7. 변수는 올바른 유형의 값입니다

- 확인 단계에서 이러한 검사를 수행하면 런타임에 이러한 검사를 수행할 필요가 없습니다.
- 연결 중 확인은 클래스 로딩 속도를 늦추지만 바이트코드를 실행할 때 이러한 검사를 여러 번 수행할 필요가 없습니다.
- **Preparing**에는 정적 저장소 및 메서드 테이블과 같이 JVM에서 사용하는 모든 데이터 구조를 위한 메모리 할당이 포함됩니다. 정적 필드가 생성되고 기본값으로 초기화되지만 초기화의 일부로 발생하므로 이 단계에서 이니셜라이저나 코드가 실행되지 않습니다.
- **Resolving**은 참조된 클래스 또는 인터페이스를 로드하고 참조가 올바른지 확인하여 기호 참조를 확인하는 선택적 단계입니다. 이것이 이 시점에서 발생하지 않으면 바이트 코드 명령어에 의해 사용되기 직전까지 기호 참조의 해결이 지연될 수 있습니다.

**Initialization** 클래스 또는 인터페이스의 초기화는 클래스 또는 인터페이스 초기화 메소드 <clinit> 실행으로 구성됩니다.

![Class_Loading_Linking_Initializing](images/Class_Loading_Linking_Initializing.png)

JVM에는 역할이 다른 여러 클래스 로더가 있습니다. 각 클래스 로더는 최상위 클래스 로더인 부트스트랩 클래스 로더를 제외하고 상위 클래스 로더(로드한)에게 위임합니다.

**Bootstrap Classloader**는 JVM이 로드될 때 매우 일찍 인스턴스화되기 때문에 일반적으로 네이티브 코드로 구현됩니다. 부트스트랩 클래스 로더는 예를 들어 rt.jar를 포함하여 기본 Java API를 로드하는 역할을 합니다. 더 높은 신뢰 수준을 가진 부트 클래스 경로에 있는 클래스만 로드합니다. 결과적으로 일반 클래스에 대해 수행되는 많은 유효성 검사를 건너뜁니다.

**Extension Classloader**는 보안 확장 기능과 같은 표준 Java 확장 API에서 클래스를 로드합니다.

**System Classloader**는 클래스 경로에서 애플리케이션 클래스를 로드하는 기본 애플리케이션 클래스 로더입니다.

**User Defined Classloaders**는 대안으로 애플리케이션 클래스를 로드하는 데 사용할 수 있습니다. user defined classloader는 런타임에 클래스를 다시 로드하거나 Tomcat과 같은 웹 서버에서 일반적으로 요구하는 로드된 클래스의 서로 다른 그룹 간의 분리를 포함하여 여러 가지 특별한 이유로 사용됩니다.

![class_loader_hierarchy](images/class_loader_hierarchy.png)

### Faster Class Loading
클래스 데이터 공유(CDS)라는 기능은 버전 5.0부터 HotSpot JMV에 도입되었습니다. JVM 설치 프로세스 동안 설치 프로그램은 rt.jar과 같은 주요 JVM 클래스 세트를 메모리 매핑된 공유 아카이브로 로드합니다. CDS는 이러한 클래스를 로드하는 데 걸리는 시간을 줄여 JVM 시작 속도를 개선하고 이러한 클래스를 JVM의 서로 다른 인스턴스 간에 공유할 수 있도록 하여 메모리 공간을 줄입니다.

### Where Is The Method Area
Java Virtual Machine 사양 Java SE 7 Edition은 "메소드 영역이 논리적으로 heap의 일부이지만 간단한 구현에서는 가비지 수집 또는 압축을 선택하지 않을 수 있습니다."라고 분명히 명시합니다. Oracle JVM용 jconsole과 달리 메서드 영역(and code cache)은 heap이 아닌 것으로 표시됩니다. OpenJDK 코드는 CodeCache가 ObjectHeap에 대한 VM의 별도 필드임을 보여줍니다.

### Classloader Reference
로드된 모든 클래스에는 해당 클래스를 로드한 클래스 로더에 대한 참조가 포함됩니다. 차례로 클래스 로더는 로드한 모든 클래스에 대한 참조도 포함합니다.

### Run Time Constant Pool
JVM은 더 많은 데이터를 포함하지만 심볼 테이블과 유사한 런타임 데이터 구조인 유형별 상수 풀을 유지합니다. Java의 바이트 코드에는 데이터가 필요합니다. 종종 이 데이터는 너무 커서 바이트 코드에 직접 저장할 수 없습니다. 대신 상수 풀에 저장되고 바이트 코드에는 상수 풀에 대한 참조가 포함됩니다. 런타임 상수 풀은 위에서 설명한 대로 동적 연결에 사용됩니다.

다음을 포함하여 여러 유형의 데이터가 상수 풀에 저장됩니다.
- numeric literals
- string literals
- class references
- field references
- method references

For example the following code:
```java
Object foo = new Object();
```

다음과 같이 바이트 코드로 작성됩니다.
```java
0: 	new #2 		    // Class java/lang/Object
1:	dup
2:	invokespecial #3    // Method java/ lang/Object "<init>"( ) V
```
- The `new` opcode(피연산자 코드) 뒤에 #2 피연산자가 옵니다. 이 피연산자는 상수 풀에 대한 인덱스이므로 상수 풀의 두 번째 항목을 참조합니다.
- 두 번째 항목은 클래스 참조이며, 이 항목은 `// Class java/lang/Object` 값을 갖는 상수 UTF8 문자열로 클래스 이름을 포함하는 상수 풀의 다른 항목을 차례로 참조합니다.
- The `new` opcode는 클래스 인스턴스를 생성하고 해당 변수를 초기화합니다. 그런 다음 새 클래스 인스턴스에 대한 참조가 피연산자 스택에 추가됩니다.
- 그런 다음 `dup` opcode는 피연산자 스택에 상위 참조의 추가 복사본을 만들고 이를 피연산자 스택의 맨 위에 추가합니다.
- 마지막으로 인스턴스 초기화 메소드는 `invokespec`에 의해 라인 2에서 호출됩니다.
- 이 피연산자는 상수 풀에 대한 참조도 포함합니다. 초기화 메서드는 메서드에 대한 인수로 피연산자 풀에서 상위 참조를 사용(pop)합니다.
- 끝에는 생성 및 초기화된 새 개체에 대한 참조가 하나 있습니다.

다음의 간단한 클래스를 컴파일하면:
```java
package org.jvminternals;

public class SimpleClass {

  public void sayHello() {
    System.out.println("Hello");
  }

}
```
생성된 클래스 파일의 상수 풀은 다음과 같습니다.
```java
Constant pool:
   #1 = Methodref          #6.#17         //  java/lang/Object."<init>":()V
   #2 = Fieldref           #18.#19        //  java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #20            //  "Hello"
   #4 = Methodref          #21.#22        //  java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #23            //  org/jvminternals/SimpleClass
   #6 = Class              #24            //  java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lorg/jvminternals/SimpleClass;
  #14 = Utf8               sayHello
  #15 = Utf8               SourceFile
  #16 = Utf8               SimpleClass.java
  #17 = NameAndType        #7:#8          //  "<init>":()V
  #18 = Class              #25            //  java/lang/System
  #19 = NameAndType        #26:#27        //  out:Ljava/io/PrintStream;
  #20 = Utf8               Hello
  #21 = Class              #28            //  java/io/PrintStream
  #22 = NameAndType        #29:#30        //  println:(Ljava/lang/String;)V
  #23 = Utf8               org/jvminternals/SimpleClass
  #24 = Utf8               java/lang/Object
  #25 = Utf8               java/lang/System
  #26 = Utf8               out
  #27 = Utf8               Ljava/io/PrintStream;
  #28 = Utf8               java/io/PrintStream
  #29 = Utf8               println
  #30 = Utf8               (Ljava/lang/String;)V
```

상수 풀에는 다음 유형이 포함됩니다.

|Value|Desc|
|---|---|
|`Integer`|4바이트 정수 상수|
|`Long`|8바이트 길이 상수|
|`Float`|4바이트 부동 상수|
|`Double`|8바이트 이중 상수|
|`String`|실제 바이트를 포함하는 상수 풀의 다른 Utf8 항목을 가리키는 문자열 상수|
|`Utf8`|Utf8로 인코딩된 문자 시퀀스를 나타내는 바이트 스트림|
|`Class`|내부 JVM 형식의 정규화된 클래스 이름을 포함하는 상수 풀의 다른 Utf8 항목을 가리키는 클래스 상수(동적 연결 프로세스에서 사용됨)|
|`NameAndType`|상수 풀의 다른 항목을 각각 가리키는 콜론으로 구분된 값 쌍입니다. 콜론 앞의 첫 번째 값은 메서드 또는 필드 이름인 Utf8 문자열 항목을 가리킵니다. 두 번째 값은 유형을 나타내는 Utf8 항목을 가리킵니다. 필드의 경우 정규화된 클래스 이름이고 메서드의 경우 매개변수당 하나씩 정규화된 클래스 이름의 목록입니다.|
|`Fieldref`, `Methodref`, `InterfaceMethodref`|상수 풀의 다른 항목을 각각 가리키는 점으로 구분된 값 쌍입니다. 첫 번째 값(점 앞)은 클래스 항목을 가리킵니다. 두 번째 값은 NameAndType 항목을 가리킵니다.|

### Exception Table
예외 테이블은 다음과 같은 예외 처리기별 정보를 저장합니다.
- Start point
- End point
- PC offset for handler code
- catch되는 예외 클래스에 대한 상수 풀 인덱스

메서드가 try-catch 또는 try-finally 예외 처리기를 정의한 경우 예외 테이블이 생성됩니다. 여기에는 처리기가 적용되는 범위, 처리 중인 예외 유형 및 처리기 코드가 있는 위치를 포함하여 각 예외 처리기 또는 finally 블록에 대한 정보가 포함됩니다.

예외가 발생하면 JVM은 현재 메소드에서 일치하는 핸들러를 찾고, 아무 것도 발견되지 않으면 메소드가 갑자기 현재 스택 프레임을 팝핑하고 호출 메소드(새로운 현재 프레임)에서 예외가 다시 발생합니다. 모든 프레임이 팝되기 전에 예외 처리기가 발견되지 않으면 스레드가 종료됩니다. 예를 들어 스레드가 기본 스레드인 경우와 같이 마지막 비 데몬 스레드에서 예외가 발생하면 JVM 자체가 종료될 수도 있습니다.

마지막으로 예외 처리기는 모든 유형의 예외와 일치하므로 예외가 throw될 때마다 항상 실행됩니다. 예외가 발생하지 않은 경우 finally 블록은 메서드 끝에서 여전히 실행되며 이는 return 문이 실행되기 직전에 finally 핸들러 코드로 점프하여 달성됩니다.

### Symbol Table
유형별 런타임 상수 풀 외에도 Hotspot JVM에는 영구 생성에 보유되는 기호 테이블이 있습니다. 기호 테이블은 기호에 대한 기호 포인터(예: Hashtable<Symbol*, Symbol>)를 매핑하는 Hashtable이며 각 클래스의 런타임 상수 풀에 포함된 기호를 포함하여 모든 기호에 대한 포인터를 포함합니다.

참조 카운팅은 기호 테이블에서 기호가 제거되는 시기를 제어하는 데 사용됩니다. 예를 들어 클래스가 언로드되면 런타임 상수 풀에 있는 모든 기호의 참조 카운트가 감소합니다. 심볼 테이블에 있는 심볼의 참조 카운트가 0이 되면 심볼 테이블은 심볼이 더 이상 참조되지 않고 심볼이 심볼 테이블에서 언로드됨을 알게 됩니다. 기호 테이블과 문자열 테이블(아래 참조) 모두 효율성을 높이고 각 항목이 한 번만 표시되도록 모든 항목이 정규화된 형식으로 유지됩니다.

### Interned Strings (String Table)
Java 언어 사양에서는 동일한 유니코드 코드 포인트 시퀀스를 포함하는 동일한 문자열 리터럴이 동일한 문자열 인스턴스를 참조해야 한다고 요구합니다. 또한 String의 인스턴스에서 String.intern()이 호출되면 문자열이 리터럴인 경우 참조 반환과 동일한 참조가 반환되어야 합니다. 따라서 다음 사항이 적용됩니다.
```java
("j" + "v" + "m").intern() == "jvm"
```
Hotspot JVM에서 인턴된 문자열은 문자열 테이블에 보관되며, 이는 Hashtable 매핑 개체 포인터를 기호(예: Hashtable<oop, Symbol>)에 매핑하고 영구 생성으로 유지됩니다. 기호 테이블(위 참조)과 문자열 테이블 모두의 경우 효율성을 높이고 각 항목이 한 번만 표시되도록 모든 항목이 정규화된 형식으로 유지됩니다.

문자열 리터럴은 컴파일러에 의해 자동으로 수용되고 클래스가 로드될 때 기호 테이블에 추가됩니다. 또한 String 클래스의 인스턴스는 String.intern()을 호출하여 명시적으로 인턴할 수 있습니다. String.intern()이 호출될 때 기호 테이블에 이미 문자열이 포함되어 있으면 이에 대한 참조가 반환되고, 그렇지 않은 경우 문자열이 문자열 테이블에 추가되고 해당 참조가 반환됩니다.


### Reference
- [JVM Internals](https://blog.jamesdbloom.com/JVMInternals.html)