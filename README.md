```c
 #include <stdio.h>
  2 #include <string.h>
  3 #include <stdlib.h>
  4 #include <unistd.h>
  5 #include <sys/stat.h>
  6 #include <dirent.h>
  7 #include <pwd.h>
  8 #include <grp.h>
  9 #include <time.h>
 10 #include <sys/utsname.h>
 11 #include <sys/stat.h>
 12 #include <sys/types.h>
 13 #include <errno.h>
 14 #include <fcntl.h>
 15 #include <utime.h>
 16 #include <sys/wait.h>
 17
 18 // get_argv_optv() 함수에서 상세히 설명함
 19 #define SZ_STR_BUF 256
 20 #define SZ_FILE_BUF 1024
 21 char *cmd;
 22 char *argv[100];
 23 char *optv[10];
 24 int  argc, optc;
 25 char cur_work_dir[SZ_STR_BUF];// 현재 작업 디렉토리 이름을 저장하는 버퍼
 26
 27 //***************************************************************************
 28 //  매크로 함수
 29 //***************************************************************************
 30 // 여러 문자들을 하나의 문장처럼 사용하기 위한 정의문
 31 // #define으로 함수처럼 작성한 정의 문장
 32 // 매크로 함수는 진짜 함수가 아니라, 그 매크로 함수를 호출(사용)한 곳에
 33 //   매크로 정의(do { ... } while(0))가 컴파일 때 삽입됨.
 34 // 일반 상수가 교체되는 것과 동일함.
 35
 36 // #define 문장은 반드시 한 줄에 다 작성해야 함
 37 //   따라서 아래 각 줄의 끝에 \는 여러 줄을 한 줄로 만들어 주는 역할을 함
 38 //   아래의 \ 뒤에서 반드시 바로 엔터를 쳐야 하고,
 39 //   스페이스가 있으면 에러 발생함
 40 //
 41 // 아래 매크로 함수에서 return은 매크로 함수 자체에서 리턴이 아니라,
 42 //   그 매크로를 호출한 곳의 해당 함수에서 리턴한다는 의미임.
 43 //   즉, cp() 함수에서 PRINT_ERR_RET()을 호출했다면 아래 return은
 44 //   cp() 함수에서 리턴한다는 의미임.
 45 //--------------------------------------------------------------------------
 46
 47 // 에러 원인을 화면에 출력하고 이 매크로를 사용하는 해당 함수에서 리턴함
 48 /* cmd가 "ls"라면 "ls: 에러원인" 형태로 출력됨 */
 49
 50 #define PRINT_ERR_RET() \
 51 do { \
 52     perror(cmd); \
 53     return; \
 54 } while (0)
 55
 56 #define EQUAL(_s1, _s2)     (strcmp(_s1, _s2) == 0) // 두개 문자열 같으면 true
 57 #define NOT_EQUAL(_s1, _s2) (strcmp(_s1, _s2) != 0) // 두개 문자열 다르면 true
 58
 59 // 명령어 사용법을 출력함
 60 // help()와 proc_cmd()에서 호출됨
 61 static void
 62 print_usage(char *msg, char *cmd, char *opt, char *arg)
 63 {
 64     // proc_cmd()에서 msg로 "사용법: "을,
 65     // help()    에서 msg로 "    "  을 넘겨 줌
 66     printf("%s%s", msg, cmd);   // "사용법: ls"
 67     if (NOT_EQUAL(opt, ""))     // 옵션이 있으면
 68         printf("  %s", opt);    // " -l"
 69     printf("  %s\n", arg);      // " [디렉토리 이름]"
 70     // 최종 출력 예) "사용법: ls  -l  [디렉토리 이름]\n"
 71 }
 72
 73 //***************************************************************************
 74 //  명령어 옵션 및 인자 개수가 정확한지 체크하는 작업을 수행함
 75 //***************************************************************************
 76
 77 // [명령어 인자의 개수]가 정확한지 체크함
 78 // argc: 사용자가 입력한 명령어 인자의 수
 79 // count: 그 명령어가 필요로 하는 인자의 수
 80 // 리턴값: 인자개수가 정확하면 0, 틀렸으면 -1
 81 static int
 82 check_arg(int count)
 83 {
 84     if(count<0){
 85         count= -count;
 86         if(argc<=count)
 87             return(0);
 88     }
 89     if(argc == count)
 90         return(0);
 91     if(argc>count)
 92         // printf("Too many arguments. \n");
 93         printf("불필요한 명령어 인자가 있습니다.\n");
 94     else
 95         printf("명령어 인자의 수가 부족합니다.\n");
 96     return(-1);
 97 }
 98
 99 // 명령어에서 정확한 [옵션]이 주어졌는지 체크함
100 // optc: 사용자가 입력한 옵션 개수
101 // optv[i]: 사용자가 입력한 옵션 문자열
102 // opt: 그 명령어가 필요로 하는 옵션 문자열
103 // 리턴값: 옵션이 없으면 0, 있으면 1, 옵션이 틀렸으면 -1
104 static int
105 check_opt(char *opt)
106 {
107     int i,err =0;
108
109     for(i=0;i<optc; ++i){
110         if(NOT_EQUAL(opt,optv[i]))  {
111             printf("지원되지 않는 명령어 옵션(%s)입니다.\n",optv[i]);
112             err=-1;
113         }
114     }
115     return(err);
116 }
117
118 //***************************************************************************
119 //  get_argv_optv(): 명령어 한줄을 명령어,옵션,명령인자를 각 단어별로 분리
120 //***************************************************************************
121 //
122 // 전역변수
123 // char *cmd;       // 명령어 문자열
124 // char *argv[100]; // 명령어 인자 문자열들 시작주소 저장
125 // char *optv[10];  // 명령어 옵션 문자열들 시작주소 저장
126 // int optc; // 옵션 토큰 수, 즉, optv[]에 저장된 원소의 개수
127 // int argc; // 명령어와 옵션을 제외한 명령어 인자의 수
128 //           // 즉, argv[]에 저장된 원소의 개수
129
130 // 입력된 명령어 행 전체를 저장하고 있는 cmd_line[]에서
131 // 각 토큰(단어)을 별개의 문자열로 자른 후, 그 토큰의 시작 주소를
132 // 명령어일 경우 cmd에, 옵션일 경우 optv[]에,
133 // 명령어 인자인 경우 argv[]에 각각 순서적으로 저장한다.
134
135 // 예제: get_argv_optv(cmd_line)를 호출하여 리턴한 후의 각 변수의 값
136 //
137 // cmd_line[]에 "ls -l pr4" 가 저장되어 있을 경우
138 // cmd -> "ls"
139 // optc = 1, optv[0] -> "-l";
140 // argc = 1, argv[0] -> "pr4";
141
142 // cmd_line[]에 "ln -s f1 f2" 가 저장되어 있을 경우
143 // cmd -> "ln"
144 // optc = 1, optv[0] -> "-s";
145 // argc = 2, argv[0] -> "f1", argv[1] -> "f1";
146
147 // cmd_line[]에 "ln f1 f2" 가 저장되어 있을 경우
148 // cmd -> "ln"
149 // optc = 0;
150 // argc = 2, argv[0] -> "f1", argv[1] -> "f2";
151
152 // cmd_line[]에 "pwd" 가 저장되어 있을 경우
153 // cmd -> "pwd"
154 // optc = 0;
155 // argc = 0;
156
157 // cmd, optv[], argv[]에 저장되는 것은 각 문자열의 시작주소,
158 // 즉, 각 문자열의 첫 글자의 주소가 저장됨
159 // 따라서 각 문자열은 여전히 cmd_line[]에 저장되어 있음
160
161 static char *
162 get_argv_optv(char *cmd_line)
163 {
164     //  사용자가 키보드에서 입력한 명령어 한 줄이 cmd_line[]에 저장되어 있음
165     //  cmd_line[]에 "ln -s file1 file2"가 저장되어 있다고 가정하자.
166     //
167     //  strtok()의 두번째 인자인 " \t\n"는 토큰(단어)을 구분하는 구분자임.
168     //  즉, 스페이스 ' ', 탭 '\t', 엔터 '\n' 글자가 나오면 토큰을 자름.
169     //  strtok() 함수는 cmd_line[]에 있는 문자열(전체가 하나의 문자열)에서
170     //  첫 단어 "ln"를 찾아 단어의 끝에 null문자('\0')를 삽입해
171     //  단어를 하나의 문자열로 만들고(단어를 자른다고 표현함),
172     //  그 문자열의 첫 글자 주소를 리턴함.
173     //  첫 단어를 잘라 내기 위해 strtok() 호출 시 첫 인자로 cmd_line을 주고,
174     //  다음에 또 호출할 땐 첫 인자로 NULL을 주는데,
175     //  이 경우 앞에서 처리한 단어의 그 다음 단어를 찾아 자른다.
176     //  만약 함수 호출 시 더 이상 처리할 단어가 없다면 NULL을 리턴함.
177     //
178     //  결국 함수 호출 [후]에는 cmd_line[]가 "ln\0-s\0file1\0file2"로 변함
179     //  '\0'는 null문자(한 글자임)를 표현한 것이고,
180     //  메모리에는 ASCI 코드 0(정수값)이 한 byte로 저장되어 있음.
181     //  모든 문자열 끝에는 이 문자가 있음(함수들이 자동으로 삽입해 줌)
182     //  즉, cmd_line[]에는 네개의 문자열이 저장되어 있고,
183     //  각 문자열의 시작 주소는 optv[]와 argv[]에 분리되어 저장되어 있음
184     char *tok;
185
186     argc =optc=0;
187     if ((cmd = strtok(cmd_line, " \t\n")) == NULL)
188         return (NULL);
189
190     for ( ; (tok = strtok(NULL, " \t\n")) != NULL; ) {
191         if (tok[0] == '-')
192             optv[optc++] = tok;
193         else
194             argv[argc++] = tok;
195     }
196     return(cmd);
197 }
198 //***************************************************************************
199 // ls() 함수에서 호출되는 함수들 시작
200 //
201 // "ls -l" 옵션 준 경우 하나의 파일에 대한 상세정보 출력하기
202 static void
203 print_attr(char *path, char *fn)
204 {
205     struct passwd *pwp;
206     struct group *grp;
207     struct stat st_buf;
208     char full_path[SZ_STR_BUF], buf[SZ_STR_BUF], c;
209     char time_buf[13];
210     struct tm *tmp;
211
212     sprintf(full_path, "%s/%s", path, fn);
213     if (lstat(full_path, &st_buf) < 0)
214         PRINT_ERR_RET();
215     if      (S_ISREG(st_buf.st_mode))   c = '-';
216     else if (S_ISDIR(st_buf.st_mode))   c = 'd';
217     else if (S_ISCHR(st_buf.st_mode))   c = 'c';
218     else if (S_ISBLK(st_buf.st_mode))   c = 'b';
219     else if (S_ISFIFO(st_buf.st_mode))  c = 'p';
220     else if (S_ISLNK(st_buf.st_mode))   c = 'l';
221     else if (S_ISSOCK(st_buf.st_mode))  c = 's';
222     buf[0] = c;
223     buf[1] = (st_buf.st_mode & S_IRUSR)? 'r': '-';
224     buf[2] = (st_buf.st_mode & S_IWUSR)? 'w': '-';
225     buf[3] = (st_buf.st_mode & S_IXUSR)? 'x': '-';
226     buf[4] = (st_buf.st_mode & S_IRUSR)? 'r': '-';
227     buf[5] = (st_buf.st_mode & S_IWGRP)? 'w': '-';
228     buf[6] = (st_buf.st_mode & S_IXGRP)? 'x': '-';
229     buf[7] = (st_buf.st_mode & S_IROTH)? 'r': '-';
230     buf[8] = (st_buf.st_mode & S_IWOTH)? 'w': '-';
231     buf[9] = (st_buf.st_mode & S_IXOTH)? 'x': '-';
232     buf[10] = '\0';
233     pwp = getpwuid(st_buf.st_uid);
234     grp = getgrgid(st_buf.st_gid);
235     tmp = localtime(&st_buf.st_mtime);
236     strftime(time_buf, 13, "%b %d %H:%M", tmp);
237     // 아래의 " " 안의 l은 영문 소문자 L임; 모든 % 개수 정확히 입력할 것
238     sprintf(buf+10, " %3ld %-8s %-8s %8ld %s %s",
239                     st_buf.st_nlink, pwp->pw_name, grp->gr_name,
240                     st_buf.st_size, time_buf, fn);
241     if (S_ISLNK(st_buf.st_mode)) {
242             int len, bytes;
243             strcat(buf, " -> ");
244             len = strlen(buf);
245             bytes = readlink(full_path, buf+len, SZ_STR_BUF-len-1);
246             buf[len+bytes] = '\0';
247     }
248     printf("%s\n", buf);
249 }
250 // "ls -l" 옵션 준 경우: 디렉토리의 모든 파일에 대해 상세정보 출력하기
251 static void
252 print_detail(DIR *dp, char *path)
253 {
254     struct dirent *dirp;
255     while ((dirp = readdir(dp)) != NULL)
256     print_attr(path, dirp->d_name);
257 }
258 // "ls"만 한 경우: 가장 긴 파일이름의 길이를 계산한 후
259 //이를 기준으로 한 줄에 출력할 수 있는 열(column)의 개수를 결정함
260 static void
261 get_max_name_len(DIR *dp, int *p_max_name_len, int *p_num_per_line)
262 {
263     struct dirent *dirp;
264     int max_name_len = 0; // 가장 긴 파일이름 길이
265     // 모든 파일 이름을 읽어 내 가장 긴 이름의 길이를 결정함
266     while ((dirp = readdir(dp)) != NULL) {
267     int name_len = strlen(dirp->d_name);
268     if (name_len > max_name_len)
269     max_name_len = name_len;
270     }
271     // 디렉토리 읽는 위치 처음으로 되돌리기
272     rewinddir(dp);
273     // 가장 긴 파일이름 + 이름 뒤의 여유 공간
274     max_name_len += 4;
275     // 한 줄에 출력할 파일이름의 개수 결정
276     *p_num_per_line = 80 / max_name_len;
277     *p_max_name_len = max_name_len;
278 }
279 // "ls"만 한 경우: 디렉토리의 모든 파일에 대해 이름만 출력하기
280 static void
281 print_name(DIR *dp)
282 {
283     struct dirent *dirp;
284     int max_name_len, num_per_line, cnt = 0;
285     // max_name_len: 가장 긴 파일이름 길이 + 4(이름 뒤의 여유 공간)
286     // num_per_line: 한 줄에 출력할 수 있는 총 파일이름의 개수
287     get_max_name_len(dp, &max_name_len, &num_per_line);
288     while ((dirp = readdir(dp)) != NULL) {
289         printf("%-*s", max_name_len, dirp->d_name);
290         // cnt: 현재까지 한 줄에 출력한 파일이름 개수
291         // 한 줄에 출력할 수 있는 파일이름의 개수만큼
292         // 이미 출력 했으면 줄 바꾸기
293         if ((++cnt % num_per_line) == 0)
294             printf("\n");
295     }
296     // 마지막 줄에 줄 바꾸기 문자 출력
297     // 앞에서 이미 한 줄에 출력할 수 있는 파일이름의 개수만큼 이미 출력하여
298     // 줄 바꾸기를 했으면 또 다시 줄 바꾸기 하지 않음
299     if ((cnt % num_per_line) != 0)
300         printf("\n");
301 }
302 // ls() 함수에서 호출되는 함수들 끝
303 //***************************************************************************
304
305 //***************************************************************************
306 //
307 //  여기서부터 각 명령어 구현 시작
308 //
309 //
310
311 // 파일을 다른 이름으로 복사하는 명령어
312 // 사용법: cp  원본파일이름  복사된파일이름
313 // argv[0] -> "원본파일이름"
314 // argv[1] -> "복사된파일이름"
315 void
316 cp(void)
317 {
318     int rfd, wfd, len;
319     char buf[SZ_FILE_BUF];
320     struct stat st_buf;
321
322     if((stat(argv[0],&st_buf)<0) || ((rfd=open(argv[0],O_RDONLY))<0))
323         PRINT_ERR_RET();
324     if((wfd= creat(argv[1],st_buf.st_mode))<0){
325         PRINT_ERR_RET();
326         close(rfd);
327         return;
328     }
329     while((len =read(rfd,buf,SZ_FILE_BUF))>0){
330         if (write(wfd,buf,len)!=len) {
331             len=-1;
332             break;
333         }
334     }
335     if(len<0)
336         perror(cmd);
337     close(rfd);
338 }
339
340
341 // echo 다음에 입력된 문자열을 화면에 바로 echo해 주는 명령어
342 // 사용법: echo [echo할 문자열 입력: 길이 제한 없음]
343 //   예) echo I love you.
344 // argv[0] -> "I"
345 // argv[1] -> "love"
346 // argv[2] -> "you."
347 // argc = 3; 에코할 문장의 총 토큰(단어)의 수
348 void
349 echo(void)
350 {
351     int i;
352     for(i=0;i<argc;i++){
353         printf("%s ",argv[i]);
354         }
355     printf("\n");
356 }
357
358 // 디렉토리 내의 파일 이름을 보여 주는 명령어
359 // 사용법: ls [-l] [디렉토리 이름]
360 //      "-l" 옵션이 없으면 파일 이름만, 있으면 파일의 상세정보를 보여 줌
361 //      [디렉토리 이름]이 없으면 현재 디렉토리를 의미함
362 // argv[0] -> "디렉토리이름"; [디렉토리 이름]을 준 경우
363 // argc = [디렉토리 이름]을 준 경우 1, 안 준 경우 0
364 // optc = "-l" 옵션을 준 경우 1, 주지 않았을 경우 0
365 void
366 ls(void)
367 {
368     char *path;
369     DIR
370     *dp;
371     // 디렉토리 이름을 주지 않았다면 현재 디렉토리 설정
372     path = (argc == 0)? ".": argv[0];
373     if ((dp = opendir(path)) == NULL)
374     PRINT_ERR_RET();
375     if (optc == 0)
376     print_name(dp);
377     else
378     print_detail(dp, path); // 디렉토리 열기
379     closedir(dp); // 디렉토리 닫기
380 }
381 // 현재 작업 디렉토리 이름을 출력하는 명령어
382 // 사용법: pwd
383 void
384 pwd(void)
385 {
386 printf("%s\n",cur_work_dir);
387 }
388 // 하나의 파일을 삭제하는 명령어
389 // 사용법: rm  파일이름
390 // argv[0] -> "파일이름"
391 void
392 rm(void)
393 {
394     struct stat buf;
395     /*if(lstat(argv[0],&buf)<0)
396         PRINT_ERR_RET();
397     int ret;
398     if(S_ISDIR(buf.st_mode))
399         ret=rmdir(argv[0]);
400     else
401         ret=unlink(argv[0]);
402     if(ret<0)
403         PRINT_ERR_RET();*/
404
405     if( (lstat(argv[0],&buf)<0) ||(( (S_ISDIR(buf.st_mode))? rmdir(argv[0]):unlink(argv[0]))<0) )
406     PRINT_ERR_RET();
407
408
409 }
410
411 // 프로그램을 종료하는 명령어
412 // 사용법: exit
413 void
414 quit(void)
415 {
416     exit(0);
417 }
418 // 현 컴퓨터의 이름을 출력해 주는 명령어
419 // 사용법: hostname
420 void
421 hostname(void)
422 {
423     char hostname[SZ_STR_BUF];
424     gethostname(hostname,SZ_STR_BUF);
425     printf("%s\n", hostname);
426 }
427 // 사용자의 계정 이름을 출력하는 명령어
428 // 사용법: whoami
429 void
430 whoami(void)
431 {
432     /*char *username;
433     username=getlogin();
434     if (username != NULL) printf("%s\n",username);
435     else                  printf("터미널 장치가 아니라서 사용자 계정정보를 구할 수 없습니다.\n");*/
436     struct passwd *pwp;
437
438     pwp= getpwuid(getuid());
439     printf("%s\n",pwp->pw_name);
440
441 }
442 // 현재 작업 디렉토리를 변경하는 명령어
443 // 사용법: cd [디렉토리이름]
444 //         []는 명령어 인자를 주어도 되고 안 주어도 됨을 의미함
445 // argv[0] -> "디렉토리이름"; [디렉토리 이름]을 준 경우
446 // argc = [디렉토리 이름]을 준 경우 1, 주지 않았을 경우 0
447 void
448 cd(void)
449 {
450     if(argc==0){
451         struct passwd *pwp;
452         //argv[0] = "/home/a223192";
453         pwp=getpwuid(getuid());
454         argv[0]=pwp->pw_dir;
455         }
456     if(chdir(argv[0]) <0)
457         PRINT_ERR_RET();    // 에러 발생
458     else
459         getcwd(cur_work_dir, SZ_STR_BUF);
460
461 }
462 // 운영체제 이름 및 버전 등 시스템 정보를 출력하는 명령어
463 // "-a" 옵션을 주면 상세정보를, 안 주면 시스템 이름만 출력함
464 // 사용법: uname [-a]
465 // optc = "-a" 옵션 주면 1, 안 주면 0
466 void
467 unixname(void)
468 {
469     struct utsname un;
470     uname(&un);
471     printf("%s ", un.sysname);
472     if(optc == 1){
473         printf("%s ", un.nodename);
474         printf("%s ", un.release);
475         printf("%s ", un.version);
476         printf("%s ", un.machine);
477     }
478     printf("\n");
479 }
480 void
481 makedir(void)
482 {
483     if(mkdir(argv[0],0755)<0)
484         PRINT_ERR_RET();
485     // 0755: rwxr-xr-x
486     // 이는 이 디렉토리를 만든 사람은 이 디렉토리에
487     // 읽고(r: ls 실행 가능), 쓰고(w: 파일 생성 및 삭제 가능),
488     // 옮겨 갈 수 있음(x: cd로 갈 수 있음).
489     // 그러나 그룹 멤버(r-x)나 다른 제3의 사용자(r-x)는
490     // cd 명령어로 옮겨(x) 가서 ls를 할 수만 있고(r),
491     // 그 디렉토리에 파일을 생성하거나 삭제(w)는 할 수 없다.
492
493 }
494 void
495 removedir(void)
496 {
497     if(rmdir(argv[0])<0)
498         PRINT_ERR_RET();
499
500 }
501 void
502 mv(void)
503 {
504 /*  if(link(argv[0],argv[1])<0)
505         PRINT_ERR_RET();
506     if(unlink(argv[0])<0)
507         PRINT_ERR_RET();*/
508
509     if(link(argv[0],argv[1])<0|| unlink(argv[0])<0)
510         PRINT_ERR_RET();
511
512
513
514 }
515 void
516 ln(void)
517 {
518     int ret;
519 /*  if(optc == 1)
520         ret=symlink(argv[0],argv[1]);
521     else
522         ret=link(argv[0],argv[1]);
523     if(ret<0)
524         PRINT_ERR_RET();*/
525     if(((optc ==1)? symlink(argv[0],argv[1]):link(argv[0],argv[1]))<0)
526         PRINT_ERR_RET();
527
528
529
530 }
531 void
532 changemod(void)
533 {
534     int mode;
535     sscanf(argv[0],"%o",&mode);
536     if(chmod(argv[1],(mode_t)mode)<0)
537         PRINT_ERR_RET();
538 }
539 void
540 id(void)
541 {
542     struct passwd *pwp;
543     struct group *grp;
544     /*if(argc==1)
545         pwp=getpwnam(argv[0]);
546     else
547         pwp=getpwuid(getuid());
548     if (pwp==NULL){
549         printf("id: 잘못된 사용자 이름: \"%s\"",pwp->pw_name);
550         return;
551     }
552     grp=getgrgid(pwp->pw_gid);
553     if(grp==NULL){
554         printf("id: 잘못된 사용자 이름: \"%s\"",grp->gr_name);
555         return;
556         }
557     printf("uid=%d(%s) gid:%d(%s) 그룹들=%d(%s)",pwp->pw_uid,pwp->pw_name,pwp->pw_gid,pwp->pw_name,grp->gr_gid,grp->gr_name);*/
558
559
560     pwp=((argc==1)? getpwnam(argv[0]): getpwuid(getuid()));
561
562     if( (pwp==NULL) || ((grp=getgrgid(pwp->pw_gid))==NULL))
563         printf("id: 잘못된 사용자 이름: \"%s\"\n",argv[0]);
564     else
565         printf("uid=%d(%s) gid:%d(%s) 그룹들=%d(%s)\n",pwp->pw_uid,pwp->pw_name,pwp->pw_gid,pwp->pw_name,grp->gr_gid,grp->gr_name);
566 }
567 void
568 date(void)
569 {
570     char stm[128];
571     time_t ttm;
572     ttm=time(NULL);
573     struct tm *ltm, *gtm;
574     ltm=localtime(&ttm);
575
576     printf("atm: %s",asctime(ltm));
577     printf("ctm: %s",ctime(&ttm));
578     strftime(stm, 128,"stm: %a %b %d %H:%M:%S %Y",ltm);
579     puts(stm);
580 }
581 void
582 touch(void)
583 {
584     int fd;
585     if (utime(argv[0],NULL)==0)
586         return;
587     if((errno != ENOENT) || (fd=creat(argv[0],0644)<0))
588         PRINT_ERR_RET();
589         return;
590     close(fd);
591 }
592 void
593 cat(void)
594 {
595     int fd, len;
596     char buf[SZ_FILE_BUF];
597
598     if((fd= open(argv[0],O_RDONLY))<0)
599         PRINT_ERR_RET();
600     while((len=read(fd,buf,SZ_FILE_BUF))>0){
601         if(write(STDOUT_FILENO,buf,len)!=len){
602             len=-1;
603             break;
604         }
605     }
606     if(len<0)
607         perror(cmd);
608     close(fd);
609 }
610 void
611 Sleep(void)
612 {
613     int sec;
614     sscanf(argv[0],"%d",&sec);
615     sleep(sec);
616 }
617
618 // 각 명령어 구현 끝
619 //
620 //***************************************************************************
621
622 // 모든 함수는 그 함수를 호출하거나 함수 이름이 사용되기 전에 먼저
623 // 선언되거나 정의되어야 한다. 따라서 아래 cmd_tbl[]의 각 배열 원소에서
624 // 명령어 처리함수 이름을 사용하기 때문에 이 함수들은 아래 cmd_tbl[] 보다
625 // 먼저 함수정의가 되어야 하고 실제 앞에 이미 정의 되었다.
626 // 그러나 help() 함수의 정의는 cmd_tbl[] 뒤에 있기 때문에, cmd_tbl[] 앞에
627 // 이 함수를 아래처럼 미리 선언해야 한다. help() 함수정의는 cmd_tbl[]을
628 // 직접 사용하기 때문에 어쩔 수 없이 cmd_tbl[] 뒤에 있어야 한다.
629
630 void help(void);
631
632 //***************************************************************************
633 //  각 명령어별 상세 정보를 저장한 구조체 배열 cmd_tbl[]
634 //***************************************************************************
635
636 #define AC_LESS_1       -1      // 명령어 인자 개수가 0 또는 1인 경우
637 #define AC_ANY          -100    // 명령어 인자 개수가 제한 없는 경우(echo)
638
639 // 하나의 명령어에 대한 정보 구조체
640 typedef struct {
641     char *cmd;          // 명령어 문자열
642     void (*func)(void); // 명령어를 처리하는 함수 시작 주소(함수 이름)
643     int  argc;          // 명령어 인자의 개수(명령어와 옵션은 제외)
644     char *opt;          // 옵션 문자열: 예) "-a", 옵션 없으면 ""
645     char *arg;          // 명령어 사용법 출력할 때 사용할 명령어 인자:
646 } cmd_tbl_t;            //      예) cp인 경우 "원본파일 복사파일""
647
648 cmd_tbl_t cmd_tbl[] = {
649 //    명령어        명령어      명령어              명령어 사용법 출력 시
650 //     이름         처리        인자        명령어  사용할 명령어
651 //    문자열        함수이름    개수        옵션    인자
652     { "cp",         cp,         2,          "",     "원본파일  복사된파일" },
653     { "echo",       echo,       AC_ANY,     "",     "[에코할 문장]" },
654     { "help",       help,       0,          "",     "" },
655     { "ls",         ls,         AC_LESS_1,  "-l",   "[디렉토리이름]" },
656     { "exit",       quit,       0,          "",     "" },
657     { "pwd",        pwd,        0,          "",     "" },
658     { "rm",         rm,         1,          "",     "파일이름" },
659     { "hostname",   hostname,   0,          "",     "컴퓨터이름"},
660     { "whoami",     whoami,     0,          "",     "계정이름"},
661     { "cd",         cd,         AC_LESS_1,  "",     "[디렉토리이름]"},
662     { "uname",      unixname,   0,          "-a",   ""},
663     { "mkdir",      makedir,    1,          "",     "디렉토리이름"},
664     { "rmdir",      removedir,  1,          "",     "디렉토리이름"},
665     { "mv",         mv,         2,          "",     "원본파일 바뀐이름"},
666     { "ln",         ln,         2,          "-s",   "원본파일 링크파일"},
667     { "chmod",      changemod,  2,          "",     "8진수모드값 파일이름"},
668     { "id",         id,         AC_LESS_1,  "",     "[계정이름]"},
669     { "date",       date,       0,          "",     ""},
670     { "touch",      touch,      1,          "",     "파일이름"},
671     { "cat",        cat,        1,          "",     "파일이름"},
672     { "sleep",      Sleep,      1,          "",     "초단위시간"},
673 };
674 // (총 명령어 개수) = (cmd_tbl[] 배열 전체 크기) / (배열의 한 원소 크기)
675 int num_cmd = sizeof(cmd_tbl) / sizeof(cmd_tbl[0]);
676
677 // 이 프로그램이 지원하는 명령어 리스트를 보여 주는 명령어
678 // 사용법: help
679 // 이 함수가 일반 명령어를 처리하는 함수들과는 달리 여기에 배치된 것은
680 // 바로 위에 있는 cmd_tbl[]을 이 함수가 직접 사용하기 때문이다.
681 void
682 help(void)
683 {
684     int  k;
685
686     printf("현재 지원되는 명령어 종류입니다.\n");
687     for (k = 0; k < num_cmd; ++k)
688         print_usage("  ", cmd_tbl[k].cmd, cmd_tbl[k].opt, cmd_tbl[k].arg);
689     printf("\n");
690 }
691 //run_cmd
692 void
693 run_cmd(void)
694 {
695     pid_t pid;
696     if ( (pid=fork())<0)
697         PRINT_ERR_RET();
698     else if (pid==0){
699         int i,cnt=0;
700         char *nargv[100];
701         nargv[cnt++] =cmd;
702         for(i=0;i<optc;++i)
703             nargv[cnt++]=optv[i];
704         for(i=0;i<argc;++i)
705             nargv[cnt++]=argv[i];
706         nargv[cnt++]=NULL;
707
708         if(execvp(cmd,nargv)<0){
709             perror(cmd);
710             exit(1);
711         }
712     }
713     else{
714         if(waitpid(pid,NULL,0)<0)
715             PRINT_ERR_RET();
716         }
717
718 }
719 // proc_cmd():
720 // 입력된 명령어와 동일한 이름의 명령어 구조체를 cmd_tbl[] 배열에서 찾아 낸 후
721 // 명령어 인자와 옵션을 제대로 입력했는지 체크하고 잘못 되었으면 사용법을
722 // 출력하고 정상이면 해당 명령어를 처리하는 함수를 호출하여 실행함
723 void
724 proc_cmd(void)
725 {
726     int  k;
727
728     for (k = 0; k < num_cmd; ++k) { // 입력한 명령어 정보를 cmd_tbl[]에서 찾음
729         // 전역변수char*cmd: 사용자가 입력한 명령어 문자열 주소, 예)cmd->"ls"
730         if (EQUAL(cmd, cmd_tbl[k].cmd)) { // 입력 명령어를 cmd_tbl에서 찾았음
731
732             if ((check_arg(cmd_tbl[k].argc) < 0) ||     // 인자 개수 체크
733                 (check_opt(cmd_tbl[k].opt)  < 0))       // 옵션 체크
734                 // 인자 개수 또는 옵션이 잘못 되었음: 사용법 출력
735                 print_usage("사용법: ", cmd_tbl[k].cmd,
736                                         cmd_tbl[k].opt, cmd_tbl[k].arg);
737                 //       예) 사용법: ls  -l  [디렉토리이름]
738             else
739                 cmd_tbl[k].func();      // 명령어를 처리하는 함수를 호출
740
741             return;
742         }
743     }
744     // 사용자가 입력한 명령어를 cmd_tbl[]에서 찾지 못한 경우
745     //printf("%s : 지원되지 않는 명령어입니다.\n", cmd);
746     run_cmd();
747 }
748
749 //***************************************************************************
750 //  main() 함수
751 //***************************************************************************
752 int
753 main(int argc, char *argv[])
754 {
755     char cmd_line[SZ_STR_BUF];  // 입력된 명령어 라인 전체를 저장 공간
756     int cmd_count = 1;
757
758     setbuf(stdout, NULL);   // 표준 출력 버퍼 제거: printf() 즉시 화면 출력
759     setbuf(stderr, NULL);   // 표준 에러 출력 버퍼 제거
760
761     help();                     // 명령어 전체 리스트 출력
762     getcwd(cur_work_dir, SZ_STR_BUF);
763
764     for (;;) {
765         // 명령 프롬프트 출력: "<현재작업디렉토리> 명령어번호: "
766         printf("<%s> %d: ", cur_work_dir,cmd_count);
767         // 키보드에서 한 행을 입력 받아 cmd_line[]에 저장
768         if (fgets(cmd_line, SZ_STR_BUF, stdin) == NULL)
769             break;
770
771         // 입력받은 한 행(cmd_line[])의 명령어, 옵션, 인자를 개별 문자열로 분리
772         if (get_argv_optv(cmd_line) != NULL) {
773             proc_cmd(); // 한 명령어 처리
774             ++cmd_count;
775         }
776         // else 명령어는 주지 않고 엔터만 친 경우 명령어 번호는 증가하지 않음
777     }
778 }
```
