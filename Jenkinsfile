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

          // 관심 경로를 정규식으로 넉넉하게 감시
          sh "git fetch origin master:refs/remotes/origin/master || true"
          def changed = sh(
            script: """
              git diff --name-only origin/master..HEAD | \\
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
          # [추가] chlicense.h가 저장소에 없을 수도 있으니(또는 경로가 다를 수 있으니)
          #        AST 파싱만 통과시키기 위한 더미 헤더를 생성
          #        (실제 파일이 있으면 아무것도 안 함)
          # ==========================================================
          mkdir -p tools/ast_ci/generated
          if [ ! -f "chlicense.h" ] && [ ! -f "os/license/chlicense.h" ]; then
            cat > tools/ast_ci/generated/chlicense.h << 'H'
#ifndef CHLICENSE_H
#define CHLICENSE_H
/* AST 파싱용 더미 헤더: 라이선스/배너 정의가 필요한 경우 여기에 최소 정의를 추가 */
#endif
H
            echo "[INFO] chlicense.h가 트리에 없어서 더미 헤더를 생성했습니다: tools/ast_ci/generated/chlicense.h"
          else
            echo "[INFO] chlicense.h가 트리에 존재합니다(더미 생성 생략)."
          fi

          # ==========================================================
          # AST 참고 경로 설정파일 생성(없을 때만)
          # - 핵심 수정: chlicense.h 탐색을 위해 os/license 및 generated include 추가
          # ==========================================================
          if [ ! -f "${AST_PATHS_CONFIG}" ]; then
            cat > "${AST_PATHS_CONFIG}" << 'JSON'
{
  "tu_roots": [
    "os/rt/src",
    "os/oslib/src",
    "os/hal/src",
    "os/common/ports/ARMv6-M-RP2"
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
    "os/hal/include/",
    "os/license/"
  ],
  "fallback_includes": [
    "-Itools/ast_ci/generated",
    "-Ios/license",
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

          # ==========================================================
          # ast_build_and_diff.py 생성(기존 유지)
          # ==========================================================
          cat > tools/ast_ci/ast_build_and_diff.py << 'PY'
#!/usr/bin/env python3
import argparse, json, subprocess, sys, hashlib
from pathlib import Path
from typing import Any, Dict, List, Set, Tuple

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

def load_config(path: str) -> Dict[str, Any]:
    p = Path(path)
    if not p.exists():
        return {}
    try:
        return json.loads(p.read_text(encoding="utf-8", errors="ignore"))
    except Exception:
        return {}

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

    for p in changed:
        for tu_root in tu_roots:
            prefix = tu_root.rstrip("/") + "/"
            if p.startswith(prefix) and p.endswith(".c"):
                tus.add(root / p)

    header_changed = any(p.startswith(watched_header_prefixes) and p.endswith(".h") for p in changed)
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

          # ==========================================================
          # merged AST 생성기(ast_merge.py) 생성(기존 유지)
          # ==========================================================
          cat > tools/ast_ci/ast_merge.py << 'PY'
#!/usr/bin/env python3
import argparse
import json
from pathlib import Path
from typing import Any, Dict, List, Set, Tuple

def load_json(p: Path) -> Any:
    return json.loads(p.read_text(encoding="utf-8", errors="ignore"))

def save_json(p: Path, obj: Any) -> None:
    p.parent.mkdir(parents=True, exist_ok=True)
    p.write_text(json.dumps(obj, ensure_ascii=False, indent=2), encoding="utf-8")

def walk(node: Any):
    if isinstance(node, dict):
        yield node
        for c in node.get("inner", []) or []:
            yield from walk(c)
    elif isinstance(node, list):
        for x in node:
            yield from walk(x)

def has_body(func_decl: Dict[str, Any]) -> bool:
    for c in func_decl.get("inner", []) or []:
        if isinstance(c, dict) and c.get("kind") == "CompoundStmt":
            return True
    kinds = [x.get("kind") for x in func_decl.get("inner", []) or [] if isinstance(x, dict)]
    return "CompoundStmt" in kinds

def index_functions(ast: Dict[str, Any], tu_rel: str) -> Dict[str, Tuple[str, Dict[str, Any]]]:
    idx: Dict[str, Tuple[str, Dict[str, Any]]] = {}
    for n in walk(ast):
        if isinstance(n, dict) and n.get("kind") == "FunctionDecl" and n.get("name"):
            name = n["name"]
            if name not in idx:
                idx[name] = (tu_rel, n)
            else:
                _, old = idx[name]
                if (not has_body(old)) and has_body(n):
                    idx[name] = (tu_rel, n)
    return idx

def extract_callee_names(call_expr: Dict[str, Any]) -> List[str]:
    names: List[str] = []
    for n in walk(call_expr):
        if not isinstance(n, dict):
            continue
        if n.get("kind") == "DeclRefExpr":
            nm = n.get("name")
            if nm:
                names.append(nm)
            ref = n.get("referencedDecl")
            if isinstance(ref, dict) and ref.get("name"):
                names.append(ref["name"])

    seen = set()
    out = []
    for x in names:
        if x not in seen:
            seen.add(x)
            out.append(x)
    return out

def build_merged_function(
    func_decl: Dict[str, Any],
    func_idx: Dict[str, Tuple[str, Dict[str, Any]]],
    max_depth: int,
) -> Dict[str, Any]:
    merged = json.loads(json.dumps(func_decl))

    visited: Set[str] = set()
    stack: List[Tuple[str, int]] = []

    def enqueue_from(node: Dict[str, Any], depth: int):
        for n in walk(node):
            if isinstance(n, dict) and n.get("kind") == "CallExpr":
                for callee in extract_callee_names(n):
                    stack.append((callee, depth))

    enqueue_from(func_decl, 1)

    merged_callees: List[Dict[str, Any]] = []

    while stack:
        callee, depth = stack.pop()
        if depth > max_depth:
            continue
        if callee in visited:
            continue
        visited.add(callee)

        if callee not in func_idx:
            continue

        callee_tu, callee_decl = func_idx[callee]
        callee_copy = json.loads(json.dumps(callee_decl))

        merged_callees.append({
            "name": callee,
            "tu": callee_tu,
            "ast": callee_copy
        })

        enqueue_from(callee_decl, depth + 1)

    merged["__merged_callees__"] = merged_callees
    merged["__merged_meta__"] = {
        "max_depth": max_depth,
        "callee_count": len(merged_callees)
    }
    return merged

def merge_one_side(out_dir: Path, side: str, max_depth: int) -> None:
    ast_files = sorted(out_dir.rglob(f"*.{side}.ast.json"))

    global_idx: Dict[str, Tuple[str, Dict[str, Any]]] = {}
    per_file_ast: Dict[Path, Dict[str, Any]] = {}

    for f in ast_files:
        ast = load_json(f)
        per_file_ast[f] = ast

        tu_rel = str(f.relative_to(out_dir)).replace(f".{side}.ast.json", "")
        idx = index_functions(ast, tu_rel)
        for k, v in idx.items():
            if k not in global_idx:
                global_idx[k] = v
            else:
                _, old = global_idx[k]
                _, new = v
                if (not has_body(old)) and has_body(new):
                    global_idx[k] = v

    for f, ast in per_file_ast.items():
        merged_root = json.loads(json.dumps(ast))

        merged_funcs: List[Dict[str, Any]] = []
        for n in walk(ast):
            if isinstance(n, dict) and n.get("kind") == "FunctionDecl" and n.get("name"):
                merged_funcs.append(build_merged_function(n, global_idx, max_depth=max_depth))

        merged_root["__merged_functions__"] = merged_funcs
        merged_root["__merged_index_size__"] = len(global_idx)

        out_file = f.with_name(f.name.replace(f".{side}.ast.json", f".{side}.merged.ast.json"))
        save_json(out_file, merged_root)

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--dir", required=True)
    ap.add_argument("--max-depth", type=int, default=3)
    args = ap.parse_args()

    out_dir = Path(args.dir).resolve()
    if not out_dir.exists():
        raise SystemExit(f"[ERROR] out_dir not found: {out_dir}")

    merge_one_side(out_dir, "before", args.max_depth)
    merge_one_side(out_dir, "after", args.max_depth)

if __name__ == "__main__":
    main()
PY
          chmod +x tools/ast_ci/ast_merge.py
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
