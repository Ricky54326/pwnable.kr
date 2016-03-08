# pwnable.kr
pwnable.kr writeups as I go

## fd
"Mommy! What is a file descriptor in Linux?" 
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
        if(argc<2){
                printf("pass argv[1] a number\n");
                return 0;
        }
        int fd = atoi( argv[1] ) - 0x1234;
        int len = 0;
        len = read(fd, buf, 32);
        if(!strcmp("LETMEWIN\n", buf)){
                printf("good job :)\n");
                system("/bin/cat flag");
                exit(0);
        }
        printf("learn about Linux file IO\n");
        return 0;

}
```

This simple program takes an integer passed in, subtracts 0x1234 from it (4660 decimal), and then tries to read 32 bytes from the file descriptor associated with the arg-0x1234. The easiest way I could think to solve this is just to make fd-4663 == 0 (`fd == argv[1]` = 4660), which would result in the program reading from stdin. You can then clearly see it checking for `LETMEWIN\n`, so I entered this, and: 

```shell
fd@ubuntu:~$ ./fd 4660
LETMEWIN
good job :)
mommy! I think I know what a file descriptor is!!

```


## random
"Daddy, teach me how to use random value in programming!"
```c
#include <stdio.h>

int main(){
        unsigned int random;
        random = rand();        // random value!

        unsigned int key=0;
        scanf("%d", &key);

        if( (key ^ random) == 0xdeadbeef ){
                printf("Good!\n");
                system("/bin/cat flag");
                return 0;
        }

        printf("Wrong, maybe you should try 2^32 cases.\n");
        return 0;
}

```
This program, I can immediately tell (recalling from my noob C days) does not seed the RNG at all. Thus, `random()` returns the same result every single time. 

To find out what that number is on this system, I copy random.c (the target) to /tmp, and edit it to dislpay the result of rand(): 
replacing:
```c
        random = rand();        // random value!
```

with
```c
        random = rand();        // random value!
        printf("rand: %d\n", random);
```
 then compiled using `gcc random.c`, followed by running the program with: `./a.out`. Receiving: `rand: 1804289383`. (hex: 0x6b8b4567)
 
 I head back to ~ (`cd ~`), then run `./random`. 
 There's one more catch, though. The password has to match 1804289383 when XOR'd with `0xdeadbeef`. To undo an XOR, you can simply XOR two of the other terms, like so:
 
 `Pass ^ 0xDEADBEEF == 0xDEADBEEF ^ 6b8b4567(our hex'd version of the rand() result)`
 And thus,
 
 `Pass = 0xdeadbeef ^ 0x6b8b4567 = 0xb526fb88` (using wolframalpha)
 
 Lastly, you'll need to convert that result back to decimal (3039230856) (as that's how integers are passed around as strings), and thus:
 ```shell
 random@ubuntu:~$ ./random
3039230856
Good!
Mommy, I thought libc random is unpredictable...

 ```
 


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


