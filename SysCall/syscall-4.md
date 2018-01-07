리눅스 커널에서 시스템 콜. Part 4.
================================================================================

리눅스 커널은 어떻게 프로그램을 수행할까?
--------------------------------------------------------------------------------

리눅스 커널에서 [system calls](https://en.wikipedia.org/wiki/System_call)을 기술하는 4번째 [챕터](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/SysCall/README.md)이고 이전 챕터에서 두 가지 개념을 언급하고 마무리 하였다.:

* `vsyscall`;
* `vDSO`;

시스템 콜 개념과 매우 유사하며 연관된 것들이다.

이 파트는 이 챕터의 마지막 파트가 될 것이고 너는 이 파트의 제목과 같은 내용을 이해할 수 있을 것이다. - 우리는 리눅스 커널에서 프로그램이 수행 될 때를 살펴 볼 것이다. 자, 시작해보자.

어떻게 우리의 프로그램이 시작되는가?
--------------------------------------------------------------------------------

사용자 관점에서 응용 프로그램을 시작하는 방법은 다양하게 많다. 예를 들어, [shell](https://en.wikipedia.org/wiki/Unix_shell)에서 수행시키거나, 응용 프로그램의 아이콘을 더블 클릭 함으로써 시작할 수 있을 것이다. 그것은 중요치 않다. 리눅스 커널은 우리가 어떻게 응용 프로긓램을 수행하는 지는 상관없이 응용 프로그램을 시하게 해준다.

이 파트에서 우리는 shell 에서 응용프로그램을 단지 수행할 때만을 고려할 것이다. 이미 알겠지만, 일반적으로 shell 에서 응용프로그램을 수행하는 방법은 다음과 같다.: 우리는 단지 [터미널 애뮬레이터](https://en.wikipedia.org/wiki/Terminal_emulator) 을 실행하고, 단지 우리가 실행하고자 하는 프로그램의 이름을 타이핑하고 인자가 필요하다면 어주면 끝이다. 예를 들면,:

![ls shell](http://s14.postimg.org/d6jgidc7l/Screenshot_from_2015_09_07_17_31_55.png)

shell 에서 응용프로그램을 수행했을 때, 프로그램 이름을 쓰면 shell 은 무엇을 하고, 리눅스 커널은 무엇을 하는지 고려해보자. 하지만 그전에, 우리는 흥미로운 몇가지를 먼저 살펴 볼 것이다. 이 책은 리눅스 커널에 관련된 내용을 기술한다는 것을 잊지 말아줬으면 한다. 그래서 리눅스 커널과 가장 관련 깊은 내용을 위주로 볼 것이라는 것도 말이다. 우리는 shell 이 무엇을 하고, 복잡한 상황에 대한-예를 들면 subshell 등- 내용은 고려하지 않을 것이다.

나의 기본 shell은 [bash](https://en.wikipedia.org/wiki/Bash_%28Unix_shell%29) 이다. 그리고 나는 어떻게 bash shell 이 프로그램을 수행하는지 고려할 것이다. 시작해보자. `bash` shell 은 어떤 C 언어로 작성된 프로그램의 [main](https://en.wikipedia.org/wiki/Entry_point) 함수에서 시작하게 해준다. 만약 당신이 `bash` shell 의 소스 코드를 보길 원한다면, 당신은 [shell.c](https://github.com/bminor/bash/blob/master/shell.c#L357) 소스 코드의 `main`함수를 찾아야 할 것이다. 이 함수는 `bash` 가 수행하기 시작하여 main thread loop 을 수행하기 전에 많은 일을 한다. 예를 들어, 이 함수는:

* `/dev/tty` 을 확인하고 연다;
* debug 모드에서 shell 이 수되는지 확인한다.;
* 커맨드 라인의 인자들을 넘겨준다.;
* shell 환경 값을 읽는다.;
* `.bashrc`, `.profile` 와 다른 설정 파일을 로드한다.;
* 그리고 다른 많은 일...

모든 일들을 다 수행하면, 우리는 `reader_loop` 함수의 호출을 볼 수 있다. 이 함수는 [eval.c](https://github.com/bminor/bash/blob/master/eval.c#L67) 소스 파일에 정의되어 있으며, main thread loop 혹은 다른 말로 명령어를 읽고 수행한다는 것이다. `reader_loop` 함수는 프로그램 이름과 인자를 확인하고 읽는다, 그것은 [execute_cmd.c](https://github.com/bminor/bash/blob/master/execute_cmd.c#L378) 소스 파일에 있는 `execute_command` 함수를 호출한다. `execute_command` 함수는 함수들의 호출의 연결고를 통해 수행된다.:

```
execute_command
--> execute_command_internal
----> execute_simple_command
------> execute_disk_command
--------> shell_execve
```

`subshell` 를 수행하기 위해 필요한 을 하는 것처럼 내부 `bash` 함수인지 아닌지 등의 많은 확인들을 한다. 이미 언급했듯이, 우리는 리눅스 커널과 관련된 내용을 자세히 살펴 보지 않을 것이다. 이 과정의 마지막에는  `shell_execve` 함수에서 `execve` 시스템 콜을 호출하는 것이다.:

```C
execve (command, args, env);
```

`execve` 시스템 콜은 다음과 같은 signature 를 가진다.:

```
int execve(const char *filename, char *const argv [], char *const envp[]);   
```

그리고 주어진 파일 이름으로 프로그램을 주어진 인자와 [환경변수](https://en.wikipedia.org/wiki/Environment_variable) 함께 실행한다. 이 시스템 콜은 우리의 경우에만 적용된다.:

```
$ strace ls
execve("/bin/ls", ["ls"], [/* 62 vars */]) = 0

$ strace echo
execve("/bin/echo", ["echo"], [/* 62 vars */]) = 0

$ strace uname
execve("/bin/uname", ["uname"], [/* 62 vars */]) = 0
```

그래서, 사용자 응용프로그은(`bash`) 시스템 콜을 호출하고, 이 다음은 아시다시피 리눅스 커널에서 수행된다.

execve system call
--------------------------------------------------------------------------------

우리는 사용자 응용 프로그램에 의해 호출되는 시스템콜이 불리기 전에 준비사항과 시스템콜이 완료되는 과정을 두번째 파트에서 살펴 보았다. 우리는 이전 단계에서 `execve` 호출에서 멈추었다. 이 시스템 콜은 [fs/exec.c](https://github.com/torvalds/linux/blob/master/fs/exec.c) 소스 코드에 구현되어 있고, 3개의 인자를 받는다.:

```
SYSCALL_DEFINE3(execve,
		const char __user *, filename,
		const char __user *const __user *, argv,
		const char __user *const __user *, envp)
{
	return do_execve(getname(filename), argv, envp);
}
```

`execve` 의 구현은 단순하다. 단지 `do_execve` 함수의 결과를 반환해주는 일만 한다. `do_execve` 함수는 같은 소스 파일에 구현되어 있으며, 다음과 같은 일을 한다.:

* 주어진 인자들과 환경 변수들과 같은 사용자 영역의 데이터를 두개의 포인터로 할당 및 초기화 한다.
* `do_execveat_common` 함수의 결과를 반환한다.

그것의 구현을 보자.:

```C
struct user_arg_ptr argv = { .ptr.native = __argv };
struct user_arg_ptr envp = { .ptr.native = __envp };
return do_execveat_common(AT_FDCWD, filename, argv, envp, 0);
```

`do_execveat_common` 함수는 주요 일을 한다 - 그것은 새로운 프로그램을 실행한다. 이 함수는 비슷한 인자들을 받는다, 하지만 3개가 아니라 5개인 것을 확인할 수 있을 것이다. 이 다섯 개 인자중 첫번째는 실행된 응용 프로그램의 디렉토리를 표현하는 파일 디스크립터이다. 그리고 우리의 경우, `AT_FDCWD` 을 사용했는데, 이것의 의미는 주어진 경로 이름은 현재 호출 한 프로세스의 working 디렉토리를 기준으로 상대적으로 번역된다는 것이다. 5번째 인자는 플래그이다. 우리의 경우는 `0` 이다. 우리는 다음 단계에서 살펴 보도록 한다.:

 `do_execveat_common` 함수는 첫번째로 `filename` 포인터를 확인하고, 만약 그것이 `NULL` 이라면 종료한다. 이 다음에서는 수행중인 프로세서들이 제한의 개수를 넘지 않았는지  플래그을 확인한다.:

```C
if (IS_ERR(filename))
	return PTR_ERR(filename);

if ((current->flags & PF_NPROC_EXCEEDED) &&
	atomic_read(&current_user()->processes) > rlimit(RLIMIT_NPROC)) {
	retval = -EAGAIN;
	goto out_ret;
}

current->flags &= ~PF_NPROC_EXCEEDED;
```

만약 이 두가지 경우가 성공적으로 확인이 되면, `execve`의 수행 실패를 방지하기 위해 현재 프로세스의 플래그 중에 `PF_NPROC_EXCEEDED` 플래그를 설정되지 않은 상태로 한다. 당신은 다음으로 호출된 `unshare_files` 함수는 [kernel/fork.c](https://github.com/torvalds/linux/blob/master/kernel/fork.c) 에 구현되어 있고, 현재 태스크의 파일디스크립터를 공유하는 것을 중지한다.:

```C
retval = unshare_files(&displaced);
if (retval)
	goto out_ret;
```

우리는 execve 된 바이너리의 [file descriptor](https://en.wikipedia.org/wiki/File_descriptor) 의 잠재적인 누수(leak)를 제거하기 위해 이 함수를 호출한다. 다음 단계에서 우리는  `struct linux_binprm` 구조체([include/linux/binfmts.h](https://github.com/torvalds/linux/blob/master/linux/binfmts.h) 선언됨)인 `bprm` 의 준비를 시작한다. `linux_binprm` 구조체는 바이너리들이 로딩될 때 사용되는 인자들을 갖고 있기 위해 사용된다. 예를 들어, 그것은 `vm_area_struct` 타입인 `vma` 항목을 가지고 우리가 실행하여야 할 프로그램이 로드되기 위한 주어진 주소 공간내에서 연속적인 간격을 단일 메모리 공간으로 표현, `mm` 항목은 바이너리의 메모리 디스크립터, 메모리의 최상위를 가리키거나 다른 메모리 영역을 표시한다.

우선 우리는 이 구조체를 위한 메모리를 `kzalloc` 함수를 통해 할당하고, 할당이 잘되었는지 확인한다.:

```C
bprm = kzalloc(sizeof(*bprm), GFP_KERNEL);
if (!bprm)
	goto out_files;
```

이 다음에는 `prepare_bprm_creds` 함수 호출로 `binprm` credentials 을 준비한다.: 

```C
retval = prepare_bprm_creds(bprm);
	if (retval)
		goto out_free;

check_unsafe_exec(bprm);
current->in_execve = 1;
```

`binprm` credentials 의 초기화는 다른 말로 `linux_binprm` 구조체의 내부에 저장된 `cred` 구조체의 초기화를 하겠다는 것이다. `cred` 구조체는 태스크의 보안 문맥(security context)를 포함한다. 예를 들어, 태스크의 [real uid](https://en.wikipedia.org/wiki/User_identifier#Real_user_ID), 태스크의 real [guid](https://en.wikipedia.org/wiki/Globally_unique_identifier),  [virtual file system](https://en.wikipedia.org/wiki/Virtual_file_system) 수행을 위한 `uid` 그리고 `guid` 같은 것이다. 다음 단계는 `bprm` credentials 의 준비가 수행되면, 우리는 이제 `check_unsafe_exec` 함수를 통해 안전하게 프로그램을 실행 할 수 있고, 현재 프로세스를 `in_execve` 상태로 한다.

이 모든 수행이 끝나면, 우리는 `do_execveat_common` 함수로 부터 넘어온 플래그들을 확인하는 `do_open_execat` 수를 호출 하고,(`flags` 부분에 우리는 `0` 을 넘겨줬다는 것을 기억하자.) 디스크에서 실행가능한 파일을 찾아 열고, `noexec` 마운트 포인트로 부터 바이너리 파일이 로드될 것인지 확인(우리는 실행 가능한 바이너리를 포함할 수 없는 [proc](https://en.wikipedia.org/wiki/Procfs) 또는 [sysfs](https://en.wikipedia.org/wiki/Sysfs)에서 실행을 회피하기 위한 확인이 필요하다), `file` 구조체를 초기화 그리고 이 구조체의 포인터를 반환한다. 다음은 `sched_exec` 의 호출을 볼 수 있다.:

```C
file = do_open_execat(fd, filename, flags);
retval = PTR_ERR(file);
if (IS_ERR(file))
	goto out_unmark;

sched_exec();
```

`sched_exec` 함수는 새로운 프로그램이 실행할 가능한 최근 로드된 프로세서를 결정하고 현재 프로세스를 그곳으로 옮긴다.

이 다음에 우리는 주어진 실행 바이너리의 [file descriptor](https://en.wikipedia.org/wiki/File_descriptor) 를 확인한다. 우리는 우리의 바이너리 일이 `/` 심볼로 시작하는지 확인하거나 주어진 실행 바이너리의 경로가 호출된 프로세스에서 현재 working 디렉토리에 상대적인 경로로 해석되어야 하는지 확인한다.(즉 `AT_FDCWD` 인지 확인)

만얀 이 중 하나라도 확인이 된다면, 우리는 바이너리 파라미터의 파일이름을 설정한다.:
```C
bprm->file = file;

if (fd == AT_FDCWD || filename->name[0] == '/') {
	bprm->filename = filename->name;
}
```

그렇지 않다면, 만약 파일 이름이 비어 있어서 우리가 바이너리 파라미터 파일 이름을 주어진 파일 디스크립터가 참조하는 실행 파일의 실행을 의미하는 주어진 실행 바이너리의 파일이름에 의존적인 `/dev/fd/%d` 나 `/dev/fd/%d/%s` 로 설정해야 한다.:  

```C
} else {
	if (filename->name[0] == '\0')
		pathbuf = kasprintf(GFP_TEMPORARY, "/dev/fd/%d", fd);
	else
		pathbuf = kasprintf(GFP_TEMPORARY, "/dev/fd/%d/%s",
		                    fd, filename->name);
	if (!pathbuf) {
		retval = -ENOMEM;
		goto out_unmark;
	}

	bprm->filename = pathbuf;
}

bprm->interp = bprm->filename;
```

우리는 `bprm->filename` 만 아니라 프로그램 인터프리테의 이름을 포함하는 `bprm->interp` 에도 해줘야 한다는 것을 확인하자. 이제, 우리는 같은 이름을 써놓았는데, 나중에 프로그램의 바이너리 포멧에 의존적인 프로그램 인터프리터의 진짜 이름으로 업데이트 해야 할 것이다. 당신은 우리가 이미 `linux_binprm` 을 위해 `cred` 를 이미 준비했다는 것을 봤을 것이다. 다음 단계는 `linux_binprm` 다른 항목들의 초기화이다. 첫번째로 `bprm_mm_init` 함수를 호출하고 `bprm` 를 그 함수로 넘겨준다.:

```C
retval = bprm_mm_init(bprm);
if (retval)
	goto out_unmark;
```

`bprm_mm_init` 함수의 구현은 같은 소스 파일에 있으며, 함수 이름으로 그 역할을 짐작할 수 있다. 그것은 메모리 디스크립터, 즉 `mm_struct` 을 초기화를 진행한다. 이 구조체는 [include/linux/mm_types.h](https://github.com/torvalds/linux/blob/master/include/mm_types.h) 에 선언되어 있으며, 프로세스의 주소공간을 표현한다. 우리는 `bprm_mm_init` 함수의 구현을 살펴보지 않을 것이다. 이유는, 리눅스 커널 메모리 관리와 연관된 중요한 많은 부분에 대해 모르기 때문이다. 하지만 우리는 이 함수가 `mm_struct` 를 초기화 하고 임시 스택인 `vm_area_struct` 를 만든다는 정도만 알아두자.

이 다음에는 우리가 실행 바이너리에게 넘겨준 명령라인 인자의 개수, 환경 변수들의 개수를 계산하고 각 각 `bprm->argc` 와 `bprm->envc` 에 설정한다.:

```C
bprm->argc = count(argv, MAX_ARG_STRINGS);
if ((retval = bprm->argc) < 0)
	goto out;

bprm->envc = count(envp, MAX_ARG_STRINGS);
if ((retval = bprm->envc) < 0)
	goto out;
```

[같은](https://github.com/torvalds/linux/blob/master/fs/exec.c) 파일에 정의되어 있는 `count` 함수의 도움으로 이와 관련된 일을 수행 할 수 있고, `argv` 배열의 문자열의 개수도 계산다. `MAX_ARG_STRINGS` 매크로는 [include/uapi/linux/binfmts.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/binfmts.h) 에 선언되어 있으며, 매크로의 이름으로 그 역할을 짐작할 수 있다. 그것은 `execve` 시스템 로 넘어오는 문자열의 최대 개수를 지한다.:

```C
#define MAX_ARG_STRINGS 0x7FFFFFFF
```

우리가 명령 인자들과 환경 변수들의 수를 계산한 다음에, 우리는  `prepare_binprm` 함수를 호출한다. 우리는 이미 이전에 이와 비슷한 함수의 호출을 했다. 그 함수는 `prepare_binprm_cred` 였고 우리는 이 함수가 `linux_bprm` 에 있는 `cred` 구조체를 초기화 했다는 것을 기억 할 것이다. 이제  `prepare_binprm` 함수를 보자:

```C
retval = prepare_binprm(bprm);
if (retval < 0)
	goto out;
```

`linux_binprm` 구조체에 [inode](https://en.wikipedia.org/wiki/Inode) 로 부터 `uid`를 채우고 실행 바이너리로 부터 `128` 바이트를 읽다. 우리는 첫 `128` 바이트를 읽어 우리의 실행 파일의 타입을 확인할 것이다. 우리는 나중에 실행 파일의 나머지를 읽어 볼 것이다.  `linux_bprm` 구조체를 준비하고 나서 우리는 실행 바이너리 파의 파일 이름, 명령라인 인자들 그리고 경 변수들을 복사하여 `copy_strings_kernel` 함수 호출로 `linux_bprm` 에게 복사한다.:

```C
retval = copy_strings_kernel(1, &bprm->filename, bprm);
if (retval < 0)
	goto out;

retval = copy_strings(bprm->envc, envp, bprm);
if (retval < 0)
	goto out;

retval = copy_strings(bprm->argc, argv, bprm);
if (retval < 0)
	goto out;
```

그리고 새로운 프로그램 스택의 맨 위를 가리키는 포인터를 `bprm_mm_init` 함수에서 설정한다.:

```C
bprm->exec = bprm->p;
```

스택의 맨 꼭대기는 프로그램의 파일이름을 포함하고 우리는 이 파일이름을 `linux_bprm` 구조체의 `exec` 목에 저장한다.

이제 우리는 `linux_bprm` 구조체를 채웠다, 우리는 `exec_binprm` 함수를 호출할 것이다.:

```C
retval = exec_binprm(bprm);
if (retval < 0)
	goto out;
```

첫번째로 우리는 [pid](https://en.wikipedia.org/wiki/Process_identifier) 를 저장하고 `exec_binprm` 에서 현재 태스크의 [namespace](https://en.wikipedia.org/wiki/Cgroups 로 부터 보여지는 `pid` 도 저장한다.:

```C
old_pid = current->pid;
rcu_read_lock();
old_vpid = task_pid_nr_ns(current, task_active_pid_ns(current->parent));
rcu_read_unlock();
```

그리고 아래의 함수를 수행:

```C
search_binary_handler(bprm);
```

이 함수는 다른 바이너리 포멧을 포함하는 핸들러의 리스트를 확인한다. 현재 리눅스 커널은 다음과 같은 바이너리 포멧을 지원한다.:

* `binfmt_script` - support for interpreted scripts that are starts from the [#!](https://en.wikipedia.org/wiki/Shebang_%28Unix%29) line;
* `binfmt_misc` - support different binary formats, according to runtime configuration of the Linux kernel;
* `binfmt_elf` - support [elf](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) format;
* `binfmt_aout` - support [a.out](https://en.wikipedia.org/wiki/A.out) format;
* `binfmt_flat` - support for [flat](https://en.wikipedia.org/wiki/Binary_file#Structure) format;
* `binfmt_elf_fdpic` - Support for [elf](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) [FDPIC](http://elinux.org/UClinux_Shared_Library#FDPIC_ELF) binaries;
* `binfmt_em86` - support for Intel [elf](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) binaries running on [Alpha](https://en.wikipedia.org/wiki/DEC_Alpha) machines.

So, the `search_binary_handler` tries to call the `load_binary` function and pass `linux_binprm` to it. If the binary handler supports the given executable file format, it starts to prepare the executable binary for execution:

```C
int search_binary_handler(struct linux_binprm *bprm)
{
	...
	...
	...
	list_for_each_entry(fmt, &formats, lh) {
		retval = fmt->load_binary(bprm);
		if (retval < 0 && !bprm->mm) {
			force_sigsegv(SIGSEGV, current);
			return retval;
		}
	}
	
	return retval;	
```

Where the `load_binary` for example for the [elf](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) checks the magic number (each `elf` binary file contains magic number in the header) in the `linux_bprm` buffer (remember that we read first `128` bytes from the executable binary file): and exit if it is not `elf` binary:

```C
static int load_elf_binary(struct linux_binprm *bprm)
{
	...
	...
	...
	loc->elf_ex = *((struct elfhdr *)bprm->buf);

	if (memcmp(elf_ex.e_ident, ELFMAG, SELFMAG) != 0)
		goto out;
```

If the given executable file is in `elf` format, the `load_elf_binary` continues to execute. The `load_elf_binary` does many different things to prepare on execution executable file. For example it checks the architecture and type of the executable file:

```C
if (loc->elf_ex.e_type != ET_EXEC && loc->elf_ex.e_type != ET_DYN)
	goto out;
if (!elf_check_arch(&loc->elf_ex))
	goto out;
```

and exit if there is wrong architecture and executable file non executable non shared. Tries to load the `program header table`:

```C
elf_phdata = load_elf_phdrs(&loc->elf_ex, bprm->file);
if (!elf_phdata)
	goto out;
```

that describes [segments](https://en.wikipedia.org/wiki/Memory_segmentation). Read the `program interpreter` and libraries that linked with the our executable binary file from disk and load it to memory. The `program interpreter` specified in the `.interp` section of the executable file and as you can read in the part that describes [Linkers](http://0xax.gitbooks.io/linux-insides/content/Misc/linkers.html) it is - `/lib64/ld-linux-x86-64.so.2` for the `x86_64`. It setups the stack and map `elf` binary into the correct location in memory. It maps the [bss](https://en.wikipedia.org/wiki/.bss) and the [brk](http://man7.org/linux/man-pages/man2/sbrk.2.html) sections and does many many other different things to prepare executable file to execute.

In the end of the execution of the `load_elf_binary` we call the `start_thread` function and pass three arguments to it:

```C
	start_thread(regs, elf_entry, bprm->p);
	retval = 0;
out:
	kfree(loc);
out_ret:
	return retval;
```

These arguments are:

* Set of [registers](https://en.wikipedia.org/wiki/Processor_register) for the new task;
* Address of the entry point of the new task;
* Address of the top of the stack for the new task.

As we can understand from the function's name, it starts new thread, but it is not so. The `start_thread` function just prepares new task's registers to be ready to run. Let's look on the implementation of this function:

```C
void
start_thread(struct pt_regs *regs, unsigned long new_ip, unsigned long new_sp)
{
        start_thread_common(regs, new_ip, new_sp,
                            __USER_CS, __USER_DS, 0);
}
```

As we can see the `start_thread` function just makes a call of the `start_thread_common` function that will do all for us:

```C
static void
start_thread_common(struct pt_regs *regs, unsigned long new_ip,
                    unsigned long new_sp,
                    unsigned int _cs, unsigned int _ss, unsigned int _ds)
{
        loadsegment(fs, 0);
        loadsegment(es, _ds);
        loadsegment(ds, _ds);
        load_gs_index(0);
        regs->ip                = new_ip;
        regs->sp                = new_sp;
        regs->cs                = _cs;
        regs->ss                = _ss;
        regs->flags             = X86_EFLAGS_IF;
        force_iret();
}
```

The `start_thread_common` function fills `fs` segment register with zero and `es` and `ds` with the value of the data segment register. After this we set new values to the [instruction pointer](https://en.wikipedia.org/wiki/Program_counter), `cs` segments etc. In the end of the `start_thread_common` function we can see the `force_iret` macro that force a system call return via `iret` instruction. Ok, we prepared new thread to run in userspace and now we can return from the `exec_binprm` and now we are in the `do_execveat_common` again. After the `exec_binprm` will finish its execution we release memory for structures that was allocated before and return.

After we returned from the `execve` system call handler, execution of our program will be started. We can do it, because all context related information already configured for this purpose. As we saw the `execve` system call does not return control to a process, but code, data and other segments of the caller process are just overwritten of the program segments. The exit from our application will be implemented through the `exit` system call.

That's all. From this point our program will be executed.

Conclusion
--------------------------------------------------------------------------------

This is the end of the fourth and last part of the about the system calls concept in the Linux kernel. We saw almost all related stuff to the `system call` concept in these four parts. We started from the understanding of the `system call` concept, we have learned what is it and why do users applications need in this concept. Next we saw how does the Linux handle a system call from a user application. We met two similar concepts to the `system call` concept, they are `vsyscall` and `vDSO` and finally we saw how does Linux kernel run a user program.

If you have questions or suggestions, feel free to ping me in twitter [0xAX](https://twitter.com/0xAX), drop me [email](anotherworldofworld@gmail.com) or just create [issue](https://github.com/0xAX/linux-insides/issues/new).

**Please note that English is not my first language and I am really sorry for any inconvenience. If you found any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [System call](https://en.wikipedia.org/wiki/System_call)
* [shell](https://en.wikipedia.org/wiki/Unix_shell)
* [bash](https://en.wikipedia.org/wiki/Bash_%28Unix_shell%29)
* [entry point](https://en.wikipedia.org/wiki/Entry_point) 
* [C](https://en.wikipedia.org/wiki/C_%28programming_language%29)
* [environment variables](https://en.wikipedia.org/wiki/Environment_variable)
* [file descriptor](https://en.wikipedia.org/wiki/File_descriptor)
* [real uid](https://en.wikipedia.org/wiki/User_identifier#Real_user_ID)
* [virtual file system](https://en.wikipedia.org/wiki/Virtual_file_system)
* [procfs](https://en.wikipedia.org/wiki/Procfs)
* [sysfs](https://en.wikipedia.org/wiki/Sysfs)
* [inode](https://en.wikipedia.org/wiki/Inode)
* [pid](https://en.wikipedia.org/wiki/Process_identifier)
* [namespace](https://en.wikipedia.org/wiki/Cgroups)
* [#!](https://en.wikipedia.org/wiki/Shebang_%28Unix%29)
* [elf](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)
* [a.out](https://en.wikipedia.org/wiki/A.out)
* [flat](https://en.wikipedia.org/wiki/Binary_file#Structure) 
* [Alpha](https://en.wikipedia.org/wiki/DEC_Alpha)
* [FDPIC](http://elinux.org/UClinux_Shared_Library#FDPIC_ELF)
* [segments](https://en.wikipedia.org/wiki/Memory_segmentation)
* [Linkers](http://0xax.gitbooks.io/linux-insides/content/Misc/linkers.html)
* [Processor register](https://en.wikipedia.org/wiki/Processor_register)
* [instruction pointer](https://en.wikipedia.org/wiki/Program_counter)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/SysCall/syscall-3.html)
