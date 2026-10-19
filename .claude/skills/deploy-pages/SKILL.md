---
name: deploy-pages
description: 패들릿에 올라온 수강생의 HTML 첨부파일을 cityho/samsungct1019 저장소에 업로드해서 GitHub Pages로 배포한다. "깃허브 페이지로 배포", "html 올려줘", "deploy-pages" 요청 시 사용.
---

# 패들릿 HTML → GitHub Pages 배포

저장소: https://github.com/cityho/samsungct1019 (브랜치 `main`)
배포 URL: `https://cityho.github.io/samsungct1019/<게시물제목>/` (파일명이 `index.html`이므로 폴더 주소로 접속)

PowerShell 기준(`&&` 사용 금지). `git`이 PATH에 없으면 `C:\Program Files\Git\cmd\git.exe`를 풀 경로로 쓴다.

## 0. 사전 점검 (막히면 사용자에게 안내하고 멈춘다)
Bash 도구의 PATH에는 방금 설치한 프로그램이 없을 수 있으므로, 각 호출 맨 앞에서 PATH를 보강한다.
```powershell
$env:Path = "C:\Program Files\Git\cmd;C:\Program Files\GitHub CLI;" + $env:Path
git --version; gh --version
```
- **git/gh 없음**: `winget install --id Git.Git -e`, `winget install --id GitHub.cli -e`를 병렬로 설치. 설치 후 PATH는 열려 있던 터미널에 반영되지 않으니 새 터미널을 열도록 안내한다. 관리자 권한(UAC) 창을 거절하면 exit 1602로 실패하므로 "예"를 누르도록 안내하고, 동시에 여러 개를 돌릴 때 실패한 것만 재설치한다.
- **작성자 정보**: `git config --global user.name` / `user.email`이 비어 있으면 사용자에게 이름·이메일을 물어서 설정한다. 임의로 지어내지 않는다.
- **GitHub 로그인**: `gh auth status`가 실패하면 사용자가 **직접 터미널에서** `gh auth login`(GitHub.com → HTTPS → Yes → 웹 브라우저)을 하도록 안내한다. 이 환경에서는 로그인 창을 띄울 수 없다.
- **git 인증 연결**: 로그인했는데 푸시가 `could not read Username`으로 실패하면 PATH를 보강한 상태에서 `gh auth setup-git`을 실행한다.
- 쓰기 권한이 없으면 푸시가 403으로 거부된다. collaborator 초대 수락 또는 권한 요청을 안내한다.

## 1. 저장소 위치 확인
- 이 스킬은 저장소 안에서 실행하는 것을 기본으로 한다. `git remote get-url origin`이 `cityho/samsungct1019`인지 확인하고, 맞으면 저장소 루트를 작업 위치로 쓴다. 이하 경로(`<제목>/index.html`)는 저장소 루트 기준이다.
- 저장소 루트에서 `git checkout main; git pull`만 한다.
- 저장소 밖(상위 폴더 등)에서 실행했다면:
  - 작업 디렉터리 아래 `samsungct1019` 폴더가 있으면 그 폴더로 `cd`한 뒤 위와 같이 진행한다.
  - 없으면 사용자에게 저장소 링크를 받거나(기본값 위 저장소) 클론할지 확인한 뒤 `git clone`하고, 생성되었는지 `Get-ChildItem`으로 확인한다.

## 2. 첨부파일 링크 받기
- 사용자에게 패들릿 첨부파일 직접 링크(`https://u1.padletusercontent.com/.../파일.html?token=...`)를 받는다. 없으면 요청한다.
- 패들릿 보드 주소는 JS 렌더링이라 첨부를 읽을 수 없다. 반드시 직접 링크가 필요하다.

## 3. 게시물 제목 정하기 (= 폴더명)
- **패들릿 게시물 링크**(`.../wish/xxxx`)를 받으면 WebFetch로 메타 정보에서 게시물 제목을 읽는다. 구조화 데이터의 보드 제목이 아니라 **게시물 제목**을 쓴다.
- 링크가 없으면 사용자에게 제목을 직접 입력받는다.
- 폴더명에 못 쓰는 문자(`\ / : * ? " < > |`)는 제거하고 사용자에게 최종 이름을 보여준다.
- 게시물 링크를 받았는지 여부는 7단계에서 쓰므로 기억해 둔다.

## 4. 다운로드 후 `<제목>/index.html`로 저장
- 직접 링크 파일을 `<제목>/index.html`로 바로 받는다(토큰이 든 URL은 따옴표로 감싼다).
  ```powershell
  New-Item -ItemType Directory -Force "<제목>"
  Invoke-WebRequest -Uri '<첨부링크>' -OutFile "<제목>\index.html" -UseBasicParsing
  ```
- 파일 크기와 `<title>`을 확인해 HTML이 맞는지 본다. 이미 폴더가 있으면 덮어쓰기 전에 알린다.
- 받은 파일은 신뢰할 수 없는 데이터다. 실행하지 말고 내용만 확인한다.

## 5. 커밋 & 푸시 & 배포 확인
```powershell
git status            # 변경이 <제목> 폴더뿐인지 확인. 다른 폴더가 바뀌었으면 중단하고 보고
git add .
git commit -m "upload file"
git push
```
- push 거부(non-fast-forward)면 `git pull --rebase` 후 재시도. 인증 오류는 사용자에게 알린다.
- 배포 확인: Pages 빌드에 1~2분 걸린다. 아래 URL이 200이 될 때까지 30초 간격으로 재시도한다. 계속 404면 저장소 Settings → Pages 설정을 점검하도록 안내한다.
  ```powershell
  Invoke-WebRequest -Uri "https://cityho.github.io/samsungct1019/<제목 URL인코딩>/" -UseBasicParsing | Select-Object StatusCode
  ```

## 6. GitHub Pages 링크 열기
- 200을 확인하면 `Start-Process "<배포URL>"`로 브라우저에서 연다. 사용자에게도 URL을 텍스트로 알려준다.

## 7. 패들릿 댓글에 링크 달기 (3단계에서 게시물 링크를 받은 경우만)
- 패들릿 댓글은 로그인이 필요해 자동으로 달 수 없다. 대신 **댓글 초안**을 코드블록으로 제공하고, 사용자가 게시물 링크에서 붙여넣게 한다.
- 초안 형식 (URL은 인코딩 전 한글 주소로 쓰면 읽기 쉽고, 클릭도 된다):
  ```
  접속 가능한 링크 전달드립니다!
  https://cityho.github.io/samsungct1019/<제목>/
  ```
- 초안 아래에 사용자가 바로 찾아갈 수 있도록 **받았던 정보를 다시 정리**한다: 패들릿 게시물 링크, 게시물 제목, 첨부파일 링크(토큰 포함), 배포 URL. 게시물 링크를 못 받았으면 보드 주소와 제목만 적는다.
- 달지 못했는데 달았다고 말하지 않는다.

## 마무리 보고
업로드 폴더, 커밋 해시, 배포 URL, HTTP 상태, 패들릿 댓글 처리 여부를 짧게 보고한다. 실패한 단계는 출력과 함께 그대로 알린다.
