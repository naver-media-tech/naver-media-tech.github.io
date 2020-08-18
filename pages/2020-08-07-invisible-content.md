---
layout: post
title: Staging 포스트 작성 가이드    # 페이지 최상단 타이틀
author: skywhite15              # authors.yml 에 작성한 본인의 이
permalink: /staging-sample1/    # staging url 로 공유할 링크. 원하는대로 입력
hide: true                      # staging 페이지의 경우, hide true 옵션을 !!!무조건!!! 줄것.
---
## Staging 포스트 작성 가이드 
`/pages/` 디렉토리 안에 `2020-08-07-invisible-content.md` 를 참고하여
개인 커스텀 url 로 만들어 등록.

md file 최상단에 아래와 같은 옵션을 주고 글을 작성한다.
```
---
layout: post
title: 자부자 안개 은신술            # 페이지 최상단 타이틀
author: skywhite15              # authors.yml 에 작성한 본인의 이
permalink: /staging-sample1/    # staging url 로 공유할 링크. 원하는대로 입력
hide: true                      # staging 페이지의 경우, hide true 옵션을 !!!무조건!!! 줄것.
---
```

## 작성 완료 후 
1. 로컬에서 띄웠을때 홈 리스트에 노출되지 않는지 확인.
2. url 정상적으로 접근 되는지 확인 후
3. pull request & master merge