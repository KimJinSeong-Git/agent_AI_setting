# Mac Studio (64GB) + Synology NAS 로컬 AI 연구·비서 마스터 구축 가이드

## 0. 전체 아키텍처 요약
- **프론트엔드:** Open WebUI (`:3000`, Docker) — 사설망(Tailscale/역방향 프록시) 접속 및 슬래시(`/`) 명령어 제공
- **메인 오케스트레이터:** Hermes Agent (`:8642`, Native) — 로컬 **Qwen 3.8 27B** 기반 일상 비서, 라우팅, MCP 제어
- **무거운 코딩/실험 엔진:** Antigravity CLI (`agy`) — 멀티 파일 수정, 리팩토링, 연구용 분류기(`text_cls`, `image_cls`) 학습 실행
- **미디어 생성 엔진:** ComfyUI (`:8188`, 기존 GUI 유지) — 64GB RAM 보호를 위해 Qwen 27B 자동 언로드 후 생성 및 VRAM 반환
- **통합 스토리지 & DB:**
  - **Synology NAS:** MariaDB (`:3306`, 메타데이터/BibTeX/Laya/페르소나/DPO/실험로그), Gitea (`:3000`, 코드 버전 관리), SMB 공유 폴더 (`/Volumes/NAS`, PDF 원본 및 모델 체크포인트)
  - **Mac Studio:** Qdrant (`:6333`, `bge-m3` 기반 시맨틱 검색), 로컬 NVMe (`~/workspace`, 데이터셋 및 학습 워킹 디렉토리)

---

## Step 1. 기존 환경 완전 초기화 및 기본 패키지 설치 (Mac Studio)

기존에 테스트하며 꼬인 설정과 세션을 모두 지우고 필수 도구를 준비합니다.

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

# 3. Computer Use용 macOS 화면 제어 CLI 설치
brew install cliclick Peekaboo

# 4. PyTorch MPS Fallback 환경 변수 등록 (미지원 연산 시 CPU 자동 우회)
grep -q "PYTORCH_ENABLE_MPS_FALLBACK" ~/.zshrc || echo 'export PYTORCH_ENABLE_MPS_FALLBACK=1' >> ~/.zshrc
source ~/.zshrc
```
> **확인 사항:**
> 1. `ls -la ~/.hermes` 실행 시 방금 만든 빈 하위 폴더만 존재하는지 확인합니다.
> 2. macOS `시스템 설정 -> 개인정보 보호 및 보안`에서 **손쉬운 사용(Accessibility)** 및 **화면 기록(Screen Recording)**에 현재 터미널 앱을 허용합니다.
> 3. Finder(`Cmd + K`)에서 `smb://<NAS_IP>/<공유폴더>`를 연결해 `/Volumes/NAS` 경로로 접근 가능한지 확인합니다.

---

## Step 2. 모든 계정·비밀번호·토큰 단일 파일 통합 (`~/.hermes/.env`)

