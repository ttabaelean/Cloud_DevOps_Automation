# 10주차 Hands-on Labs: Git과 GitHub를 활용한 형상 관리

| 항목 | 구성 |
| --- | --- |
| 실습 PC | Windows 10/11 (Mac은 별도 안내 참고) |
| 편집기 | Visual Studio Code |
| Git | Git for Windows (Git Bash 포함) |
| 터미널 | VSCode 터미널 → Git Bash |
| 실습 저장소 | 로컬 `~/kcu-git-lab` ↔ GitHub `kcu-git-lab` |

> ⚠️ 3강에서 만드는 GitHub `kcu-git-lab` 저장소는 **11주차 GitHub Actions 실습에서 계속 사용**하므로 삭제하지 않습니다.
> 

---

## 10주 1강 **Hands-on Labs : Git 실습 환경 구성**

---

### **1단계: VSCode 설치**

- 공식 사이트에서 Windows용 설치 파일 다운로드
    - https://code.visualstudio.com
- **추가 작업 선택** 화면에서 확인
    - ☑ **"Code(으)로 열기" 작업을 Windows 탐색기 메뉴에 추가**
    - ☑ **PATH에 추가** (기본 선택)
- 설치 완료 후 VSCode가 실행되는지 확인

> **참고**: VSCode를 먼저 설치해야 2단계 Git 설치 화면에서 VSCode를 기본 편집기로 선택할 수 있습니다.
> 

### **2단계: Git 설치**

- Windows용 Git 설치 파일 다운로드
    - https://git-scm.com/downloads → **Windows** → **64-bit Git for Windows Setup**
- 설치 화면은 대부분 기본값(**Next**)으로 진행
- 아래 **2가지 옵션은 반드시 변경**
    
    
    | 설치 화면 | 선택 |
    | --- | --- |
    | Choosing the default editor used by Git | **Use Visual Studio Code as Git's default editor** |
    | Adjusting the name of the initial branch in new repositories | **Override the default branch name for new repositories** → `main` |
- PATH 설정 화면은 **권장값 그대로** 유지
    - Adjusting your PATH environment → **Git from the command line and also from 3rd-party software (Recommended)**

### **3단계: 설치 확인**

<aside>
📌**명령어 정리**

- `git --version` : 설치된 Git 버전 확인 (버전이 나오면 설치 완료)
- `where.exe git` : git 실행 파일이 설치된 위치 확인
</aside>

- Git 설치 중 VSCode가 열려 있었다면 **완전히 종료한 후 다시 실행**
- VSCode 메뉴 **Terminal → New Terminal** (단축키 `Ctrl + \``)
- Git 버전 확인
    
    ```bash
    git --version
    ```
    
    - 결과 예: `git version 2.xx.x.windows.x`
- Git 설치 위치 확인
    
    ```bash
    where.exe git
    ```
    
    - 결과 예: `C:\Program Files\Git\cmd\git.exe`

### **4단계: PATH 환경변수 확인·설정**

<aside>
📌**용어 정리**

- `PATH` : 명령어를 입력했을 때 Windows가 프로그램을 찾아보는 폴더 목록
- `C:\Program Files\Git\cmd` : git.exe가 들어 있는 폴더 (PATH에 있어야 git 명령 사용 가능)
</aside>

> `git`을 찾을 수 없다는 오류가 나올 때만 진행합니다. 3단계에서 버전이 정상 출력되었다면 5단계로 넘어갑니다.
> 
- Windows 검색 → **시스템 환경 변수 편집** 실행
- **[환경 변수]** → **사용자 변수**의 `Path` 선택 → **[편집]**
- **[새로 만들기]** → 아래 경로 입력 → **[확인]**
    
    ```
    C:\Program Files\Git\cmd
    ```
    
- VSCode를 완전히 종료한 후 다시 실행하고, 새 터미널에서 재확인
    
    ```bash
    git --version
    ```
    

### **5단계: 기본 터미널을 Git Bash로 지정**

<aside>
📌**명령어 정리**

- `pwd` : 현재 작업 위치(폴더) 출력
- `ls` : 현재 폴더의 파일 목록 보기
- `~` : 사용자 홈 폴더 (`C:\Users\사용자이름`)
</aside>

- `Ctrl + Shift + P` → 명령 팔레트에 입력 → **Git Bash** 선택
    
    ```
    Terminal: Select Default Profile
    ```
    
