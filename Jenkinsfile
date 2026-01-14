// Jenkinsfile (v02 최적화 반영본)
pipeline {
  agent any

  options {
    // 플러그인 의존 옵션은 환경에 따라 파싱 실패할 수 있어 기본 비활성 권장
    // timestamps()
    // ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '30'))
  }

  environment {
    // AST 결과를 서버에 영구 저장할 경로 (Jenkins 실행 계정이 쓰기 권한을 가져야 함)
    AST_STORE = "/var/lib/jenkins/ast/chibios-os-rt"

    // 사용 도구
    CLANG = "clang"
    PY = "python3"

    // ⚠️ 실제 ChibiOS RT 빌드에 맞게 조정 필요 (compile_commands.json 생성 목적)
    // (incremental은 필요할 때만 켜는 게 좋음)
    BUILD_CMD = "make -C testrt"

    // ==========================================================
    // [baseline에서 build-cmd 기본 비활성]
    // - baseline은 처음 1회 전체 TU를 다 도는 구간이라 시간이 길어짐
    // - bear + make가 더 오래 걸릴 수 있어서 기본은 ""(비활성)
    // - 필요하면 아래 값을 BUILD_CMD와 같게 바꿔서 켜면 됨
    //   예: BUILD_CMD_BASELINE = "make -C testrt"
    // ==========================================================
    BUILD_CMD_BASELINE = ""
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
          # ==========================================================
          # [B 방식 전제]
          # Jenkinsfile에서 apt 설치를 위해 sudo를 사용하므로,
          # jenkins 유저가 비밀번호 없이 sudo 실행 가능해야 함(NOPASSWD).
          # -n 옵션으로 "비밀번호 요구" 시 즉시 실패하게 함.
          # ==========================================================
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
          // baseline 디렉터리 보장
          sh "mkdir -p '${env.AST_STORE}/baseline'"

          // ==========================================================
          // [BASELINE 존재 여부 판단]
          // - 최초 실행이거나 ${AST_STORE}/baseline/summary.json 이 없으면
          //   → baseline AST를 처음부터 생성해야 함
          // ==========================================================
          def baselineExists = fileExists("${env.AST_STORE}/baseline/summary.json")

          // ==========================================================
          // [변경 사항 감지]
          // - origin/master..HEAD 범위에서 os/rt/** 변경이 있는지 확인
          // ==========================================================
          sh "git fetch origin master:refs/remotes/origin/master || true"
          def changed = sh(
            script: "git diff --name-only origin/master..HEAD | grep '^os/rt/' || true",
            returnStdout: true
          ).trim()

          if (!baselineExists) {
            echo "Baseline AST가 존재하지 않음 → 최초 baseline 생성"
            env.DO_AST = "1"
            env.AST_MODE = "baseline"
          } else if (changed == "") {
            echo "os/rt 변경 없음 → AST 단계 스킵"
            env.DO_AST = "0"
          } else {
            echo "os/rt 변경 감지 → incremental AST 수행"
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

          # ==========================================================
          # [시간 최적화]
          # - baseline에서 10분+ 걸리는 주범이 apt-get update/install인 경우가 많음
          # - 이미 도구가 설치돼 있으면 apt 단계를 스킵해서 시간을 크게 줄임
          # ==========================================================
          if command -v clang >/dev/null \
             && command -v bear  >/dev/null \
             && command -v jq    >/dev/null \
             && command -v python3 >/dev/null; then
            echo "[SKIP] 의존성 이미 설치됨 (clang/bear/jq/python3)"
            exit 0
          fi

          # ==========================================================
          # [B 방식 반영]
          # - sudo -n : 비밀번호 입력 요구 시 즉시 실패 (대화형 입력 금지)
          # ==========================================================
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

          cat > tools/ast_ci/ast_build_and_diff.py << 'PY'
#!/usr/bin/env python3
import argparse, json, subprocess, sys, hashlib
from pathlib import Path
from typing import Any, Dict, List, Set

def run(cmd: List[str], check: bool = True) -> str:
    p = subprocess.run(cmd, text=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    if check and p.returncode != 0:
        sys.stderr.write(p.stderr)
        raise SystemExit(p.returncode)
    return p.stdout

def run_shell(cmd: str) -> str:
    p = subprocess.run(["bash","-lc",cmd], text=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    # bear/make 실패 시에도 stderr를 보는 게 유용하므로 출력 합쳐서 반환
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

def build_compile_db(build_cmd: str) -> bool:
    # ==========================================================
    # [compile_commands.json 생성]
    # - build_cmd가 비어있으면 아예 수행하지 않음(시간 절약)
    # - 수행하더라도 compile_commands.json이 없으면 fallback include로 진행
    # ==========================================================
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
        # 컴파일러 제거(첫 토큰이 컴파일러인 경우가 많음)
        if args and (args[0].endswith("clang") or args[0].endswith("gcc") or args[0].endswith("cc")):
            args = args[1:]
        mapping[absf] = args
    return mapping

def filter_args(args: List[str]) -> List[str]:
    # 빌드 플래그 중 clang AST에 불필요/위험한 항목 제거
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

def list_all_rt_tus(root: Path) -> List[Path]:
    return sorted((root/"os/rt/src").rglob("*.c"))

def select_incremental_tus(changed: List[str], root: Path) -> List[Path]:
    tus: Set[Path] = set()
    header_changed = any(p.startswith("os/rt/include/") and p.endswith(".h") for p in changed)
    for p in changed:
        if p.startswith("os/rt/src/") and p.endswith(".c"):
            tus.add(root/p)
    # 헤더 변경은 보수적으로 전체 TU 재생성
    if header_changed:
        tus |= set(list_all_rt_tus(root))
    return sorted(tus)

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--outdir", required=True)
    ap.add_argument("--base", required=True)
    ap.add_argument("--head", required=True)
    ap.add_argument("--mode", choices=["baseline","incremental"], required=True)
    ap.add_argument("--clang", default="clang")
    ap.add_argument("--build-cmd", default="")
    ap.add_argument("--fallback-includes", nargs="*", default=["-Ios/rt/include"])
    args = ap.parse_args()

    root = Path(".").resolve()
    out = Path(args.outdir); out.mkdir(parents=True, exist_ok=True)

    base_commit = git_rev(args.base)
    head_commit = git_rev(args.head)

    compile_db: Dict[str, List[str]] = {}
    if args.build_cmd and build_compile_db(args.build_cmd):
        compile_db = read_compile_db()

    # ==================================================
    # [MODE 분기]
    # ==================================================
    if args.mode == "baseline":
        # 최초 실행: os/rt/src 전체 AST 생성
        tus = list_all_rt_tus(root)
        changed_files = ["(baseline 초기 생성)"]
    else:
        # 이후 실행: 변경 파일 기반 TU 선택
        changed_files = list_changed_files(args.base, args.head)
        tus = select_incremental_tus(changed_files, root)

    results = []
    for tu in tus:
        rel = tu.relative_to(root)

        before = out / f"{rel}.before.c"
        after  = out / f"{rel}.after.c"
        diffp  = out / f"{rel}.diff.json"

        ensure_parent(before); ensure_parent(after); ensure_parent(diffp)

        before.write_text(run(["git","show",f"{base_commit}:{rel}"], check=False), encoding="utf-8", errors="ignore")
        after.write_text(run(["git","show",f"{head_commit}:{rel}"], check=False), encoding="utf-8", errors="ignore")

        flags = compile_db.get(str(tu.resolve()), args.fallback_includes)
        flags = filter_args(flags)

        ast_b = clang_ast(args.clang, str(before), flags)
        ast_a = clang_ast(args.clang, str(after),  flags)

        diff = diff_functions(ast_b, ast_a)
        diffp.write_text(json.dumps(diff, indent=2), encoding="utf-8")

        results.append({"tu": str(rel), **diff})

    summary = {
        "mode": args.mode,
        "base_commit": base_commit,
        "head_commit": head_commit,
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

      // ==========================================================
      // [무한 대기 방지]
      // - baseline/build/bear/clang 중 어디서 멈추든 일정 시간 지나면 종료
      // ==========================================================
      options { timeout(time: 25, unit: 'MINUTES') }

      steps {
        sh '''
          set -euo pipefail

          COMMIT=$(git rev-parse --short HEAD)
          AST_ROOT="${AST_STORE}"
          BASELINE_DIR="${AST_ROOT}/baseline"

          mkdir -p "${AST_ROOT}"
          mkdir -p "ast_out"

          # ==========================================================
          # [롤백/원자적 갱신 전략]
          #
          # 1) 최종 저장소(AST_STORE)에는 "임시 디렉터리"로 먼저 저장한다.
          # 2) AST 생성 + rsync가 모두 성공한 뒤에만 최종 경로로 mv(원자적 교체) 한다.
          # 3) 실패하면:
          #    - 임시 디렉터리 삭제
          #    - baseline을 건드리던 중이면 이전 baseline 복구
          # ==========================================================

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
            # ======================================================
            # [최초 실행 / baseline 생성]
            # - os/rt/src 전체 AST 생성
            # - baseline은 성공했을 때만 원자적으로 교체
            # ======================================================

            OUT="ast_out/baseline_${COMMIT}"
            mkdir -p "$OUT"

            echo "[BASELINE] 전체 TU AST 생성 시작"

            # ======================================================
            # [baseline build-cmd 선택]
            # - 기본은 BUILD_CMD_BASELINE="" 이므로 bear/make를 수행하지 않음(시간 절약)
            # - 필요하면 Jenkinsfile 상단에서 BUILD_CMD_BASELINE 값을 조정해서 켜면 됨
            # ======================================================
            BASELINE_BUILD_CMD="${BUILD_CMD_BASELINE}"

            ${PY} tools/ast_ci/ast_build_and_diff.py \
              --outdir "$OUT" \
              --base "HEAD" \
              --head "HEAD" \
              --mode "baseline" \
              --build-cmd "${BASELINE_BUILD_CMD}"

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
            # ======================================================
            # [이후 실행 / incremental + diff]
            # - baseline 커밋을 기준으로 diff 수행
            # - commit 결과 저장도 임시 저장 후 성공 시에만 이동
            # ======================================================

            BASE_COMMIT=""
            if [ -f "${BASELINE_DIR}/summary.json" ]; then
              BASE_COMMIT=$(jq -r '.head_commit // .headCommit // empty' "${BASELINE_DIR}/summary.json" || true)
            fi
            if [ -z "${BASE_COMMIT}" ] || [ "${BASE_COMMIT}" = "null" ]; then
              echo "[INCREMENTAL] baseline 기준 커밋을 못 읽음 → origin/master 사용"
              BASE_REF="origin/master"
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
              --build-cmd "${BUILD_CMD}"

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
