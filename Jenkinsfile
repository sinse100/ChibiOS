// Jenkinsfile
pipeline {
  agent any

  options {
    // (플러그인 의존 옵션은 환경에 따라 파싱 실패할 수 있어 제거/비활성 권장)
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

    // ⚠️ 실제 ChibiOS RT 빌드에 맞는 명령으로 조정 필요
    // 목적: bear로 compile_commands.json 생성 시도
    BUILD_CMD = "make -C testrt"

    // (사용 안 해도 유지)
    MERGE_DEPTH = "2"
  }

  triggers {
    // GitHub Webhook / Multibranch 사용 권장
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
          # [중요]
          # Jenkinsfile에서 apt 설치를 위해 sudo를 사용하므로,
          # jenkins 유저가 비밀번호 없이 sudo 실행 가능해야 함(NOPASSWD).
          #
          # 아래 명령은 "비밀번호 입력 요구"가 발생하면 즉시 실패(-n).
          # ==========================================================
          if sudo -n true 2>/dev/null; then
            echo "[OK] jenkins 계정이 비밀번호 없이 sudo 사용 가능"
          else
            echo "[ERROR] jenkins 계정이 NOPASSWD sudo 설정이 되어있지 않습니다."
            echo "다음 중 하나로 해결하세요:"
            echo "  (방법 B) sudoers에 jenkins의 apt 권한을 NOPASSWD로 추가"
            echo "    sudo visudo -f /etc/sudoers.d/jenkins-apt"
            echo "    내용 예시:"
            echo "      Defaults:jenkins !requiretty"
            echo "      jenkins ALL=(root) NOPASSWD: /usr/bin/apt-get, /usr/bin/apt, /usr/bin/dpkg"
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

          // baseline 존재 여부
          def baselineExists = fileExists("${env.AST_STORE}/baseline/summary.json")

          // origin/master..HEAD 범위에서 os/rt/** 변경 여부 확인
          sh "git fetch origin master:refs/remotes/origin/master || true"
          def changed = sh(
            script: "git diff --name-only origin/master..HEAD | grep '^os/rt/' || true",
            returnStdout: true
          ).trim()

          if (!baselineExists) {
            // 최초 실행: baseline 생성
            echo "Baseline AST가 존재하지 않음 → 최초 baseline 생성"
            env.DO_AST = "1"
            env.AST_MODE = "baseline"
          } else if (changed == "") {
            // 변경 없음: 스킵
            echo "os/rt 변경 없음 → AST 단계 스킵"
            env.DO_AST = "0"
          } else {
            // 변경 있음: incremental
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
          # [B 방식 반영]
          # - sudo -n : 비밀번호 입력을 요구하면 즉시 실패 (대화형 입력 금지)
          # - NOPASSWD sudo 설정이 되어있어야 통과
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
import argparse
import json
import subprocess
import sys
import hashlib
from pathlib import Path
from typing import Any, Dict, List, Set

def run(cmd: List[str], check: bool = True) -> str:
    p = subprocess.run(cmd, text=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    if check and p.returncode != 0:
        sys.stderr.write(p.stderr)
        raise SystemExit(p.returncode)
    return p.stdout

def run_shell(cmd: str) -> str:
    p = subprocess.run(["bash", "-lc", cmd], text=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    return p.stdout

def sha1_text(s: str) -> str:
    return hashlib.sha1(s.encode("utf-8", errors="ignore")).hexdigest()

def git_rev(ref: str) -> str:
    return run(["git", "rev-parse", ref]).strip()

def ensure_parent_dir(p: Path) -> None:
    p.parent.mkdir(parents=True, exist_ok=True)

def list_changed_files(base: str, head: str) -> List[str]:
    out = run(["git", "diff", "--name-only", f"{base}..{head}"])
    return [x for x in out.splitlines() if x]

def build_compile_db(build_cmd: str) -> bool:
    run_shell(f"bear -- {build_cmd}")
    return Path("compile_commands.json").exists()

def read_compile_db() -> Dict[str, List[str]]:
    db = Path("compile_commands.json")
    if not db.exists():
        return {}

    data = json.loads(db.read_text(encoding="utf-8", errors="ignore"))
    mapping: Dict[str, List[str]] = {}

    for e in data:
        file_path = e.get("file")
        if not file_path:
            continue
        abs_file = str(Path(file_path).resolve())

        args = e.get("arguments")
        if not args:
            cmd = e.get("command", "")
            args = cmd.split()

        if args and (args[0].endswith("clang") or args[0].endswith("gcc") or args[0].endswith("cc")):
            args = args[1:]

        mapping[abs_file] = args
    return mapping

def filter_args(args: List[str]) -> List[str]:
    skip = {"-c", "-MMD", "-MP"}
    out: List[str] = []
    i = 0
    while i < len(args):
        if args[i] in skip:
            i += 1
        elif args[i] in ("-o", "-MF", "-MT", "-MQ"):
            i += 2
        elif args[i].endswith(".c"):
            i += 1
        else:
            out.append(args[i])
            i += 1
    return out

def clang_ast(clang: str, src: str, flags: List[str]) -> Dict[str, Any]:
    cmd = [clang, "-Xclang", "-ast-dump=json", "-fsyntax-only", src] + flags
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
            if n.get("kind") == "FunctionDecl" and n.get("name"):
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
        "only_after": sorted(set(fb) - set(fa)),
        "changed": sorted(
            f for f in (fa.keys() & fb.keys())
            if sha1_text(normalise(fa[f])) != sha1_text(normalise(fb[f]))
        )
    }

def list_all_rt_tus(root: Path) -> List[Path]:
    return sorted((root / "os/rt/src").rglob("*.c"))

def select_incremental_tus(changed: List[str], root: Path) -> List[Path]:
    tus: Set[Path] = set()
    header_changed = any(p.startswith("os/rt/include/") for p in changed)

    for p in changed:
        if p.startswith("os/rt/src/") and p.endswith(".c"):
            tus.add(root / p)

    if header_changed:
        tus |= set(list_all_rt_tus(root))

    return sorted(tus)

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--outdir", required=True)
    ap.add_argument("--base", required=True)
    ap.add_argument("--head", required=True)
    ap.add_argument("--mode", choices=["baseline", "incremental"], required=True)
    ap.add_argument("--clang", default="clang")
    ap.add_argument("--build-cmd", default="")
    ap.add_argument("--fallback-includes", nargs="*", default=["-Ios/rt/include"])
    args = ap.parse_args()

    root = Path(".").resolve()
    out = Path(args.outdir)
    out.mkdir(parents=True, exist_ok=True)

    base_commit = git_rev(args.base)
    head_commit = git_rev(args.head)

    compile_db: Dict[str, List[str]] = {}
    if args.build_cmd and build_compile_db(args.build_cmd):
        compile_db = read_compile_db()

    if args.mode == "baseline":
        tus = list_all_rt_tus(root)
        changed_files = ["(baseline 초기 생성)"]
    else:
        changed_files = list_changed_files(args.base, args.head)
        tus = select_incremental_tus(changed_files, root)

    results = []

    for tu in tus:
        rel = tu.relative_to(root)

        before = out / f"{rel}.before.c"
        after  = out / f"{rel}.after.c"
        diffp  = out / f"{rel}.diff.json"

        ensure_parent_dir(before)
        ensure_parent_dir(after)
        ensure_parent_dir(diffp)

        before.write_text(run(["git", "show", f"{base_commit}:{rel}"], check=False), encoding="utf-8", errors="ignore")
        after.write_text(run(["git", "show", f"{head_commit}:{rel}"], check=False), encoding="utf-8", errors="ignore")

        flags = compile_db.get(str(tu.resolve()), args.fallback-includes) if False else compile_db.get(str(tu.resolve()), args.fallback_includes)
        flags = filter_args(flags)

        ast_b = clang_ast(args.clang, str(before), flags)
        ast_a = clang_ast(args.clang, str(after), flags)

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
    (out / "summary.json").write_text(json.dumps(summary, indent=2), encoding="utf-8")

if __name__ == "__main__":
    main()
PY

          chmod +x tools/ast_ci/ast_build_and_diff.py
        '''
      }
    }

    stage('AST 생성 및 diff') {
      when { expression { return env.DO_AST == "1" } }
      steps {
        sh '''
          set -eux
          COMMIT=$(git rev-parse --short HEAD)
          BASELINE_DIR="${AST_STORE}/baseline"
          mkdir -p "${BASELINE_DIR}"

          if [ "${AST_MODE}" = "baseline" ]; then
            # [최초 실행] os/rt/src 전체 AST 생성 → baseline 저장
            OUT="ast_out/baseline_${COMMIT}"
            mkdir -p "$OUT"

            python3 tools/ast_ci/ast_build_and_diff.py \
              --outdir "$OUT" \
              --base "HEAD" \
              --head "HEAD" \
              --mode "baseline" \
              --build-cmd "${BUILD_CMD}"

            rsync -a "$OUT/" "${BASELINE_DIR}/"
          else
            # [이후 실행] 변경된 TU만 AST 생성 + baseline 대비 diff
            OUT="ast_out/${COMMIT}"
            mkdir -p "$OUT"

            BASE_COMMIT=$(jq -r '.head_commit' "${BASELINE_DIR}/summary.json")

            python3 tools/ast_ci/ast_build_and_diff.py \
              --outdir "$OUT" \
              --base "${BASE_COMMIT}" \
              --head "HEAD" \
              --mode "incremental" \
              --build-cmd "${BUILD_CMD}"

            mkdir -p "${AST_STORE}/${COMMIT}"
            rsync -a "$OUT/" "${AST_STORE}/${COMMIT}/"
          fi
        '''
      }
    }
  }
}
