# 커널 초기화 과정

커널이 압축을 해제하고 커널 자체로 실행되는 첫번째 프로세스에서 부터 커널 초기화의 모든 내용을 기술한다.

*Note* 여기에 모든 커널 초기화 단계의 내용이 있다는 것이 아니라, 일반적인 커널의 부분이며, 인터럽트 핸들링, ACPI 그리고 많은 다른 부분들이 제외 되었다. 내가 뺀 모든 파트는 다른 챕터에서 다룰 예정이다.

* [커널 압축 해제 후 첫 단계](linux-initialization-1.md) - 커널에서 첫 단계를 기술
* [초기 인터럽트와 Exception 핸들링](linux-initialization-2.md) - 초기 인터럽트 초기화와 초기 페이지 폴트 핸들러에 대한 기술
* [커널 엔트리 포이트 전에 마지막 준비](linux-initialization-3.md) - `start_kernel` 을 호출 하기 전에 마지막 준비 단계를 기술
* [커널 엔트리 포인트](linux-initialization-4.md) - 커널의 일반적인(generic) 코드에서 첫 단계
* [아키텍쳐 의존적인 초기화 계속](linux-initialization-5.md) - 아키텍처 특유의 초기화.
* [아키텍처 의존적인 초기화, 다시...](linux-initialization-6.md) - 아키텍처 의존적인 초기화에 대한 내용을 계속 다룸
* [거의 마지막 단계의 아키텍처 의존적인 초기화](linux-initialization-7.md) - `setup_arch` 에 관련된 마지막 일들을 기술
* [스케줄러 초기화](linux-initialization-8.md) - 스케줄러 초기화전에 준비 기술\
* [RCU 초기화](linux-initialization-9.md) - [RCU](http://en.wikipedia.org/wiki/Read-copy-update) 초기화
* [초기화 마지막](linux-initialization-10.md) - 리눅스 커널 초기화 마무리
