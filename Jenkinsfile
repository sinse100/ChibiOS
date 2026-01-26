// Jenkinsfile (v14 FIX: chlicense.h include 경로 보강 + 없으면 더미 생성)
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

    // AST 참고 경로 설정파일
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
          sudo -n true
          echo "[OK] jenkins 계정이 비밀번호 없이 sudo 사용 가능"
        '''
      }
    }

    stage('AST 실행 여부 판단') {
      steps {
        script {
          sh "mkdir -p ${AST_STORE}/baseline"
          def baselineExists = fileExists("${AST_STORE}/baseline/summary.json")

          sh "git fetch origin master:refs/remotes/origin/master"
          def changed = sh(
            script: '''
              set -e
              git diff --name-only origin/master..HEAD | grep -E "^(os/rt/|os/common/ports/ARMv6-M-RP2/|os/rt/templates/|os/oslib/src/|os/hal/src/)" || true
            ''',
            returnStdout: true
          ).trim()

          if (!baselineExists) {
            env.AST_MODE = "baseline"
            echo "Baseline AST가 존재하지 않음 → 최초 baseline 생성"
          } else if (changed) {
            env.AST_MODE = "incremental"
            echo "변경 감지됨 → incremental AST 생성"
          } else {
            env.AST_MODE = "skip"
            echo "변경 없음 → AST 생성 스킵"
          }
        }
      }
    }

    stage('의존성 설치') {
      when { expression { env.AST_MODE != "skip" } }
      steps {
        sh '''
          set -eux
          export DEBIAN_FRONTEND=noninteractive
          command -v clang || sudo -n apt-get update
          sudo -n apt-get install -y clang bear jq python3
        '''
      }
    }

    stage('AST 분석 스크립트 생성') {
      when { expression { env.AST_MODE != "skip" } }
      steps {
        sh '''
          set -eux

          mkdir -p tools/ast_ci
          mkdir -p tools/ast_ci/generated

          # ------------------------------------------------------------
          # chlicense.h 처리:
          # 1) repo root에 있으면 OK
          # 2) os/license/chlicense.h가 있으면 OK(단, include path에 os/license 추가 필요)
          # 3) 둘 다 없으면 tools/ast_ci/generated/chlicense.h 더미 생성
          # ------------------------------------------------------------
          if [ ! -f "chlicense.h" ] && [ ! -f "os/license/chlicense.h" ]; then
            echo "[INFO] chlicense.h가 없어 더미 생성: tools/ast_ci/generated/chlicense.h"
            cat > tools/ast_ci/generated/chlicense.h <<'EOF'
#ifndef CHLICENSE_H
#define CHLICENSE_H
/* Dummy license header for CI AST parsing */
#endif
EOF
          else
            echo "[INFO] chlicense.h가 트리에 존재합니다(더미 생성 생략)."
          fi

          # ------------------------------------------------------------
          # AST 경로 설정 파일(사용자 설정):
          # 이미 있으면 그대로 사용.
          # ------------------------------------------------------------
          if [ ! -f tools/ast_ci/ast_paths.json ]; then
            echo "[INFO] tools/ast_ci/ast_paths.json 생성"
            cat > tools/ast_ci/ast_paths.json <<'JSON'
{
  "ast_scan_dirs": [
    "os/rt",
    "os/common/ports/ARMv6-M-RP2",
    "os/rt/templates",
    "os/oslib/src",
    "os/hal/src"
  ],
  "fallback_includes": [
    "-Ios/rt/include",
    "-Ios/rt/templates",
    "-Ios/common/ports/ARMv6-M-RP2",
    "-Ios/license",
    "-Itools/ast_ci/generated"
  ]
}
JSON
          else
            echo "[INFO] tools/ast_ci/ast_paths.json 가 존재하므로 해당 설정을 사용합니다."
          fi

          # ------------------------------------------------------------
          # AST 생성 + diff 스크립트 (build-cmd 있으면 compile_commands.json 사용)
          # ------------------------------------------------------------
          cat > tools/ast_ci/ast_build_and_diff.py <<'PY'
#!/usr/bin/env python3
import argparse, json, os, re, shutil, subprocess, sys
from pathlib import Path

def run(cmd, check=True, capture=False):
    if capture:
        return subprocess.check_output(cmd, text=True)
    p = subprocess.run(cmd)
    if check and p.returncode != 0:
        raise SystemExit(p.returncode)
    return p.returncode

def git_show(ref, path):
    return run(["git", "show", f"{ref}:{path}"], capture=True)

def read_cfg(cfg_path: Path):
    if not cfg_path.exists():
        return {}
    return json.loads(cfg_path.read_text())

def ensure_dir(p: Path):
    p.mkdir(parents=True, exist_ok=True)

def write_text(p: Path, s: str):
    ensure_dir(p.parent)
    p.write_text(s)

