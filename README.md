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

## 글 작성 방법
### 1. 작성자 등록
- 이미 등록했다면, 이 과정은 생략해도 됩니다.
- 먼저, `_data` 디렉토리에 있는 `authors.yml` 파일에 자신의 정보를 입력합니다.
```yml
kimtaeng: # 포스팅의 `author`와 매핑됨
  name: kimtaeng # 작성자 이름
  username: madplay # github 계정
  email: itsmetaeng@gmail.com
```

### 2. 글 작성을 위한 파일 생성
- `_post` 디렉토리에 `YYYY-mm-dd-파일이름.md` 형태로 파일을 만들어주세요.

### 3. 파일 상단에 메타 정보 입력
- 포스팅 파일에는 반드시 아래와 같은 정보가 필요합니다.
```bash
---
layout: post
title:  "테스트로 만들고 있는 이것저것"
author: kimtaeng
description: "여기에는 글 설명이 들어갈 위치랍니다."
---
```