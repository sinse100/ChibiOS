pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '30'))
  }

  parameters {
    string(name: 'REPO_URL', defaultValue: 'https://github.com/sinse100/ChibiOS', description: 'Target repository URL')
    string(name: 'BRANCH', defaultValue: 'master', description: 'Git branch to build')
    string(name: 'C_SRC_DIR', defaultValue: 'os/rt/src', description: 'Directory containing .c files')
    string(name: 'H_INC_DIR', defaultValue: 'os/rt/include', description: 'Directory containing .h files (include path)')
    string(name: 'EXTRA_INCLUDE_DIRS', defaultValue: '', description: 'Extra include dirs (comma-separated, relative to repo root)')
    string(name: 'CLANG_BIN', defaultValue: 'clang', description: 'clang executable name/path')
    string(name: 'PYTHON_BIN', defaultValue: 'python3', description: 'python executable name/path')
  }

  environment {
    OUT_DIR = "out_ast"
    RAW_AST_DIR = "out_ast/raw_ast"
    MERGED_AST_DIR = "out_ast/merged_ast"
    NEO4J_DIR = "out_ast/neo4j"
  }

  stages {
    stage('Checkout') {
      steps {
        deleteDir()
        checkout([$class: 'GitSCM',
          branches: [[name: "*/${params.BRANCH}"]],
          userRemoteConfigs: [[url: params.REPO_URL]]
        ])
        sh '''
          set -euxo pipefail
          git rev-parse --abbrev-ref HEAD
          git rev-parse HEAD
        '''
      }
    }

    // ✅ 요청 반영: AST 생성 + 병합 + Neo4j import 파일 변환에 필요한 툴을 설치하는 stage를 완전 반영
    stage('Install toolchain (AST + Neo4j export)') {
      steps {
        sh '''
          set -euxo pipefail

          # 0) Basic
          sudo apt-get update

          # 1) AST generation toolchain
          sudo apt-get install -y --no-install-recommends \
            git \
            clang llvm \
            build-essential

          # 2) Transform/export toolchain
          sudo apt-get install -y --no-install-recommends \
            python3 python3-pip \
            jq

          # 3) Python libs used by merge/export script
          python3 -m pip install --upgrade pip
          python3 -m pip install networkx

          # 4) Version checks (helps debugging in Jenkins logs)
          echo "---- Versions ----"
          clang --version || true
          python3 --version || true
          pip --version || true
          jq --version || true
        '''
      }
    }

    stage('Prepare output dirs & scripts') {
      steps {
        sh '''
          set -euxo pipefail
          mkdir -p "${OUT_DIR}" "${RAW_AST_DIR}" "${MERGED_AST_DIR}" "${NEO4J_DIR}" scripts

          # -------- scripts/gen_ast.sh --------
          cat > scripts/gen_ast.sh << 'EOF'
          #!/usr/bin/env bash
          set -euxo pipefail

          CLANG_BIN="$1"
          REPO_ROOT="$2"
          C_FILE_REL="$3"
          RAW_AST_DIR="$4"
          INC_DIR_REL="$5"
          EXTRA_INCS_CSV="$6"

          C_FILE_ABS="${REPO_ROOT}/${C_FILE_REL}"

          # Build include flags
          INC_FLAGS="-I${REPO_ROOT}/${INC_DIR_REL}"
          if [[ -n "${EXTRA_INCS_CSV}" ]]; then
            IFS=',' read -ra EXTRA <<< "${EXTRA_INCS_CSV}"
            for d in "${EXTRA[@]}"; do
              d_trim="$(echo "$d" | xargs)"
              [[ -n "$d_trim" ]] && INC_FLAGS="${INC_FLAGS} -I${REPO_ROOT}/${d_trim}"
            done
          fi

          # Output name: path-safe
          SAFE_NAME="$(echo "${C_FILE_REL}" | sed 's#[/ ]#_#g')"
          OUT_JSON="${RAW_AST_DIR}/${SAFE_NAME}.ast.json"

          # NOTE: You may need additional -D macros depending on ChibiOS config.
          # Keep it light for syntax-only parsing.
          "${CLANG_BIN}" -x c -fsyntax-only \
            ${INC_FLAGS} \
            -Xclang -ast-dump=json \
            "${C_FILE_ABS}" > "${OUT_JSON}" || {
              echo "[WARN] AST dump failed for ${C_FILE_REL} (skipping)."
              rm -f "${OUT_JSON}"
              exit 0
            }

          echo "${OUT_JSON}"
          EOF
          chmod +x scripts/gen_ast.sh

          # -------- scripts/merge_and_export.py --------
          cat > scripts/merge_and_export.py << 'EOF'
          import argparse
          import glob
          import json
          import os
          import re

          def load_json(path):
            with open(path, 'r', encoding='utf-8') as f:
              return json.load(f)

          def iter_nodes(node):
            if isinstance(node, dict):
              yield node
              for k in ("inner",):
                if k in node and isinstance(node[k], list):
                  for ch in node[k]:
                    yield from iter_nodes(ch)
            elif isinstance(node, list):
              for it in node:
                yield from iter_nodes(it)

          def node_kind(node):
            return node.get("kind")

          def find_function_decls(ast_root):
            funcs = {}
            for n in iter_nodes(ast_root):
              if node_kind(n) == "FunctionDecl" and "name" in n:
                funcs.setdefault(n["name"], n)
                if any(isinstance(ch, dict) and ch.get("kind") == "CompoundStmt" for ch in n.get("inner", []) or []):
                  funcs[n["name"]] = n
            return funcs

          def find_calls_in_function(func_node):
            calls = set()
            for n in iter_nodes(func_node):
              if node_kind(n) == "CallExpr":
                for ch in n.get("inner", []) or []:
                  if not isinstance(ch, dict):
                    continue
                  k = node_kind(ch)
                  if k == "DeclRefExpr":
                    rd = ch.get("referencedDecl")
                    if isinstance(rd, dict) and rd.get("kind") == "FunctionDecl" and rd.get("name"):
                      calls.add(rd["name"])
                    if ch.get("name"):
                      calls.add(ch["name"])
            return calls

          def safe_id(s):
            return re.sub(r'[^A-Za-z0-9_]+', '_', s)

          def merge_asts(global_funcs):
            merged = {}
            for fname, fnode in global_funcs.items():
              caller = json.loads(json.dumps(fnode))
              calls = find_calls_in_function(fnode)

              inline_nodes = []
              for callee in sorted(calls):
                if callee == fname:
                  continue
                callee_node = global_funcs.get(callee)
                if not callee_node:
                  # unresolved -> skip
                  continue
                inline_nodes.append({
                  "kind": "InlineCallee",
                  "name": callee,
                  "inner": [callee_node],
                })

              caller.setdefault("inner", [])
              caller["inner"].extend(inline_nodes)
              merged[fname] = caller
            return merged

          def export_neo4j_csv(merged_funcs, out_nodes_csv, out_edges_csv):
            import csv

            nodes = []
            edges = []

            def add_node(node_id, label, name, kind):
              nodes.append({
                "id:ID": node_id,
                "label:LABEL": label,
                "name": name,
                "kind": kind,
              })

            def add_edge(src, dst, rel_type):
              edges.append({
                ":START_ID": src,
                ":END_ID": dst,
                ":TYPE": rel_type,
              })

            for func_name, ast in merged_funcs.items():
              f_id = f"FUNC_{safe_id(func_name)}"
              add_node(f_id, "Function", func_name, "FunctionDecl")

              for n in ast.get("inner", []) or []:
                if isinstance(n, dict) and n.get("kind") == "InlineCallee" and n.get("name"):
                  callee = n["name"]
                  inl_id = f"INL_{safe_id(func_name)}__{safe_id(callee)}"
                  add_node(inl_id, "Inline", f"{func_name} -> {callee}", "InlineCallee")
                  add_edge(f_id, inl_id, "INLINE")

                  c_id = f"FUNC_{safe_id(callee)}"
                  add_node(c_id, "Function", callee, "FunctionDecl")
                  add_edge(inl_id, c_id, "TARGET")

            uniq = {}
            for n in nodes:
              uniq[n["id:ID"]] = n
            nodes = list(uniq.values())

            with open(out_nodes_csv, "w", newline="", encoding="utf-8") as f:
              w = csv.DictWriter(f, fieldnames=["id:ID","label:LABEL","name","kind"])
              w.writeheader()
              for r in sorted(nodes, key=lambda x: x["id:ID"]):
                w.writerow(r)

            with open(out_edges_csv, "w", newline="", encoding="utf-8") as f:
              w = csv.DictWriter(f, fieldnames=[":START_ID",":END_ID",":TYPE"])
              w.writeheader()
              for r in edges:
                w.writerow(r)

          def main():
            ap = argparse.ArgumentParser()
            ap.add_argument("--raw_ast_dir", required=True)
            ap.add_argument("--merged_ast_dir", required=True)
            ap.add_argument("--neo4j_dir", required=True)
            args = ap.parse_args()

            raw_files = sorted(glob.glob(os.path.join(args.raw_ast_dir, "*.ast.json")))
            if not raw_files:
              raise SystemExit(f"No AST JSON files found in {args.raw_ast_dir}")

            global_funcs = {}
            for p in raw_files:
              ast = load_json(p)
              funcs = find_function_decls(ast)
              for k, v in funcs.items():
                global_funcs[k] = v

            merged_funcs = merge_asts(global_funcs)

            os.makedirs(args.merged_ast_dir, exist_ok=True)
            for func_name, merged_ast in merged_funcs.items():
              outp = os.path.join(args.merged_ast_dir, f"{safe_id(func_name)}.merged.ast.json")
              with open(outp, "w", encoding="utf-8") as f:
                json.dump(merged_ast, f, ensure_ascii=False)

            os.makedirs(args.neo4j_dir, exist_ok=True)
            nodes_csv = os.path.join(args.neo4j_dir, "nodes.csv")
            edges_csv = os.path.join(args.neo4j_dir, "edges.csv")
            export_neo4j_csv(merged_funcs, nodes_csv, edges_csv)

            print(f"[OK] merged_ast_dir={args.merged_ast_dir}")
            print(f"[OK] neo4j_csv={nodes_csv}, {edges_csv}")

          if __name__ == "__main__":
            main()
          EOF
        '''
      }
    }

    stage('Generate AST per .c file') {
      steps {
        sh '''
          set -euxo pipefail
          REPO_ROOT="$PWD"
          mkdir -p "${RAW_AST_DIR}"

          mapfile -t CFILES < <(find "${C_SRC_DIR}" -type f -name "*.c" | sort)
          echo "Found ${#CFILES[@]} C files under ${C_SRC_DIR}"

          for f in "${CFILES[@]}"; do
            scripts/gen_ast.sh "${CLANG_BIN}" "${REPO_ROOT}" "${f}" "${RAW_AST_DIR}" "${H_INC_DIR}" "${EXTRA_INCLUDE_DIRS}" || true
          done

          echo "Generated AST count:"
          ls -1 "${RAW_AST_DIR}"/*.ast.json 2>/dev/null | wc -l || true
        '''
      }
    }

    stage('Merge ASTs (caller <- callee) & Export Neo4j CSV') {
      steps {
        sh '''
          set -euxo pipefail
          "${PYTHON_BIN}" scripts/merge_and_export.py \
            --raw_ast_dir "${RAW_AST_DIR}" \
            --merged_ast_dir "${MERGED_AST_DIR}" \
            --neo4j_dir "${NEO4J_DIR}"
        '''
      }
    }

    stage('Archive artifacts') {
      steps {
        archiveArtifacts artifacts: 'out_ast/**', fingerprint: true
      }
    }
  }

  post {
    always {
      sh '''
        set +e
        echo "Workspace: $PWD"
        echo "Artifacts under out_ast/"
        find out_ast -maxdepth 3 -type f | sed 's#^# - #' || true
      '''
    }
  }
}
