---
layout: post
title: Staging 포스트 작성 가이드       # 페이지 최상단 타이틀
author: skywhite15                  # authors.yml 에 작성한 본인의 이
thumbnail: thumbnail/rsocket.png    # thumbnail
permalink: /staging-sample1/        # staging url 로 공유할 링크. 원하는대로 입력
staging: true                       # staging 페이지의 경우, staging true 옵션을 !!!무조건!!! 줄것.
---
## Staging 포스트 작성 가이드 <br><br>
### 0. fork 를 뜨고, 체크아웃 받는다.
- fork https://github.com/naver-media-tech/naver-media-tech.github.io
- 추후 작업이 완료되었을 때, pull request를 main media tech repo로 날려준다.

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
`/_posts/` 디렉토리 안에 `2020-08-07-invisible-content.md` 를 복사하여 <br>
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

#### 글 작성시 유의사항
- 게시물 thumbnail 노출 사이즈는 100 x 70 입니다. 
- 블로그 홈 리스트에서는 아래처럼 게시물 발췌 요약본을 노출합니다. 이 때, **요약본 발췌는 게시물의 첫 문단을 활용합니다.** 
- 따라서 게시물의 첫 문단은 신경써서 작성해주세요.
<img src="https://user-images.githubusercontent.com/20153890/90952833-b7a2f880-e4a1-11ea-86d4-9da6790a4c06.png" width=500>

`staging: false` 옵션으로 테스트 하시면 됩니다. (테스트 종료 후 꼭 `staging: true` 로 변경해주세요!!)

### 4. 작성 완료 후 
1. 로컬에서 띄웠을때 홈 리스트에 노출되지 않는지 확인.
2. url 정상적으로 접근 되는지 확인 후
3. pull request & master merge
