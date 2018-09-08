---

layout : single
title : "Reversecore Issue chap 16 - Base Relocation Table"
categories : [Reversing]
tags : [reversecore]
comment : true

---

### '리버싱 핵심 원리'의 내용 및 이슈들과 해결책을 다룹니다.

---

<br/>


PE 파일의 재배치(Relocation)과정에 사용되는 Base Relocation Table의 구조와 동작 원리에 대해서 알아보자.


---

### PE 재배치

PE 파일(EXE/DLL/SYS)이 프로세스 가상 메모리에 로딩될때 PE헤더의 ImageBase 주소에 로딩된다. DLL(SYS)파일의 경우 ImageBase 위치에 이미 다른 DLL(SYS) 파일이 로딩되어 있다면 다른 비어 있는 주소 공간에 로딩된다. 이것을 PE 파일 재배치라고 한다. 즉 PE 재배치란 PE 파일이 ImageBase에 로딩되지 못하고 다른 주소에 로딩될 때 수행되는 일련의 작업들을 의미한다.


> SDK(Software Development Kit) 또는 Visual C++로 PE 파일을 생성하면 기본적으로 EXE의 경우 ImageBase = 00400000, DLL의 경우 ImageBase = 10000000으로 설정된다. 또한 DDK(Driver Development Kit)로 생성된 SYS 파일은 기본적으로 ImageBase = 10000으로 설정된다.

- DLL

![16-1](https://user-images.githubusercontent.com/26838115/45201219-68ad2a00-b2af-11e8-9f7e-3a47fcf86b1e.png)

B.dll이 A.dll과 같은 주소에 로딩을 시작하면, 재배치가 이루어진다.


- EXE


예전에는 프로세스가 생성될 때 EXE 파일이 가장 먼저 메모리에 로딩되기 때문에 EXE에서는 재배치를 고려할 필요가 없었다. 그러나 Vista 이후에는 보안 강화를 위해 **ASLR(Address Space Layout Ramdomizatoin)** 기능이 추가되었다. 이는 EXE 파일이 실행될 때마다 랜덤한 주소에 로딩하는 것이다. 


> ASLR 기능은 DLL/SYS 파일에도 적용되는 개념이다. MS 에서는 각 OS의 주요 시스템 DLL들에 대해 버전별로 각자 고유한 ImageBase를 부여하였다. 따라서 한 시스템에서 nernel32.dll, user32.dll 등은 자신만의 고유 ImageBase에 로딩되기 때문에 실제로 시스템 DLL들끼리는 Relocation이 발생할 일이 없습니다.

---

### PE 재배치 발생시 수행되는 작업

![16-2](https://user-images.githubusercontent.com/26838115/45201592-13721800-b2b1-11e8-9de9-673d3b2bfcc7.png)

PE viewer를 이용해 보면, notepad.exe의 ImageBase는 01000000임을 알 수 있다.

![16-3](https://user-images.githubusercontent.com/26838115/45201734-64820c00-b2b1-11e8-9db5-b3e4766bbc43.png)

OllyDbg를 통해 코드 영역을 보면, 선택된 instruction에서 프로세스 메모리 주소가 하드코딩되 었는 것을 볼 수 있다. OllyDbg에서 notepad.exe를 재실행할 때 마다 위 그림에 표시된 주소 값은 로딩 주소에 맞게 매번 변경된다. 이렇게 프로그램에 하드코딩된 메모리 주소를 현재 로딩된 주소에 맞게 변경해주는 작업이 바로 PE 재배치이다!

ImageBase 주소에 로딩되지 않는 경우에 이 PE 재배치 작업이 없다면 프로그램은 정상적으로 실행될 수 없다('잘못된 메모리 주소 참조 에러'에 의해서 비정상 종료된다).

---

### PE 재배치 동작 원리

PE 재배치 작업의 기본 동작 원리는 다음과 같다.

1. 프로그램에서 하드딩된 주소 위치를 찾는다.
2. 값을 읽은 후 ImageBase만큼 뺀다(VA -> RVA)
3. 실제 로딩 주소를 더한다(RVA -> VA)

여기서 핵심은 하드코딩된 주소 위치를 찾는 것인데, 이를 위해 PE 파일 내부에 Relocation Table이라고 하는 하드코딩 주소들의 옵셋(위치)을 모아 놓은 목록이 존재한다(Relocation Table로 찾아가는 방법은 PE 헤더의 Base Relocation Table 항목을 따라가는 것이다).

- Base Relocation Table

Base Relocation Table 주소는 PE 헤더에서 DataDirectory 배열의 여섯 번째 항목에 들어있다(배열 Index는 5).

IMAGE_NT_HEADERS \ IMAGE_OPTIONAL_HEADER \ IMAGE_DATA_DIRECTORY[5]

![16-4](https://user-images.githubusercontent.com/26838115/45202555-54b7f700-b2b4-11e8-8c31-fa2b0123dbfd.png)




























<br/>

---



### Issue #1

- 해결해야 할 이슈 : 
