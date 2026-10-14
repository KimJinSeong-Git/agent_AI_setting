# Mac Studio (64GB) + Synology NAS 로컬 AI 연구·비서 구축 가이드 (MariaDB 단독 버전)

## 0. 전체 아키텍처 요약 (Qdrant 제외 버전)
- **프론트엔드:** Open WebUI (`:3000`, Docker 또는 `uvx`) — 사설망 접속 및 슬래시(`/`) 명령어 제공
- **메인 오케스트레이터:** Hermes Agent (`:8642`, Native) — 로컬 **Qwen 3.8 27B** 기반 일상 비서, 라우팅, MCP 제어
- **무거운 코딩/실험 엔진:** Antigravity CLI (`agy`) — 멀티 파일 수정, 리팩토링, 연구용 분류기(`text_cls`, `image_cls`) 학습 실행
- **미디어 생성 엔진:** ComfyUI (`:8188`, 기존 GUI 유지) — 64GB RAM 보호를 위해 Qwen 27B 자동 언로드 후 생성 및 VRAM 반환
- **통합 스토리지 & DB (Synology NAS 단독):**
  - **MariaDB (`:3306`):** 논문 메타데이터/BibTeX, Full-Text 지식 검색, Laya(System 1) 로그, 페르소나, DPO 선호도, 실험 트래커 통합 처리
  - **Gitea (`:3000`):** 에이전트 작성 코드 자동 레포지토리 생성 및 버전 관리
  - **SMB 공유 폴더 (`/Volumes/NAS`):** 논문 PDF 원본 및 학습 완료된 모델 체크포인트 보관

---

## Step 1. 기존 환경 완전 초기화 및 기본 패키지 설치 (Mac Studio)

```bash
# 1. 실행 중인 프로세스 종료 및 설정/세션 초기화 (Factory Reset)
hermes gateway stop 2>/dev/null
pkill -f "hermes"
pkill -f "agy"
rm -rf ~/.hermes ~/.gemini/antigravity-cli

# 2. 필수 디렉토리 구조 생성
mkdir -p ~/.hermes/skills/paper-review-committee \
         ~/.hermes/scripts \
         ~/.hermes/workflows \
         ~/.hermes/datasets \
         ~/workspace/common \
         ~/workspace/datasets

# 3. Computer Use용 macOS 화면 제어 CLI 설치 및 보조 비전 모델 준비
brew install cliclick Peekaboo
ollama pull qwen2.5-vl:7b

# 4. PyTorch MPS Fallback 환경 변수 등록 (미지원 연산 시 CPU 자동 우회)
grep -q "PYTORCH_ENABLE_MPS_FALLBACK" ~/.zshrc || echo 'export PYTORCH_ENABLE_MPS_FALLBACK=1' >> ~/.zshrc
source ~/.zshrc
```
> **필수 확인:**
> 1. macOS `시스템 설정 -> 개인정보 보호 및 보안`에서 **손쉬운 사용(Accessibility)** 및 **화면 기록(Screen Recording)**에 현재 터미널 앱을 체크합니다.
> 2. Finder(`Cmd + K`)에서 `smb://<NAS_IP>/<공유폴더>`를 연결해 `/Volumes/NAS` 경로로 마운트되어 있는지 확인합니다.

---

## Step 2. 모든 계정·비밀번호·토큰 단일 파일 통합 (`~/.hermes/.env`)

```bash
cat << 'EOF' > ~/.hermes/.env
# 1. Hermes 로컬 API 서버 인증키
API_SERVER_ENABLED=true
API_SERVER_KEY=my-local-secret-key-1234

# 2. Synology NAS & MariaDB 접속 정보
NAS_HOST=192.168.x.x
NAS_MOUNT_PATH=/Volumes/NAS
MARIADB_HOST=192.168.x.x
MARIADB_PORT=3306
MARIADB_USER=kb_user
MARIADB_PASSWORD=여기에_MariaDB_비밀번호_입력
MARIADB_DB=ai_kb

# 3. Synology NAS Gitea 접속 및 API 토큰 (Gitea 설정 -> 애플리케이션에서 발급)
GITEA_URL=[http://192.168.](http://192.168.)x.x:3000
GITEA_USER=본인_Gitea_아이디
GITEA_TOKEN=여기에_Gitea_Access_Token_입력

# 4. Mac Studio 로컬 서비스 엔드포인트
OLLAMA_URL=[http://127.0.0.1:11434](http://127.0.0.1:11434)
COMFY_URL=[http://127.0.0.1:8188](http://127.0.0.1:8188)
LLM_MODEL_NAME=qwen3.8:27b
PYTORCH_ENABLE_MPS_FALLBACK=1
EOF

# 권한 잠금(소유자 전용) 및 macOS 키체인에 Gitea 자격증명 등록
chmod 600 ~/.hermes/.env
source ~/.hermes/.env
git config --global credential.helper osxkeychain
printf "protocol=http\nhost=${NAS_HOST}:3000\nusername=${GITEA_USER}\npassword=${GITEA_TOKEN}\n\n" | git credential-osxkeychain store
```
> **확인:** `nc -zv $MARIADB_HOST 3306` 실행 시 `succeeded!`가 출력되어야 합니다.

---

## Step 3. Synology NAS MariaDB 통합 스키마 생성 (6개 테이블)

Qdrant 없이도 빠르고 정확한 한국어/영어 검색이 가능하도록 `kb_documents`와 `persona_vault` 모두에 **n-gram Full-Text 인덱스**를 적용한 스키마입니다. (NAS phpMyAdmin 또는 MySQL CLI에서 실행)

```sql
CREATE DATABASE IF NOT EXISTS ai_kb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE ai_kb;

-- 1. 논문/뉴스/문서 메타데이터 및 BibTeX (Citation 자동화)
CREATE TABLE IF NOT EXISTS kb_documents (
    doc_id VARCHAR(64) PRIMARY KEY,
    doc_type ENUM('paper', 'news', 'code', 'note', 'media') NOT NULL,
    title VARCHAR(500) NOT NULL,
    authors VARCHAR(500),
    venue VARCHAR(100),
    pub_year INT,
    pub_month TINYINT NULL,
    doi VARCHAR(150) NULL,
    arxiv_id VARCHAR(50) NULL,
    source_url VARCHAR(1000) NULL,
    tags JSON,
    nas_file_path VARCHAR(1000),
    bibtex TEXT NULL,
    summary_md MEDIUMTEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE INDEX idx_doi (doi),
    UNIQUE INDEX idx_arxiv (arxiv_
