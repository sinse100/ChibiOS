// Jenkinsfile
pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '30'))
  }

  environment {
    // AST 결과를 서버에 영구 저장할 경로
    // (Jenkins 실행 계정이 쓰기 권한을 가져야 함)
    AST_STORE = "/var/lib/jenkins/ast/chibios-os-rt"

    // 사용 도구
    CLANG = "clang"
    PY = "python3"

    // ⚠️ 반드시 실제 ChibiOS RT 빌드에 맞는 명령으로 조정 필요
    // 이 빌드를 통해 compile_commands.json을 생성함
    BUILD_CMD = "make -C testrt"

    // 병합 AST 생성 시, 함수 호출을 몇 단계까지 따라갈지
    MERGE_DEPTH = "2"
  }

  triggers {
    // GitHub Webhook / Multibranch Pipeline 사용을 권장
    // pollSCM은 예시용
    pollSCM('H/5 * * * *')
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        sh 'git rev-parse --short HEAD'
      }
    }

    stage('AST 실행 여부 판단') {
      steps {
        script {
          // baseline 디렉터리가 없으면 생성
          sh "mkdir -p '${env.AST_STORE}/baseline'"

          // ==========================================================
          // [BASELINE 존재 여부 판단]
          // - 최초 실행이거나
          // - ${AST_STORE}/baseline/summary.json 이 없는 경우
          //   → baseline AST를 처음부터 생성해야 함
          // ==========================================================
          def baselineExists = fileExists("${env.AST_STORE}/baseline/summary.json")

          // ==========================================================
          // [변경 사항 감지]
          // - origin/master..HEAD 범위에서
          // - os/rt/** 하위에 변경이 있는지 확인
          // ==========================================================
          sh "git fetch origin master:refs/remotes/origin/master || true"
          def changed = sh(
            script: "git diff --name-only origin/master..HEAD | grep '^os/rt/' || true",
            returnStdout: true
          ).trim()

          if (!baselineExists) {
            // ======================================================
            // [최초 실행 경로]
            // - baseline AST가 아직 없음
            // - os/rt/src/**/*.c 전체에 대해 AST 생성
            // ======================================================
            echo "Baseline AST가 존재하지 않음 → 최초 baseline 생성"
            env.DO_AST = "1"
            env.AST_MODE = "baseline"

          } else if (changed == "") {
            // ======================================================
            // [실행 불필요]
            // - baseline 존재
            // - os/rt/** 변경 없음
            // ======================================================
            echo "os/rt 변경 없음 → AST 단계 스킵"
            env.DO_AST = "0"

          } else {
            // ======================================================
            // [이후 실행 경로]
            // - baseline 존재
            // - os/rt/** 변경 존재
            // - 변경된 TU만 AST 생성 + diff
            // ======================================================
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
          sudo apt-get update
          sudo apt-get install -y clang bear jq python3
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
"""
ChibiOS RT os/rt 영역에 대한 AST 생성 및 비교 스크립트

동작 모드:
  1) baseline 모드
     - 최초 실행 시
     - os/rt/src/**/*.c 전체를 대상으로 AST 생성
  2) incremental 모드
     - 이후 실행 시
     - 변경된 TU만 AST 생성
     - baseline 커밋 대비 함수 단위 AST diff 수행
"""

import argparse
import json
import subprocess
import sys
import hashlib
from pathlib import Path
from typing import Any, Dict, List, Set, Optional

# -------------------------------
# 공통 유틸리티 함수
# -------------------------------

def run(cmd: List[str], check: bool = True) -> str:
    """외부 명령 실행"""
    p = subprocess.run(cmd, text=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    if check and p.returncode != 0:
        sys.stderr.write(p.stderr)
        raise SystemExit(p.returncode)
    return p.stdout

def run_shell(cmd: str) -> str:
    """쉘 명령 실행"""
    return run(["bash", "-lc", cmd], check=False)

def sha1_text(s: str) -> str:
    """문자열 해시"""
    return hashlib.sha1(s.encode("utf-8", errors="ignore")).hexdigest()

def git_rev(ref: str) -> str:
    """Git 커밋 해시 조회"""
    return run(["git", "rev-parse", ref]).strip()

# -------------------------------
# Git 변경 파일 탐지
# -------------------------------

def list_changed_files(base: str, head: str) -> List[str]:
    """base..head 범위의 변경 파일 목록"""
    out = run(["git", "diff", "--name-only", f"{base}..{head}"])
    return [x for x in out.splitlines() if x]

# -------------------------------
# compile_commands.json 처리
# -------------------------------

def build_compile_db(build_cmd: str) -> bool:
    """bear를 사용해 compile_commands.json 생성"""
    run_shell(f"bear -- {build_cmd}")
    return Path("compile_commands.json").exists()

def read_compile_db() -> Dict[str, List[str]]:
    """compile_commands.json 파싱"""
    db = Path("compile_commands.json")
    if not db.exists():
        return {}

    data = json.loads(db.read_text())
    mapping = {}
    for e in data:
        f = str(Path(e["file"]).resolve())
        args = e.get("arguments") or e.get("command", "").split()
        mapping[f] = args[1:]  # 컴파일러 제거
    return mapping

def filter_args(args: List[str]) -> List[str]:
    """AST 생성에 불필요한 옵션 제거"""
    skip = {"-c", "-MMD", "-MP"}
    out = []
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

# -------------------------------
# AST 처리 로직
# -------------------------------

def clang_ast(clang: str, src: str, flags: List[str]) -> Dict[str, Any]:
    """clang AST(JSON) 생성"""
    cmd = [clang, "-Xclang", "-ast-dump=json", "-fsyntax-only", src] + flags
    return json.loads(run(cmd))

def normalise(node: Any) -> str:
    """AST 노드를 문자열로 정규화"""
    if not isinstance(node, dict):
        return ""
    s = f"{node.get('kind','')}|{node.get('name','')}"
    for c in node.get("inner", []) or []:
        s += normalise(c)
    return s

def index_functions(ast: Dict[str, Any]) -> Dict[str, Dict[str, Any]]:
    """함수 선언 노드 인덱싱"""
    out = {}
    def walk(n):
        if isinstance(n, dict):
            if n.get("kind") == "FunctionDecl" and n.get("name"):
                out[n["name"]] = n
            for c in n.get("inner", []) or []:
                walk(c)
    walk(ast)
    return out

def diff_functions(a, b):
    """함수 단위 AST diff"""
    fa = index_functions(a)
    fb = index_functions(b)
    return {
        "only_before": sorted(set(fa) - set(fb)),
        "only_after": sorted(set(fb) - set(fa)),
        "changed": sorted(
            f for f in fa.keys() & fb.keys()
            if sha1_text(normalise(fa[f])) != sha1_text(normalise(fb[f]))
        )
    }

# -------------------------------
# TU 선택 로직
# -------------------------------

def list_all_rt_tus(root: Path) -> List[Path]:
    """os/rt/src 전체 TU"""
    return sorted((root / "os/rt/src").rglob("*.c"))

def select_incremental_tus(changed: List[str], root: Path) -> List[Path]:
    """변경 기반 TU 선택"""
    tus = set()
    header_changed = any(p.startswith("os/rt/include/") for p in changed)

    for p in changed:
        if p.startswith("os/rt/src/") and p.endswith(".c"):
            tus.add(root / p)

    if header_changed:
        # 헤더 변경 시 보수적으로 전체 TU 재생성
        tus |= set(list_all_rt_tus(root))

    return sorted(tus)

# -------------------------------
# 메인 진입점
# -------------------------------

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

    compile_db = {}
    if args.build_cmd and build_compile_db(args.build_cmd):
        compile_db = read_compile_db()

    # ==================================================
    # [MODE 분기]
    # ==================================================
    if args.mode == "baseline":
        # 최초 실행: os/rt/src 전체 AST 생성
        tus = list_all_rt_tus(root)
        changed = ["(baseline 초기 생성)"]
    else:
        # 이후 실행: 변경 파일 기반 TU 선택
        changed = list_changed_files(args.base, args.head)
        tus = select_incremental_tus(changed, root)

    results = []

    for tu in tus:
        rel = tu.relative_to(root)
        before = out / f"{rel}.before.c"
        after  = out / f"{rel}.after.c"

        before.write_text(run(["git", "show", f"{base_commit}:{rel}"], check=False))
        after.write_text(run(["git", "show", f"{head_commit}:{rel}"], check=False))

        flags = compile_db.get(str(tu.resolve()), args.fallback_includes)
        flags = filter_args(flags)

        ast_b = clang_ast(args.clang, str(before), flags)
        ast_a = clang_ast(args.clang, str(after), flags)

        diff = diff_functions(ast_b, ast_a)

        (out / f"{rel}.diff.json").write_text(json.dumps(diff, indent=2))
        results.append({"tu": str(rel), **diff})

    summary = {
        "mode": args.mode,
        "base_commit": base_commit,
        "head_commit": head_commit,
        "changed_files": changed,
        "results": results
    }
    (out / "summary.json").write_text(json.dumps(summary, indent=2))

if __name__ == "__main__":
    main()
PY
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

          if [ "${AST_MODE}" = "baseline" ]; then
            # ======================================================
            # [최초 실행]
            # - os/rt/src/**/*.c 전체 AST 생성
            # - baseline으로 저장
            # ======================================================
            OUT="ast_out/baseline_${COMMIT}"
            python3 tools/ast_ci/ast_build_and_diff.py \
              --outdir "$OUT" \
              --base "HEAD" \
              --head "HEAD" \
              --mode "baseline" \
              --build-cmd "${BUILD_CMD}"

            rsync -a "$OUT/" "${BASELINE_DIR}/"

          else
            # ======================================================
            # [이후 실행]
            # - 변경된 TU만 AST 생성
            # - baseline 대비 함수 단위 diff
            # ======================================================
            OUT="ast_out/${COMMIT}"
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
