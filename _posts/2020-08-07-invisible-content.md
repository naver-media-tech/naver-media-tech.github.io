---
layout: post
title: Staging 포스트 작성 가이드       # 페이지 최상단 타이틀
author: skywhite15                  # authors.yml 에 작성한 본인의 이
thumbnail: thumbnail/rsocket.png    # thumbnail
permalink: /staging-sample1/        # staging url 로 공유할 링크. 원하는대로 입력
staging: true                       # staging 페이지의 경우, staging true 옵션을 !!!무조건!!! 줄것.
---
## Staging 포스트 작성 가이드 
### 1. 작성자 등록
- 이미 등록했다면, 이 과정은 생략해도 됩니다.
- 먼저, `_data` 디렉토리에 있는 `authors.yml` 파일에 자신의 정보를 입력합니다.
```yml
kimtaeng: # 포스팅의 `author`와 매핑됨
  name: kimtaeng # 작성자 이름
  username: madplay # github 계정
  email: itsmetaeng@gmail.com
```

### 2. 게시물 생성
`/_posts/` 디렉토리 안에 `2020-08-07-invisible-content.md` 를 복사하여 
`날짜-{url에 붙을 title}.md` 패턴으로 생성합니다. 

### 3. 게시물 작성 
md file 최상단에 아래와 같은 옵션을 주고 글을 작성합니다.
```
---
layout: post
title: Staging 포스트 작성 가이드       # 페이지 최상단 타이틀
author: skywhite15                  # authors.yml 에 작성한 본인의 이름
thumbnail: thumbnail/rsocket.png    # assets/img/thumbnail/ 하위에 페이지 썸네일 놓기
permalink: /staging-sample1/        # staging url 로 공유할 링크. 원하는대로 입력
staging: true                       # staging 페이지의 경우, staging true 옵션을 !!!무조건!!! 줄것.
---
```

### 4. 작성 완료 후 
1. 로컬에서 띄웠을때 홈 리스트에 노출되지 않는지 확인.
2. url 정상적으로 접근 되는지 확인 후
3. pull request & master merge