// Jenkinsfile (v17 FIX: Python 백슬래시 이스케이프 문제 해결 + ast_paths.json 표준 덮어쓰기)
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
    BUILD_CMD_BASELINE = "${BUILD_CMD}"

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
          sh '''
            mkdir -p "${AST_STORE}/baseline"
            git fetch origin master:refs/remotes/origin/master
            set -e
            git diff --name-only origin/master..HEAD | grep -E '^(os/rt/|os/common/ports/ARMv6-M-RP2/|os/rt/templates/|os/oslib/src/|os/hal/src/)' || true
          '''
          echo "Baseline AST가 존재하지 않음 → 최초 baseline 생성"
        }
      }
    }

    stage('의존성 설치') {
      steps {
        sh '''
          set -eux
          export DEBIAN_FRONTEND=noninteractive
          command -v clang || (sudo -n apt-get update && sudo -n apt-get install -y clang bear jq python3)
          sudo -n apt-get update
          sudo -n apt-get install -y clang bear jq python3
        '''
      }
    }

    stage('AST 분석 스크립트 생성') {
      steps {
        sh '''
          set -eux
          mkdir -p tools/ast_ci
          mkdir -p tools/ast_ci/generated

          # (기존 로직) chlicense.h 처리
          if [ ! -f chlicense.h ] && [ ! -f os/license/chlicense.h ]; then
            echo "[INFO] chlicense.h 없음 → 더미 생성"
            cat > tools/ast_ci/generated/chlicense.h <<'EOF'
#ifndef CHLICENSE_H
#define CHLICENSE_H
#endif
EOF
          else
            echo "[INFO] chlicense.h가 트리에 존재합니다(더미 생성 생략)."
          fi

          # ast_paths.json: repo 파일이 있더라도, 업로드본과 항상 동일하게 맞추기 위해 매 빌드마다 덮어씀
          echo "[INFO] tools/ast_ci/ast_paths.json 을 표준 설정으로 덮어씁니다."
          cat > tools/ast_ci/ast_paths.json <<'EOF'
{
  "watched_prefixes": [
    "os/rt/",
    "os/common/ports/ARMv6-M-RP2/",
    "os/rt/templates/",
    "os/oslib/src/",
    "os/hal/src/"
  ],
  "extra_dep_prefixes": [
    "os/common/ports/ARMv6-M-RP2/",
    "os/rt/templates/",
    "os/oslib/src/",
    "os/hal/src/"
  ],
  "extra_dep_headers": [
    "os/common/ports/ARMv6-M-RP2/",
    "os/rt/templates/",
    "os/oslib/include/",
    "os/hal/include/"
  ],
  "rt_tu_root": "os/rt/src",
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
EOF

          # chtypes.h 위치를 자동 탐색해 fallback include에 추가 (중복 방지)
          CTYPES_FILE=$(git ls-files | grep -E '(^|/)chtypes\\.h$' | head -n 1 || true)
          if [ -n "$CTYPES_FILE" ]; then
            CTYPES_DIR=$(dirname "$CTYPES_FILE")
            echo "[INFO] chtypes.h 발견: $CTYPES_FILE (dir=$CTYPES_DIR) → fallback_includes에 반영 시도"

            tmp=$(mktemp)
            jq --arg inc "-I${CTYPES_DIR}" '
              if (.fallback_includes | index($inc)) then .
              else .fallback_includes += [$inc]
              end
            ' tools/ast_ci/ast_paths.json > "$tmp" && mv "$tmp" tools/ast_ci/ast_paths.json

            echo "[INFO] fallback_includes 업데이트 완료"
          else
            echo "[WARN] chtypes.h를 git 트리에서 찾지 못함"
          fi

          # Python 스크립트 생성 (FIX: 백슬래시 리터럴 제거 → chr(92) 사용)
          cat > tools/ast_ci/ast_build_and_diff.py <<'PY'
#!/usr/bin/env python3
import argparse, os, json, subprocess, shutil, sys, glob, hashlib
from datetime import datetime

def run(cmd, **kw):
    print("[CMD]", " ".join(cmd), flush=True)
    return subprocess.run(cmd, **kw)

def sha256_file(p):
    h = hashlib.sha256()
    with open(p, "rb") as f:
        for b in iter(lambda: f.read(1024*1024), b""):
            h.update(b)
    return h.hexdigest()

def clang_ast(src, out_json, includes, clang="clang"):
    cmd = [clang, "-Xclang", "-ast-dump=json", "-fsyntax-only"] + includes + [src]
    with open(out_json, "w") as f:
        subprocess.run(cmd, stdout=f, check=True)

def load_config(path):
    with open(path, "r") as f:
        return json.load(f)

def collect_sources(globs_):
    out = []
    for g in globs_:
        out += glob.glob(g, recursive=True)
    return sorted(set(out))

def ensure_compile_db(build_cmd, workdir="."):
    if not build_cmd:
        return None
    if shutil.which("bear") is None:
        raise RuntimeError("bear not found. please install bear.")
    if os.path.exists("compile_commands.json"):
        os.remove("compile_commands.json")
    run(["bash", "-lc", f"bear -- {build_cmd}"], cwd=workdir, check=True)
    if os.path.exists("compile_commands.json"):
        return os.path.abspath("compile_commands.json")
    return None

def parse_compile_db(path):
    with open(path, "r") as f:
        return json.load(f)

def incs_from_compile_db(cdb, src_path):
    sp = os.path.abspath(src_path)
    best = None
    for e in cdb:
        f = e.get("file")
        if not f:
            continue
        ap = os.path.abspath(os.path.join(e.get("directory","."), f)) if not os.path.isabs(f) else os.path.abspath(f)
        if ap == sp:
            best = e
            break
    if not best:
        return []
    cmd = best.get("command")
    if not cmd and "arguments" in best:
        args = best["arguments"]
    else:
        args = cmd.split()
    out = []
    it = iter(args)
    for a in it:
        if a.startswith("-I") or a.startswith("-D") or a in ("-isystem", "-iquote"):
            out.append(a)
            if a in ("-isystem","-iquote"):
                try:
                    out.append(next(it))
                except StopIteration:
                    pass
    return out

def write_summary(outdir, base, head, mode, files, head_commit):
    summary = {
        "mode": mode,
        "base": base,
        "head": head,
        "head_commit": head_commit,
        "generated_at": datetime.utcnow().isoformat()+"Z",
        "files": files
    }
    with open(os.path.join(outdir, "summary.json"), "w") as f:
        json.dump(summary, f, indent=2)

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--outdir", required=True)
    ap.add_argument("--base", required=True)
    ap.add_argument("--head", required=True)
    ap.add_argument("--mode", choices=["baseline","incremental"], required=True)
    ap.add_argument("--build-cmd", default="")
    ap.add_argument("--config", required=True)
    args = ap.parse_args()

    cfg = load_config(args.config)
    globs_ = cfg.get("source_globs", [])
    fallback_includes = cfg.get("fallback_includes", [])
    extra_defines = cfg.get("extra_defines", [])

    outdir = args.outdir
    os.makedirs(outdir, exist_ok=True)

    head_commit = subprocess.check_output(["git","rev-parse","--short",args.head]).decode().strip()

    cdb = None
    if args.build_cmd:
        try:
            cdb_path = ensure_compile_db(args.build_cmd)
            if cdb_path:
                cdb = parse_compile_db(cdb_path)
        except Exception as e:
            print("[WARN] compile DB 생성 실패, fallback includes로 진행:", e, file=sys.stderr)

    sources = collect_sources(globs_)
    files_meta = []

    def incs_for(src):
        incs = []
        if cdb:
            incs += incs_from_compile_db(cdb, src)
        if not incs:
            incs += fallback_includes
        incs += extra_defines
        seen = set()
        dedup = []
        for x in incs:
            if x not in seen:
                dedup.append(x)
                seen.add(x)
        return dedup

    for src in sources:
        rel = src.replace(chr(92), "/")
        before_src = os.path.join(outdir, rel + ".before.c")
        after_src  = os.path.join(outdir, rel + ".after.c")
        os.makedirs(os.path.dirname(before_src), exist_ok=True)
        shutil.copyfile(src, before_src)
        shutil.copyfile(src, after_src)

        before_ast = before_src + ".ast.json"
        after_ast  = after_src + ".ast.json"

        clang_ast(before_src, before_ast, incs_for(src), clang=os.environ.get("CLANG","clang"))
        clang_ast(after_src,  after_ast,  incs_for(src), clang=os.environ.get("CLANG","clang"))

        files_meta.append({
            "source": rel,
            "before_ast": before_ast.replace(chr(92), "/"),
            "after_ast":  after_ast.replace(chr(92), "/"),
            "before_sha256": sha256_file(before_ast),
            "after_sha256":  sha256_file(after_ast),
        })

    write_summary(outdir, args.base, args.head, args.mode, files_meta, head_commit)
    print("[OK] summary.json written:", os.path.join(outdir, "summary.json"))

if __name__ == "__main__":
    main()
PY
          chmod +x tools/ast_ci/ast_build_and_diff.py

          cat > tools/ast_ci/ast_merge.py <<'PY'
#!/usr/bin/env python3
import argparse, os, json, glob

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--dir", required=True)
    ap.add_argument("--max-depth", type=int, default=3)
    args = ap.parse_args()

    ast_files = glob.glob(os.path.join(args.dir, "**", "*.ast.json"), recursive=True)
    merged = {"files": []}
    for f in sorted(ast_files):
      merged["files"].append(f.replace(chr(92), "/"))

    with open(os.path.join(args.dir, "merged_ast_index.json"), "w") as out:
      json.dump(merged, out, indent=2)

    print("[OK] merged_ast_index.json written:", os.path.join(args.dir, "merged_ast_index.json"))

if __name__ == "__main__":
    main()
PY
          chmod +x tools/ast_ci/ast_merge.py
        '''
      }
    }

    stage('AST 생성 및 diff (롤백 포함))') {
      steps {
        timeout(time: 25, unit: 'MINUTES') {
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

            AST_MODE="incremental"
            if [ ! -f "${BASELINE_DIR}/summary.json" ]; then
              AST_MODE="baseline"
            fi

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
  }

  post {
    always {
      echo "빌드 결과: ${currentBuild.currentResult}"
    }
  }
}
