# 4주차 - 자동화와 협업 / 네트워크

4주차에는 두 가지를 배웠다.
1. **자동화와 협업**: 셸 스크립트로 반복 작업 자동화, 백업 스크립트, 브랜치 → PR → Conflict 해결 → Merge
2. **네트워크**: IP, 포트, DNS, 네트워크 명령어, 내 컴퓨터에 웹 서버 띄우기, 포트 충돌 해결

## 파일 구성
| 파일 | 설명 |
|---|---|
| `test/diary.txt` | `head`, `tail`, `wc` 연습용 텍스트 |
| `test/server.log` | `grep` 연습용 로그 |
| `test/errors.txt` | `grep ERROR server.log > errors.txt`로 저장한 결과 |
| `test/var.sh` | 변수와 명령 치환 |
| `test/greet.sh` | 인자 개수 확인과 종료 코드 |
| `test/check.sh` | 파일/디렉터리 검사 조건문 |
| `backup.sh` | 폴더를 `tar.gz`로 백업하고 로그를 남기는 스크립트 |
| `site/index.html` | `python3 -m http.server`로 띄운 웹 페이지 |

---

# Part 1. 자동화와 협업

## 1. 경로와 검색
| 표현 | 의미 |
|---|---|
| `/home/.../week04/backup.sh` | 절대 경로 (루트 `/`부터) |
| `./week04/backup.sh` | 상대 경로 (현재 위치 기준) |
| `.` / `..` / `~` / `/` | 현재 / 상위 / 홈 / 루트 |

```bash
find . -type f -name "*.sh"          # 셸 스크립트 파일 찾기
grep -R -n "exit 1" week03           # -R 하위까지, -n 줄 번호
grep -c "ERROR" server.log           # ERROR 줄 개수
grep -v "INFO" server.log            # INFO가 없는 줄만
cat server.log | grep ERROR | wc -l  # 파이프: 앞 출력 → 뒤 입력
grep ERROR server.log > errors.txt   # 결과를 파일로 저장
```

## 2. 변수, 인자, 조건, 반복
```bash
# var.sh - 변수와 명령 치환
USERNAME="황현우"                 # = 앞뒤에 공백 금지
TODAY=$(date +%Y%m%d)
echo "파일명: backup_${TODAY}.tar.gz"

# greet.sh - 인자 확인
if [ $# -eq 0 ]; then             # [ ] 안쪽에 공백 필요
  echo "사용법: $0 [이름]"; exit 1
fi

# 반복문
for f in *.sh; do echo "스크립트: $f"; done
```

| 조건 | 의미 |
|---|---|
| `-f` / `-d` | 파일 / 디렉터리가 존재하는지 |
| `-z` / `-n` | 문자열 길이가 0인지 / 0이 아닌지 |
| `-eq` `-ne` `-gt` `-lt` | 정수 같음 / 다름 / 큼 / 작음 |

## 3. 프로세스
- **프로그램**: 디스크에 저장된 실행 코드 / **프로세스**: 실행 중인 프로그램
```bash
sleep 300 &              # 백그라운드 실행
ps aux | grep sleep      # PID 확인
kill <PID>               # 정상 종료 요청 (SIGTERM, 15)
kill -9 <PID>            # 강제 종료 (SIGKILL, 9) - 최후의 수단
```

## 4. 백업 스크립트 (backup.sh)
1. 인자로 받은 폴더 확인 (없으면 기본값 `test`, `${1:-test}`)
2. `backups/` 폴더 생성
3. `날짜_시간_PID.tar.gz`로 압축 (`$$`로 이름 충돌 방지)
4. `$?`로 성공 여부 확인 후 크기 출력
5. 백업 목록 출력 + `backup.log`에 기록

```bash
./backup.sh test
tar -tzf backups/<파일명>.tar.gz             # 내용 확인
mkdir -p restore-check
tar -xzf backups/<파일명>.tar.gz -C restore-check
diff -rq test restore-check/test && echo $?  # 0이면 원본과 같음
```
백업 결과물은 Git으로 관리하지 않도록 `.gitignore`에 `week04/backups/`, `week04/restore-check/`를 추가했다.

## 5. Git 협업: 브랜치 → PR → Merge
```bash
git switch -c feature/backup-log         # 내 작업실(브랜치) 만들기
# backup.sh에 로그 기록 기능 추가
git status && git diff                   # 바뀐 내용 확인 (습관)
git add week04/backup.sh
git commit -m "feat: 백업 로그 기록 기능 추가"
git push -u origin feature/backup-log
gh pr create --title "백업 로그 기록 기능 추가" --base main
gh pr merge --squash --delete-branch     # 합치고 브랜치 삭제
git switch main && git pull --ff-only
```
→ 결과: PR #1 `feat: 백업 로그 기능 추가`로 main에 반영

