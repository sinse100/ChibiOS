// Jenkinsfile (v10 보완본: AST 생성/감지/Include 경로를 설정파일(ast_paths.json)로 분리)
pipeline {
  agent any

  options {
    buildDiscarder(logRotator(numToKeepStr: '30'))
  }

  environment {
    AST_STORE = "/var/lib/jenkins/ast/chibios-os-rt"

    CLANG = "clang"
    PY = "python3"

    BUILD_CMD = "make -C testrt"

    // baseline에서 build-cmd 기본 비활성
    BUILD_CMD_BASELINE = ""

    // ==========================================================
    // [추가] AST 참고 경로 설정파일 경로
    // - repo에서 이 파일만 수정/커밋하면, Jenkinsfile 수정 없이 경로 튜닝 가능
    // ==========================================================
    AST_PATHS_CONFIG = "tools/ast_ci/ast_paths.json"
  }

  triggers {
    pollSCM('H/5 * * * *')
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        sh 'git rev-parse --short HEAD'
      }
    }

    stage('sudo 권한 사전 점검') {
      steps {
        sh '''
          set -eux
          if sudo -n true 2>/dev/null; then
            echo "[OK] jenkins 계정이 비밀번호 없이 sudo 사용 가능"
          else
            echo "[ERROR] jenkins 계정이 NOPASSWD sudo 설정이 되어있지 않습니다."
            echo "예시(서버에서 실행):"
            echo "  sudo visudo -f /etc/sudoers.d/jenkins-apt"
            echo "  Defaults:jenkins !requiretty"
            echo "  jenkins ALL=(root) NOPASSWD: /usr/bin/apt-get, /usr/bin/apt, /usr/bin/dpkg, /usr/bin/true"
            exit 1
          fi
        '''
      }
    }

    stage('AST 실행 여부 판단') {
      steps {
        script {
          sh "mkdir -p '${env.AST_STORE}/baseline'"

          def baselineExists = fileExists("${env.AST_STORE}/baseline/summary.json")

          // ==========================================================
          // [주의]
          // - 이 단계는 "AST를 돌릴지 말지" 빠르게 결정하는 용도라,
          //   설정파일을 아직 만들기 전(다음 stage에서 생성)일 수 있음.
          // - 따라서 여기서는 기존처럼 하드코딩된 관심 경로로만 변경 감지함.
          //   (정교하게 하려면 Groovy로 JSON 읽어서 regex를 만들 수도 있음)
          // ==========================================================
          sh "git fetch origin main:refs/remotes/origin/main || true"
          def changed = sh(
            script: """
              git diff --name-only origin/main..HEAD | \\
              grep -E '^(os/rt/|os/common/ports/ARMv6-M-RP2/|os/rt/templates/|os/oslib/src/|os/hal/src/)' || true
            """,
            returnStdout: true
          ).trim()

          if (!baselineExists) {
            echo "Baseline AST가 존재하지 않음 → 최초 baseline 생성"
            env.DO_AST = "1"
            env.AST_MODE = "baseline"
          } else if (changed == "") {
            echo "관심 경로 변경 없음 → AST 단계 스킵"
            env.DO_AST = "0"
          } else {
            echo "관심 경로 변경 감지 → incremental AST 수행"
            env.DO_AST = "1"
            env.AST_MODE = "incremental"
          }
        }
      }
    }

    stage('의존성 설치') {
      when { expression { return env.DO_AST == "1" } }
      steps {
        sh '''
          set -eux
          export DEBIAN_FRONTEND=noninteractive

          if command -v clang >/dev/null \
             && command -v bear  >/dev/null \
             && command -v jq    >/dev/null \
             && command -v python3 >/dev/null; then
            echo "[SKIP] 의존성 이미 설치됨 (clang/bear/jq/python3)"
            exit 0
          fi

          sudo -n apt-get update
          sudo -n apt-get install -y clang bear jq python3
        '''
      }
    }

    stage('AST 분석 스크립트 생성') {
      when { expression { return env.DO_AST == "1" } }
      steps {
        sh '''
          set -eux
          mkdir -p tools/ast_ci

          # ==========================================================
          # [핵심] AST 참고 경로 설정파일 생성(없을 때만)
          # - 이미 repo에 파일이 있으면 그대로 사용(덮어쓰지 않음)
          # - 사용자는 이 파일만 수정해서 경로를 자유롭게 튜닝 가능
          # ==========================================================
          if [ ! -f "${AST_PATHS_CONFIG}" ]; then
            cat > "${AST_PATHS_CONFIG}" << 'JSON'
{
  "tu_roots": [
    "os/rt/src"
  ],
  "watched_prefixes": [
    "os/rt/",
    "os/common/ports/ARMv6-M-RP2/",
    "os/rt/templates/",
    "os/oslib/src/",
    "os/hal/src/"
  ],
  "watched_header_prefixes": [
    "os/rt/include/",
    "os/common/ports/ARMv6-M-RP2/",
    "os/rt/templates/",
    "os/oslib/include/",
    "os/hal/include/"
  ],
  "fallback_includes": [
    "-Ios/rt/include",
    "-Ios/rt/templates",
    "-Ios/common/ports/ARMv6-M-RP2",
    "-Ios/oslib/include",
    "-Ios/oslib/src",
    "-Ios/hal/include",
    "-Ios/hal/src"
  ]
}
JSON
            echo "[INFO] ${AST_PATHS_CONFIG} 가 없어서 기본값으로 생성했습니다. 필요하면 이 파일만 수정하세요."
          else
            echo "[INFO] ${AST_PATHS_CONFIG} 가 존재하므로 해당 설정을 사용합니다."
          fi

          cat > tools/ast_ci/ast_build_and_diff.py << 'PY'
#!/usr/bin/env python3
import argparse, json, subprocess, sys, hashlib
from pathlib import Path
from typing import Any, Dict, List, Set, Tuple

# ==========================================================
# 유틸 함수들
# ==========================================================
def run(cmd: List[str], check: bool = True) -> str:
    p = subprocess.run(cmd, text=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    if check and p.returncode != 0:
        sys.stderr.write(p.stderr)
        raise SystemExit(p.returncode)
    return p.stdout

def run_shell(cmd: str) -> str:
    p = subprocess.run(["bash","-lc",cmd], text=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    return (p.stdout or "") + (p.stderr or "")

def sha1_text(s: str) -> str:
    return hashlib.sha1(s.encode("utf-8", errors="ignore")).hexdigest()

def git_rev(ref: str) -> str:
    return run(["git","rev-parse",ref]).strip()

def ensure_parent(p: Path) -> None:
    p.parent.mkdir(parents=True, exist_ok=True)

def list_changed_files(base: str, head: str) -> List[str]:
    out = run(["git","diff","--name-only",f"{base}..{head}"])
    return [x for x in out.splitlines() if x]

# ==========================================================
# 설정파일 로드
# ==========================================================
def load_config(path: str) -> Dict[str, Any]:
    p = Path(path)
    if not p.exists():
        return {}
    try:
        return json.loads(p.read_text(encoding="utf-8", errors="ignore"))
    except Exception:
        # 설정파일이 깨져 있어도 파이프라인이 즉시 죽지 않게 방어
        return {}

# ==========================================================
# compile_commands.json 생성/로드 (가능하면 사용)
# ==========================================================
def build_compile_db(build_cmd: str) -> bool:
    if not build_cmd.strip():
        return False
    _ = run_shell(f"bear -- {build_cmd}")
    return Path("compile_commands.json").exists()

def read_compile_db() -> Dict[str, List[str]]:
    db = Path("compile_commands.json")
    if not db.exists():
        return {}
    data = json.loads(db.read_text(encoding="utf-8", errors="ignore"))
    mapping: Dict[str, List[str]] = {}
    for e in data:
        fp = e.get("file")
        if not fp:
            continue
        absf = str(Path(fp).resolve())
        args = e.get("arguments") or e.get("command","").split()
        if args and (args[0].endswith("clang") or args[0].endswith("gcc") or args[0].endswith("cc")):
            args = args[1:]
        mapping[absf] = args
    return mapping

def filter_args(args: List[str]) -> List[str]:
    skip = {"-c","-MMD","-MP"}
    out: List[str] = []
    i=0
    while i < len(args):
        a=args[i]
        if a in skip:
            i+=1
        elif a in ("-o","-MF","-MT","-MQ"):
            i+=2
        elif a.endswith(".c"):
            i+=1
        else:
            out.append(a); i+=1
    return out

# ==========================================================
# clang AST 덤프
# ==========================================================
def clang_ast(clang: str, src: str, flags: List[str]) -> Dict[str, Any]:
    cmd = [clang,"-Xclang","-ast-dump=json","-fsyntax-only",src] + flags
    return json.loads(run(cmd))

def normalise(node: Any) -> str:
    if not isinstance(node, dict):
        return ""
    s = f"{node.get('kind','')}|{node.get('name','')}"
    for c in node.get("inner", []) or []:
        s += normalise(c)
    return s

def index_functions(ast: Dict[str, Any]) -> Dict[str, Dict[str, Any]]:
    out: Dict[str, Dict[str, Any]] = {}
    def walk(n: Any):
        if isinstance(n, dict):
            if n.get("kind")=="FunctionDecl" and n.get("name"):
                out[n["name"]] = n
            for c in n.get("inner", []) or []:
                walk(c)
    walk(ast)
    return out

def diff_functions(a: Dict[str, Any], b: Dict[str, Any]) -> Dict[str, List[str]]:
    fa = index_functions(a)
    fb = index_functions(b)
    return {
        "only_before": sorted(set(fa) - set(fb)),
        "only_after":  sorted(set(fb) - set(fa)),
        "changed": sorted(
            f for f in (fa.keys() & fb.keys())
            if sha1_text(normalise(fa[f])) != sha1_text(normalise(fb[f]))
        )
    }

# ==========================================================
# TU 선택 로직 (설정파일 기반)
# ==========================================================
def list_all_tus(root: Path, tu_roots: List[str]) -> List[Path]:
    tus: List[Path] = []
    for r in tu_roots:
        tus.extend((root / r).rglob("*.c"))
    return sorted(set(tus))

def select_incremental_tus(
    changed: List[str],
    root: Path,
    tu_roots: List[str],
    watched_prefixes: Tuple[str, ...],
    watched_header_prefixes: Tuple[str, ...],
) -> List[Path]:
    tus: Set[Path] = set()

    # 1) TU 루트들 아래에서 변경된 .c 는 해당 TU만 대상으로
    for p in changed:
        for tu_root in tu_roots:
            prefix = tu_root.rstrip("/") + "/"
            if p.startswith(prefix) and p.endswith(".c"):
                tus.add(root / p)

    # 2) 헤더 변경은 보수적으로 전체 TU 재생성
    header_changed = any(p.startswith(watched_header_prefixes) and p.endswith(".h") for p in changed)

    # 3) 감시 경로(외부 의존 포함)에서 변경이 발생하면 전체 TU 재생성(보수적)
    dep_changed = any(p.startswith(watched_prefixes) for p in changed)

    if header_changed or dep_changed:
        tus |= set(list_all_tus(root, tu_roots))

    return sorted(tus)

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--outdir", required=True)
    ap.add_argument("--base", required=True)
    ap.add_argument("--head", required=True)
    ap.add_argument("--mode", choices=["baseline","incremental"], required=True)
    ap.add_argument("--clang", default="clang")
    ap.add_argument("--build-cmd", default="")
    ap.add_argument("--config", default="tools/ast_ci/ast_paths.json")
    args = ap.parse_args()

    cfg = load_config(args.config)

    tu_roots = cfg.get("tu_roots") or ["os/rt/src"]
    watched_prefixes = tuple(cfg.get("watched_prefixes") or ["os/rt/"])
    watched_header_prefixes = tuple(cfg.get("watched_header_prefixes") or ["os/rt/include/"])
    fallback_includes = cfg.get("fallback_includes") or ["-Ios/rt/include"]

    root = Path(".").resolve()
    out = Path(args.outdir); out.mkdir(parents=True, exist_ok=True)

    base_commit = git_rev(args.base)
    head_commit = git_rev(args.head)

    compile_db: Dict[str, List[str]] = {}
    if args.build_cmd and build_compile_db(args.build_cmd):
        compile_db = read_compile_db()

    if args.mode == "baseline":
        tus = list_all_tus(root, tu_roots)
        changed_files = ["(baseline 초기 생성)"]
    else:
        changed_files = list_changed_files(args.base, args.head)
        tus = select_incremental_tus(
            changed_files, root,
            tu_roots=tu_roots,
            watched_prefixes=watched_prefixes,
            watched_header_prefixes=watched_header_prefixes,
        )

    results = []
    for tu in tus:
        rel = tu.relative_to(root)

        before_c = out / f"{rel}.before.c"
        after_c  = out / f"{rel}.after.c"

        before_ast = out / f"{rel}.before.ast.json"
        after_ast  = out / f"{rel}.after.ast.json"

        diffp  = out / f"{rel}.diff.json"

        ensure_parent(before_c); ensure_parent(after_c)
        ensure_parent(before_ast); ensure_parent(after_ast)
        ensure_parent(diffp)

        before_c.write_text(run(["git","show",f"{base_commit}:{rel}"], check=False), encoding="utf-8", errors="ignore")
        after_c.write_text(run(["git","show",f"{head_commit}:{rel}"], check=False), encoding="utf-8", errors="ignore")

        flags = compile_db.get(str(tu.resolve()), fallback_includes)
        flags = filter_args(flags)

        ast_b = clang_ast(args.clang, str(before_c), flags)
        ast_a = clang_ast(args.clang, str(after_c),  flags)

        before_ast.write_text(json.dumps(ast_b, indent=2), encoding="utf-8")
        after_ast.write_text(json.dumps(ast_a, indent=2), encoding="utf-8")

        diff = diff_functions(ast_b, ast_a)
        diffp.write_text(json.dumps(diff, indent=2), encoding="utf-8")

        results.append({"tu": str(rel), **diff})

    summary = {
        "mode": args.mode,
        "base_commit": base_commit,
        "head_commit": head_commit,
        "config": args.config,
        "tu_roots": tu_roots,
        "watched_prefixes": list(watched_prefixes),
        "watched_header_prefixes": list(watched_header_prefixes),
        "fallback_includes": fallback_includes,
        "changed_files": changed_files,
        "results": results
    }
    (out/"summary.json").write_text(json.dumps(summary, indent=2), encoding="utf-8")

if __name__ == "__main__":
    main()
PY

          chmod +x tools/ast_ci/ast_build_and_diff.py
        '''
      }
    }

    stage('AST 생성 및 diff (롤백 포함)') {
      when { expression { return env.DO_AST == "1" } }
      options { timeout(time: 25, unit: 'MINUTES') }

      steps {
        sh '''
          bash -lc '
            set -euo pipefail

            COMMIT=$(git rev-parse --short HEAD)
            AST_ROOT="${AST_STORE}"
            BASELINE_DIR="${AST_ROOT}/baseline"

            mkdir -p "${AST_ROOT}"
            mkdir -p "ast_out"

            TS=$(date +%Y%m%d_%H%M%S)
            TMP_BASE="${AST_ROOT}/.tmp_baseline_${COMMIT}_${TS}"
            TMP_COMMIT="${AST_ROOT}/.tmp_commit_${COMMIT}_${TS}"

            rollback() {
              echo "[ROLLBACK] 빌드 실패 감지 → 임시 결과물 정리 및(필요 시) baseline 복구"
              rm -rf "${TMP_BASE}" "${TMP_COMMIT}" || true

              if [ -n "${BASELINE_BACKUP:-}" ] && [ -d "${BASELINE_BACKUP}" ]; then
                echo "[ROLLBACK] baseline 복구 수행: ${BASELINE_BACKUP} → ${BASELINE_DIR}"
                rm -rf "${BASELINE_DIR}" || true
                mv "${BASELINE_BACKUP}" "${BASELINE_DIR}" || true
              fi
            }
            trap rollback ERR

            if [ "${AST_MODE}" = "baseline" ]; then
              OUT="ast_out/baseline_${COMMIT}"
              mkdir -p "$OUT"

              echo "[BASELINE] 전체 TU AST 생성 시작"

              BASELINE_BUILD_CMD="${BUILD_CMD_BASELINE:-}"

              ${PY} tools/ast_ci/ast_build_and_diff.py \
                --outdir "$OUT" \
                --base "HEAD" \
                --head "HEAD" \
                --mode "baseline" \
                --build-cmd "${BASELINE_BUILD_CMD}" \
                --config "${AST_PATHS_CONFIG}"

              mkdir -p "${TMP_BASE}"
              rsync -a --delete "$OUT/" "${TMP_BASE}/"

              if [ -d "${BASELINE_DIR}" ] && [ "$(ls -A "${BASELINE_DIR}" 2>/dev/null || true)" != "" ]; then
                BASELINE_BACKUP="${AST_ROOT}/.backup_baseline_${TS}"
                echo "[BASELINE] 기존 baseline 백업: ${BASELINE_DIR} → ${BASELINE_BACKUP}"
                mv "${BASELINE_DIR}" "${BASELINE_BACKUP}"
              fi

              echo "[BASELINE] baseline 원자적 교체: ${TMP_BASE} → ${BASELINE_DIR}"
              rm -rf "${BASELINE_DIR}" || true
              mv "${TMP_BASE}" "${BASELINE_DIR}"

              if [ -n "${BASELINE_BACKUP:-}" ] && [ -d "${BASELINE_BACKUP}" ]; then
                echo "[BASELINE] 교체 성공 → 이전 baseline 백업 정리: ${BASELINE_BACKUP}"
                rm -rf "${BASELINE_BACKUP}"
              fi

              echo "[BASELINE] 완료: ${BASELINE_DIR}/summary.json 생성 여부 확인"
              ls -la "${BASELINE_DIR}" || true

            else
              BASE_COMMIT=""
              if [ -f "${BASELINE_DIR}/summary.json" ]; then
                BASE_COMMIT=$(jq -r ".head_commit // .headCommit // empty" "${BASELINE_DIR}/summary.json" || true)
              fi
              if [ -z "${BASE_COMMIT}" ] || [ "${BASE_COMMIT}" = "null" ]; then
                echo "[INCREMENTAL] baseline 기준 커밋을 못 읽음 → origin/main 사용"
                BASE_REF="origin/main"
              else
                BASE_REF="${BASE_COMMIT}"
                echo "[INCREMENTAL] baseline 기준 커밋: ${BASE_REF}"
              fi

              OUT="ast_out/${COMMIT}"
              mkdir -p "$OUT"

              echo "[INCREMENTAL] 변경 TU AST 생성 + diff 시작"
              ${PY} tools/ast_ci/ast_build_and_diff.py \
                --outdir "$OUT" \
                --base "${BASE_REF}" \
                --head "HEAD" \
                --mode "incremental" \
                --build-cmd "${BUILD_CMD}" \
                --config "${AST_PATHS_CONFIG}"

              mkdir -p "${TMP_COMMIT}"
              rsync -a --delete "$OUT/" "${TMP_COMMIT}/"

              FINAL_COMMIT_DIR="${AST_ROOT}/${COMMIT}"

              if [ -d "${FINAL_COMMIT_DIR}" ] && [ "$(ls -A "${FINAL_COMMIT_DIR}" 2>/dev/null || true)" != "" ]; then
                COMMIT_BACKUP="${AST_ROOT}/.backup_commit_${COMMIT}_${TS}"
                echo "[INCREMENTAL] 기존 커밋 결과 백업: ${FINAL_COMMIT_DIR} → ${COMMIT_BACKUP}"
                mv "${FINAL_COMMIT_DIR}" "${COMMIT_BACKUP}"
              fi

              echo "[INCREMENTAL] 커밋 결과 원자적 교체: ${TMP_COMMIT} → ${FINAL_COMMIT_DIR}"
              rm -rf "${FINAL_COMMIT_DIR}" || true
              mv "${TMP_COMMIT}" "${FINAL_COMMIT_DIR}"

              if [ -n "${COMMIT_BACKUP:-}" ] && [ -d "${COMMIT_BACKUP}" ]; then
                echo "[INCREMENTAL] 교체 성공 → 이전 커밋 결과 백업 정리: ${COMMIT_BACKUP}"
                rm -rf "${COMMIT_BACKUP}"
              fi

              echo "[INCREMENTAL] 완료: ${FINAL_COMMIT_DIR}/summary.json 확인"
              ls -la "${FINAL_COMMIT_DIR}" || true
            fi

            trap - ERR
          '
        '''
      }
    }
  }

  post {
    always {
      echo "빌드 결과: ${currentBuild.currentResult}"
    }
  }
}
