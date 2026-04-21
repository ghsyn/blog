---
title: Quartz로 Obsidian 블로그 발행하기
date: 2026-04-21
tags:
  - quartz
  - obsidian
  - blog
  - github
---
# Obsidian & Quartz
## 블로그 플랫폼 선정

개발 서적을 읽다가 기록을 남기고 싶어졌다.
티스토리, 벨로그, 노션, 옵시디언, 깃허브 온갖 군데에서 방황하던 지식들을 하나의 블로그로 모아서 관리하고 공유하기로 했다.

처음에는 자체 플랫폼을 생각했지만, 현재 지식 관리 도구로 [obsidian](https://obsidian.md/)을 사용하던 내겐 서비스 종료나 문제 발생 시 기록들을 영구 저장할 수 없는 점이 플랫폼의 큰 단점으로 다가왔다.
(사실 Github로 나만의 블로그를 만드는 것은 개인적인 숙원사업이기도 했다.)

## 왜 쿼츠인가?

AI 기술이 나날이 갱신되며 프로젝트의 맥락, 규칙, 제한 사항, 요구사항 등을 프롬프트만으로 명령하기엔 부족해 다시 `.md` 파일이 활기를 띄는 요즘이다.[^1]
**obsidian**은 마크다운 노트 앱의 대표 주자인만큼 백링크로 이어진 그래프 뷰나 잘 만들어진 플러그인을 활용하여 문서를 작성하기 매우 유용하다. 그동안 작성해뒀던 파일들을 정리해서 나만의 지식 정원을 만들겠다는 내 목적에 딱 맞았다.

쿼츠(Quartz)는 ==Github을 활용해서 obsidian을 publish 하는 도구==이다. 양방향 백링크를 그대로 웹에서 구현하거나 콜아웃, 각주 등의 [옵시디언 맛(?) markdown 문법을 지원](https://khy07181.github.io/2025/building-blog-with-obsidian-and-quartz#obsidian--quartz-%EB%A1%9C-%EB%B8%94%EB%A1%9C%EA%B7%B8-%EB%A7%8C%EB%93%A4%EA%B8%B0) 하고 다양한 플러그인도 추가 제공하며 마크다운 형식의 활용도를 높였다. 이는 이미 옵시디언과 깃허브를 사용중이던 내게 매력적으로 다가왔다.[^2]

Quartz, Jekyll, Hugo는 모두 `.md` 파일을 웹사이트로 만들어준다는 점은 같지만
Jekyll은 깃허브가 공식 지원하지만 빌드 속도가 다소 느리다는 단점이 있고, Hugo는 단순한 블로그 보다는 방대한 기술 문서를 만들기에 적합한 것 같다.

# 구축 로드맵
### 1. Quartz 레포 생성
> 요약: 본인 Github 내 레포지토리 생성 후 생성한 레포에 [Quartz 레포지토리](https://github.com/jackyzha0/quartz.git)[^3] fork

1. [https://github.com/jackyzha0/quartz.git](https://github.com/jackyzha0/quartz.git) 접속
2. 상단 초록색 `Use this template` > `Create a new repository` 버튼 클릭
	![[스크린샷 2026-04-21 오후 8.36.57.png]]
3. 본인 Github 내 새로운 repository 생성

### 2. Obsidian 연동 + 작업 환경 구성
> 요약: 로컬에 내려받은 후 Obsidian에서 열고 메인 글 추가

1. 터미널(혹은 cmd)에서 프로젝트를 내려받을 위치로 이동
2. 본인 레포지토리 clone
	   ```
	   git clone https://github.com/{user-name}/{repo-name}.git
	   ```
3. Obsidian 실행 보관함 `열기` > 해당 파일의 루트 경로 폴더[^4] 선택<br>
   ![[스크린샷 2026-04-21 오후 8.53.09.png]]
4. Obsidian 에서 파일 구조 확인 -> 모든 게시글은 `/content` 에 작성한다.<br>
	![[스크린샷 2026-04-21 오후 8.54.32.png]]
5. `content` 폴더 아래 `index.md` 파일[^5] 생성 후 글 작성
   - *이 글이 바로 대문 화면이 된다.*
   - *properties로 title을 지정해주어야 제목을 원하는대로 작성할 수 있다.*
	```markdown
  	---
	title: 블로그 제목입니다.
	---
	소개글입니다.
	```

### 3. `Node.js` 설치 및 배포
> 요약: Quartz 사용은 `Node.js`와 `npm` 설치가 필수(mac 사용자는 `nvm`을 통해 관리하는 것이 편하다)
> 
> 참고: https://nodejs.org/ko/download [^6]

1. 터미널 접속 후 `nvm`[^7] 다운로드 및 설치
	- *설치가 완료되면 터미널 한번 껐다가 켜기*
	```shell
	curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
	```
2. `Node.js` 다운로드
	- *nvm 다운로드는 어느 위치던 상관 없으나 이 작업은 반드시 본인 프로젝트의 root 경로에서 실행해야함*
	```shell
	cd ~/blog # 예시
	nvm install 24
	```
3. 버전 확인
	```shell
	node -v     # v24.15.0 (실행 중인 Node.js의 버전)
	nvm current # v24.15.0 (현재 nvm이 가리키고 있는 활성 버전)
	npm -v      # 11.12.1
	```
4. 배포 전 로컬에서 빌드 후 확인
	- `localhost:8080` 접속했을 때 `index` 글이 잘 나와야함
	```shell
	npx quartz build --serve
	```
	
> [!CAUTION] 마주쳤던 문제
> 만약 위 명령어 실행 시 `Cannot find package '~'` 와 같은 문제가 발생한다면<br>
> `pacakage.json` 목록 파일에 작성된 패키지 중 `~`가 누락된 것이다.<br>
> 모든 패키지를 쿼츠 폴더 안에 다운로드 하는 과정이 필요하므로 **본인 프로젝트 위치에서** 아래 명령어 실행
> ```
> npm install
> ```

5. 배포
	- `git commit` 과 `git push`를 동시에 해주는 작업이라고 생각하면됨
	- 로그인이 필요할 수 있음. Token으로 진행.
	```shell
	npx quartz sync
	```

### 4. Page 열기
> 요약: 빌드+배포 방법을 `Github Actions` 로 자동화하고 배포 설정파일 추가

1. Github Repository에서 `Settings` > `Pages` 로 이동
2. `Build and deployment` 의 `Source` 항목을 `GitHub Actions`로 변경
	![[스크린샷 2026-04-21 오후 10.00.58.png]]
3. `.github/workflows` 폴더에 [`/deploy.yml`](https://github.com/ghsyn/blog/blob/v4/.github/workflows/deploy.yml) 파일 생성
	- *없으면 `Github Actions` 작동 안함*
	```yml
	on:
	  push:
	    branches:
	      - v4.   # 여기를 자동 배포될 브랜치명으로 작성하는게 핵심
	```
4. `Actions` 탭에서 ✅ 표시가 나오면 배포 성공
	- **`https://{user-name}.github.io/{repo-name}` 접속**

# 운영 계획

앞으로 글 업로드는 아래 3단계 루틴으로 진행될 예정이다.

1. obsidian에서 `.md` 로 자유롭게 작성
2. 로컬 빌드: `npx quartz build --serve`
3. 배포: `npx quartz sync`

모바일에 최적화 되지 않은 건 아쉬운 부분이다.

폴더를 나눠서 개인 프로젝트를 진행하며 습득한 기술적인 학습 내용, 개발 도서 독서 후기, 새로운 개발 지식에 관한 견해 등 다양한 글로 개발 자서전을 작성해야지.

### 마치며
앞으로 파편화된 자료들을 모으며 Quartz의 다양한 설정들을 추가해볼 예정이다.
구현하는데는 1시간도 안걸렸지만 글 작성만 3시간이 넘게 걸렸다.
생각을 글로 정리하는 건 내겐 아직 너무 어렵다.
내 문장방식과 템플릿을 찾아 나가면서 글 쓰기 연습을 해야겠다.

---
**관련 링크**
- [Obsidian](https://obsidian.md/)
- [github.com/jackyzha0/quartz](https://github.com/jackyzha0/quartz)
- [Node.js® 다운로드](https://nodejs.org/ko/download)
- [Obsidian과 Quartz로 블로그 만들기](https://khy07181.github.io/2025/building-blog-with-obsidian-and-quartz#obsidian--quartz-%EB%A1%9C-%EB%B8%94%EB%A1%9C%EA%B7%B8-%EB%A7%8C%EB%93%A4%EA%B8%B0)
- [Obsidian을 무료로 발행해보자! (feat. Quartz, Cloudflare)](https://hel-p.tistory.com/56)


[^1]: `.md` 로 코딩하는 시대라고도 하던데

[^2]: 그리고 무엇보다 무료 호스팅이다 ㅎㅎ

[^3]: Quartz를 사용할 수 있도록 구현한 템플릿 repository

[^4]: repository명과 같음

[^5]: obsidian에서 작업중이라면 파일명이 `index.md.md` 가 되지 않도록 주의,,

[^6]: 필자는 macOS 환경에서 nvm 방식으로 npm을 사용하여 설치함

[^7]: Node Version Manager, 여러 버전의 Node.js를 설치하고 상황에 따라 바꿔 쓸 수 있게 해주는 설치 스크립트