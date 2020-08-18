# Media Tech.

<br>

## 개발 환경 세팅
### 요구사항
- ruby 2.1.X 이상 버전
- jekyll, bundler 설치 필요(아래 명령어로 한 번에 설치)
- 빌드
```bash
$ sudo gem install jekyll bundler
```

<br>

### 로컬에 세팅하기
- 먼저, 해당 프로젝트는 `git clone` 합니다.
- 해당 프로젝트의 루트 디렉토리로 이동해서 아래 명령어 실행
  - Gemfile에 명시된 의존성 기반으로 프로젝트 세팅
```bash
$ bundle install 
```

<br>

### 로컬에서 실행
- 프로젝트 실행
  - `watch` 옵션은 MacOS 에서만 사용 가능하며, 변경사항 자동 리로드 옵션
  - 단, `_config.yml` 변경시에는 재실행 필요
```bash
$ bundle exec jekyll serve --watch
```

### 기타 참고
- Windows 환경에서는 아래 링크를 참고해서 진행
  - https://madplay.github.io/post/install-jekyll-on-windows
- 테마 소스: 기본 설정 관련해서 참고
  - https://github.com/rohanchandra/type-theme

<br>

## 글 작성 가이드
https://naver-media-tech.github.io/staging-sample1/