- 터미널 오른쪽 **[+]** 버튼으로 새 터미널을 열고 `bash`로 표시되는지 확인
- Linux 명령 동작 확인
    
    ```bash
    pwd      # 현재 위치 (예: /c/Users/사용자이름)
    ls       # 파일 목록
    ```
    

> **참고**: Git Bash에서 `~`는 사용자 폴더(`C:\Users\사용자이름`)를 의미합니다.
> 

### **6단계: Git 사용자 설정**

<aside>
📌**명령어 정리**

- `git config --global <항목> <값>` : 이 PC의 모든 저장소에 적용되는 Git 설정
- `user.name` : Commit에 작성자로 표시될 이름 (자유롭게 입력)
- `user.email` : GitHub 계정과 Commit을 연결하는 기준 (**GitHub 가입 이메일**)
- `init.defaultBranch main` : 새 저장소를 만들 때 기본 브랜치 이름을 main으로 지정
- `git config --global --list` : 현재 설정 목록 확인
</aside>

- Commit 작성자 정보 등록 (본인 이름과 **GitHub 가입 이메일**로 변경해서 입력)
    
    ```bash
    git config --global user.name "Seongmi Lee"
    git config --global user.email "seongmi.lee@kcu.ac"
    ```
    
- 새 저장소의 기본 Branch 이름을 `main`으로 지정
    
    ```bash
    git config --global init.defaultBranch main
    ```
    
- 설정 확인
    
    ```bash
    git config --global --list
    ```
    
    ```
    core.editor="C:\Users\seong\AppData\Local\Programs\Microsoft VS Code\bin\code" --wait
    user.name=Seongmi Lee
    user.email=seongmi.lee@kcu.ac
    init.defaultbranch=main
    ```
    

### **7단계: GitHub 계정 준비**

- git : 내 컴퓨터에서 사용하는 버전 관리 프로그램
- github: git 저장소를 인터넷에 올려 다른 사람과 공유하게 해주는 서비스
- https://github.com → **[Sign up]**
- 이메일 · 비밀번호 · Username 입력 → 이메일 인증 코드 입력 → 가입 완료
    - username: **`seongmi-lee`**
    - email: `seongmi.lee@kuc.ac`
- **Username**은 저장소 주소에 사용되므로 기억해 둡니다.
    - 예: `https://github.com/<seongmi-lee>/kcu-git-lab`

> **Mac 사용자**: 터미널에서 `git --version` 입력 → Command Line Tools 설치 창이 뜨면 **[설치]**. PATH 설정(4단계)과 Git Bash 지정(5단계)은 필요 없으며, 6단계부터 진행합니다.
> 

---

## 10주 2강 **Hands-on Labs : 로컬 저장소 버전 관리**

---

> 모든 명령은 **VSCode 터미널(Git Bash)** 에서 실행합니다. 
git add, git diff, git status, git commit, git log 등 기본 명령어 사용을 학습한다.
버전관리 실습 - main branch외 feature branch를 생성하여 변경 사항을 기록. 이후  main branch로 병합
main branch (index.html) → feature branch(index.html) - marge
> 

### **1단계: 실습 폴더와 저장소 생성**

<aside>
📌**명령어 정리**

- `git init` : 현재 폴더를 Git 저장소로 만들기 (`.git` 폴더 생성)
- `code -r .` : 현재 폴더를 VSCode의 현재 창에서 열기
</aside>

- 실습 폴더 생성 및 이동
    
    ```bash
    mkdir ~/kcu-git-lab
    cd ~/kcu-git-lab
    ```
    
- Git 저장소 생성 → 숨김 폴더 `.git` 생성 확인
    
    ```bash
    git init
    ls -a
    ```
    
- VSCode에서 현재 폴더 열기
    
    ```bash
    code -r .
    ```
    

> **참고**: `.git` 폴더가 Local Repository입니다. 직접 수정하거나 삭제하지 않습니다.
> 

### **2단계: 첫 번째 Commit**

<aside>
📌**명령어 정리**

- `git status` : 파일 상태 확인 (Untracked → Changes to be committed → clean)
- `git add <파일>` : 파일을 Staging Area에 등록 (다음 Commit에 포함)
- `git rm --cached <파일>` : add 취소 (Staging Area에서만 빼고 파일은 유지)
- `git commit -m "메시지"` : Staging된 내용을 하나의 버전(Commit)으로 저장
</aside>

