# pwnable.kr
pwnable.kr writeups as I go



## mistake
The hint given is "hint: operator priority". Here's the exploitable .c file (mistake.c) from the target:

```c
#include <stdio.h>
#include <fcntl.h>

#define PW_LEN 10
#define XORKEY 1

void xor(char* s, int len){
	int i;
	for(i=0; i<len; i++){
		s[i] ^= XORKEY;
	}
}

int main(int argc, char* argv[]){
	
	int fd;
	if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
		printf("can't open password %d\n", fd);
		return 0;
	}

	printf("do not bruteforce...\n");
	sleep(time(0)%20);

	char pw_buf[PW_LEN+1];
	int len;
	if(!(len=read(fd,pw_buf,PW_LEN) > 0)){
		printf("read error\n");
		close(fd);
		return 0;		
	}

	char pw_buf2[PW_LEN+1];
	printf("input password : ");
	scanf("%10s", pw_buf2);

	// xor your input
	xor(pw_buf2, 10);

	if(!strncmp(pw_buf, pw_buf2, PW_LEN)){
		printf("Password OK\n");
		system("/bin/cat flag\n");
	}
	else{
		printf("Wrong Password\n");
	}

	close(fd);
	return 0;
}
```

The mistake(s), which may or may not be obvious, are:
```c 
if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0)
``` 
and
```c 
if(!(len=read(fd,pw_buf,PW_LEN) > 0))
```

Because the first one will (if it can open the file) always assign 0 as the fd, and then the second line will open fd(0) for reading, rather than the actual file. 

Therefore:
```c 
if(!strncmp(pw_buf, pw_buf2, PW_LEN))
```
is actually checking against stdin twice rather than stdin vs the password file. However, there's a catch.
The second input (pw_buf) has each byte XOR'd by 1. So you can't just enter AAAAAAAAAA and AAAAAAAAAA as the passwords, as the second one will get XOR'd by 1 in each block.

I decided to use I (ASCII 73) for the "password", and H (ASCII 72) for the inputted password. Therefore my H string will get XOR'd by 1 at each character, essentially turning it into a string of Is. This will match, and thus:

```
mistake@ubuntu:~$ ./mistake
do not bruteforce...
IIIIIIIIII
input password : HHHHHHHHHH
Password OK
Mommy, the operator priority always confuses me :(
```