## 6. Conflict 해결
main과 `fix/message` 브랜치에서 **같은 줄을 서로 다르게** 고쳐 일부러 충돌을 만들었다.
```
<<<<<<< HEAD
echo "[완료] 로그 기록"
=======
echo "로그 기록"
>>>>>>> origin/main
```
Git이 고장 난 것이 아니라, 무엇을 남길지 **사람이 결정하라**는 표시다. 표시 줄 3개를 지우고 원하는 내용(`echo "[완료] 로그 기록"`)만 남긴 뒤 다시 커밋했다.

```bash
gh pr checkout fix/message
git fetch origin && git merge origin/main   # CONFLICT 발생
# 파일 수정
git add week04/backup.sh
git commit -m "resolve: merge conflict"
git push
gh pr merge --squash --delete-branch
```
흐름: **해결 → add → commit → push → PR 업데이트 → merge** (PR #2 `문구 수정`)

| Merge 옵션 | 동작 |
|---|---|
| `--merge` | 기존 커밋 유지 + merge 커밋 생성 |
| `--squash` | 여러 커밋을 하나로 합쳐 반영 |
| `--rebase` | 브랜치 커밋을 main 뒤에 일렬로 다시 적용 |

---

# Part 2. 네트워크

## 1. 기본 개념
| 개념 | 설명 | 비유 |
|---|---|---|
| IP 주소 | 네트워크에서 장치를 찾는 주소 | 건물 주소 |
| 포트 | 컴퓨터 안의 서비스를 구분하는 번호 (0~65535) | 건물의 호수 |
| DNS | 도메인 이름 → IP 주소 변환 | 전화번호부 |
| NAT | 사설 IP를 공인 IP로 바꿔 인터넷과 통신 | 공유기 |

- `127.0.0.1` = `localhost` = 내 컴퓨터 자신
- `0.0.0.0` = 모든 네트워크 인터페이스에서 연결을 받겠다
- 자주 쓰는 포트: 22(SSH), 80(HTTP), 443(HTTPS), 3306(MySQL), 6379(Redis), 8080(개발용 웹)

## 2. 네트워크 명령어
| 명령 | 확인하는 것 |
|---|---|
| `ifconfig`, `ipconfig getifaddr en0` (맥) | 내 IP |
| `ping -c 4 8.8.8.8` | 상대 장치 응답 여부 (방화벽이 막으면 실패할 수 있음) |
| `dig cs.inhatc.ac.kr +short` | DNS로 IP 조회 |
| `curl -I https://google.com` | HTTP 응답 코드와 헤더 |
| `lsof -nP -iTCP -sTCP:LISTEN` (맥) / `ss -tlnp` | LISTEN 중인 포트와 프로세스 |

응답 코드: 200(성공), 400(잘못된 요청), 404(없음), 500(서버 에러), 503(서비스 불가)

## 3. 내 컴퓨터에 웹 서버 띄우기
```bash
cd ~/devops/week04/site
python3 -m http.server 8080
# 브라우저 → http://localhost:8080 ("Server / This page is running." 확인)

curl -I http://localhost:8080       # 200 OK
lsof -nP -iTCP:8080 -sTCP:LISTEN    # 8080 LISTEN 확인
```

## 4. 포트 충돌과 해결
서버를 켜둔 채 다른 터미널에서 같은 8080 포트로 서버를 또 띄우면 `Address already in use` 오류가 난다.
- 해결 1: 다른 포트 사용 → `python3 -m http.server 8000`
- 해결 2: 기존 프로세스 종료 → PID 확인 후 `kill <PID>`

> 포트를 열어도 방화벽이 막으면 외부에서는 접속할 수 없다.

---

## 핵심 정리
- 자동화는 **반복 가능한 작업**을 만드는 것, 협업은 **변경 사항을 안전하게 합치는** 과정
- 브랜치에서 작업 → PR로 검토 → Merge, 충돌은 사람이 판단해 해결
- 서버는 특정 포트에서 요청을 기다리며, 같은 포트를 두 번 쓰면 충돌이 난다.
- 4주차에 배운 `-p 8080:80` 같은 포트 개념은 이후 도커 실습으로 이어진다.

## 느낀 점
백업처럼 매일 해야 하는 일을 스크립트로 만들고, 여러 사람이 같은 파일을 고칠 때 브랜치와 PR로 안전하게 합치는 과정을 직접 해보면서 DevOps가 "자동화 + 협업"이라는 말을 이해할 수 있었다.
