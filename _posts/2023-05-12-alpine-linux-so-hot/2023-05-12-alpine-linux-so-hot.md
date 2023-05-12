---
layout: post
title: 'Alpine Linux에 크게 데인 날 🥵'
comments: true
author: 'dongzoolee'
tags: 일기
description: 나는 아직 Alpine Linux의 A도 모른다.
sticky: true
hidden: true
---

오늘 아침부터 회사가 시끌시끌했습니다.

CI에서 체크하는 백엔드 테스트가 전부 실패했기 때문인데요, 분명 테스트 코드나 테스트 데이터에는 문제가 하나도 없었습니다.

![](/assets/post-images{{ page.url }}/2023-05-13-03-00-52.png)

순간 머리 속으로 “제 2의 [left-pad 사건](https://github.com/left-pad/left-pad/issues/4)인가..?” 하는 생각이 스쳐 지나갔습니다.. 그리고 결론적으로 나름? 유사한 사건이었습니다 😇

회사 동료인 [@StationSoen](https://github.com/StationSoen)님과 함께 4시간을 쏟아 문제의 원인을 분석하고 해결한 과정을 기록해보려고 합니다.

## 원인 찾기 🧐

실패하는 테스트는 문서 스냅샷 테스트와 DB Call을 사용하는 일부 테스트였습니다.

### 📸 스냅샷 테스트

```python
======================================================================
FAIL: test_xxx_document
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/home/runner/work/captain/captain/zuzu/tests/xx/text_xx_document.py", line 111, in test_xx_form_document
    self.assertPdfEqual(pdf, "option-consent-form.pdf")
  File "/home/runner/work/captain/captain/zuzu/tests/xx/base.py", line 95, in assertPdfEqual
    raise e
  File "/home/runner/work/captain/captain/zuzu/tests/xx/base.py", line 79, in assertPdfEqual
    self.assertEqual(
AssertionError: 0.0006538025445077359 != 0 : The PDFs are different.
----------------------------------------------------------------------
```

보통 스냅샷 테스트가 실패하는 원인은 두 가지입니다. 

1. 테스트 시 생성하는 데이터가 실행할 때마다 다른 데이터가 생성되는 경우 (랜덤하게 실패하는 테스트)
2. 글자 크기나 자간 옵션을 고정하지 않아서 스타일이 살짝 다른 문서가 생성되는 경우

아무리 찾아봐도 해당 문서를 생성하는 코드는 잘못이 없었습니다.

그리고 로컬에서는 해당 테스트를 아무리 실행해도 성공하고, CI 환경에서는 100% 실패했습니다.

문서 스냅샷 테스트의 비교 대상 스냅샷과 테스트 과정에 생성한 스냅샷의 diff 이미지를 출력해보니 다음과 같았습니다.

![](/assets/post-images{{ page.url }}/2023-05-13-03-01-20.png)

이 정도면.. 두 문서의 차이는 없다고 봐야겠지요..?

아무리 봐도 알 수가 없어서 위 diff 이미지에 나와 있는 글자 중 하나를 크게 확대해서 확인해봤습니다.

<div style="display:flex">
    <img width="45%" src="/assets/post-images{{ page.url }}/2023-05-13-03-01-32.png"/>
    <img width="45%" src="/assets/post-images{{ page.url }}/2023-05-13-03-01-53.png"/>
</div>

차이가 보이시나요? 글자 주변에 번진 픽셀의 분포가 아주 미세하게 차이가 있습니다 😓

바로 저희가 Word 문서를 PDF로 변환할 때 사용하는 LibreOffice의 버전이 달라지지는 않았는지 확인하러갔습니다.

로컬이나 프로덕션 환경에서는 이미 빌드된 Docker image를 사용하기 때문에 영향은 없지만, CI 환경에서는 이미지를 매번 빌드하기 때문에 뭔가 다른 버전의 라이브러리가 설치되었을 가능성을 의심했습니다. 

이미지를 로컬에서 직접 빌드해보니 역시나 마이너 버전이 업데이트된 LibreOffice 패키지가 설치되어 있었습니다. 

**기존 이미지의 패키지 목록**

```python
/app # apk list
libreoffice-7.3.7.2-r0 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-base-7.3.7.2-r0 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-calc-7.3.7.2-r0 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-common-7.3.7.2-r0 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-connector-postgres-7.3.7.2-r0 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-draw-7.3.7.2-r0 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-impress-7.3.7.2-r0 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-lang-en_us-7.3.7.2-r0 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-math-7.3.7.2-r0 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-writer-7.3.7.2-r0 aarch64 {libreoffice} (MPL-2.0) [installed]
libreofficekit-7.3.7.2-r0 aarch64 {libreoffice} (MPL-2.0) [installed]
```

**새로 빌드한 이미지의 패키지 목록**

```python
/app # apk list
libreoffice-7.5.3.2-r2 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-base-7.5.3.2-r2 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-calc-7.5.3.2-r2 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-common-7.5.3.2-r2 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-connector-postgres-7.5.3.2-r2 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-draw-7.5.3.2-r2 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-impress-7.5.3.2-r2 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-lang-en_us-7.5.3.2-r2 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-math-7.5.3.2-r2 aarch64 {libreoffice} (MPL-2.0) [installed]
libreoffice-writer-7.5.3.2-r2 aarch64 {libreoffice} (MPL-2.0) [installed]
libreofficekit-7.5.3.2-r2 aarch64 {libreoffice} (MPL-2.0) [installed]
```

그런데 아무리 생각을 해봐도 리눅스 환경에서 버전이 고정된 패키지를 설치해본 적이 없습니다 🤔 오히려 패키지를 설치할 때마다 매번 습관적으로 `apk update`를 실행하던 기억 밖에 없고 말이죠.

Alpine Linux 패키지 목록 페이지에 가보아도 이전 버전에 대한 내용은 눈을 씻어도 찾기 힘들었습니다. 이전 버전의 패키지도 제공하는 npm이나 다른 패키지 매니저들과는 다르게 말이죠.

![](/assets/post-images{{ page.url }}/2023-05-13-03-02-05.png)

찾아보니 Alpine Linux는 패키지 버전을 고정하거나, 이전 버전의 패키지를 설치하는 것을 지원하지 않았습니다.

Alpine Linux용 패키지를 배포할 때 어떤 버전(Branch)의 Alpine Linux에 link할지를 선택해서, 해당 버전의 branch에 push하면 해당 버전의 Alpine Linux에서 밖에 이용하지 못 하는 패키지가 되는 것입니다.

(참고: [https://stschindler.medium.com/the-problem-with-docker-and-alpines-package-pinning-18346593e891](https://stschindler.medium.com/the-problem-with-docker-and-alpines-package-pinning-18346593e891))

### 첫 시도 - 실패 ❌

처음에 이 사실을 깨닫고 OS의 레파지토리 목록에 `v3.17`을 추가하여 어떻게든 강제로 version fixing을 해보려고 시도해봤습니다.

```docker
RUN echo "http://dl-cdn.alpinelinux.org/alpine/v3.17/main/" >> /etc/apk/repositories
RUN apk add -U --no-cache "libreoffice==7.5.3.2-r2"
```

뭐, 결과는 당연했습니다. world가 맞지 않아 설치할 수 없답니다.

```docker
ERROR: unsatisfiable constraints:
  libreoffice-7.5.3.2-r2:
    breaks: world[libreoffice=7.5.3.2-r2]
```

더 찾아보면 어떻게든 해결해볼 수는 있을 것 같았지만, Alpine Linux의 패키지 관리 원칙에 맞지 않는 우회책이라서 그냥 포기했습니다.

### 두번째 시도 - 성공 ✌🏼

계속 `7.5.3.2-r2` 버전을 Alpine Linux 패키지 목록 사이트에서 찾아다니다보니, 결국 저희가 찾던 버전은 Alpine Linux 3.17 branch에 숨어 있는 것을 발견했습니다 😭

![](/assets/post-images{{ page.url }}/2023-05-13-03-02-38.png)

그리고 저희의 Dockerfile에서도 Alpine Linux의 메이저 버전 3만 명시한 상태여서 Alpine Linux 3.18의 배포로 인해 자동으로 3.17 → 3.18을 사용하게 된 것입니다. 당연히 3.17 버전과 link된 LibreOffice 2.7.2 버전이 아니라 3.18 버전과 link된 LibreOffice 2.7.5가 설치된 것이구요. 

![](/assets/post-images{{ page.url }}/2023-05-13-03-02-53.png)

Alpine Linux 버전을 3.17로 고정시켜줌으로써 문제를 해결할 수 있었습니다. 

### 🤖 PostgreSQL DB Call 오류

```python
File "/home/runner/.local/share/virtualenvs/captain-CVyYfSri/lib/python3.11/site-packages/django/db/models/query.py", line 57, in __iter__
    results = compiler.execute_sql(
              ^^^^^^^^^^^^^^^^^^^^^
  File "/home/runner/.local/share/virtualenvs/captain-CVyYfSri/lib/python3.11/site-packages/django/db/models/sql/compiler.py", line 1361, in execute_sql
    cursor.execute(sql, params)
  File "/home/runner/.local/share/virtualenvs/captain-CVyYfSri/lib/python3.11/site-packages/django/db/backends/utils.py", line 67, in execute
    return self._execute_with_wrappers(
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/runner/.local/share/virtualenvs/captain-CVyYfSri/lib/python3.11/site-packages/django/db/backends/utils.py", line 80, in _execute_with_wrappers
    return executor(sql, params, many, context)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/runner/.local/share/virtualenvs/captain-CVyYfSri/lib/python3.11/site-packages/django/db/backends/utils.py", line 84, in _execute
    with self.db.wrap_database_errors:
  File "/home/runner/.local/share/virtualenvs/captain-CVyYfSri/lib/python3.11/site-packages/django/db/utils.py", line 91, in __exit__
    raise dj_exc_value.with_traceback(traceback) from exc_value
  File "/home/runner/.local/share/virtualenvs/captain-CVyYfSri/lib/python3.11/site-packages/django/db/backends/utils.py", line 89, in _execute
    return self.cursor.execute(sql, params)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
django.db.utils.OperationalError: could not load library "/usr/local/lib/postgresql/llvmjit.so": Error relocating /usr/local/lib/postgresql/llvmjit.so: LLVMBuildGEP: symbol not found
```

느닷없이 DB call에 실패했습니다. 에러 로그를 보니 딱 봐도 Django의 탓은 아닙니다.

제일 먼저 저희가 사용하고 있는 `postgres:12-alpine`의 커밋 로그를 찾으러 떠났습니다. (위에서 이미 Alpine Linux 3.18으로 뒤통수를 한 대 맞았기 때문에, 이번에도 아주 쎄했습니다)

아니나다를까, 오늘 오전 3시에 Alpine Linux 3.17 → 3.18을 사용하도록 변경된 `postgres:12.15-alpine`이 출시되었고, CI에서 `postgres:12-alpine`를 사용 중이었던 저희는 자동으로 `12.14`에서 `12.15`를 사용하게 된 것입니다.

![](/assets/post-images{{ page.url }}/2023-05-13-03-03-06.png)

그런데 뭔가가 이상합니다. Alpine Linux 3.17에서 3.18을 사용하게 되었다고 DB Call이 작동 안 할 이유가 있나요?

그래서 [@StationSoen](https://github.com/StationSoen)님과 위 에러 로그의 `llvmjit.so` 바이너리, 그리고 이 바이너리 파일에서 사용하는 `LLVMBuildGEP` 함수가 무엇인지를 제일 먼저 찾아 보았습니다.

힘들게 찾아보니

1. LLVM이라는 컴파일러의 16버전이 3월에 출시되었고, 이 버전에서 `LLVMBuildGEP` 함수가 `LLVMBuildGEP**2**`로 rename된 것을 확인했습니다. 
    
    [https://releases.llvm.org/16.0.0/docs/ReleaseNotes.html#changes-to-the-c-api](https://releases.llvm.org/16.0.0/docs/ReleaseNotes.html#changes-to-the-c-api)
    
2. Alpine Linux 3.18 버전부터 LLVM 15 → 16을 사용하도록 변경되었던 겁니다.

결국 PostgreSQL에서는 업데이트된 LLVM의 인터페이스에 맞게 코드를 수정하지 않고 12.15 버전을 릴리즈한 것입니다 😢

(함수를 deprecate시키지 않고 rename해버린 LLVM 측도 너무하긴 했지만요..)

회사 동료인 [@StationSoen](https://github.com/StationSoen)님과 “우리도 PostgreSQL 컨트리뷰터?” 라고 이야기하며 Pull Request를 만들어보자고 했었지만 ㅎ 일단 공식 레포지토리에 이슈만 생성해놓았습니다. 

역시나 뿔이 난 전 세계 개발자들이 [@StationSoen](https://github.com/StationSoen)님께서 만드신 이슈에 달려들어 분노를 표출해주셨습니다 👿

[https://github.com/docker-library/postgres/issues/1076](https://github.com/docker-library/postgres/issues/1076)

12시간 밖에 안 되었는데 그는 벌써 글로벌 스타가 되었습니다 🔥

![](/assets/post-images{{ page.url }}/2023-05-13-03-03-22.png)

해결책은 간단했습니다. CI에서 사용되는 PostgreSQL 이미지를 마이너 버전까지 fix하는 것으로 문제는 바로 해결되었습니다.

![](/assets/post-images{{ page.url }}/2023-05-13-03-03-34.png)

## TIL

1. 지금까지 Alpine Linux는 “빠르고”, “가벼운” OS라는 생각만 하며 사용을 했었는데, 빠르고 가벼운 데에는 다 이유가 있었다.
2. Alpine Linux 버전마다 설치할 수 있는 Package의 version이 fix되어 있다.
3. 의존성은 minor version까지 fix하는 것을 잊지 말자. (RC든 정식 Release version이든 테스트 전까지 믿을 게 못 된다..)

마지막으로, 굳이 Alpine Linux가 이러한 방식으로 패키지를 관리하는 이유를 알고 계신 분이 있으시다면 댓글로 알려주세요 🤔 

패키지 개발자가 OS 버전에 link된 패키지 버전을 업데이트하면 Alpine Linux의 버전을 fix해도 언제든지 호환성 문제가 발생할 수 있는 risky한 구조가 아닌가 싶습니다.

**저와 하루종일 함께 건설적인 논의와 다양한 시도를 펼쳐주신 [@StationSoen](https://github.com/StationSoen)님께 영광을 돌립니다 🎉**