def collect_tus(scan_dirs):
    tus = []
    for d in scan_dirs:
        root = Path(d)
        if not root.exists():
            continue
        for c in root.rglob("*.c"):
            tus.append(c)
    return sorted(set(tus))

def build_compile_db(build_cmd: str, outdir: Path):
    if not build_cmd.strip():
        return None
    # bear 로 compile_commands.json 생성
    run(["bash","-lc", f"bear -- {build_cmd}"], check=True)
    db = Path("compile_commands.json")
    if not db.exists():
        return None
    shutil.copy(db, outdir / "compile_commands.json")
    return outdir / "compile_commands.json"

def load_compile_db(db_path: Path):
    db = json.loads(db_path.read_text())
    # file -> (directory, arguments/command)
    m = {}
    for ent in db:
        f = ent.get("file")
        d = ent.get("directory", ".")
        args = ent.get("arguments")
        cmd = ent.get("command")
        if args:
            m[f] = (d, args)
        elif cmd:
            m[f] = (d, ["bash","-lc", cmd])
    return m

def clang_ast(src: Path, out_json: Path, incs, clang="clang"):
    ensure_dir(out_json.parent)
    cmd = [clang, "-Xclang", "-ast-dump=json", "-fsyntax-only"] + incs + [str(src)]
    # stderr는 CI 로그에 보이도록 그대로 둔다
    with open(out_json, "w") as f:
        subprocess.run(cmd, stdout=f, check=True)

def diff_json(a: Path, b: Path, out_path: Path):
    # 단순 diff: json pretty + unified diff 저장
    import difflib
    ja = json.loads(a.read_text())
    jb = json.loads(b.read_text())
    sa = json.dumps(ja, indent=2, sort_keys=True).splitlines(keepends=True)
    sb = json.dumps(jb, indent=2, sort_keys=True).splitlines(keepends=True)
    diff = difflib.unified_diff(sa, sb, fromfile=str(a), tofile=str(b))
    ensure_dir(out_path.parent)
    out_path.write_text("".join(diff))

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--outdir", required=True)
    ap.add_argument("--base", required=True)
    ap.add_argument("--head", required=True)
    ap.add_argument("--mode", choices=["baseline","incremental"], required=True)
    ap.add_argument("--build-cmd", default="")
    ap.add_argument("--config", default="tools/ast_ci/ast_paths.json")
    args = ap.parse_args()

    outdir = Path(args.outdir)
    ensure_dir(outdir)

    cfg = read_cfg(Path(args.config))
    scan_dirs = cfg.get("ast_scan_dirs") or ["os/rt"]

    # 사용자 config가 불완전하더라도 필수 include는 강제로 보강
    fallback_includes = cfg.get("fallback_includes") or ["-Ios/rt/include"]

    # Ensure essential include paths exist even when user config is incomplete.
    mandatory_includes = [
        "-I.",
        "-Itools/ast_ci/generated",
        "-Ios/license",
        "-Ios/rt/include",
        "-Ios/rt/templates",
        "-Ios/common/ports/ARMv6-M-RP2",
    ]
    for inc in mandatory_includes:
        if inc not in fallback_includes:
            fallback_includes.append(inc)

    # compile db (있으면 우선)
    db_path = build_compile_db(args.build_cmd, outdir)
    compile_db = load_compile_db(db_path) if db_path else {}

    tus = collect_tus(scan_dirs)

    summary = {
        "mode": args.mode,
        "base": args.base,
        "head": args.head,
        "tu_count": len(tus),
        "head_commit": run(["git","rev-parse", args.head], capture=True).strip(),
    }

    # baseline: 전체 TU 생성
    # incremental: 변경된 TU만 생성(단순히 git diff로 잡힘)
    changed = set()
    if args.mode == "incremental":
        names = run(["bash","-lc", f"git diff --name-only {args.base}..{args.head}"], capture=True).splitlines()
        for n in names:
            if n.endswith(".c"):
                changed.add(Path(n))
    else:
        changed = set(tus)

    for tu in tus:
        if tu not in changed:
            continue

        base_src = outdir / f"{tu}.before.c"
        head_src = outdir / f"{tu}.after.c"

        # 파일 내용 스냅샷
        try:
            write_text(base_src, git_show(args.base, str(tu)))
        except subprocess.CalledProcessError:
            continue
        try:
            write_text(head_src, git_show(args.head, str(tu)))
        except subprocess.CalledProcessError:
            continue

        # include 결정: compile_commands가 있으면 거기서 -I/-D만 추출
        def incs_for(src_path: Path):
            key = str(tu)
            incs = []
            # compile_db 키는 absolute/relative 혼재 가능 → 부분 매칭
            ent = None
            if key in compile_db:
                ent = compile_db[key]
            else:
                for k,v in compile_db.items():
                    if k.endswith(key):
                        ent = v; break

            if ent:
                d, argv = ent
                # argv에서 -I/-D만 추출
                if isinstance(argv, list) and argv[:2] != ["bash","-lc"]:
                    for a in argv:
                        if a.startswith("-I") or a.startswith("-D"):
                            incs.append(a)
                else:
                    # command string 케이스는 fallback
                    incs = list(fallback_includes)
            else:
                incs = list(fallback_includes)
            return incs

        before_ast = outdir / f"{tu}.before.ast.json"
        after_ast  = outdir / f"{tu}.after.ast.json"

        clang_ast(base_src, before_ast, incs_for(base_src), clang=os.environ.get("CLANG","clang"))
        clang_ast(head_src, after_ast,  incs_for(head_src), clang=os.environ.get("CLANG","clang"))

        diff_path = outdir / f"{tu}.ast.diff.txt"
        diff_json(before_ast, after_ast, diff_path)

    (outdir / "summary.json").write_text(json.dumps(summary, indent=2))