- 파일 생성 후 상태 확인 → **Untracked files** (빨간색)
    
    ```bash
    echo "<h1>KCU Cloud DevOps</h1>" > index.html
    ```
    
    ```bash
    git status
    
    On branch main                 <- main Branch에서 작업 중
    
    No commits yet                 <- commit 하기 전. git init으로 저장소만 만들었고 Local Repository는 비어 있는 상태
    
    Untracked files:               <- Git이 아직 관리하지 않는 새 파일.  index.html은 Working Directory에만 있고, Git은 이 파일을 모르는 상태
      (use "git add <file>..." to include in what will be committed)  
            index.html
    
    nothing added to commit but untracked files present (use "git add" to track)    <- 다음 Commit에 포함하려면 git add <파일>을 실행하라는 안내
    ```
    
- Staging Area 등록 → **Changes to be committed** (초록색)
    
    ```bash
    git add index.html
    warning: in the working copy of 'index.html', LF will be replaced by CRLF the next time Git touches it
    ```
    
    ```bash
    git status
    On branch main
    
    No commits yet
    
    Changes to be committed:
      (use "git rm --cached <file>..." to unstage)  <- "잘못 올렸으면 이 명령으로 Staging Area에서 다시 빼라"는 안내
            new file:   index.html                  <- new file은 "저장소에 처음 추가되는 파일"
    ```
    
- Commit 생성
    
    ```bash
    git commit -m "Create index page"
    ```
    

### **3단계: 파일 수정과 git diff**

<aside>
📌**명령어 정리**

- `git diff` : 아직 Staging하지 않은 변경 내용 비교 (`-` 이전 줄, `+` 바뀐 줄)
- 참고 : Working Directory(작업 공간) / Staging Area(다음 Commit 목록) / Local Repository(Commit 이력 저장)
</aside>

- `index.html` 내용 수정 (VSCode 편집기에서 직접 수정·저장하거나 아래 명령 실행)
    - 변경 전: `<h1>KCU Cloud DevOps</h1>`
    - 변경 후: `<h1>KCU Cloud DevOps Automation</h1>`
    
    ```bash
    echo "<h1>KCU Cloud DevOps Automation</h1>" > index.html
    ```
    
- 아직 Staging하지 않은 변경 내용 확인
    
    ```bash
    git diff
    ```
    
- 두 번째 Commit
    
    ```bash
    git add index.html
    git commit -m "Update index page"
    ```
    

### **4단계: 새 파일 추가와 이력 확인**

<aside>
📌**명령어 정리**

- `git log` : Commit 이력 자세히 보기 (Commit ID, 작성자, 날짜, 메시지)
- `git log --oneline` : Commit 이력을 한 줄씩 간단히 보기 (최신 Commit이 맨 위)
</aside>

- 새 파일 추가 후 세 번째 Commit
    
    ```bash
    echo "<h1>About KCU</h1>" > about.html
    git add about.html
    git commit -m "Add about page"
    ```
    
- Commit 이력 확인 (**최신 Commit이 맨 위**, Commit ID는 실습자마다 다름)
    
    ```bash
    git log
    ```
    
    ```bash
    git log --oneline
    ```
    

### **5단계: feature Branch 작업**

<aside>
📌 **명령어 정리**

- `git branch <브랜치이름>` : 브랜치 만들기 (이동은 안 함)
- `git switch <브랜치이름>` : 그 브랜치로 이동
- `git switch -c <브랜치이름>` : 브랜치 생성 후 이동
- `git branch` : 브랜치 목록 보기 (`*` = 현재 브랜치)
- `git log --oneline --graph --all` : 모든 브랜치의 Commit 이력을 한 줄씩, 그래프로 보기
</aside>

- Branch 생성과 동시에 이동 → 현재 Branch 앞에 `*` 표시
    
    ```bash
    git branch
    
    git switch -c feature
    
    git branch
    * feature
      main
    ```
    
- feature Branch에서 index.html 파일 수정 후 Commit (`>>`는 파일 끝에 내용 추가)
    
    ```bash
    echo "<p>New Feature</p>" >> index.html
    git add index.html
    git commit -m "Add new feature"
    ```
    

### **6단계: main으로 Merge**

<aside>
📌**명령어 정리**

- `git switch main` : main 브랜치로 이동
- `git merge <브랜치이름>` : 지정한 브랜치의 변경 내용을 현재 브랜치에 합치기
</aside>

- main으로 이동 → feature에서 추가한 `New Feature` 줄이 **없음**을 확인
    
    ```bash
    git switch main
    cat index.html
    ```
    
