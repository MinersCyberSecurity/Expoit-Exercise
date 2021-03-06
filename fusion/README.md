# Fusion Challenges

These challenges were taken from [exploit-exercises](https://exploit-exercises.lains.me).  
For this level it is recommeneded that you download the [fusion virtual](https://drive.google.com/drive/u/0/folders/0B9RbZkKdRR8qV1BsVkRxak0zcVU) machine, or the [vagrant image](https://app.vagrantup.com/mjwhitta/boxes/exploit-exercises-fusion-2/versions/1.0.0).

## Fusion

Fusion is the next step from the protostar setup, and covers more advanced styles of exploitation, and covers a variety of anti-exploitation mechanisms such as:

+ Address Space Layout Randomisation
+ Position Independent Executables
+ Non-executable Memory
+ Source Code Fortification (_DFORTIFY_SOURCE=)
+ Stack Smashing Protection (ProPolice / SSP)

In addition to the above, there are a variety of other challenges and things to explore, such as:

+ Cryptographic issues
+ Timing attacks
+ Variety of network protocols (such as Protocol Buffers and Sun RPC)

At the end of Fusion, the participant will have a through understanding of exploit prevention strategies, associated weaknesses, various cryptographic weaknesses, numerous heap implementations.

## INDEX

### Challenges

| Levels                 |
|:----------------------:|
|[Level00](#Level00)     |
|[Level01](#Level01)     |
|[Level02](#Level02)     |
|[Level03](#Level03)     |
|[Level04](#Level04)     |
|[Level05](#Level05)     |
|[Level06](#Level06)     |
|[Level07](#Level07)     |
|[Level08](#Level08)     |
|[Level09](#Level09)     |
|[Level10](#Level10)     |
|[Level11](#Level11)     |
|[Level12](#Level12)     |
|[Level13](#Level13)     |
|[Level14](#Level14)     |

## Level00

### About

This is a simple introduction to get you warmed up. The return address is supplied in case your memory needs a jog :)

Hint: Storing your shellcode inside of the fix_path ‘resolved’ buffer might be a bad idea due to character restrictions due to realpath(). Instead, there is plenty of room after the HTTP/1.1 that you can use that will be ideal (and much larger).

| Protections                         |      |
|:------------------------------------|:----:|
| Vulnerability Type:                 | Stack|
| Position Independent Executable:    |    No|
| Read only relocations:              |    No|
| Non-Executable stack:               |    No|
| Non-Executable heap:                |    No|
| Address Space Layout Randomisation: |    No|
| Source Fortification:               |    No|

### Source Code:

```c
#include "../common/common.c"

int fix_path(char *path)
{
  char resolved[128];
  
  if(realpath(path, resolved) == NULL) return 1; // can't access path. will error trying to open
  strcpy(path, resolved);
}

char *parse_http_request()
{
  char buffer[1024];
  char *path;
  char *q;

  printf("[debug] buffer is at 0x%08x :-)\n", buffer);

  if(read(0, buffer, sizeof(buffer)) <= 0) errx(0, "Failed to read from remote host");
  if(memcmp(buffer, "GET ", 4) != 0) errx(0, "Not a GET request");

  path = &buffer[4];
  q = strchr(path, ' ');
  if(! q) errx(0, "No protocol version specified");
  *q++ = 0;
  if(strncmp(q, "HTTP/1.1", 8) != 0) errx(0, "Invalid protocol");

  fix_path(path);

  printf("trying to access %s\n", path);

  return path;
}

int main(int argc, char **argv, char **envp)
{
  int fd;
  char *p;

  background_process(NAME, UID, GID);
  fd = serve_forever(PORT);
  set_io(fd);

  parse_http_request();
}
```

## Level01

### About

level00 with stack/heap/mmap aslr, without info leak :)

| Protections                         |      |
|:------------------------------------|:----:|
| Vulnerability Type:                 | Stack|
| Position Independent Executable:    |    No|
| Read only relocations:              |    No|
| Non-Executable stack:               |    No|
| Non-Executable heap:                |    No|
| Address Space Layout Randomisation: |   Yes|
| Source Fortification:               |    No|

### Source Code:

```c
#include "../common/common.c"

int fix_path(char *path)
{
  char resolved[128];
  
  if(realpath(path, resolved) == NULL) return 1; // can't access path. will error trying to open
  strcpy(path, resolved);
}

char *parse_http_request()
{
  char buffer[1024];
  char *path;
  char *q;

  // printf("[debug] buffer is at 0x%08x :-)\n", buffer); :D

  if(read(0, buffer, sizeof(buffer)) <= 0) errx(0, "Failed to read from remote host");
  if(memcmp(buffer, "GET ", 4) != 0) errx(0, "Not a GET request");

  path = &buffer[4];
  q = strchr(path, ' ');
  if(! q) errx(0, "No protocol version specified");
  *q++ = 0;
  if(strncmp(q, "HTTP/1.1", 8) != 0) errx(0, "Invalid protocol");

  fix_path(path);

  printf("trying to access %s\n", path);

  return path;
}

int main(int argc, char **argv, char **envp)
{
  int fd;
  char *p;

  background_process(NAME, UID, GID);
  fd = serve_forever(PORT);
  set_io(fd);

  parse_http_request();
}
```

## Level02

### About

This level deals with some basic obfuscation / math stuff.

This level introduces non-executable memory and return into libc / .text / return orientated programming (ROP).

| Protections                         |      |
|:------------------------------------|:----:|
| Vulnerability Type:                 | Stack|
| Position Independent Executable:    |    No|
| Read only relocations:              |    No|
| Non-Executable stack:               |   Yes|
| Non-Executable heap:                |   Yes|
| Address Space Layout Randomisation: |   Yes|
| Source Fortification:               |    No|

### Source Code:

```c
#include "../common/common.c"

#define XORSZ 32

void cipher(unsigned char *blah, size_t len)
{
  static int keyed;
  static unsigned int keybuf[XORSZ];

  int blocks;
  unsigned int *blahi, j;

  if(keyed == 0) {
      int fd;
      fd = open("/dev/urandom", O_RDONLY);
      if(read(fd, &keybuf, sizeof(keybuf)) != sizeof(keybuf)) exit(EXIT_FAILURE);
      close(fd);
      keyed = 1;
  }

  blahi = (unsigned int *)(blah);
  blocks = (len / 4);
  if(len & 3) blocks += 1;

  for(j = 0; j < blocks; j++) {
      blahi[j] ^= keybuf[j % XORSZ];
  }
}

void encrypt_file()
{
  // http://thedailywtf.com/Articles/Extensible-XML.aspx
  // maybe make bigger for inevitable xml-in-xml-in-xml ?
  unsigned char buffer[32 * 4096];

  unsigned char op;
  size_t sz;
  int loop;

  printf("[-- Enterprise configuration file encryption service --]\n");
  
  loop = 1;
  while(loop) {
      nread(0, &op, sizeof(op));
      switch(op) {
          case 'E':
              nread(0, &sz, sizeof(sz));
              nread(0, buffer, sz);
              cipher(buffer, sz);
              printf("[-- encryption complete. please mention "
              "474bd3ad-c65b-47ab-b041-602047ab8792 to support "
              "staff to retrieve your file --]\n");
              nwrite(1, &sz, sizeof(sz));
              nwrite(1, buffer, sz);
              break;
          case 'Q':
              loop = 0;
              break;
          default:
              exit(EXIT_FAILURE);
      }
  }

}

int main(int argc, char **argv, char **envp)
{
  int fd;
  char *p;

  background_process(NAME, UID, GID);
  fd = serve_forever(PORT);
  set_io(fd);

  encrypt_file();
}
```

## Level03

### About

This level introduces partial hash collisions (hashcash) and more stack corruption

| Protections                         |      |
|:------------------------------------|:----:|
| Vulnerability Type:                 | Stack|
| Position Independent Executable:    |    No|
| Read only relocations:              |    No|
| Non-Executable stack:               |   Yes|
| Non-Executable heap:                |   Yes|
| Address Space Layout Randomisation: |   Yes|
| Source Fortification:               |    No|

### Source Code:

```c
#include "../common/common.c"

#include "json/json.h"

unsigned char *gRequest; // request buffer
int gRequestMax = 4096;      // maximum buffer size
int gRequestSize;       // current buffer size
char *token;

char *gServerIP;

unsigned char *gContents;
int gContents_len;

unsigned char *gTitle;
int gTitle_len;

json_object *gObj;

void generate_token()
{
  struct sockaddr_in sin;
  int len;

  len = sizeof(struct sockaddr_in);
  if(getpeername(0, (void *)&sin, &len) == -1)
      err(EXIT_FAILURE, "Unable to getpeername(0, ...): ");
  
  srand((getpid() << 16) ^ (getppid() + (time(NULL) ^
      sin.sin_addr.s_addr) + sin.sin_port));

  asprintf(&token, "// %s:%d-%d-%d-%d-%d", inet_ntoa(sin.sin_addr),
      ntohs(sin.sin_port), (int)time(NULL), rand(), rand(),
      rand());
}

void send_token()
{
  generate_token();

  printf("\"%s\"\n", token);
  fflush(stdout); 
}

void read_request()
{
  int ret;

  gRequest = malloc(gRequestMax);
  if(!gRequest) errx(EXIT_FAILURE, "Failed to allocate %d bytes",
      gRequestMax);

  while(1) {
      ret = read(0, gRequest + gRequestSize, gRequestMax - gRequestSize);
      if(ret == -1) err(EXIT_FAILURE, "Failed to read %d bytes ... ",
          gRequestMax - gRequestSize);

      if(ret == 0) break;

      gRequestSize += ret;

      if(gRequestSize == gRequestMax) {
          gRequest = realloc(gRequest, gRequestMax * 2);
          if(gRequest == NULL) {
              errx(EXIT_FAILURE, "Failed to realloc from %d bytes "
              "to %d bytes ", gRequestMax, gRequestMax * 2);
          }
          gRequestMax *= 2;  
      }
  }

  close(0); close(1); close(2);
}

#include <openssl/hmac.h>

void validate_request()
{
  unsigned char result[20];
  unsigned char invalid;
  int len;

  if(strncmp(gRequest, token, strlen(token)) != 0)
      errx(EXIT_FAILURE, "Token not found!"); 
      // XXX won't be seen by user

  len = sizeof(result);

  HMAC(EVP_sha1(), token, strlen(token), gRequest, gRequestSize, result,
      &len); // hashcash with added hmac goodness
  
  invalid = result[0] | result[1]; // Not too bad :>
  if(invalid)
      errx(EXIT_FAILURE, "Checksum failed! (got %02x%02x%02x%02x...)",
      result[0], result[1], result[2], result[3]);
      // XXX won't be seen by user.
}

void parse_request()
{
  json_object *new_obj;
  new_obj = json_tokener_parse(gRequest);
  if(is_error(new_obj)) errx(EXIT_FAILURE, "Unable to parse request");
  gObj = new_obj;
}

void decode_string(const char *src, unsigned char *dest, int *dest_len)
{
  char swap[5], *p;
  int what;
  unsigned char *start, *end;
  
  swap[4] = 0;
  start = dest;
  // make sure we don't over the end of the allocated space.
  end = dest + *dest_len;

  while(*src && dest != end) {
      // printf("*src = %02x, dest = %p, end = %p\n", (unsigned char) 
      // *src, dest, end);

      if(*src == '\\') {
          *src++;
          // printf("-> in src == '\\', next byte is %02x\n", *src);

          switch(*src) {
              case '"':
              case '\\':
              case '/':
                  *dest++ = *src++;
                  break;
              case 'b': *dest++ = '\b'; src++; break;
              case 'f': *dest++ = '\f'; src++; break;
              case 'n': *dest++ = '\n'; src++; break;
              case 'r': *dest++ = '\r'; src++; break;
              case 't': *dest++ = '\t'; src++; break;
              case 'u':
                  src++;

                  // printf("--> in \\u handling. got %.4s\n",
                  // src);

                  memcpy(swap, src, 4);
                  p = NULL;
                  what = strtol(swap, &p, 16);

                  // printf("--> and in hex, %08x\n", what);

                  *dest++ = (what >> 8) & 0xff;
                  *dest++ = (what & 0xff);
                  src += 4;
                  break;
              default:
                  errx(EXIT_FAILURE, "Unhandled encoding found");
                  break;
          }
      } else {
          *dest++ = *src++;
      }
  }

  // and record the actual space taken up
  *dest_len = (unsigned int)(dest) - (unsigned int)(start);
  // printf("and the length of the function is ... %d bytes", *dest_len);

}

void handle_request()
{
  unsigned char title[128];
  char *tags[16];
  unsigned char contents[1024];

  int tag_cnt = 0;
  int i;
  int len;

  memset(title, 0, sizeof(title));
  memset(contents, 0, sizeof(contents));

  json_object_object_foreach(gObj, key, val) {
      if(strcmp(key, "tags") == 0) {
          for(i=0; i < json_object_array_length(val); i++) {
              json_object *obj = json_object_array_get_idx(val, i);
              tags[tag_cnt + i] = json_object_get_string(obj);
          }
          tag_cnt += i;
      } else if(strcmp(key, "title") == 0) {
          len = sizeof(title);
          decode_string(json_object_get_string(val), title, &len);

          gTitle = calloc(len+1, 1);
          gTitle_len = len;
          memcpy(gTitle, title, len);

      } else if(strcmp(key, "contents") == 0) {
          len = sizeof(contents);
          decode_string(json_object_get_string(val), contents, &len);

          gContents = calloc(len+1, 1);
          gContents_len = len;
          memcpy(gContents, contents, len);

      } else if(strcmp(key, "serverip") == 0) {
          gServerIP = json_object_get_string(val);
      }
  }
  printf("and done!\n");
}

void post_blog_article()
{
  char *port = "80", *p;
  struct sockaddr_in sin;
  int fd;
  int len, cl;

  unsigned char *post, *data;

  // We can't post if there is no information available
  if(! gServerIP || !gContents || !gTitle) return;

  post = calloc(128 * 1024, 1);
  cl = gTitle_len + gContents_len + strlen("\r\n\r\n");

  len = sprintf(post, "POST /blog/post HTTP/1.1\r\n");
  len += sprintf(post + len, "Connection: close\r\n");
  len += sprintf(post + len, "Host: %s\r\n", gServerIP);
  len += sprintf(post + len, "Content-Length: %d\r\n", cl);
  len += sprintf(post + len, "\r\n");

  memcpy(post + len, gTitle, gTitle_len);
  len += gTitle_len;
  len += sprintf(post + len, "\r\n");
  memcpy(post + len, gContents, gContents_len);
  len += gContents_len;

  p = strchr(gServerIP, ':');
  if(p) {
      *p++ = 0;
      port = p;
  }

  memset(&sin, 0, sizeof(struct sockaddr_in));
  sin.sin_family = AF_INET;
  sin.sin_addr.s_addr = inet_addr(gServerIP);
  sin.sin_port = htons(atoi(port));

  fd = socket(AF_INET, SOCK_STREAM, 0);
  if(fd == -1) err(EXIT_FAILURE, "socket(): ");
  if(connect(fd, (void *)&sin, sizeof(struct sockaddr_in)) == -1)
      err(EXIT_FAILURE, "connect(): ");
  nwrite(fd, post, len);
  close(fd);
}

int main(int argc, char **argv, char **envp)
{
  int fd;
  char *p;

  signal(SIGPIPE, SIG_IGN);

  background_process(NAME, UID, GID);
  fd = serve_forever(PORT);
  set_io(fd);

  send_token();
  read_request();
  validate_request();
  parse_request();
  handle_request();
  post_blog_article();
}
```

## Level04

### About

Level04 introduces timing attacks, position independent executables (PIE), and stack smashing protection (SSP). Partial overwrites ahoy!

| Protections                         |      |
|:------------------------------------|:----:|
| Vulnerability Type:                 | Stack|
| Position Independent Executable:    |   Yes|
| Read only relocations:              |    No|
| Non-Executable stack:               |   Yes|
| Non-Executable heap:                |   Yes|
| Address Space Layout Randomisation: |   Yes|
| Source Fortification:               |   Yes|

### Source Code:

```c
#include "../common/common.c"

// original code from micro_httpd_12dec2005.tar.gz -- acme.com. added vulnerabilities etc ;)

/* micro_httpd - really small HTTP server
**
** Copyright (c) 1999,2005 by Jef Poskanzer <[email protected]>.
** All rights reserved.
**
** Redistribution and use in source and binary forms, with or without
** modification, are permitted provided that the following conditions
** are met:
** 1. Redistributions of source code must retain the above copyright
**    notice, this list of conditions and the following disclaimer.
** 2. Redistributions in binary form must reproduce the above copyright
**    notice, this list of conditions and the following disclaimer in the
**    documentation and/or other materials provided with the distribution.
**
** THIS SOFTWARE IS PROVIDED BY THE AUTHOR AND CONTRIBUTORS ``AS IS'' AND
** ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
** IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
** ARE DISCLAIMED.  IN NO EVENT SHALL THE AUTHOR OR CONTRIBUTORS BE LIABLE
** FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
** DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
** OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
** HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
** LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY
** OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF
** SUCH DAMAGE.
*/


#define SERVER_NAME "level04.c"
#define SERVER_URL "https://gist.github.com/b69116098bcc6ef7dfb4"
#define PROTOCOL "HTTP/1.0"
#define RFC1123FMT "%a, %d %b %Y %H:%M:%S GMT"


/* Forwards. */
static void file_details( char* dir, char* name );
static void send_error( int status, char* title, char* extra_header, char* text );
static void send_headers( int status, char* title, char* extra_header,
  char* mime_type, off_t length, time_t mod );
static char* get_mime_type( char* name );
static void strdecode( char* to, char* from );
static int hexit( char c );
static void strencode( char* to, size_t tosize, const char* from );

int webserver(int argc, char **argv);

// random decoder 
void build_decoding_table();

char *password;
int password_size = 16;

int main(int argc, char **argv)
{
  int fd, i;
  char *args[6];

  /* Securely generate a password for this session */

  secure_srand();
  password = calloc(password_size, 1);
  for(i = 0; i < password_size; i++) {
      switch(rand() % 3) {
          case 0: password[i] = (rand() % 25) + 'a'; break;
          case 1: password[i] = (rand() % 25) + 'A'; break;
          case 2: password[i] = (rand() % 9) + '0'; break;
      }
  }

  // printf("password is %s\n", password);

  background_process(NAME, UID, GID); 
  fd = serve_forever(PORT);
  set_io(fd);
  alarm(15);

  args[0] = "/opt/fusion/bin/stack06";
  args[1] = "/opt/fusion/run";
  args[2] = NULL;

  build_decoding_table();

  webserver(2, args);
}

// random decoder from stackoverflow
// modified to make more vulnerable

#include <math.h>
#include <stdint.h>
#include <stdlib.h>

static char encoding_table[] = {'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H',
                                'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P',
                                'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X',
                                'Y', 'Z', 'a', 'b', 'c', 'd', 'e', 'f',
                                'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n',
                                'o', 'p', 'q', 'r', 's', 't', 'u', 'v',
                                'w', 'x', 'y', 'z', '0', '1', '2', '3',
                                '4', '5', '6', '7', '8', '9', '+', '/'};
static char *decoding_table = NULL;
static int mod_table[] = {0, 2, 1};

void base64_decode(const char *data,
                    size_t input_length,
                    unsigned char *output,
                    size_t *output_length) {

    if (decoding_table == NULL) build_decoding_table();

    // printf("data: %p, input_length: %d, output: %p, output_length: %p\n",
    // data, input_length, output, output_length);

    if ((input_length % 4) != 0) {
  // printf("len % 4 = fail\n");
  return;
    }

    *output_length = input_length / 4 * 3;
    if (data[input_length - 1] == '=') (*output_length)--;
    if (data[input_length - 2] == '=') (*output_length)--;

    for (int i = 0, j = 0; i < input_length;) {

        uint32_t sextet_a = data[i] == '=' ? 0 & i++ : decoding_table[data[i++]];
        uint32_t sextet_b = data[i] == '=' ? 0 & i++ : decoding_table[data[i++]];
        uint32_t sextet_c = data[i] == '=' ? 0 & i++ : decoding_table[data[i++]];
        uint32_t sextet_d = data[i] == '=' ? 0 & i++ : decoding_table[data[i++]];

        uint32_t triple = (sextet_a << 3 * 6)
                        + (sextet_b << 2 * 6)
                        + (sextet_c << 1 * 6)
                        + (sextet_d << 0 * 6);

        if (j < *output_length) output[j++] = (triple >> 2 * 8) & 0xFF;
        if (j < *output_length) output[j++] = (triple >> 1 * 8) & 0xFF;
        if (j < *output_length) output[j++] = (triple >> 0 * 8) & 0xFF;
    }
}


void build_decoding_table() {

    decoding_table = malloc(256);

    for (int i = 0; i < 0x40; i++)
        decoding_table[encoding_table[i]] = i;
}


void base64_cleanup() {
    free(decoding_table);
}


// end random decoder

int validate_credentials(char *line)
{
  char *p, *pw;
  unsigned char details[2048];
  int bytes_wrong;
  int l;
  struct timeval tv;
  int output_len;


  memset(details, 0, sizeof(details));

  output_len = sizeof(details);

  p = strchr(line, '\n');
  if(p) *p = 0;
  p = strchr(line, '\r');
  if(p) *p = 0;

  // printf("%d\n", strlen(line));
  base64_decode(line, strlen(line), details, &output_len);
  // printf("%s -> %s\n", line, details);
  // fflush(stdout);
  
  p = strchr(details, ':');
  pw = (p == NULL) ? (char *)details : p + 1;

  for(bytes_wrong = 0, l = 0; pw[l] && l < password_size; l++) {
      if(pw[l] != password[l]) {

#if 0
          char *buf;
          asprintf(&buf, "[%d] wrong byte (%02x vs %02x)\n", l,
                          password[l], pw[l]);
          write(58, buf, strlen(buf));
#endif

          bytes_wrong++;
      }
  }

  // anti bruteforce mechanism. good luck ;>
  
  tv.tv_sec = 0;
  tv.tv_usec = 2500 * bytes_wrong;

  select(0, NULL, NULL, NULL, &tv);

  // printf("%d bytes wrong!\n", bytes_wrong);

  if(l < password_size || bytes_wrong)
      send_error(401, "Unauthorized",
      "WWW-Authenticate: Basic realm=\"stack06\"",
      "Unauthorized");

  return 1;
}

int
webserver( int argc, char** argv )
    {
    char line[10000], method[10000], path[10000], protocol[10000], idx[20000];
    char location[20000], command[20000];
    char* file;
    size_t len;
    int ich;
    struct stat sb;
    FILE* fp;
    struct dirent **dl;
    int i, n;
    int authed = 0;

    if ( argc != 2 )
  send_error( 500, "Internal Error", (char*) 0,
  "Config error - no dir specified." );
    if ( chdir( argv[1] ) < 0 )
  send_error( 500, "Internal Error", (char*) 0,
  "Config error - couldn't chdir()." );
    if ( fgets( line, sizeof(line), stdin ) == (char*) 0 )
  send_error( 400, "Bad Request", (char*) 0,
  "No request found." );
    if ( sscanf( line, "%[^ ] %[^ ] %[^ ]", method, path, protocol ) != 3 )
  send_error( 400, "Bad Request", (char*) 0, "Can't parse request." );
    while ( fgets( line, sizeof(line), stdin ) != (char*) 0 )
  {
        if ( strncmp ( line, "Authorization: Basic ", 21) == 0)
      authed = validate_credentials(line + 21);

  if ( strcmp( line, "\n" ) == 0 || strcmp( line, "\r\n" ) == 0 )
      break;
  }
    if ( ! authed) send_error(401, "Unauthorized",
      "WWW-Authenticate: Basic realm=\"stack06\"",
      "Unauthorized");

    if ( strcasecmp( method, "get" ) != 0 )
  send_error( 501, "Not Implemented", (char*) 0,
  "That method is not implemented." );
    if ( path[0] != '/' )
  send_error( 400, "Bad Request", (char*) 0, "Bad filename." );
    file = &(path[1]);
    strdecode( file, file );
    if ( file[0] == '\0' )
  file = "./";
    len = strlen( file );
    if ( file[0] == '/' || strcmp( file, ".." ) == 0 ||
      strncmp( file, "../", 3 ) == 0 || strstr( file, "/../" ) != (char*) 0 ||
      strcmp( &(file[len-3]), "/.." ) == 0 )
  send_error( 400, "Bad Request", (char*) 0, "Illegal filename." );
    if ( stat( file, &sb ) < 0 )
  send_error( 404, "Not Found", (char*) 0, "File not found." );
    if ( S_ISDIR( sb.st_mode ) )
  {
  if ( file[len-1] != '/' )
      {
      (void) snprintf(
      location, sizeof(location), "Location: %s/", path );
      send_error( 302, "Found", location, "Directories must end with a slash." );
      }
  (void) snprintf( idx, sizeof(idx), "%sindex.html", file );
  if ( stat( idx, &sb ) >= 0 )
      {
      file = idx;
      goto do_file;
      }
  send_headers( 200, "Ok", (char*) 0, "text/html", -1, sb.st_mtime );
  (void) printf( "<html><head><title>Index of %s</title></head>\n"
      "<body bgcolor=\"#99cc99\"><h4>Index of %s</h4>\n<pre>\n",
      file, file );
  n = scandir( file, &dl, NULL, alphasort );
  if ( n < 0 )
      perror( "scandir" );
  else
      for ( i = 0; i < n; ++i )
      file_details( file, dl[i]->d_name );
  (void) printf( "</pre>\n<hr>\n<address><a href=\"%s\">%s</a></address>"
      "\n</body></html>\n", SERVER_URL, SERVER_NAME );
  }
    else
  {
  do_file:
  fp = fopen( file, "r" );
  if ( fp == (FILE*) 0 )
      send_error( 403, "Forbidden", (char*) 0, "File is protected." );
  send_headers( 200, "Ok", (char*) 0, get_mime_type( file ), sb.st_size,
      sb.st_mtime );
  while ( ( ich = getc( fp ) ) != EOF )
      putchar( ich );
  }

    (void) fflush( stdout );
    exit( 0 );
    }


static void
file_details( char* dir, char* name )
    {
    static char encoded_name[1000];
    static char path[2000];
    struct stat sb;
    char timestr[16];

    strencode( encoded_name, sizeof(encoded_name), name );
    (void) snprintf( path, sizeof(path), "%s/%s", dir, name );
    if ( lstat( path, &sb ) < 0 )
  (void) printf( "<a href=\"%s\">%-32.32s</a>    ???\n",
      encoded_name, name );
    else
  {
  (void) strftime( timestr, sizeof(timestr), "%d%b%Y %H:%M",
      localtime( &sb.st_mtime ) );
  (void) printf( "<a href=\"%s\">%-32.32s</a>    %15s %14lld\n",
      encoded_name, name, timestr, (int64_t) sb.st_size );
  }
    }


static void
send_error( int status, char* title, char* extra_header, char* text )
    {
    send_headers( status, title, extra_header, "text/html", -1, -1 );
    (void) printf( "<html><head><title>%d %s</title></head>\n<body "
      "bgcolor=\"#cc9999\"><h4>%d %s</h4>\n", status,
      title, status, title );
    (void) printf( "%s\n", text );
    (void) printf( "<hr>\n<address><a href=\"%s\">%s</a></address>"
      "\n</body></html>\n", SERVER_URL, SERVER_NAME );
    (void) fflush( stdout );
    exit( 1 );
    }


static void
send_headers( int status, char* title, char* extra_header,
  char* mime_type, off_t length, time_t mod )
    {
    time_t now;
    char timebuf[100];

    (void) printf( "%s %d %s\015\012", PROTOCOL, status, title );
    (void) printf( "Server: %s\015\012", SERVER_NAME );
    now = time( (time_t*) 0 );
    (void) strftime( timebuf, sizeof(timebuf), RFC1123FMT, gmtime( &now ) );
    (void) printf( "Date: %s\015\012", timebuf );
    if ( extra_header != (char*) 0 )
  (void) printf( "%s\015\012", extra_header );
    if ( mime_type != (char*) 0 )
  (void) printf( "Content-Type: %s\015\012", mime_type );
    if ( length >= 0 )
  (void) printf( "Content-Length: %lld\015\012", (int64_t) length );
    if ( mod != (time_t) -1 )
  {
  (void) strftime( timebuf, sizeof(timebuf), RFC1123FMT, gmtime( &mod ) );
  (void) printf( "Last-Modified: %s\015\012", timebuf );
  }
    (void) printf( "Connection: close\015\012" );
    (void) printf( "\015\012" );
    }


static char*
get_mime_type( char* name )
    {
    char* dot;

    dot = strrchr( name, '.' );
    if ( dot == (char*) 0 )
  return "text/plain; charset=iso-8859-1";
    if ( strcmp( dot, ".html" ) == 0 || strcmp( dot, ".htm" ) == 0 )
  return "text/html; charset=iso-8859-1";
    if ( strcmp( dot, ".jpg" ) == 0 || strcmp( dot, ".jpeg" ) == 0 )
  return "image/jpeg";
    if ( strcmp( dot, ".gif" ) == 0 )
  return "image/gif";
    if ( strcmp( dot, ".png" ) == 0 )
  return "image/png";
    if ( strcmp( dot, ".css" ) == 0 )
  return "text/css";
    if ( strcmp( dot, ".au" ) == 0 )
  return "audio/basic";
    if ( strcmp( dot, ".wav" ) == 0 )
  return "audio/wav";
    if ( strcmp( dot, ".avi" ) == 0 )
  return "video/x-msvideo";
    if ( strcmp( dot, ".mov" ) == 0 || strcmp( dot, ".qt" ) == 0 )
  return "video/quicktime";
    if ( strcmp( dot, ".mpeg" ) == 0 || strcmp( dot, ".mpe" ) == 0 )
  return "video/mpeg";
    if ( strcmp( dot, ".vrml" ) == 0 || strcmp( dot, ".wrl" ) == 0 )
  return "model/vrml";
    if ( strcmp( dot, ".midi" ) == 0 || strcmp( dot, ".mid" ) == 0 )
  return "audio/midi";
    if ( strcmp( dot, ".mp3" ) == 0 )
  return "audio/mpeg";
    if ( strcmp( dot, ".ogg" ) == 0 )
  return "application/ogg";
    if ( strcmp( dot, ".pac" ) == 0 )
  return "application/x-ns-proxy-autoconfig";
    return "text/plain; charset=iso-8859-1";
    }


static void
strdecode( char* to, char* from )
    {
    for ( ; *from != '\0'; ++to, ++from )
  {
  if ( from[0] == '%' && isxdigit( from[1] ) && isxdigit( from[2] ) )
      {
      *to = hexit( from[1] ) * 16 + hexit( from[2] );
      from += 2;
      }
  else
      *to = *from;
  }
    *to = '\0';
    }


static int
hexit( char c )
    {
    if ( c >= '0' && c <= '9' )
  return c - '0';
    if ( c >= 'a' && c <= 'f' )
  return c - 'a' + 10;
    if ( c >= 'A' && c <= 'F' )
  return c - 'A' + 10;
    return 0;       /* shouldn't happen, we're guarded by isxdigit() */
    }


static void
strencode( char* to, size_t tosize, const char* from )
    {
    int tolen;

    for ( tolen = 0; *from != '\0' && tolen + 4 < tosize; ++from )
  {
  if ( isalnum(*from) || strchr( "/_.-~", *from ) != (char*) 0 )
      {
      *to = *from;
      ++to;
      ++tolen;
      }
  else
      {
      (void) sprintf( to, "%%%02x", (int) *from & 0xff );
      to += 3;
      tolen += 3;
      }
  }
    *to = '\0';
    }
```

## Level05

### About

Even more information leaks and stack overwrites. This time with random libraries / evented programming styles :>

| Protections                         |      |
|:------------------------------------|:----:|
| Vulnerability Type:                 | Stack|
| Position Independent Executable:    |   Yes|
| Read only relocations:              |    No|
| Non-Executable stack:               |   Yes|
| Non-Executable heap:                |   Yes|
| Address Space Layout Randomisation: |   Yes|
| Source Fortification:               |   Yes|

### Source Code:

```c
#include "../common/common.c"

#include <task.h>

#define STACK (4096 * 8)

unsigned int hash(unsigned char *str, int length, unsigned int mask)
{
  unsigned int h = 0xfee13117;
  int i;

  for(h = 0xfee13117, i = 0; i < length; i++) {
      h ^= str[i];
      h += (h << 11);
      h ^= (h >> 7);
      h -= str[i];
  }
  h += (h << 3);
  h ^= (h >> 10);
  h += (h << 15);
  h -= (h >> 17);

  return (h & mask);
}

void fdprintf(int fd, char *fmt, ...)
{
  va_list ap;
  char *msg = NULL;

  va_start(ap, fmt);
  vasprintf(&msg, fmt, ap);
  va_end(ap);

  if(msg) {
      fdwrite(fd, msg, strlen(msg));
      free(msg);
  }
}

struct registrations {
  short int flags;
  in_addr_t ipv4;
} __attribute__((packed));

#define REGDB (128)
struct registrations registrations[REGDB];

static void addreg(void *arg)
{
  char *name, *sflags, *ipv4, *p;
  int h, flags;
  char *line = (char *)(arg);
  
  name = line;
  p = strchr(line, ' ');
  if(! p) goto bail;
  *p++ = 0;
  sflags = p;
  p = strchr(p, ' ');
  if(! p) goto bail;
  *p++ = 0;
  ipv4 = p;

  flags = atoi(sflags);
  if(flags & ~0xe0) goto bail;

  h = hash(name, strlen(name), REGDB-1);
  registrations[h].flags = flags;
  registrations[h].ipv4 = inet_addr(ipv4);

  printf("registration added successfully\n");

bail:
  free(line);
}

static void senddb(void *arg)
{
  unsigned char buffer[512], *p;
  char *host, *l;
  char *line = (char *)(arg);
  int port;
  int fd;
  int i;
  int sz;

  p = buffer;
  sz = sizeof(buffer);
  host = line;
  l = strchr(line, ' ');
  if(! l) goto bail;
  *l++ = 0;
  port = atoi(l);
  if(port == 0) goto bail;

  printf("sending db\n");

  if((fd = netdial(UDP, host, port)) < 0) goto bail;

  for(sz = 0, p = buffer, i = 0; i < REGDB; i++) {
      if(registrations[i].flags | registrations[i].ipv4) {
          memcpy(p, &registrations[i], sizeof(struct registrations));
          p += sizeof(struct registrations);
          sz += sizeof(struct registrations);
      }
  }
bail:
  fdwrite(fd, buffer, sz);
  close(fd);
  free(line);
}

int get_and_hash(int maxsz, char *string, char separator)
{
  char name[32];
  int i;
  
  if(maxsz > 32) return 0;

  for(i = 0; i < maxsz, string[i]; i++) {
      if(string[i] == separator) break;
      name[i] = string[i];
  }

  return hash(name, strlen(name), 0x7f);
}


struct isuparg {
  int fd;
  char *string;
};


static void checkname(void *arg)
{
  struct isuparg *isa = (struct isuparg *)(arg);
  int h;

  h = get_and_hash(32, isa->string, '@');
  
  fdprintf(isa->fd, "%s is %sindexed already\n", isa->string, registrations[h].ipv4 ? "" : "not ");

}

static void isup(void *arg)
{
  unsigned char buffer[512], *p;
  char *host, *l;
  struct isuparg *isa = (struct isuparg *)(arg);
  int port;
  int fd;
  int i;
  int sz;

  // skip over first arg, get port
  l = strchr(isa->string, ' ');
  if(! l) return;
  *l++ = 0;

  port = atoi(l);
  host = malloc(64);

  for(i = 0; i < 128; i++) {
      p = (unsigned char *)(& registrations[i]);
      if(! registrations[i].ipv4) continue;

      sprintf(host, "%d.%d.%d.%d",
          (registrations[i].ipv4 >> 0) & 0xff,
          (registrations[i].ipv4 >> 8) & 0xff,
          (registrations[i].ipv4 >> 16) & 0xff,
          (registrations[i].ipv4 >> 24) & 0xff);

      if((fd = netdial(UDP, host, port)) < 0) {
          continue;
      }

      buffer[0] = 0xc0;
      memcpy(buffer + 1, p, sizeof(struct registrations));
      buffer[5] = buffer[6] = buffer[7] = 0;

      fdwrite(fd, buffer, 8);

      close(fd);
  }

  free(host);
}

static void childtask(void *arg)
{
  int cfd = (int)(arg);
  char buffer[512], *n;
  int r;
  

  n = "** welcome to level05 **\n";

  if(fdwrite(cfd, n, strlen(n)) < 0) goto bail;

  while(1) {
      if((r = fdread(cfd, buffer, 512)) <= 0) goto bail;

      n = strchr(buffer, '\r');
      if(n) *n = 0;
      n = strchr(buffer, '\n');
      if(n) *n = 0;

      if(strncmp(buffer, "addreg ", 7) == 0) {
          taskcreate(addreg, strdup(buffer + 7), STACK);
          continue;
      }

      if(strncmp(buffer, "senddb ", 7) == 0) {
          taskcreate(senddb, strdup(buffer + 7), STACK);
          continue;
      }

      if(strncmp(buffer, "checkname ", 10) == 0) {
          struct isuparg *isa = calloc(sizeof(struct isuparg), 1);

          isa->fd = cfd;
          isa->string = strdup(buffer + 10);

          taskcreate(checkname, isa, STACK);
          continue;
      }
  
      if(strncmp(buffer, "quit", 4) == 0) {
          break;
      }

      if(strncmp(buffer, "isup ", 5) == 0) {
          struct isuparg *isa = calloc(sizeof(struct isuparg), 1);
          isa->fd = cfd;
          isa->string = strdup(buffer + 5);
          taskcreate(isup, isa, STACK);
      }
  }

bail:
  close(cfd);
}

void taskmain(int argc, char **argv)
{
  int fd, cfd;
  char remote[16];
  int rport;

  signal(SIGPIPE, SIG_IGN);
  background_process(NAME, UID, GID);

  if((fd = netannounce(TCP, 0, PORT)) < 0) {
      fprintf(stderr, "failure on port %d: %s\n", PORT, strerror(errno));
      taskexitall(1);
  }

  fdnoblock(fd);

  while((cfd = netaccept(fd, remote, &rport)) >= 0) {
      fprintf(stderr, "accepted connection from %s:%d\n", remote, rport);
      taskcreate(childtask, (void *)(cfd), STACK);
  }

}
```

## Level06

### About

| Protections                         |      |
|:------------------------------------|:----:|
| Vulnerability Type:                 | Stack|
| Position Independent Executable:    |   Yes|
| Read only relocations:              |    No|
| Non-Executable stack:               |   Yes|
| Non-Executable heap:                |   Yes|
| Address Space Layout Randomisation: |   Yes|
| Source Fortification:               |   Yes|

### Source Code:

```c
#define THREADED
#include "../common/common.c"

// Taken some code from gnu tls documentation, 
// This example is a very simple echo server which supports X.509
// authentication, using the RSA ciphersuites. 
// This file has the leading comment of... /* This example code is
// placed in the public domain. */
// so there :>

#include <gcrypt.h>
#include <gnutls/gnutls.h>

#include <libHX/init.h>
#include <libHX/defs.h>
#include <libHX/map.h>
#include <libHX/string.h>

#define KEYFILE "/opt/fusion/ssl/key.pem"
#define CERTFILE "/opt/fusion/ssl/cert.pem"
#define CAFILE "/opt/fusion/ssl/ca.pem"
#define CRLFILE "/opt/fusion/ssl/crl.pem"

gnutls_certificate_credentials_t x509_cred;
gnutls_priority_t priority_cache;

static gnutls_session_t
initialize_tls_session (void)
{
  gnutls_session_t session;

  gnutls_init (&session, GNUTLS_SERVER);

  gnutls_priority_set (session, priority_cache);

  gnutls_credentials_set (session, GNUTLS_CRD_CERTIFICATE, x509_cred);

  /*
   *request client certificate if any.
   */
  gnutls_certificate_server_set_request (session, GNUTLS_CERT_REQUEST);

  return session;
}


struct HXmap *dict;

struct data {
  void *data;
  size_t length;
};

struct data *gather_data(gnutls_session_t session, char *key, size_t length)
{
  unsigned char buffer[length];
      offset) > 65535 ? 65535 : (length - offset));
  int offset, ret;
  struct data *data;

  for(offset = 0; offset < length; ) {
    ret = gnutls_record_recv(session, buffer + offset, (length -
    if(ret <= 0) return NULL;
    offset += ret;
  }

  data = malloc(sizeof(struct data));
  if(! data) return NULL;
  data->data = HX_memdup(buffer, length);
  if(!data->data) {
    free(data);
    return NULL;
  }
  data->length = length;

  //printf("gather data: returning %08x, data->length = %d\n", data,
  // data->length);
  //fflush(stdout);

  return data;
}

#define NOKEY "// No key was specified\n"
#define NOTFOUND "// Key was not found\n"
#define KEYFOUND "// Key exists\n"
#define NOMEM "// Not enough memory to allocate\n"
#define UPDATEOK "// Updated successfully\n"

int update_data(gnutls_session_t session, char *key, size_t length)
{
  struct data *data;
  size_t offset;
  int ret;

  data = HXmap_get(dict, key);
  if(! data) {
    gnutls_record_send(session, NOTFOUND, strlen(NOTFOUND));
    return -1;
  }

  if(length > data->length) {
    void *tmp;
    tmp = realloc(data->data, length);
    if(! tmp) {
      gnutls_record_send(session, NOMEM, strlen(NOMEM));
      return -1;
    }
    data->data = tmp;
  }  

  for(offset = 0; offset < length; ) {
    ret = gnutls_record_recv(session, data->data + offset,
      (length - offset) > 65535 ? 65535 : (length - offset));
    if(ret <= 0) return 0;
    offset += ret;
  }

  gnutls_record_send(session, UPDATEOK, strlen(UPDATEOK));

  data->length = length;
  return 0;
}

int send_data(gnutls_session_t session, char *key, struct data *data)
{
  int offset, ret;
  int to_send;

  char *msg;

  asprintf(&msg, "// Sending %d bytes\n", data->length);
  gnutls_record_send(session, msg, strlen(msg));
  free(msg);

  for(offset = 0; offset < data->length; ) {
    int tosend;
    tosend = (data->length - offset) > 65535 ? 65535 :
      (data->length - offset);
    ret = gnutls_record_send(session, data->data + offset,
       tosend);
    if(ret <= 0) return -1;
    offset += ret;
  }
  return 0;
}

void *free_data(void *ptr)
{
  struct data *data;
  data = (struct data *)(ptr);

  //printf("in free data, got %08x\n", (unsigned int)data);
  if(data) {
    if(data->data) {
      free(data->data);
    }
    free(data);
  }
}

void new_dict()
{
  struct HXmap_ops mops;
  if(dict) HXmap_free(dict);
  
  memset(&mops, 0, sizeof(mops));
  mops.d_free = free_data;
  
  dict = HXmap_init5(HXMAPT_HASH, HXMAP_SKEY | HXMAP_CKEY, &mops,
    0, sizeof(struct data));
}


void *keyval_thread(void *arg)
{
  int fd = (int)arg;
  int ret;
  struct data *data;
  int cont;

  gnutls_session_t session;
  session = initialize_tls_session ();

  gnutls_transport_set_ptr (session, (gnutls_transport_ptr_t) fd);
  ret = gnutls_handshake (session);

  if (ret < 0) {
    char *msg;

    close (fd);
    gnutls_deinit (session);
  
    msg = NULL;
    asprintf(&msg, "*** Handshake has failed (%s)\n\n",
      gnutls_strerror(ret));
    write(fd, msg, strlen(msg));
    close(fd);
    free(msg);
        }

#define BANNER "// Welcome to KeyValDaemon. Type 'h' for help information\n"
  gnutls_record_send(session, BANNER, strlen(BANNER));

  cont = 1;
    char cmdbuf[512], *p;
  while(cont) {
    char *args[6], *msg;
    int argcnt, i;

    memset(cmdbuf, 0, sizeof(cmdbuf));
    ret = gnutls_record_recv(session, cmdbuf, sizeof(cmdbuf));
    if(ret <= 0) break;

    p = strchr(cmdbuf, '\r');
    if(p) *p = 0;
    p = strchr(cmdbuf, '\n');
    if(p) *p = 0;

    memset(args, 0, sizeof(args));
    argcnt = HX_split5(cmdbuf, " ", 6, args);

#if 0
    for(i = 0; i < argcnt; i++) {
      asprintf(&msg, "args[%d] = \"%s\"\n", i, args[i]);
      gnutls_record_send(session, msg, strlen(msg));
      free(msg);
    }
#endif



    switch(args[0][0]) {
      case 'h':
#define HELP \
"// f <key> - find entry and see if it exists\n" \
"// s <key> <bytes> - store an entry with key and <bytes> lenght of data\n" \
"// g <key> - read data from key\n" \
"// d <key> - delete key/data\n" \
"// X - delete all data and restart\n"
// XXX, loop over HXmap and display data?
  
        gnutls_record_send(session, HELP, strlen(HELP));
        break;
      case 'd':
        if(! args[1]) {
          gnutls_record_send(session, NOKEY, strlen(NOKEY));
        } else {
          void *data;

          data = HXmap_del(dict, args[1]);
          if(data) {
            gnutls_record_send(session, KEYFOUND,
              strlen(KEYFOUND));
          } else {
            gnutls_record_send(session, NOTFOUND,
              strlen(NOTFOUND));
          }
        }
        break;
      case 's': // set
        data = gather_data(session, args[1], atoi(args[2]));
        if(data != NULL) {
#define NEWKEY "// New key added!\n"
          printf("args[1] = %08x/%s, data = %08x\n",
            args[1], args[1], data);
          HXmap_add(dict, args[1], data);
          gnutls_record_send(session, NEWKEY,
            strlen(NEWKEY));
        } else {
#define ADDERROR "// Unable to add new entry, problem getting data\n"
          gnutls_record_send(session, ADDERROR,
            strlen(ADDERROR));
        }
        break;
      case 'u': // update
        update_data(session, args[1], atoi(args[2]));
        break;
      case 'f': // find
        if(! args[1]) {
          gnutls_record_send(session, NOKEY,
            strlen(NOKEY));
        } else {
          if(HXmap_find(dict, args[1]) == NULL) {
            gnutls_record_send(session,
            NOTFOUND, strlen(NOTFOUND));
          } else {
            gnutls_record_send(session,
            KEYFOUND, strlen(KEYFOUND));
          }
        }

        break;

      case 'g': // get
        if(! args[1]) {
          gnutls_record_send(session, NOKEY,
            strlen(NOKEY));
        } else {
          if((data = HXmap_get(dict, args[1]))
            == NULL) {
            gnutls_record_send(session, NOTFOUND,
            strlen(NOTFOUND));
          } else {
            send_data(session, args[1], data);
          }
        }
        break;
      case 'e':
        cont = 0;
        break;
      case 'X':
        new_dict();
#define NEWDICT "// New dictionary installed\n"
        gnutls_record_send(session, NEWDICT,
        strlen(NEWDICT));
        break;
      default:
#define UC "// Unknown Command, please see 'h' for help information\n"

        gnutls_record_send(session, UC, strlen(UC));
        break;
    }
  }

#define GB "// Good bye!\n"
  gnutls_record_send(session, GB, strlen(GB));
  gnutls_bye(session, GNUTLS_SHUT_WR);

  close(fd);
  gnutls_deinit(session);

  return NULL;
}

#define DH_BITS 512

static gnutls_dh_params_t dh_params;

static int generate_dh_params (void)
{
  /* 
   * Generate Diffie-Hellman parameters - for use with DHE
   * kx algorithms. When short bit length is used, it might
   * be wise to regenerate parameters.
   *
   */
  gnutls_dh_params_init (&dh_params);
  gnutls_dh_params_generate2 (dh_params, DH_BITS);

  return 0;
}

GCRY_THREAD_OPTION_PTHREAD_IMPL;

int main(int argc, char **argv)
{
  int fd, i;

  HX_init();

  gcry_control(GCRYCTL_SET_THREAD_CBS, &gcry_threads_pthread);
  gnutls_global_init();

  gnutls_certificate_allocate_credentials (&x509_cred);
  gnutls_certificate_set_x509_trust_file (x509_cred, CAFILE,
            GNUTLS_X509_FMT_PEM);

  gnutls_certificate_set_x509_crl_file (x509_cred, CRLFILE,
            GNUTLS_X509_FMT_PEM);

  gnutls_certificate_set_x509_key_file (x509_cred, CERTFILE, KEYFILE,
          GNUTLS_X509_FMT_PEM);

  generate_dh_params ();

  gnutls_priority_init (&priority_cache, "NORMAL", NULL);
  gnutls_certificate_set_dh_params (x509_cred, dh_params);

  new_dict();

  signal(SIGPIPE, SIG_IGN);

  background_process(NAME, UID, GID);  
  serve_forever_threaded(PORT, keyval_thread);
}

```

## Level07

### About

| Protections                         |      |
|:------------------------------------|:----:|
| Vulnerability Type:                 | Stack|
| Position Independent Executable:    |   Yes|
| Read only relocations:              |    No|
| Non-Executable stack:               |   Yes|
| Non-Executable heap:                |   Yes|
| Address Space Layout Randomisation: |   Yes|
| Source Fortification:               |   Yes|

### Source Code:

```c
#include "../common/common.c"  

#include <pth.h>
#include <openssl/rsa.h>

#include "utlist.h"

struct ops {
  void (*register_cmd)(unsigned int opcode, unsigned int flags, void *(*fp)(void *));
  void (*unregister_cmd)(unsigned int opcode);
};

int parse_pak(unsigned char *pakaddr, size_t paklen, size_t base, struct ops *ops);

#define DB (572)

int udp;

struct pa {
  unsigned char *buf;
  ssize_t len;
  struct sockaddr_in sin;

  unsigned char *p;
  ssize_t remainder;
  
};

void free_pa(struct pa *pa)
{
  if(! pa) return;

  if(! pa->buf) {
    memset(pa->buf, 0, pa->len);
    free(pa->buf);
  }

  memset(pa, 0, sizeof(struct pa));
  free(pa);
}

typedef struct cmdtab {
  unsigned int opcode;
  unsigned int flags;
  void *(* fp)(void *);
  struct cmdtab *prev, *next;
} cmdtab;

cmdtab *cmdtab_head;

void *dispatch(void *arg)
{
  struct pa *p = (struct pa *)(arg);
  int *ip;
  cmdtab *c = NULL;

  if(p->len < sizeof(int)) goto bail;

  ip = (int *)(p->buf);

  p->p = p->buf + 4;
  p->remainder = p->len - 4;

  DL_FOREACH(cmdtab_head, c) {
    if(c->opcode == ip[0]) {
      c->fp(p);
      break;
    }
  }

bail:
  free_pa(p);
  return NULL;
}

void register_cmd(unsigned int opcode, unsigned int flags, void *(*fp)(void *))
{
  cmdtab *c;

  c = calloc(1, sizeof(cmdtab));
  c->opcode = opcode;
  c->flags = flags;
  c->fp = fp;

  DL_APPEND(cmdtab_head, c);
}

void unregister_cmd(unsigned int opcode)
{
  cmdtab *c, *tmp;

  DL_FOREACH_SAFE(cmdtab_head, c, tmp) {
    if(c->opcode == opcode) {
      DL_DELETE(cmdtab_head, c);
    }
  }
}


struct ops regops = {
  .register_cmd = register_cmd,
  .unregister_cmd = unregister_cmd
};

#define PAKFILE "/opt/fusion/res/level07.pak"

void load_and_parse_default_pak()
{
  void *m;
  int fd;
  struct stat statbuf;
  int status;
  unsigned int base;

  fd = open(PAKFILE, O_RDONLY);
  if(! fd) err(1, "Unable to open %s", PAKFILE);
  if(fstat(fd, &statbuf) == -1) err(1, "Unable to fstat %s", PAKFILE);

  m = mmap(NULL, statbuf.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
  if(m == MAP_FAILED) err(1, "Unable to mmap %s", PAKFILE);

  // printf("got %d bytes to process\n", statbuf.st_size);

  status = parse_pak(m, statbuf.st_size, 0, &regops);

  // printf("parse_pak result: %08x\n", status);

}

int download_pak_file(char *host, char *port, unsigned char *key, unsigned char **pakfile, size_t *pakfile_len)
{
  struct sockaddr_in sin;
  size_t blue;
  int ret;
  size_t alloc;
  int status;
  int keyidx;
  int keylen;
  int i;
  int fd;

  status = -1;

  keylen = strlen(key);
  keyidx = 0;

  memset(&sin, 0, sizeof(struct sockaddr_in));

  sin.sin_addr.s_addr = inet_addr(host);
  sin.sin_port = htons(atoi(port));
  sin.sin_family = AF_INET;

  *pakfile = NULL;
  *pakfile_len = 0;

  fd = socket(AF_INET, SOCK_STREAM, 0);
  if(fd == -1) return;
  if(pth_connect(fd, (void *)(&sin), sizeof(struct sockaddr_in)) == -1) goto closefd;

  if(pth_read(fd, &alloc, sizeof(alloc)) != sizeof(alloc)) goto closefd;

  blue = 0;
  *pakfile = calloc(alloc, 1);
  if(*pakfile == NULL) goto closefd;
  *pakfile_len = alloc;

  while(alloc - blue) {
    ret = pth_read(fd, (*pakfile) + blue, alloc - blue);

    if(ret == -1) goto freemem;
    if(ret == 0) goto freemem;

    for(i = 0; i < ret; i++) {
      //printf("key byte is %02x/%c\n", key[keyidx], key[keyidx]);
      (*pakfile)[blue + i] ^= key[keyidx];
      keyidx = (keyidx + 1) % keylen;
    }

    blue += ret;
  }

  status = 0;
  goto closefd;

freemem:
  free(*pakfile);
  *pakfile = NULL;
  *pakfile_len = 0;
  
closefd:
  close(fd);
  return status;

}

void *load_new_pakfile(void *arg)
{
  struct pa *p = (struct pa *)(arg);
  unsigned char *q;
  unsigned char *host, *port, *key = NULL;
  unsigned char *pakfile;
  size_t pakfile_len;

  host = p->p;
  q = strchr(p->p, '|');
  if(! q) return NULL;
  *q++ = 0;
  port = q;
  q = strchr(q, '|');
  if(! q) return NULL;
  *q++;
  key = q;

  if(strlen(key) < 8) return NULL;

  // printf("key is '%s'\n", key);

  if(download_pak_file((char *)(host), (char *)(port), key, &pakfile, &&pakfile_len) == 0) {
    parse_pak(pakfile, pakfile_len, 0, &regops);
    free(pakfile);
  }

  return NULL;
}

void *execute_command(void *arg)
{
  struct pa *p = (struct pa *)(arg);
  if(fork() != 0) {
    system(p->p);
  }
}

int main(int argc, char **argv, char **envp)
{
  background_process(NAME, UID, GID);  

  pth_init();

  udp = get_udp_server_socket(PORT);

  register_cmd(1347961165, 0, load_new_pakfile);
  register_cmd(2280059729, 0, execute_command);

  load_and_parse_default_pak();

  while(1) {
    struct pa *p;
    int l;

    p = calloc(sizeof(struct pa), 1);
    p->buf = calloc(DB, 1);
    l = sizeof(struct sockaddr_in);
    p->len = pth_recvfrom(udp, p->buf, DB, 0, (void *)(&p->sin), &l); 

    pth_spawn(PTH_ATTR_DEFAULT, dispatch, p);
  }  
}

```

## Level08

### About

This process runs chrooted in it's home directory, can you break out of it?

As a side note, it's always interesting to watch how things fail, and what you can do with that :)

| Protections                         |      |
|:------------------------------------|:----:|
| Vulnerability Type:                 | Stack|
| Position Independent Executable:    |   Yes|
| Read only relocations:              |    No|
| Non-Executable stack:               |   Yes|
| Non-Executable heap:                |   Yes|
| Address Space Layout Randomisation: |   Yes|
| Source Fortification:               |    No|

### Source Code:

```c
#include "../common/common.c"  
#include <tcr/threaded_cr.h>
#include "crypto_box.h"

/*
 * Design:
 *
 * One stdin-reader
 * Many decryption processes
 * Many command parsers / executors
 * Many encryptor processes
 * One stdout-writer
 */


struct buffer {
  unsigned char *data;
  size_t len;
  CIRCLEQ_ENTRY(buffer) ll;
};

#define free_buf(x) \
  do { \
    if((x)) { \
      if((x)->data) free(x->data); \
      memset((x), 0xdf, sizeof(struct buffer)); \
      free((x)); \
      (x) = NULL; \
    } \
  } while(0)

unsigned char public_key[crypto_box_PUBLICKEYBYTES];
unsigned char secret_key[crypto_box_SECRETKEYBYTES];

static struct tc_waitq decryption_waitq;
static CIRCLEQ_HEAD(, buffer) decryption_queue;
static spinlock_t decryption_sl;

static struct tc_waitq process_waitq;
static CIRCLEQ_HEAD(, buffer) process_queue;
static spinlock_t process_sl;

static struct tc_waitq encryption_waitq;
static CIRCLEQ_HEAD(, buffer) encryption_queue;
static spinlock_t encryption_sl;

static struct tc_waitq output_waitq;
static CIRCLEQ_HEAD(, buffer) output_queue;
static spinlock_t output_sl;

int devurandom_fd = -1;


ssize_t tc_read(struct tc_fd *t, void *buf, ssize_t buflen)
{
  ssize_t ret, len = buflen;
  enum tc_rv rv;

  while (1) {
    ret = read(tc_fd(t), buf, len);
    if (ret > 0) {
      buf += ret;
      len -= ret;
      if (len == 0)
        return buflen;
    } else if (ret < 0) {
      if (errno != EAGAIN)
        return buflen - len ? buflen - len : ret;
    } else /* ret == 0 */
      return buflen - len;

    rv = tc_wait_fd_prio(EPOLLIN, t);
    if (rv != RV_OK) {
      errno = EINTR;
      return -1;
    }
  }
}

ssize_t tc_write(struct tc_fd *t, void *buf, ssize_t buflen)
{
  ssize_t ret, len = buflen;
  enum tc_rv rv;

  while (1) {
    ret = write(tc_fd(t), buf, len);
    if (ret > 0) {
      buf += ret;
      len -= ret;
      if (len == 0)
        return buflen;
    } else if (ret < 0) {
      if (errno != EAGAIN)
        return buflen - len ? buflen - len : ret;
    } else /* ret == 0 */
      return buflen - len;

    rv = tc_wait_fd_prio(EPOLLIN, t);
    if (rv != RV_OK) {
      errno = EINTR;
      return -1;
    }
  }
}

ssize_t randombytes(void *buf, ssize_t len)
{
  struct tc_fd *tcfd;
  ssize_t ret;
  ssize_t max;

  if(devurandom_fd == -1) errx(EXIT_FAILURE, "devurandom_fd has not been setup");
  max = len;
  while(len) {
    ret = read(devurandom_fd, buf, len);
    if(ret == -1) {
      if(errno == EAGAIN || errno == EINTR) continue;
      err(EXIT_FAILURE, "randombytes failed");
    }

    buf += ret;
    len -= ret;
  }

  return max;
}

static void read_stdin(void *arg)
{
  struct buffer *buf;

  struct tc_fd *tcfd = (struct tc_fd *)(arg);

  ssize_t rr;
  size_t ltr;
  
  // printf("in read_stdin\n");

  while(1) {
    /* no clean ups since program will exit once this function returns */

    if(tc_read(tcfd, &ltr, sizeof(ltr)) != sizeof(ltr)) {
      fprintf(stderr, "unable to tc_read %d bytes\n", sizeof(ltr));
      break;
    }

    buf = calloc(sizeof(struct buffer), 1);
    if(! buf) {
      fprintf(stderr, "unable to calloc %d bytes\n", sizeof(struct buffer));
      break;
    }

    buf->data = malloc(ltr);
    if(! buf->data) {
      fprintf(stderr, "unable to malloc %d bytes\n", ltr);
      break;
    }

    buf->len = ltr;

    if(tc_read(tcfd, buf->data, ltr) != ltr) {
      fprintf(stderr, "Unable to tc_read %d bytes\n", ltr);
      break;
    }

    /* Insert the buffer into the queue */

    spin_lock(&decryption_sl);
    CIRCLEQ_INSERT_TAIL(&decryption_queue, buf, ll);
    spin_unlock(&decryption_sl);

    /* And wake up one of them */

    tc_waitq_wakeup_one(&decryption_waitq);
  }


}

unsigned char peer_remote_pk[crypto_box_PUBLICKEYBYTES];

static void decryption_worker(void *arg)
{
  // XXX, exit_all
  struct buffer *input_buffer;
  struct buffer *output_buffer;

  size_t tl;
  unsigned char *tmpout;

  while(1) {
    input_buffer = output_buffer = NULL;
    tmpout = NULL;

    // printf("decryption_worker: about to wait\n");

    // XXX, decryption_queue should probably be locked in this case :/

    if(tc_waitq_wait_event(&decryption_waitq, ! CIRCLEQ_EMPTY(&decryption_queue)) != RV_OK) {
      fprintf(stderr, "decryption_worker: failed to wait\n");
      break;
    }

    // printf("decryption_worker: removing from list\n");

    spin_lock(&decryption_sl);
    input_buffer = CIRCLEQ_FIRST(&decryption_queue);
    CIRCLEQ_REMOVE(&decryption_queue, input_buffer, ll);
    spin_unlock(&decryption_sl);

    // printf("decryption_worker: processing\n");

    // XXX, valgrind this :p
    // XXX, more sanity checks? :p

    // calculate how many bytes will be processed.
    tl = input_buffer->len - crypto_box_NONCEBYTES;

    tmpout = malloc(tl);
    if(! tmpout) {
      fprintf(stderr, "decryption_worker: unable to malloc %d bytes for tmpout, skipping\n", tl);
      goto failure;
    }

    output_buffer = calloc(sizeof(struct buffer), 1);
    if(! output_buffer) {
      fprintf(stderr, "decryption_worker: unable to calloc new buffer, skipping\n");
      goto failure;
    }
    output_buffer->len = tl - crypto_box_ZEROBYTES;
    output_buffer->data = malloc(output_buffer->len);
    if(! output_buffer->data) {
      fprintf(stderr, "decryption_worker: unable to malloc new data buffer of "
      "%d bytes, skipping\n", tl - crypto_box_ZEROBYTES);
      goto failure;
    }

    // printf("attempting to decrypt with length of %d bytes\n", tl);

    if(crypto_box_open(tmpout, input_buffer->data + crypto_box_NONCEBYTES, tl, input_buffer->data,
      peer_remote_pk, secret_key) != 0) {
      fprintf(stderr, "decryption_worker: unable to crypto_box_open :s, skipping\n");
      goto failure;
    }

    // printf("decryption_worker: outputting buffer\n");

    memcpy(output_buffer->data, tmpout + crypto_box_ZEROBYTES, output_buffer->len);
    
    /* Insert the buffer into the queue */

    spin_lock(&process_sl);
    CIRCLEQ_INSERT_TAIL(&process_queue, output_buffer, ll);
    spin_unlock(&process_sl);

    /* And wake up one of them */

    tc_waitq_wakeup_one(&process_waitq);

    // printf("decryption_worker: finished, signaled process worker\n");

    goto alright;
failure:
    free_buf(output_buffer);
alright:
    free_buf(input_buffer);
    if(tmpout) free(tmpout);
  }
}

static int sanity_check_name(char *s)
{
  char buf[256];

  int error;
   error = 0;

  error |= (!! strstr(s, "ssh"));
  error |= (!! strstr(s, "/."));

  memset(buf, 0, sizeof(buf));
  if((realpath(s, buf) == NULL) && errno != ENOENT) error |= 1;

  error |= (buf[0] == '.');

  return error;
}

#define set_str_buffer(tag, buf, string) \
do { \
  size_t *t; \
  buf->len = sizeof(size_t) + strlen(string); \
  buf->data = malloc(buf->len + 1); \
  if(!buf->data) { \
    fprintf(stderr, "process_worker (set_str_buffer): failure to malloc buffer->data\n"); \
    goto failure; \
  } \
  t = (size_t *)(buf->data); \
  t[0] = tag; \
  strcpy(buf->data + 4, string); \
} while(0)



static void process_worker(void *arg)
{
  struct buffer *input_buffer;
  struct buffer *output_buffer;
  size_t *tag;
  int ret;
  mode_t mode;
  unsigned char *f, *g, *h;
  int fd;
  char msg[64];
  size_t offset;
  int len;
  struct tc_fd *tcfd;


  while(1) {
    input_buffer = NULL;

    if(tc_waitq_wait_event(&process_waitq, ! CIRCLEQ_EMPTY(&process_queue)) != RV_OK) {
      fprintf(stderr, "process_worker: failed to wait\n");
      break;
    }

    spin_lock(&process_sl);
    input_buffer = CIRCLEQ_FIRST(&process_queue);
    CIRCLEQ_REMOVE(&process_queue, input_buffer, ll);
    spin_unlock(&process_sl);

    // printf("process_worker: got one!\n");

    output_buffer = calloc(sizeof(struct buffer), 1);
    if(! output_buffer) {
      fprintf(stderr, "process_worker: can't allocate output buffer :(\n");
      goto append_output;
    }

    tag = ((size_t *)(input_buffer->data))[0];

    if(input_buffer->len < 4) {
      set_str_buffer(-1, output_buffer, "input packet length specified");
      goto append_output;
    }

    // printf("process_worker: handling %02x .. input is %s\n", input_buffer->data[4],
    // input_buffer->data + 4);
    
    switch(input_buffer->data[4]) {
      case 'm':
        // make a directory
        mode = strtoul(input_buffer->data + 5, (char **) &f, 8);
        if(f == (input_buffer->data + 5)) {
          printf("invalid mode\n");
          set_str_buffer(tag, output_buffer, "invalid mode specified");
          break;
        }
        if(f[0] == '\0') {
          printf("no filename\n");
          set_str_buffer(tag, output_buffer, "no filename specified");
          break;
        }

        if(sanity_check_name(f) != 0) {
          printf("filename specification\n");
          set_str_buffer(tag, output_buffer, "invalid filename specified");
          break;
        }

        ret = mkdir(f, mode);
        set_str_buffer(tag, output_buffer, ret == -1 ? "error in creating directory" : 
          "successfully created directory");

        break;
      case 'o':
        // open specified file, with mode.
        mode = strtoul(input_buffer->data + 5, (char **) &f, 8);
        if(f == (input_buffer->data + 5)) {
          set_str_buffer(tag, output_buffer, "invalid mode specified");
          break;
        }
        if(f[0] == '\0') {
          printf("input = %s\n", (input_buffer->data + 5));
          set_str_buffer(tag, output_buffer, "no filename specified");
          break;
        }

        if(sanity_check_name(f) != 0) {
          set_str_buffer(tag, output_buffer, "invalid filename specified");
          break;
        }

        fd = open(f, O_CREAT|O_TRUNC|O_WRONLY, mode);
        if(fd == -1) {
          set_str_buffer(tag, output_buffer, "failed to open file");
          break;
        }

        sprintf(msg, "fd is %d", fd);

        set_str_buffer(tag, output_buffer, msg);

        break;

      case 'w':
        // write to file, offset, data

        fd = strtoul(input_buffer->data + 5, (char **)&f, 10);
        if(f == (input_buffer->data + 5)) {
          set_str_buffer(tag, output_buffer, "invalid fd specified");
          break;
        }

        if(f[0] != ',') {
          set_str_buffer(tag, output_buffer, "invalid protocol");
          break;
        }

        f++;  

        offset = strtoul(f, (char **) &g, 10);
        if(f == g) {
          set_str_buffer(tag, output_buffer, "no fd specified");
          break;
        }
        len = input_buffer->len - ((unsigned int)(g) - (unsigned int)(input_buffer->data));

        while(len) {
          size_t ret;

          ret = pwrite(fd, g, len, offset);
          if(ret == -1) {
            if(errno == EAGAIN || errno == EINTR) continue;
            break;

          }

          g += ret;
          len -= ret;
        }

        set_str_buffer(tag, output_buffer, len == 0 ? "successfully wrote contents to fd\x00" : 
          "failed to write contents to fd\x00");

        break;

      case 'c':
        // close file

        fd = strtoul(input_buffer->data + 5, (char **)&f, 10);
        if(f == (input_buffer->data + 5)) {
          set_str_buffer(tag, output_buffer, "no fd specified");
          break;
        }

        if(f[0] != '\0') {
          set_str_buffer(tag, output_buffer, "fd not specified");
          break;
        }

        set_str_buffer(tag, output_buffer, "fd closed");
        close(fd);

        break;
      default:
        set_str_buffer(tag, output_buffer, "malformed input");
        break;

    }

append_output:
    spin_lock(&encryption_sl);
    CIRCLEQ_INSERT_TAIL(&encryption_queue, output_buffer, ll);
    spin_unlock(&encryption_sl);
    tc_waitq_wakeup_one(&encryption_waitq);

failure:
    free_buf(input_buffer);
  }
}

static void encryption_worker(void *arg)
{
  struct buffer *input_buffer;
  struct buffer *output_buffer;
  unsigned char *tmp;

  while(1) {
    tmp = NULL;

    // dequeue, and remove
    if(tc_waitq_wait_event(&encryption_waitq, ! CIRCLEQ_EMPTY(&encryption_queue)) != RV_OK) {
      fprintf(stderr, "encryption_worker: failed to wait\n");
      break;
    }

    spin_lock(&encryption_sl);
    input_buffer = CIRCLEQ_FIRST(&encryption_queue);
    CIRCLEQ_REMOVE(&encryption_queue, input_buffer, ll);
    spin_unlock(&encryption_sl);

    output_buffer = calloc(sizeof(struct buffer), 1);
    if(output_buffer == NULL) goto failure;
    output_buffer->len = input_buffer->len + crypto_box_ZEROBYTES + crypto_box_NONCEBYTES;
    output_buffer->data = malloc(output_buffer->len);
    if(output_buffer->data == NULL) goto failure;

    randombytes(output_buffer->data, crypto_box_NONCEBYTES);

    tmp = malloc(input_buffer->len + crypto_box_ZEROBYTES);
    if(! tmp) goto failure;


    // printf("input_buffer is %08x\n", input_buffer);
    // printf("input_buffer->data = %s, input_buffer->len = %d\n", input_buffer->data, input_buffer->len);

    memset(tmp, 0, crypto_box_ZEROBYTES);
    memcpy(tmp + crypto_box_ZEROBYTES, input_buffer->data, input_buffer->len);    

    if(crypto_box(output_buffer->data + crypto_box_NONCEBYTES, tmp, crypto_box_ZEROBYTES + input_buffer->len,
      output_buffer->data, peer_remote_pk, secret_key) != 0) {
      fprintf(stderr, "crypto_box failed\n");
      goto failure;
    }

    spin_lock(&output_sl);
    CIRCLEQ_INSERT_TAIL(&output_queue, output_buffer, ll);
    spin_unlock(&output_sl);
    tc_waitq_wakeup_one(&output_waitq);
    
    goto alright;
failure:
    free_buf(output_buffer);
alright:
    free_buf(input_buffer);
    if(tmp) { free(tmp); tmp = NULL; }
  }
}

static void write_stdout(void *arg)
{
  struct tc_fd *tcfd = (struct tc_fd *)(arg);
  struct buffer *buffer;
  size_t len;

  while(1) {
    if(tc_waitq_wait_event(&output_waitq, ! CIRCLEQ_EMPTY(&output_queue)) != RV_OK) {
      fprintf(stderr, "output_worker: failed to wait\n");
      break;
    }

    spin_lock(&output_sl);
    buffer = CIRCLEQ_FIRST(&output_queue);
    CIRCLEQ_REMOVE(&output_queue, buffer, ll);
    spin_unlock(&output_sl);

    len = buffer->len;

    if(tc_write(tcfd, &len, sizeof(size_t)) != sizeof(size_t)) {
      errx(1, "failed to write");
      // xxx, return to tc_main
    }

    if(tc_write(tcfd, buffer->data, buffer->len) != buffer->len) {
      errx(1, "failed to write data");
    }

    free_buf(buffer);

  }
}

static void tc_main(void *arg)
{
  struct tc_thread *reader;
  struct tc_thread_pool decryption_pool;
  struct tc_thread_pool process_pool;
  struct tc_thread_pool encryption_pool;
  struct tc_thread *writer;
  struct tc_fd *tcfd;

  CIRCLEQ_INIT(&decryption_queue);
  spin_lock_init(&decryption_sl);
  tc_waitq_init(&decryption_waitq);

  CIRCLEQ_INIT(&process_queue);
  spin_lock_init(&process_sl);
  tc_waitq_init(&process_waitq);

  CIRCLEQ_INIT(&encryption_queue);
  spin_lock_init(&encryption_sl);
  tc_waitq_init(&encryption_waitq);

  CIRCLEQ_INIT(&output_queue);
  spin_lock_init(&output_sl);
  tc_waitq_init(&output_waitq);

  tcfd = tc_register_fd(fileno(stdin));

  if(write(fileno(stdin), public_key, crypto_box_PUBLICKEYBYTES) != crypto_box_PUBLICKEYBYTES)
    err(EXIT_FAILURE, "writing public key");

  if(tc_read(tcfd, peer_remote_pk, sizeof(peer_remote_pk)) != sizeof(peer_remote_pk)) {
    fprintf(stderr, "failed to read public key\n");
    goto failure;
  }

  tc_thread_pool_new(&decryption_pool, decryption_worker, NULL, "decryption_worker_%d", 0);
  tc_thread_pool_new(&process_pool, process_worker, NULL, "process_worker_%d", 0);
  tc_thread_pool_new(&encryption_pool, encryption_worker, NULL, "encryption_worker_%d", 0);

  if((reader = tc_thread_new(read_stdin, tcfd, "stdin_reader")) == NULL) {
    errx(1, "tc_thread_new failed to create stdin_reader");
  }

  if((writer = tc_thread_new(write_stdout, tcfd, "stdout_writer")) == NULL) {
    errx(1, "tc_thread_new failed to create stdout_writer");
  }

  tc_thread_wait(reader);
    
failure:
  tc_unregister_fd(tcfd);

  // signal exit

  // printf("waited on read_stdin\n");

}

int main(int argc, char **argv, char **envp)
{
  int fd;
  char *p;

  if(is_restarted_process(argc, argv)) {
    devurandom_fd = open("/dev/urandom", O_RDONLY);

    crypto_box_keypair(public_key, secret_key);

    if(chroot("/home/level08") == -1) {
      err(1, "Unable to chroot");
    }

    if(chdir("/") == -1) {
      err(1, "chdir(\"/\")");
    }

    drop_privileges(UID, GID);

    tc_run(tc_main, NULL, "tc_main", sysconf(_SC_NPROCESSORS_ONLN));
    return 0;    
  }

  if(daemon(0, 0) == -1) {
    err(EXIT_FAILURE, "Unable to daemon(0, 0)");
  }

  fd = serve_forever(PORT);
  set_io(fd);
  restart_process(NAME);

}
```

## Level09

### About

This level is another reverse engineering level, this time bundled with some elementary binary packing to make things more fun.

This level follows on from level08 in that you can sometimes turn things completely in your favour, like turning an otherwise annoying function call into a no-op :)

| Protections                         |      |
|:------------------------------------|:----:|
| Vulnerability Type:                 | Stack|
| Position Independent Executable:    |    No|
| Read only relocations:              |    No|
| Non-Executable stack:               |   Yes|
| Non-Executable heap:                |   Yes|
| Address Space Layout Randomisation: |   Yes|
| Source Fortification:               |    No|

### Source Code:

```c
There is no source code available for this level
```

## Level10

### About

Introductory format string level that covers basic expansion.

| Protections                         |      |
|:------------------------------------|:----:|
| Vulnerability Type:                 |Format|
| Position Independent Executable:    |   Yes|
| Read only relocations:              |    No|
| Non-Executable stack:               |   Yes|
| Non-Executable heap:                |   Yes|
| Address Space Layout Randomisation: |   Yes|
| Source Fortification:               |   Yes|

### Source Code:

```c
#include "../common/common.c"

void expand_the_input()
{
  volatile int target;
  char output[1024];
  char input[12];

  target = 0;
  memset(input, 0, sizeof(input));
  memset(output, 0, sizeof(output));

  fgets(input, sizeof(input)-1, stdin);
  if(strlen(input) == 0) exit(0);
  
  sprintf(output, input);

  if(target == 0xdea110c8) {
    printf("\n[ critical hit! :> ]\n");
    system("exec /bin/sh");
    exit(0);
  }

  printf("\n[ target contains 0x%08x, wanted 0xdea110c8 ]\n", target);
  exit(0);
}

int main(int argc, char **argv, char **envp)
{
  int fd;
  char *p;

  background_process(NAME, UID, GID);  
  fd = serve_forever(PORT);
  set_io(fd);

  expand_the_input();
}
```

## Level11

### About

Another warm up level that covers writing arbitrary values to memory.

| Protections                         |      |
|:------------------------------------|:----:|
| Vulnerability Type:                 |Format|
| Position Independent Executable:    |    No|
| Read only relocations:              |    No|
| Non-Executable stack:               |   Yes|
| Non-Executable heap:                |   Yes|
| Address Space Layout Randomisation: |   Yes|
| Source Fortification:               |    No|

### Source Code:

```c
#include "../common/common.c"

int target;

void expand_the_input()
{
  char input[256];
  
  target = 0;
  memset(input, 0, sizeof(input));

  fgets(input, sizeof(input)-1, stdin);
  if(strlen(input) == 0) exit(0);

  printf(input);  

  if(target == 0x0ddba11) {
    printf("\n[ critical hit! :> ]\n");
    system("exec /bin/sh");
    exit(0);
  }
}

int main(int argc, char **argv, char **envp)
{
  int fd;
  char *p;

  background_process(NAME, UID, GID);  
  fd = serve_forever(PORT);
  set_io(fd);

  while(1) {
    printf("[ &target = 0x%08x, we want 0x0ddba11, currently is 0x%0x ]\n", &target, target);
    expand_the_input();
  }

}
```

## Level12

### About

There is no description available for this level.

| Protections                         |      |
|:------------------------------------|:----:|
| Vulnerability Type:                 | Stack|
| Position Independent Executable:    |    No|
| Read only relocations:              |    No|
| Non-Executable stack:               |   Yes|
| Non-Executable heap:                |   Yes|
| Address Space Layout Randomisation: |   Yes|
| Source Fortification:               |    No|

### Source Code:

```c
#include "../common/common.c"
/*
 * The aim of this level is to redirect code execution by overwriting an entry in the global offset table.
 */

void callme()
{
  printf("Hmmm, how did this happen?\n");
  system("exec /bin/sh");
}

void echo(char *string)
{
  printf("You said, \"");
  printf(string);
  printf("\"\n");
  fflush(stdout);
}

int main(int argc, char **argv, char **envp)
{
  int fd;
  char *p;

  background_process(NAME, UID, GID);  
  fd = serve_forever(PORT);
  set_io(fd);

  printf("Basic echo server. Type 'quit' to exit\n");

  while(1) {
    char input[1024];
    memset(input, 0, sizeof(input));

    fgets(input, sizeof(input)-1, stdin);
    if(strlen(input) == 0 || strncmp(input, "quit", 4) == 0) {
      exit(0);
    }

    if((p = strchr(input, '\r')) != NULL) *p = 0;
    if((p = strchr(input, '\n')) != NULL) *p = 0;

    echo(input);
  }
}
```

## Level13

### About

There is no description available for this level.

| Protections                         |      |
|:------------------------------------|:----:|
| Vulnerability Type:                 |Format|
| Position Independent Executable:    |    No|
| Read only relocations:              |    No|
| Non-Executable stack:               |    No|
| Non-Executable heap:                |    No|
| Address Space Layout Randomisation: |    No|
| Source Fortification:               |    No|

### Source Code:

```c
There is no source code available for this level
```

## Level14

### About

There is no description available for this level.

| Protections                         |      |
|:------------------------------------|:----:|
| Vulnerability Type:                 |  Heap|
| Position Independent Executable:    |    No|
| Read only relocations:              |    No|
| Non-Executable stack:               |    No|
| Non-Executable heap:                |    No|
| Address Space Layout Randomisation: |   Yes|
| Source Fortification:               |    No|

### Source Code:

```c
There is no source code available for this level
```