if __name__ == "__main__":
    main()
PY
          chmod +x tools/ast_ci/ast_build_and_diff.py

          # ------------------------------------------------------------
          # merged AST 생성 스크립트 (확장 AST)
          # ------------------------------------------------------------
          cat > tools/ast_ci/ast_merge.py <<'PY'
#!/usr/bin/env python3
import argparse, json
from pathlib import Path

def load(p: Path):
    return json.loads(p.read_text())

def save(p: Path, obj):
    p.write_text(json.dumps(obj, indent=2))

def index_functions(ast_obj):
    # 간단 인덱싱: FunctionDecl 이름 -> node
    idx = {}
    def walk(node):
        if not isinstance(node, dict):
            return
        if node.get("kind") == "FunctionDecl":
            name = node.get("name")
            if name:
                idx[name] = node
        for k,v in node.items():
            if isinstance(v, dict):
                walk(v)
            elif isinstance(v, list):
                for it in v:
                    if isinstance(it, dict):
                        walk(it)
    walk(ast_obj)
    return idx

def find_callees(ast_obj):
    callees = set()
    def walk(node):
        if not isinstance(node, dict):
            return
        if node.get("kind") == "CallExpr":
            # clang json에서 directCallee 또는 referencedDecl 등에 이름이 있을 수 있음
            for key in ("directCallee", "referencedDecl", "callee", "name"):
                v = node.get(key)
                if isinstance(v, dict) and "name" in v:
                    callees.add(v["name"])
                elif isinstance(v, str) and key == "name":
                    # name이 함수명인 케이스는 보수적으로 제외
                    pass
        for k,v in node.items():
            if isinstance(v, dict):
                walk(v)
            elif isinstance(v, list):
                for it in v:
                    if isinstance(it, dict):
                        walk(it)
    walk(ast_obj)
    return sorted(callees)

def merge_one(root_ast, all_func_idx, max_depth=3):
    merged = json.loads(json.dumps(root_ast))
    visited = set()

    def attach(node, depth):
        if depth <= 0:
            return
        for callee in find_callees(node):
            if callee in visited:
                continue
            if callee in all_func_idx:
                visited.add(callee)
                # callee AST를 node에 붙임
                node.setdefault("mergedCallees", [])
                node["mergedCallees"].append(all_func_idx[callee])
                attach(all_func_idx[callee], depth-1)

    attach(merged, max_depth)
    return merged

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--dir", required=True)
    ap.add_argument("--max-depth", type=int, default=3)
    args = ap.parse_args()

    d = Path(args.dir)
    ast_files = list(d.rglob("*.ast.json"))
    # 동일 디렉토리의 ast들을 대상으로 함수 인덱싱
    all_idx = {}
    for f in ast_files:
        try:
            obj = load(f)
        except Exception:
            continue
        all_idx.update(index_functions(obj))

    for f in ast_files:
        if f.name.endswith(".before.ast.json"):
            out = f.with_name(f.name.replace(".before.ast.json", ".before.merged.ast.json"))
        elif f.name.endswith(".after.ast.json"):
            out = f.with_name(f.name.replace(".after.ast.json", ".after.merged.ast.json"))
        else:
            continue

        try:
            root = load(f)
            merged = merge_one(root, all_idx, max_depth=args.max_depth)
            save(out, merged)
        except Exception:
            continue

if __name__ == "__main__":
    main()
PY
          chmod +x tools/ast_ci/ast_merge.py
        '''
      }
    }

    stage('AST 생성 및 diff (롤백 포함))') {
      when { expression { env.AST_MODE != "skip" } }
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

              echo "[BASELINE] merged AST 생성 시작"
              ${PY} tools/ast_ci/ast_merge.py --dir "$OUT" --max-depth 3

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
                --build-cmd "${BUILD_CMD}" \
                --config "${AST_PATHS_CONFIG}"

              echo "[INCREMENTAL] merged AST 생성 시작"
              ${PY} tools/ast_ci/ast_merge.py --dir "$OUT" --max-depth 3

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