- feature 변경 내용을 main에 병합
    
    ```bash
    git merge feature
    cat index.html
    ```    

---

## 10주 3강 **Hands-on Labs : GitHub 원격 저장소 연동**

---

> 2강에서 관리중인 로컬 저장소를 github 원격 저장소와 연동
- github 인증 token 생성 및 repository 생성 
local PC에서 관리하는 git 파일을 github repository에 push - main
local PC에서 feature-contact 브랜치 생성, contect.html 문서 생성 후 push
- github 에서 main branch로 병합
> 

### **1단계: GitHub 준비 (Token 생성 · Repository 생성)**

<aside>
📌

**용어 정리**

- `Personal Access Token(PAT)` : Git 명령으로 GitHub에 접근할 때 **비밀번호 대신** 사용하는 인증 문자열
- GitHub는 `git push` 할 때 계정 비밀번호 로그인을 지원하지 않으므로 토큰이 필요함
- `Scope(권한 범위)` : 토큰으로 할 수 있는 일의 범위 (`repo` : 저장소 읽기·쓰기, `workflow` : GitHub Actions 파일 수정)
- `Repository` : GitHub에 만드는 프로젝트 저장 공간 (2강의 로컬 저장소를 올릴 빈 저장소)
</aside>

#### **① Personal Access Token 생성**

- GitHub 로그인 → 오른쪽 위 **프로필 사진** → **Settings**
- 왼쪽 메뉴 맨 아래 **Developer settings** → **Personal access tokens** → **Tokens (classic)**
- **[Generate new token]** → **Generate new token (classic)**
    - 본인 확인 창이 뜨면 GitHub 비밀번호 또는 인증 코드 입력
- 토큰 설정
    
    
    | 항목 | 설정 |
    | --- | --- |
    | Note | `kcu-git-lab-token` |
    | Expiration | `90 days` (학기 중 계속 사용) |
    | Select scopes | ☑ `repo` , ☑ `workflow` |
    
    <img width="1151" height="700" alt="Image" src="https://github.com/user-attachments/assets/26e6438a-1556-495a-9d3a-6a06ddfbd430" />
    
- 맨 아래 **[Generate token]** 클릭
- 생성된 토큰(`ghp_`로 시작)을 **바로 복사**해서 메모장 등에 보관
    
    ```
    ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    ```
    
    > ⚠️ 토큰은 생성 화면에서 **한 번만** 보입니다. 페이지를 벗어나면 다시 볼 수 없으므로 반드시 복사해 둡니다. 토큰은 비밀번호와 같으므로 다른 사람과 공유하거나 파일에 적어 Commit하지 않습니다.
    > 
    
    > 💡 `workflow` 권한은 11주차 GitHub Actions 실습에서 `.github/workflows` 파일을 push할 때 필요합니다. 토큰을 잃어버렸거나 만료되면 같은 방법으로 새로 만들면 됩니다.
    > 

#### **② GitHub Repository 생성**

- GitHub 로그인 → 오른쪽 위 **[+]** → **New repository**
    - Repository name : `kcu-git-lab`
    - Visibility : **Public**
    - Add README / .gitignore / license : **선택하지 않음** (빈 저장소)
    
    <img width="751" height="761" alt="Image" src="https://github.com/user-attachments/assets/299cdfea-63bc-4510-86e9-e07acd3d4217" />
    
- **[Create repository]** → HTTPS 주소 복사
    
    <img width="1273" height="706" alt="Image" src="https://github.com/user-attachments/assets/0c6e190d-1eee-4e4d-985d-abbcbc420a4e" />
    
    ```
    https://github.com/<Username>/kcu-git-lab.git
    ```
    

> ⚠️ README를 함께 만들면 첫 push가 거부(rejected)됩니다. 반드시 빈 저장소로 만듭니다.
> 

### **2단계: 원격 저장소 연결**

<aside>
📌

**명령어 정리**

- `git remote add origin <주소>` : GitHub 저장소 주소를 `origin`이라는 이름으로 등록
- `git remote -v` : 등록된 원격 저장소 주소 확인 (fetch · push 2줄)
- `origin` : 원격 저장소에 관례적으로 붙이는 기본 별명
</aside>

- 2강에서 만든 저장소로 이동 후 Commit 이력 확인
    
    ```bash
    cd ~/kcu-git-lab
    git log --oneline
    ```
    
