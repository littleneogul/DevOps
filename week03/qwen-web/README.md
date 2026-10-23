# 3주차 - 셸 스크립팅 (Qwen + Ollama 로컬 대화방)

## 학습 목표
- 반복해서 쓰는 명령어를 셸 스크립트로 작성하고 실행한다.
- 실행 권한, shebang, 경로 처리 방식을 이해한다.
- 환경 변수와 조건문으로 실행 환경을 설정하고 오류에 대응한다.
- Qwen + Ollama로 로컬 AI 서비스를 실행하며 자동화의 필요성을 이해한다.

## 구조
```
브라우저 (http://localhost:8000)
    │  질문 전송 / 답변 표시
    ▼
chat.py (Python 웹 앱, 127.0.0.1:8000)
    │  질문을 API로 전달
    ▼
Ollama 서버 (127.0.0.1:11434)
    │  모델 실행
    ▼
qwen3:0.6b
```

| 구성 | 역할 |
|---|---|
| **Qwen** | 답변을 만드는 모델 (알리바바 클라우드 개발, 0.6b = 약 6억 개 파라미터) |
| **Ollama** | 모델을 내 컴퓨터에서 실행하고 관리하는 로컬 서버 |
| **chat.py** | 파이썬 표준 라이브러리만 쓴 웹 앱 (Flask 등 설치 불필요) |

## 파일 구성
| 파일 | 설명 |
|---|---|
| `chat.py` | 질문을 받아 Ollama API로 보내고 답변을 보여주는 웹 앱 |
| `start.sh` | 어디서 실행하든 이 폴더로 이동해 `chat.py`를 실행하는 스크립트 |
| `run_py.sh` | shebang을 `python3`로 지정한 실습 파일 |

> 모델 파일은 Ollama가 따로 보관하므로 이 폴더에 복사하거나 GitHub에 올리지 않는다.

## 실행 방법
```bash
# 1) Ollama 설치 및 서버 확인 (맥)
curl -fsSL https://ollama.com/install.sh | sh
open -a Ollama
curl -fsS http://127.0.0.1:11434/api/tags   # JSON이 나오면 서버 응답 OK

# 2) 모델 다운로드
ollama pull qwen3:0.6b
ollama list

# 3) 실행
chmod u+x start.sh
./start.sh
# 웹 접속: http://localhost:8000
# 종료: Ctrl+C
```

## start.sh 코드 분석
```bash
#!/bin/bash
cd "$(dirname "$0")" || exit 1
exec python3 chat.py
```

| 코드 | 의미 |
|---|---|
| `#!/bin/bash` | shebang. "이 파일은 Bash로 실행하라"고 OS에 알려줌 (`#` 하나면 주석) |
| `$0` | 지금 실행한 스크립트의 이름 또는 경로 |
| `dirname` | 경로에서 파일 이름을 뺀 디렉터리 부분만 출력 |
| `$( ... )` | 명령어 치환. 괄호 안 명령의 출력을 그 자리에 넣음 |
| `"..."` | 경로에 공백이 있어도 안전하도록 따옴표로 감쌈 |
| `\|\| exit 1` | 앞 명령(cd)이 실패하면 실패 코드 1을 남기고 종료 |
| `exec` | 현재 bash를 python3로 **교체** (PID 유지). 그래서 아래 줄은 실행되지 않음 |

종료 상태는 성공이면 `0`, 실패면 `0 이외`의 값이다. 숫자 0이 거짓인 것과 반대이므로 헷갈리지 않도록 주의한다.

### shebang 실습 (run_py.sh)
```bash
#!/usr/bin/env python3
print('Hello, DevOps!')
```
```bash
chmod +x run_py.sh
./run_py.sh      # Hello, DevOps!
```
확장자가 `.sh`여도 첫 줄의 shebang이 `python3`이면 파이썬으로 실행된다.

## 자주 만난 오류
| 오류 | 원인 | 해결 |
|---|---|---|
| `Permission denied` | 실행 권한 없음 | `chmod u+x start.sh` |
| `command not found` | 이름만 쓰면 PATH에서 찾음 | `./start.sh`처럼 경로를 붙여 실행 |

## 환경 변수로 설정 바꾸기
코드를 고치지 않고 모델이나 포트를 바꿀 수 있다.
```bash
cat > start_with_export_2.sh << 'EOF'
#!/bin/bash
cd "$(dirname "$0")" || exit 1
export WEB_PORT="8080"
exec ./start.sh
EOF
chmod u+x start_with_export_2.sh
./start_with_export_2.sh     # → http://localhost:8080
```
- `<< 'EOF'` (heredoc): `EOF`가 나올 때까지 여러 줄을 입력받아 파일로 저장
- `'EOF'`처럼 따옴표를 쓰면 `$변수`가 현재 값으로 바뀌지 않고 글자 그대로 저장됨
- `export`는 이 스크립트 안에서만 적용되고, 원래 터미널의 환경 변수는 바뀌지 않음

## 조건문으로 친절한 스크립트 만들기
```bash
if ! command -v python3 >/dev/null 2>&1; then
  echo "Python3를 먼저 설치하세요." >&2
  exit 1
fi
exec python3 chat.py
```
| 코드 | 의미 |
|---|---|
| `command -v python3` | python3가 있으면 경로 출력 + 성공, 없으면 실패 |
| `!` | 결과 뒤집기 → "python3를 찾을 수 **없다면**" |
| `>/dev/null` | 표준 출력을 버림 (있는지 없는지만 알면 되므로) |
| `2>&1` | 표준 오류(2)도 표준 출력(1)이 가는 곳으로 보냄 |
| `>&2` | 안내 문구를 오류 메시지(stderr)로 출력 |
| `exit 1` | 오류 상태로 종료 |

## 핵심 정리
- 셸 스크립트는 여러 명령을 파일에 적어 **매번 같은 절차로** 실행하게 해준다.
- 실행 권한과 shebang을 확인하고, `cd "$(dirname "$0")"`로 작업 경로를 고정한다.
- 환경 변수로 실행 설정을 전달하고, 종료 상태로 성공/실패를 판단한다.
- 표준 출력, 표준 오류, 실패 종료를 구분해서 사용한다.

## 느낀 점
매번 `cd`로 경로를 외워서 들어가야 했던 실행 과정을 `./start.sh` 한 줄로 줄이면서, 사람이 실수할 수 있는 반복 작업을 기계에 맡기는 것이 자동화의 시작이라는 것을 느꼈다.