비밀번호를 코드에 하드코딩하지 않도록 `~/.hermes/.env` 한 곳에 모으고 소유자 전용 권한(`600`)으로 잠급니다.

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
QDRANT_URL=[http://127.0.0.1:6333](http://127.0.0.1:6333)
OLLAMA_URL=[http://127.0.0.1:11434](http://127.0.0.1:11434)
COMFY_URL=[http://127.0.0.1:8188](http://127.0.0.1:8188)
LLM_MODEL_NAME=qwen3.8:27b
PYTORCH_ENABLE_MPS_FALLBACK=1
EOF

# 권한 잠금 및 macOS 키체인에 Gitea 자격증명 등록
chmod 600 ~/.hermes/.env
source ~/.hermes/.env
git config --global credential.helper osxkeychain
printf "protocol=http\nhost=${NAS_HOST}:3000\nusername=${GITEA_USER}\npassword=${GITEA_TOKEN}\n\n" | git credential-osxkeychain store
```
> **확인 사항:** `nc -zv $MARIADB_HOST 3306` 실행 시 `succeeded!`가 출력되어야 합니다.

---

## Step 3. Synology NAS MariaDB 통합 스키마 생성 (6개 테이블)

NAS의 phpMyAdmin 또는 MySQL 클라이언트에서 아래 SQL을 실행합니다.

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
    UNIQUE INDEX idx_arxiv (arxiv_id),
    FULLTEXT INDEX idx_ft_search (title, summary_md) WITH PARSER ngram
);

-- 2. Hermes & Antigravity 실행 로그
CREATE TABLE IF NOT EXISTS agent_logs (
    log_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    executor ENUM('hermes-qwen', 'agy', 'subagent') NOT NULL,
    project_path VARCHAR(500),
    user_prompt TEXT,
    result_summary TEXT,
    status ENUM('success', 'error', 'timeout') DEFAULT 'success',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 3. System 1 (Laya) 학습용 의사결정/라우팅 로그
CREATE TABLE IF NOT EXISTS sys1_decision_logs (
    decision_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    state_text TEXT NOT NULL,
    question_type ENUM('Choice', 'Score', 'Noul') NOT NULL DEFAULT 'Choice',
    question_prompt VARCHAR(500) NOT NULL,
    options JSON NOT NULL,
    selected_option VARCHAR(100) NOT NULL,
    outcome_reward FLOAT DEFAULT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 4. 나의 페르소나 및 철학 아카이브 (Digital Twin)
CREATE TABLE IF NOT EXISTS persona_vault (
    persona_id VARCHAR(64) PRIMARY KEY,
    category ENUM('philosophy', 'research_view', 'life_rule', 'writing_style', 'decision_criteria') NOT NULL,
    question TEXT NOT NULL,
    my_answer TEXT NOT NULL,
    core_principle VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 5. 선호도 피드백 루프 (DPO / Preference Dataset)
CREATE TABLE IF NOT EXISTS preference_feedback (
    pref_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    context_prompt TEXT NOT NULL,
    rejected_response MEDIUMTEXT NOT NULL,
    chosen_response MEDIUMTEXT NOT NULL,
    feedback_reason VARCHAR(500),
    task_domain VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 6. 연구용 분류기(Text/Image Classifier) 및 공격/방어 실험 트래커
CREATE TABLE IF NOT EXISTS research_experiments (
    exp_id VARCHAR(64) PRIMARY KEY,
    project_name VARCHAR(100) NOT NULL,
    task_type ENUM('text_cls', 'image_cls', 'attack', 'defense', 'sys1_ft') NOT NULL,
    backbone VARCHAR(100) NOT NULL,
    dataset_name VARCHAR(100) NOT NULL,
    hyperparams JSON,
    metrics JSON,
    checkpoint_nas_path VARCHAR(1000),
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## Step 4. Mac Studio Qdrant 구동 및 ComfyUI API 포맷 추출

### 4-1. Qdrant 및 로컬 보조 모델 준비
```bash
# 다국어 임베딩 모델 및 Computer Use용 경량 비전 모델 다운로드
ollama pull bge-m3
ollama pull qwen2.5-vl:7b

# Qdrant Vector DB 컨테이너 실행
docker rm -f qdrant 2>/dev/null
docker run -d -p 6333:6333 -p 6334:6334 \
  -v ~/.qdrant_storage:/qdrant/storage:z \
  --name qdrant \
  --restart always \
  qdrant/qdrant
```

### 4-2. 기존 ComfyUI GUI에서 워크플로우 추출 (기존 환경 그대로 유지)
1. 평소 사용하는 ComfyUI GUI(`http://127.0.0.1:8188`) 접속 $\rightarrow$ 우측 톱니바퀴(Settings) $\rightarrow$ **`Enable Dev mode Options`** 체크.
2. 이미지 생성 워크플로우를 열고 **`Save (API Format)`** 클릭 $\rightarrow$ `~/.hermes/workflows/image_api.json`으로 저장.
3. 영상 생성 워크플로우를 열고 **`Save (API Format)`** 클릭 $\rightarrow$ `~/.hermes/workflows/video_api.json`으로 저장.
   *(팁: 긍정 프롬프트 입력 노드의 Title에 `Positive`가 포함되어 있는지 확인하세요.)*

---

## Step 5. 올인원 MCP 브릿지 서버 및 실험 로거 생성

### 5-1. `~/.hermes/scripts/kb_mcp_server.py` 생성
MariaDB, Qdrant, Gitea, arXiv 크롤러, ComfyUI 64GB 메모리 스위칭을 모두 전담하는 핵심 스크립트입니다.

```python
import os, time, uuid, json, subprocess, urllib.request, urllib.parse
import xml.etree.ElementTree as ET
import pymysql
import httpx
from dotenv import load_dotenv
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct
from fastmcp import FastMCP

load_dotenv(os.path.expanduser("~/.hermes/.env"))
mcp = FastMCP("Master-Research-Assistant-MCP")

DB_CONFIG = {
    "host": os.environ.get("MARIADB_HOST", "127.0.0.1"),
    "port": int(os.environ.get("MARIADB_PORT", 3306)),
    "user": os.environ.get("MARIADB_USER", "root"),
    "password": os.environ.get("MARIADB_PASSWORD", ""),
    "database": os.environ.get("MARIADB_DB", "ai_kb"),
    "charset": "utf8mb4",
    "cursorclass": pymysql.cursors.DictCursor,
    "autocommit": True
}
QDRANT_URL = os.environ.get("QDRANT_URL", "[http://127.0.0.1:6333](http://127.0.0.1:6333)")
OLLAMA_URL = os.environ.get("OLLAMA_URL", "[http://127.0.0.1:11434](http://127.0.0.1:11434)")
COMFY_URL = os.environ.get("COMFY_URL", "[http://127.0.0.1:8188](http://127.0.0.1:8188)")
LLM_MODEL_NAME = os.environ.get("LLM_MODEL_NAME", "qwen3.8:27b")
NAS_MOUNT_PATH = os.environ.get("NAS_MOUNT_PATH", "/Volumes/NAS")
GITEA_URL = os.environ.get("GITEA_URL", "[http://127.0.0.1:3000](http://127.0.0.1:3000)").rstrip("/")
GITEA_USER = os.environ.get("GITEA_USER", "")
GITEA_TOKEN = os.environ.get("GITEA_TOKEN", "")

qdrant = QdrantClient(url=QDRANT_URL)

def get_db():
    return pymysql.connect(**DB_CONFIG)

def get_embedding(text: str) -> list[float]:
    resp = httpx.post(f"{OLLAMA_URL}/api/embeddings", json={"model": "bge-m3", "prompt": text}, timeout=30.0)
    return resp.json()["embedding"]

def ensure_collection(name: str, size: int = 1024):
    cols = [c.name for c in qdrant.get_collections().collections]
    if name not in cols:
        qdrant.create_collection(collection_name=name, vectors_config=VectorParams(size=size, distance=Distance.COSINE))

def unload_ollama_llm():
    try:
        httpx.post(f"{OLLAMA_URL}/api/generate", json={"model": LLM_MODEL_NAME, "keep_alive": 0}, timeout=10.0)
    except Exception:
        pass

def free_comfyui_memory():
    try:
        httpx.post(f"{COMFY_URL}/free", json={"unload_models": True, "free_memory": True}, timeout=10.0)
    except Exception:
        pass

@mcp.tool()
def check_system_health() -> str:
    """MariaDB, Qdrant, Gitea, ComfyUI, NAS 마운트 상태를 한 번에 점검합니다."""
    status = {}
    try:
        with get_db() as conn:
            with conn.cursor() as cur:
                cur.execute("SHOW TABLES")
                status["MariaDB"] = f"OK ({len(cur.fetchall())} tables)"
    except Exception as e:
        status["MariaDB"] = f"ERROR: {e}"
    try:
        ensure_collection("kb_docs")
        ensure_collection("persona_vault")
        status["Qdrant"] = "OK (kb_docs, persona_vault ready)"
    except Exception as e:
        status["Qdrant"] = f"ERROR: {e}"
    try:
        r = httpx.get(f"{GITEA_URL}/api/v1/version", timeout=3.0)
        status["Gitea"] = "OK" if r.status_code == 200 else f"HTTP {r.status_code}"
    except Exception as e:
        status["Gitea"] = f"ERROR: {e}"
    try:
        r = httpx.get(f"{COMFY_URL}/system_stats", timeout=3.0)
        status["ComfyUI"] = "OK (Running)" if r.status_code == 200 else f"HTTP {r.status_code}"
    except Exception:
        status["ComfyUI"] = "Offline (필요 시 GUI 실행)"
    status["NAS_Mount"] = "OK" if os.path.exists(NAS_MOUNT_PATH) else f"Warning: {NAS_MOUNT_PATH} not mounted"
    return json.dumps(status, ensure_ascii=False, indent=2)

@mcp.tool()
def save_document(title: str, summary_md: str, doc_type: str = "paper", venue: str = "", pub_year: int = 2026, pub_month: int = 10, doi: str = None, tags: list[str] = [], nas_file_path: str = "", bibtex: str = None) -> str:
    """문서/논문 요약본을 MariaDB와 Qdrant에 동시에 저장합니다."""
    ensure_collection("kb_docs")
    doc_id = str(uuid.uuid4())
    with get_db() as conn:
        with conn.cursor() as cur:
            cur.execute(
                """INSERT INTO kb_documents (doc_id, doc_type, title, venue, pub_year, pub_month, doi, tags, nas_file_path, bibtex, summary_md)
                   VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)""",
                (doc_id, doc_type, title, venue, pub_year, pub_month, doi, json.dumps(tags, ensure_ascii=False), nas_file_path, bibtex, summary_md)
            )
    vec = get_embedding(f"{title}\n{summary_md}")
    qdrant.upsert(collection_name="kb_docs", points=[PointStruct(id=doc_id, vector=vec, payload={"doc_id": doc_id, "title": title, "doc_type": doc_type, "venue": venue, "pub_year": pub_year})])
    return f"Saved document: {title} (ID: {doc_id})"

@mcp.tool()
def search_knowledge(query: str, top_k: int = 5) -> str:
    """Qdrant 시맨틱 검색 후 MariaDB에서 전체 요약본, 파일 경로, BibTeX를 반환합니다."""
    ensure_collection("kb_docs")
    hits = qdrant.search(collection_name="kb_docs", query_vector=get_embedding(query), limit=top_k)
    if not hits:
        return "검색 결과가 없습니다."
    doc_ids = [h.payload["doc_id"] for h in hits]
    placeholders = ",".join(["%s"] * len(doc_ids))
    with get_db() as conn:
        with conn.cursor() as cur:
            cur.execute(f"SELECT doc_id, title, authors, venue, pub_year, pub_month, doi, nas_file_path, bibtex, summary_md FROM kb_documents WHERE doc_id IN ({placeholders})", doc_ids)
            return json.dumps(cur.fetchall(), ensure_ascii=False, indent=2)

@mcp.tool()
def fetch_and_ingest_arxiv(query: str, max_results: int = 3) -> str:
    """arXiv 최신 논문을 검색해 PDF는 NAS에, 서지정보(연/월, DOI, BibTeX)는 MariaDB와 Qdrant에 저장합니다."""
    url = f"[http://export.arxiv.org/api/query?search_query=all](http://export.arxiv.org/api/query?search_query=all):{urllib.parse.quote(query)}&sortBy=submittedDate&sortOrder=descending&max_results={max_results}"
    root = ET.fromstring(urllib.request.urlopen(url).read())
    ns = {'atom': '[http://www.w3.org/2005/Atom](http://www.w3.org/2005/Atom)', 'arxiv': '[http://arxiv.org/schemas/atom](http://arxiv.org/schemas/atom)'}
    ingested = []
    for entry in root.findall('atom:entry', ns):
        arxiv_id = entry.find('atom:id', ns).text.split('/abs/')[-1].split('v')[0]
        title = entry.find('atom:title', ns).text.strip().replace('\n', ' ')
        summary = entry.find('atom:summary', ns).text.strip()
        published = entry.find('atom:published', ns).text
        pub_year, pub_month = int(published[:4]), int(published[5:7])
        authors = ", ".join([a.find('atom:name', ns).text for a in entry.findall('atom:author', ns)])
        doi_el = entry.find('arxiv:doi', ns)
        doi = doi_el.text if doi_el is not None else f"10.48550/arXiv.{arxiv_id}"
        first_author = authors.split(",")[0].split()[-1].lower()
        cite_key = f"{first_author}{pub_year}{title.split()[0].lower()}"
        bibtex = f"@article{{{cite_key},\n  title={{{title}}},\n  author={{{authors}}},\n  year={{{pub_year}}},\n  month={{{pub_month}}},\n  doi={{{doi}}},\n  eprint={{{arxiv_id}}},\n  archivePrefix={{arXiv}}\n}}"

        save_dir = f"{NAS_MOUNT_PATH}/papers/{pub_year}"
        os.makedirs(save_dir, exist_ok=True)
        pdf_path = f"{save_dir}/{arxiv_id}.pdf"
        if os.path.exists(NAS_MOUNT_PATH) and not os.path.exists(pdf_path):
            urllib.request.urlretrieve(f"[https://arxiv.org/pdf/](https://arxiv.org/pdf/){arxiv_id}.pdf", pdf_path)

        doc_id = str(uuid.uuid5(uuid.NAMESPACE_DNS, f"arxiv:{arxiv_id}"))
        with get_db() as conn:
            with conn.cursor() as cur:
                cur.execute(
                    """INSERT IGNORE INTO kb_documents (doc_id, doc_type, title, authors, venue, pub_year, pub_month, doi, arxiv_id, source_url, nas_file_path, bibtex, summary_md)
                       VALUES (%s, 'paper', %s, %s, 'arXiv', %s, %s, %s, %s, %s, %s, %s, %s)""",
                    (doc_id, title, authors, pub_year, pub_month, doi, arxiv_id, f"[https://arxiv.org/abs/](https://arxiv.org/abs/){arxiv_id}", pdf_path, bibtex, summary)
                )
        ensure_collection("kb_docs")
        qdrant.upsert(collection_name="kb_docs", points=[PointStruct(id=doc_id, vector=get_embedding(f"{title}\n{summary}"), payload={"doc_id": doc_id, "title": title, "pub_year": pub_year, "venue": "arXiv"})])
        ingested.append(f"- {title} ({pub_year}.{pub_month:02d}) [cite: {cite_key}]")
    return "수집 완료:\n" + "\n".join(ingested)

@mcp.tool()
def save_persona_thought(category: str, question: str, my_answer: str, core_principle: str) -> str:
    """사용자의 철학, 연구 관점, 의사결정 원칙을 페르소나 DB와 Qdrant에 기록합니다."""
    ensure_collection("persona_vault")
    p_id = str(uuid.uuid4())
    with get_db() as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO persona_vault (persona_id, category, question, my_answer, core_principle) VALUES (%s, %s, %s, %s, %s)", (p_id, category, question, my_answer, core_principle))
    vec = get_embedding(f"Q: {question}\nA: {my_answer}\nPrinciple: {core_principle}")
    qdrant.upsert(collection_name="persona_vault", points=[PointStruct(id=p_id, vector=vec, payload={"persona_id": p_id, "category": category, "core_principle": core_principle})])
    return f"Saved persona entry (ID: {p_id})"

@mcp.tool()
def search_persona(topic_query: str, top_k: int = 3) -> str:
    """특정 주제에 대한 사용자의 철학과 과거 답변을 검색합니다."""
    ensure_collection("persona_vault")
    hits = qdrant.search(collection_name="persona_vault", query_vector=get_embedding(topic_query), limit=top_k)
    if not hits:
        return "관련된 페르소나 기록이 없습니다."
    p_ids = [h.payload["persona_id"] for h in hits]
    placeholders = ",".join(["%s"] * len(p_ids))
    with get_db() as conn:
        with conn.cursor() as cur:
            cur.execute(f"SELECT category, question, my_answer, core_principle FROM persona_vault WHERE persona_id IN ({placeholders})", p_ids)
            return json.dumps(cur.fetchall(), ensure_ascii=False, indent=2)

@mcp.tool()
def log_routing_and_task(state_text: str, selected_option: str, project_path: str = "", result_summary: str = "", outcome_reward: float = 1.0) -> str:
    """Laya(System 1) 라우팅 로그 및 에이전트 작업 내역을 저장합니다."""
    options = ["hermes_local", "agy_heavy", "kb_search"]
    with get_db() as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO sys1_decision_logs (state_text, question_type, question_prompt, options, selected_option, outcome_reward) VALUES (%s, 'Choice', 'Which executor should handle this task?', %s, %s, %s)", (state_text, json.dumps(options), selected_option, outcome_reward))
            executor = "agy" if selected_option == "agy_heavy" else "hermes-qwen"
            cur.execute("INSERT INTO agent_logs (executor, project_path, user_prompt, result_summary) VALUES (%s, %s, %s, %s)", (executor, project_path, state_text, result_summary))
    return "Logged routing decision and task."

@mcp.tool()
def log_preference_dpo(context_prompt: str, rejected_response: str, chosen_response: str, feedback_reason: str, task_domain: str = "general") -> str:
    """사용자 피드백/교정 내역을 DPO 선호도 테이블에 저장합니다."""
    with get_db() as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO preference_feedback (context_prompt, rejected_response, chosen_response, feedback_reason, task_domain) VALUES (%s, %s, %s, %s, %s)", (context_prompt, rejected_response, chosen_response, feedback_reason, task_domain))
    return "Saved DPO preference feedback."

@mcp.tool()
def gitea_sync_project(project_dir: str, commit_message: str, repo_name: str = "", is_private: bool = True) -> str:
    """로컬 프로젝트 폴더를 NAS Gitea 레포지토리와 자동 동기화(생성, .gitignore 설정, 커밋, 푸시)합니다."""
    target_path = os.path.expanduser(project_dir)
    if not os.path.exists(target_path):
        return f"경로 없음: {target_path}"
    if not repo_name:
        repo_name = os.path.basename(os.path.normpath(target_path))

    headers = {"Authorization": f"token {GITEA_TOKEN}", "Content-Type": "application/json"}
    check_resp = httpx.get(f"{GITEA_URL}/api/v1/repos/{GITEA_USER}/{repo_name}", headers=headers, timeout=10.0)
    if check_resp.status_code == 404:
        httpx.post(f"{GITEA_URL}/api/v1/user/repos", headers=headers, json={"name": repo_name, "private": is_private, "auto_init": False}, timeout=10.0)

    gitignore_path = os.path.join(target_path, ".gitignore")
    default_ignores = [".env", "*.env", "__pycache__/", "*.pt", "*.pth", "*.safetensors", "*.bin", "*.gguf", "datasets/", "checkpoints/", "runs/", ".DS_Store"]
    if not os.path.exists(gitignore_path):
        with open(gitignore_path, "w", encoding="utf-8") as f:
            f.write("\n".join(default_ignores) + "\n")

    remote_url = f"{GITEA_URL}/{GITEA_USER}/{repo_name}.git"
    for cmd in [["git", "init"], ["git", "branch", "-M", "main"], ["git", "remote", "remove", "origin"], ["git", "remote", "add", "origin", remote_url], ["git", "add", "."], ["git", "commit", "-m", commit_message], ["git", "push", "-u", "origin", "main"]]:
        subprocess.run(cmd, cwd=target_path, capture_output=True, text=True)

    commit_hash = subprocess.run(["git", "rev-parse", "--short", "HEAD"], cwd=target_path, capture_output=True, text=True).stdout.strip()
    return f"Gitea 동기화 완료: {GITEA_URL}/{GITEA_USER}/{repo_name} (Commit: {commit_hash})"

@mcp.tool()
def generate_media_with_memory_switch(prompt_en: str, media_type: str = "image", workflow_path: str = "") -> str:
    """64GB RAM 보호를 위해 Qwen 27B를 언로드한 뒤 ComfyUI로 미디어를 생성하고 완료 후 ComfyUI VRAM을 즉시 비웁니다."""
    wf_file = workflow_path if workflow_path else os.path.expanduser(f"~/.hermes/workflows/{media_type}_api.json")
    if not os.path.exists(wf_file):
        return f"워크플로우 파일 없음: {wf_file}"
    with open(wf_file, "r", encoding="utf-8") as f:
        workflow = json.load(f)

    for node in workflow.values():
        if node.get("class_type") == "CLIPTextEncode" and "negative" not in node.get("_meta", {}).get("title", "").lower():
            node["inputs"]["text"] = prompt_en
            break

    try:
        unload_ollama_llm()
        time.sleep(2)
        resp = httpx.post(f"{COMFY_URL}/prompt", json={"prompt": workflow}, timeout=30.0).json()
        prompt_id = resp["prompt_id"]
        output_files = []
        while True:
            hist = httpx.get(f"{COMFY_URL}/history/{prompt_id}", timeout=10.0).json()
            if prompt_id in hist:
                for node_out in hist[prompt_id].get("outputs", {}).values():
                    for item in node_out.get("images", []) + node_out.get("gifs", []) + node_out.get("videos", []):
                        output_files.append(item["filename"])
                break
            time.sleep(3)
        result_file = output_files[0] if output_files else "unknown"
        save_document(title=f"[ComfyUI {media_type}] {prompt_en[:30]}", summary_md=f"Prompt: {prompt_en}\nFile: {result_file}", doc_type="media", tags=["comfyui", media_type], nas_file_path=result_file)
        return f"미디어 생성 완료: {result_file}"
    finally:
        free_comfyui_memory()

@mcp.tool()
def query_research_experiments(project_name: str) -> str:
    """연구용 분류기(Text/Image Classifier) 및 실험 결과를 MariaDB에서 조회합니다."""
    with get_db() as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT exp_id, task_type, backbone, dataset_name, hyperparams, metrics, checkpoint_nas_path, notes, created_at FROM research_experiments WHERE project_name = %s ORDER BY created_at DESC", (project_name,))
            return json.dumps(cur.fetchall(), default=str, ensure_ascii=False, indent=2)

if __name__ == "__main__":
    mcp.run()
```

### 5-2. 연구용 실험 로거 유틸리티 (`~/workspace/common/exp_logger.py`) 생성
Text/Image Classifier 학습 완료 시 체크포인트를 NAS로 백업하고 Git 커밋 해시와 함께 MariaDB에 기록하는 모듈입니다.

```python
import os, shutil, uuid, json, subprocess, pymysql
from dotenv import load_dotenv

load_dotenv(os.path.expanduser("~/.hermes/.env"))

def get_git_commit(project_dir: str) -> str:
    try:
        return subprocess.run(["git", "rev-parse", "--short", "HEAD"], cwd=project_dir, capture_output=True, text=True).stdout.strip() or "uncommitted"
    except Exception:
        return "no_git"

def log_experiment(project: str, task_type: str, backbone: str, dataset: str, hyperparams: dict, metrics: dict, local_ckpt_dir: str = "", notes: str = ""):
    exp_id = f"{project}_{backbone.split('/')[-1]}_{uuid.uuid4().hex[:6]}"
    nas_mount = os.environ.get("NAS_MOUNT_PATH", "/Volumes/NAS")
    nas_dest = f"{nas_mount}/models/{project}/{exp_id}" if local_ckpt_dir else ""

    if local_ckpt_dir and os.path.exists(local_ckpt_dir) and os.path.exists(nas_mount):
        shutil.copytree(local_ckpt_dir, nas_dest, dirs_exist_ok=True)

    commit_hash = get_git_commit(os.getcwd())
    conn = pymysql.connect(
        host=os.environ["MARIADB_HOST"],
        port=int(os.environ.get("MARIADB_PORT", 3306)),
        user=os.environ["MARIADB_USER"],
        password=os.environ["MARIADB_PASSWORD"],
        database=os.environ.get("MARIADB_DB", "ai_kb"),
        charset="utf8mb4",
        autocommit=True
    )
    with conn.cursor() as cur:
        cur.execute(
            """INSERT INTO research_experiments (exp_id, project_name, task_type, backbone, dataset_name, hyperparams, metrics, checkpoint_nas_path, notes)
               VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)""",
            (exp_id, project, task_type, backbone, dataset, json.dumps(hyperparams), json.dumps(metrics), nas_dest, f"[Commit: {commit_hash}] {notes}")
        )
    conn.close()
    return exp_id
```

---

## Step 6. Hermes Agent 스킬, `config.yaml`, `SOUL.md` 설정

### 6-1. Antigravity 스킬 및 논문 심사 위원회 스킬 생성
```bash
# 내장 antigravity-cli 스킬 활성화
HERMES_ROOT=$(python3 -c "import hermes_agent, os; print(os.path.dirname(os.path.dirname(hermes_agent.__file__)))" 2>/dev/null)
cp -r "$HERMES_ROOT/optional-skills/autonomous-ai-agents/antigravity-cli" ~/.hermes/skills/ 2>/dev/null || true

# 3인 리뷰어 + Chair 피어 리뷰 위원회 스킬 생성
cat << 'EOF' > ~/.hermes/skills/paper-review-committee/SKILL.md
---
name: paper-review-committee
description: 논문 초안에 대해 Reviewer 1, 2, 3 및 Area Chair로 구성된 블라인드 피어 리뷰 위원회를 가동합니다.
---
# Peer Review Committee Workflow
논문 리뷰 요청 시 `delegate_task`로 서로 독립된 3개의 서브에이전트를 생성해 평가한 뒤 네가 Area Chair로서 종합 판정을 내려라.
- **Reviewer 1 (Threat Model & Security Critic)**: 위협 모델의 현실성, 방어 우회 가능성, 보안 가정의 허점 공격.
- **Reviewer 2 (Methodology & Novelty Critic)**: 기존 연구 대비 차별성(Novelty), 수식/알고리즘 전개의 논리적 결함 지적.
- **Reviewer 3 (Experiments & Baselines Skeptic)**: 최신 베이스라인 누락, Ablation study 부족, 데이터셋/지표 편향성 공격.
- **Area Chair (본체)**: 3인의 리뷰를 종합해 Fatal Flaw 추출, 판정(Strong Accept ~ Reject), 리버설 및 수정 Action Plan 작성 후 `save_document`로 DB에 저장.
EOF
```

### 6-2. `~/.hermes/config.yaml` 작성
*(주의: `/Users/본인계정명` 부분을 실제 Mac 계정명으로 변경하세요.)*

```yaml
agent:
  provider: custom
  model: qwen3.8:27b
  base_url: [http://127.0.0.1:11434/v1](http://127.0.0.1:11434/v1)
  reasoning_effort: medium

delegation:
  orchestrator_enabled: true
  max_spawn_depth: 1

auxiliary:
  vision:
    provider: custom
    model: qwen2.5-vl:7b
    base_url: [http://127.0.0.1:11434/v1](http://127.0.0.1:11434/v1)

mcp_servers:
  personal_kb:
    command: "uv"
    args: ["run", "--with", "fastmcp,pymysql,qdrant-client,httpx,python-dotenv", "/Users/본인계정명/.hermes/scripts/kb_mcp_server.py"]
  playwright:
    command: "npx"
    args: ["-y", "@playwright/mcp@latest"]
```

### 6-3. `~/.hermes/SOUL.md` 작성
```markdown
# Core Identity
- 아부, 빈말, 미사여구를 배제하고 객관적·직관적·전문적으로 답변해라. 시키지 않은 부가 작업은 임의로 하지 마라.

# 1. Task Routing & Direct AGY Override (@agy)
- 메시지가 `@agy`로 시작하면 즉시 `terminal(command="agy -p '<지시사항>' --print-timeout 20m", workdir="<타겟_경로>", timeout=1200)`를 실행하고 결과만 요약 보고해라.
- `@agy` 호출 전 프롬프트에 API Key(`sk-`, `hf_`)나 비밀번호가 있으면 마스킹 처리해라.
- `@agy`가 없어도 2개 이상 파일 수정, 리팩토링, 디버깅, 연구용 분류기(`text_cls`, `image_cls`) 학습 스크립트 작성 및 실행은 `agy`에 위임하고, 완료 후 `log_routing_and_task`로 기록해라.
- 직전에 네가 직접 답한 질문을 사용자가 곧바로 `@agy`로 재요청하면 직전 판단의 `outcome_reward=0.0`으로 기록해라.

# 2. Gitea Auto-Versioning & Secret Protection
- 모든 계정 정보와 토큰은 `~/.hermes/.env`에만 존재하므로 코드 내에 시크릿을 절대 하드코딩하지 말고 `os.environ`으로 불러와라.
- 새 프로젝트를 만들거나 `@agy` 작업 및 테스트 통과가 완료되면 즉시 `gitea_sync_project`를 호출해 NAS Gitea에 커밋·푸시해라.

# 3. Knowledge Base, Persona & DPO Feedback Loop
- 논문 검색은 `search_knowledge`, arXiv 최신 논문 수집은 `fetch_and_ingest_arxiv`를 호출해라.
- 사용자가 `@persona` 또는 자신의 철학/원칙을 말하면 `save_persona_thought`로 저장하고, 내 관점의 글쓰기 요청 시 `search_persona`를 먼저 조회해라.
- 사용자가 너의 직전 답변 스타일이나 내용을 교정하면 수정본 제시와 함께 반드시 `log_preference_dpo`를 호출해 기록해라.

# 4. Computer Use & ComfyUI Memory Switching (64GB RAM)
- 웹 조작은 무조건 `playwright` MCP 도구를 최우선으로 쓰고, macOS 네이티브 앱 제어 시에만 `cliclick`을 쓰되 Retina 논리 해상도(1x) 기준으로 좌표를 계산해라.
- 이미지/영상 생성 요청 시 영문 프롬프트로 변환하여 `generate_media_with_memory_switch` 도구를 호출해라.
```

---

## Step 7. Hermes Gateway 및 Open WebUI 구동

```bash
# 1. Hermes 데몬 설치 및 시작 (:8642)
hermes gateway install
hermes gateway start
curl -s [http://127.0.0.1:8642/health](http://127.0.0.1:8642/health)

# 2. Open WebUI 컨테이너 실행 (:3000)
docker rm -f open-webui 2>/dev/null
docker run -d -p 3000:8080 \
  -e OPENAI_API_BASE_URL=[http://host.docker.internal:8642/v1](http://host.docker.internal:8642/v1) \
  -e OPENAI_API_KEY=my-local-secret-key-1234 \
  -e ENABLE_OLLAMA_API=false \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

### Open WebUI 슬래시 명령어(`/`) 4종 등록 (`http://localhost:3000` -> Workspace -> Prompts)
1. **`/agy`**
   ```markdown
   @agy
   - 타겟 경로: ~/workspace/{{project_dir}}
   - 참조 파일: {{target_files}}
   - 목표 작업: {{task_description}}
   - 제약 사항: 작업 완료 후 gitea_sync_project로 커밋할 것. 실험 코드인 경우 ~/workspace/common/exp_logger.py를 호출해 MariaDB와 NAS에 기록할 것.
   ```
2. **`/persona`**
   ```markdown
   @persona
   - 카테고리: {{category}}
   - 질문/상황: {{question}}
   - 나의 생각: {{my_answer}}
   - 핵심 원칙(한 줄 요약): {{core_principle}}
   ```
3. **`/review`**
   ```markdown
   paper-review-committee 스킬을 가동해 줘.
   - 타겟 학회: {{target_venue}}
   - 논문 파일 경로 또는 초안 내용: {{paper_content}}
   ```
4. **`/media`**
   ```markdown
   generate_media_with_memory_switch 도구를 사용해서 아래 내용을 영문 프롬프트로 구체화한 뒤 생성해 줘.
   - 타입: {{media_type}} (image 또는 video)
   - 설명: {{description}}
   ```

---

## Step 8. 세팅 완료 후 Hermes에게 건넬 '초기 온보딩 3단계 대화'

Open WebUI에서 상단 모델을 **`hermes-agent`**로 선택한 뒤, 아래 3개의 프롬프트를 순서대로 입력하여 전체 연결을 검증하고 장기 메모리를 고정합니다.

### [대화 1] 영구 메모리 각인 및 전체 시스템 헬스 체크
```markdown
지금부터 로컬 에이전트 비서 시스템 초기 온보딩 및 전체 연결 상태 점검을 시작한다.
다음 두 가지 작업을 순서대로 수행하고 결과만 간결하게 보고해라.

1. 너의 영구 메모리(USER.md 및 MEMORY.md)에 아래 핵심 컨텍스트를 기록해라:
   - 사용자: 연세대학교 인공지능보안연구실(AIS Lab) 박사과정 김진성
   - 핵심 연구 분야: AI Security & Privacy (Model Inversion, LLM Jailbreaking, PEFT Privacy Leakage, Supply Chain Security)
   - 작업 환경: Mac Studio (64GB Unified Memory, PyTorch MPS) + Synology NAS (MariaDB `ai_kb`, Gitea, `/Volumes/NAS` 마운트)
   - 핵심 원칙: 빈말과 아부를 절대 하지 말고, 라우팅 규칙(가벼운 작업은 직접 수행, 무거운 코딩 및 분류기 학습은 `@agy` 위임 후 Gitea 자동 커밋)과 DPO/Persona/Laya 로깅 규칙을 엄격히 준수할 것.

2. `check_system_health` MCP 도구를 호출해서 MariaDB, Qdrant, Gitea, ComfyUI, NAS 마운트 연결 상태를 점검하고 결과를 표로 보고해라.
```

### [대화 2] 페르소나 DB + AGY 패스스루 + Gitea 자동 커밋 통합 검증
```markdown
이제 DB 파이프라인과 AGY 연동, Gitea 버전 관리가 정상 작동하는지 실전 테스트를 진행한다.

1. `save_persona_thought` 도구를 호출해서 나의 첫 번째 연구 원칙을 저장해라:
   - category: "research_view"
   - question: "AI 보안 연구에서 가장 중요하게 생각하는 기준은 무엇인가?"
   - my_answer: "단순한 이론적 가능성에 그치지 않고, 실제 배포 환경과 현실적인 Threat Model 하에서 재현 가능한 공격/방어 실효성을 코드로 증명해야 한다."
   - core_principle: "현실적 위협 모델 기반의 실증적 보안 연구"

2. `@agy ~/workspace/test_init 파이썬으로 PyTorch MPS 가속 가능 여부(torch.backends.mps.is_available())를 출력하는 짧은 스크립트 mps_check.py를 작성하고 실행 결과를 확인해 줘.`
   완료 후 `gitea_sync_project`로 NAS Gitea에 Private 레포지토리로 푸시하고, `log_routing_and_task`로 라우팅 로그까지 남겨라.
```

### [대화 3] 일일 자동화 스케줄러(Cron) 등록
```markdown
모든 파이프라인이 확인되었다. 마지막으로 아래 두 개의 정기 스케줄(Cron)을 등록해라:

1. 매일 아침 08:30: `fetch_and_ingest_arxiv`로 "Model Inversion", "LLM Jailbreak", "PEFT Privacy" 관련 최신 논문을 수집하고 핵심 요약 브리핑 남기기.
2. 매일 밤 22:30: `search_persona`로 기존 기록을 확인한 뒤, 중복되지 않는 연구 철학이나 의사결정 딜레마 질문을 나에게 하루 하나씩 던지기.
```