- `origin`이라는 이름으로 GitHub 저장소 주소 등록
    
    ```bash
    git remote add origin https://github.com/<Username>/kcu-git-lab.git
    git remote -v
    ```
    
    ```
    origin  https://github.com/<Username>/kcu-git-lab.git (fetch) <- pull, fetch 에 적용되는 주소
    origin  https://github.com/<Username>/kcu-git-lab.git (push)  <- push에 적용되는 주소
    ```
    

> **참고**: Git Bash에 붙여넣기는 `Shift + Insert` 또는 마우스 오른쪽 버튼을 사용합니다.
> 

### **3단계: GitHub로 Push (main branch)**

[!NOTE]
📌**명령어 정리**
- `git branch -M main` : 현재 브랜치 이름을 main으로 변경 (master인 경우만)
- `git push -u origin main` : 로컬 main의 Commit을 origin(GitHub)으로 전송 (`-u` : 이 연결을 기억)
- `git push` : 두 번째부터는 이것만 입력
- 인증 : 비밀번호 대신 **1단계에서 만든 Personal Access Token** 사용


- 현재 Branch가 `main`인지 확인
    
    ```bash
    git branch
    git branch -M main 
    ```
    
- 최초 Push (`-u`: origin/main을 기본 업로드 대상으로 기억)
    
    ```bash
    git push -u origin main
    ```
    
- 인증 창이 뜨면 **1단계에서 만든 토큰**으로 로그인
    - **Windows** (Git Credential Manager 창) : **Token** 항목 선택 → 토큰 붙여넣기 → **[Sign in]**
        
        <img width="627" height="548" alt="Image" src="https://github.com/user-attachments/assets/3d0b93d0-2007-4a97-b89b-1e29142805a8" />
        
        <img width="504" height="399" alt="Image" src="https://github.com/user-attachments/assets/b05a625c-de73-4d96-883b-bf865d934e6c" />
        
        <img width="504" height="537" alt="Image" src="https://github.com/user-attachments/assets/6f5ce07f-050b-4379-9063-3456a98598c6" />
        
        <img width="519" height="818" alt="Image" src="https://github.com/user-attachments/assets/19c84700-6a7b-4524-a882-55dc3381e80a" />
        
    - **터미널에서 직접 묻는 경우 (Mac 등)**
        - `Username` : GitHub Username 입력
        - `Password` : GitHub 비밀번호가 아닌 **토큰** 붙여넣기 (입력해도 화면에 표시되지 않음)
    - 한 번 인증하면 PC에 저장되어 다음 push부터는 다시 묻지 않음
        
        > 💡 인증 창이 뜨지 않고 `403` 또는 `Permission ... denied to <다른계정>` 오류가 나면, PC에 저장된 다른 GitHub 계정 정보 때문입니다. 
        Windows 검색 → **자격 증명 관리자** → **Windows 자격 증명** → `git:https://github.com` 제거 후 다시 push합니다.
        > 
    - github reposiotry 에 업로드된 파일 확인
        
        <img width="935" height="574" alt="Image" src="https://github.com/user-attachments/assets/13b71ce7-dff2-4251-b69e-506b63d2c5cf" />
        

### **4단계: GitHub에서 확인**

<aside>
📌

**확인 포인트**

- `git log --oneline` 결과의 `origin/main` : GitHub(원격) main이 가리키는 Commit
- `HEAD -> main, origin/main`이 같은 줄에 있으면 로컬과 GitHub가 같은 상태
</aside>

- GitHub 저장소 페이지 새로고침
    - `index.html`, `about.html` 파일 확인
    - **Commits** 클릭 → Commit 4개 확인
- 로컬에서도 확인 → `origin/main` 표시
    
    ```bash
    git log --oneline
    ```
    
    ```
    xxxxxxx (HEAD -> main, origin/main) Add new feature
    ```
    

> **참고**: 이후에는 `git push`만 입력하면 됩니다.
> 

### **5단계: Clone**

<aside>
📌

**명령어 정리**

- `git clone <주소> <폴더이름>` : 원격 저장소 전체(파일 + Commit 이력 + origin 연결)를 지정한 폴더로 복제
- 폴더 이름을 생략하면 저장소 이름(`kcu-git-lab`)으로 폴더가 만들어짐
</aside>

- 새 개발자 PC라고 가정하고, 다른 폴더(`clone-lab`)에 저장소 복제
    
    ```bash
    cd ~
    git clone https://github.com/<Username>/kcu-git-lab.git clone-lab
    #git clone https://github.com/seongmi-lee/kcu-git-lab.git clone-lab
    cd clone-lab
    ls
    git log --oneline
    ```
    
