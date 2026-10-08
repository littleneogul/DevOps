내 이미지 주소 : ghcr.io/littleneogul/guestbook:v2

# ---------------------------------------------------------------
# 학생 과제: 아래 각 줄이 무엇을 하는지 주석으로 설명을 달아보세요.
# ---------------------------------------------------------------

FROM python:3.12-slim	# 기반 이미지 입니다

WORKDIR /app		# 작업 디렉터리를 지정합니다

COPY requirements.txt .	# 호스트 파일 -> 이미지
RUN pip install --no-cache-dir -r requirements.txt	# 빌드할 때 실행합니다

COPY . .	

RUN useradd -m appuser
USER appuser

ENV APP_TITLE="황현우 DevOps 방명록" \
    THEME_COLOR="#50C878"		# 환경변수 설정입니다

EXPOSE 5000				# 포트 번호를 문서화 합니다

CMD ["python", "app.py"]		# 컨테이너를 시작할 때 실행시킬 명령어 입니다