- GitHub의 파일과 Commit 이력이 그대로 복제되었는지 확인

### **6단계: Pull**

<aside>
📌

**명령어 정리**

- `git pull` : GitHub에 새로 생긴 Commit을 가져와 현재 브랜치에 합치기
- Push는 올리기(로컬 → GitHub), Pull은 받기(GitHub → 로컬)
</aside>

- GitHub 웹에서 원격 저장소 변경 (다른 개발자가 Push한 상황)
    - **[Add file]** → **Create new file**
    - 파일 이름: `README.md`
    - 내용: `# KCU Git Lab`
    - **[Commit changes]** 클릭
- 로컬 저장소에서 변경 내용 가져오기 → `README.md` 생성 확인
    
    ```bash
    cd ~/kcu-git-lab
    git pull
    ls
    git log --oneline
    ```
    

### **7단계: Branch Push와 Pull Request (feature-contact branch 생성)**

<aside>
📌

**명령어 정리**

- `git switch -c feature-contact` : 작업용 브랜치 생성 후 이동
- `git push -u origin feature-contact` : 새 브랜치를 GitHub에 올리기
- Pull Request : 내 브랜치를 main에 합쳐 달라고 GitHub에서 검토를 요청하는 기능
</aside>

- 새 Branch에서 파일 추가 후 GitHub로 Push
    
    ```bash
    cd ~/kcu-git-lab
    git switch -c feature-contact
    echo "<h1>Contact</h1>" > contact.html
    git add contact.html
    git commit -m "Add contact page"
    git push -u origin feature-contact
    ```
    
- GitHub 저장소 화면 → **[Compare & pull request]**
    - 새 브랜치 feature-conact 를 push하면 github에는 [Compare & pull request]가 표시됨.
    - 이는 현재 브랜치에서 바꾼 내용을 main에 합쳐 달라고 요청서를 만드는 것이다.
    
    <img width="1235" height="249" alt="image" src="https://github.com/user-attachments/assets/cd5772e1-b88c-4a5e-9d22-702e53ad7bad" />

    
- base: `main` ← compare: `feature-contact` 확인 → 제목·설명 작성 → **[Create pull request]**
    - 제목:  Add contact page
    - 설명:
        
        ```bash
        ## 변경 내용
        - 연락처 페이지(contact.html) 추가
        
        ## 확인 방법
        - contact.html 파일이 추가되었는지 Files changed 탭에서 확인
        ```
        
    
    <img width="1894" height="1022" alt="Image" src="https://github.com/user-attachments/assets/2587aa29-00ce-4127-a33c-cd511785dd3b" />
    
- **[Files changed]** 탭에서 변경 내용 확인
    
    <img width="638" height="340" alt="Image" src="https://github.com/user-attachments/assets/d43ee9da-dc20-420c-a5e5-608d5e585e57" />
    

### **8단계: Merge 후 로컬 반영**

<aside>
📌

**명령어 정리**

- [Merge pull request] : GitHub에서 브랜치를 main에 합치기 (GitHub의 main만 바뀜)
- `git switch main` → `git pull` : GitHub에서 합쳐진 결과를 내 PC의 main에도 반영
</aside>

- Pull Request 화면 → **[Merge pull request]** → **[Confirm merge]**
    
    <img width="919" height="732" alt="Image" src="https://github.com/user-attachments/assets/64c335e5-28f6-4821-98b7-1a2bb882b552" />
    
- 로컬 main으로 이동 후 최신 내용 가져오기 → `contact.html` 확인
    
    ```bash
    git switch main
    ls
    
    # github에서 다시 pull
    git pull
    ls
    ```
    

---

### 리소스 삭제

- 내 PC의 Git 저장소 (Git Bash)
    - **VSCode에서 폴더 닫기 :** 메뉴 → **File → Close Folder** 를 선택
    - 터미널에서 디렉토리 삭제
    
    ```bash
    cd ~
    rm -rf ~/kcu-git-lab ~/clone-lab
    ls ~        # 두 폴더가 없어졌는지 확인
    ```
    
- github repository 삭제
    - `https://github.com/<username>/kcu-git-lab`에 접속
    - 위쪽 **Settings** 탭 → 맨 아래 **Danger Zone** → **Delete this repository**를 누릅니다.
    - 확인 창에서 저장소 이름 `<username>/kcu-git-lab`을 입력하고 삭제합니다.
