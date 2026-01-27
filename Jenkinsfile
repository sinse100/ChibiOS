pipeline {
  agent any

  triggers {
    githubPush()
  }

  options {
    disableConcurrentBuilds()
    timeout(time: 60, unit: 'MINUTES')
  }

  environment {
    SRC_DIR = "os/rt/src"
    INC_DIR = "os/rt/include"
    OUT_DIR = "ast_out"
    CLANG_BIN = "clang"
    PY = "python3"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'git rev-parse --short HEAD > GIT_SHA.txt'
      }
    }

    stage('Prepare workspace') {
      steps {
        sh """#!/usr/bin/env bash
          set -euxo pipefail
          mkdir -p ${OUT_DIR}/ast_json
          mkdir -p ${OUT_DIR}/tu_wrappers
          mkdir -p ${OUT_DIR}/merged
          mkdir -p ${OUT_DIR}/neo4j
          mkdir -p ${OUT_DIR}/dfd
        """
      }
    }

    stage('Write helper scripts') {
      steps {
        writeFile file: "${OUT_DIR}/extract_ast.py", text: '''\
#!/usr/bin/env python3
import os, sys, json, subprocess, hashlib
from pathlib import Path

CLANG = os.environ.get("CLANG_BIN", "clang")
SRC_DIR = os.environ.get("SRC_DIR", "os/rt/src")
INC_DIR = os.environ.get("INC_DIR", "os/rt/include")
OUT_DIR = os.environ.get("OUT_DIR", "ast_out")

TARGET_DIRS = [SRC_DIR, INC_DIR]
EXTS = {".c", ".h"}

# ChibiOS는 설정 헤더/템플릿 경로가 필요한 경우가 많아 include 경로를 보강
EXTRA_INCLUDES = [
    "os/rt/include",
    "os/rt/templates",
    "os/hal/include",
    "os/hal/templates",
    "os/common/osal/include",
]

def run(cmd):
    p = subprocess.run(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
    return p.returncode, p.stdout, p.stderr

def file_hash(p: Path) -> str:
    h = hashlib.sha1()
    with p.open("rb") as f:
        while True:
            b = f.read(1024 * 1024)
            if not b: break
            h.update(b)
    return h.hexdigest()

def common_clang_args(repo_root: Path):
    args = [
        "-fsyntax-only",
        "-Xclang", "-ast-dump=json",
        "-ferror-limit=0",
    ]
    # include path 보강
    for inc in EXTRA_INCLUDES:
        ip = (repo_root / inc).resolve()
        if ip.exists():
            args += ["-I", str(ip)]
    # 기존 v1 include 유지
    args += ["-I", str((repo_root / INC_DIR).resolve())]
    args += ["-I", str((repo_root / SRC_DIR).resolve())]
    return args

def make_header_wrapper(header_rel: Path, wrapper_path: Path):
    # 헤더(.h)는 단독으로 파싱하면 thread_t/msg_t 같은 타입 미정의로 깨지기 쉬움
    # 그래서 임시 .c(TU)를 만들어, ch.h를 먼저 include한 상태에서 해당 헤더를 include하도록 한다.
    code = f"""/* auto-generated wrapper TU for header parsing */
#include "ch.h"
#include "{header_rel.as_posix()}"
"""
    wrapper_path.write_text(code, encoding="utf-8")

def clang_ast_dump_json(parse_input: Path, out_json: Path, repo_root: Path):
    cmd = [CLANG] + common_clang_args(repo_root) + [str(parse_input)]
    rc, out, err = run(cmd)
    if rc != 0:
        return False, err.strip()
    out_json.write_text(out, encoding="utf-8")
    return True, ""

def main():
    repo_root = Path(".").resolve()
    out_root = Path(OUT_DIR).resolve() / "ast_json"
    wrap_root = Path(OUT_DIR).resolve() / "tu_wrappers"
    out_root.mkdir(parents=True, exist_ok=True)
    wrap_root.mkdir(parents=True, exist_ok=True)

    targets = []
    for d in TARGET_DIRS:
        base = repo_root / d
        if not base.exists():
            continue
        for p in base.rglob("*"):
            if p.is_file() and p.suffix in EXTS:
                targets.append(p)

    manifest = {
        "repo_root": str(repo_root),
        "targets": [],
        "errors": []
    }

    for p in sorted(targets):
        rel = p.relative_to(repo_root)
        h = file_hash(p)
        out_json = out_root / (str(rel).replace("/", "__") + f".{h}.ast.json")

        # 핵심 수정: .h 는 wrapper TU(.c)를 만들어 그걸 clang 입력으로 사용
        if p.suffix == ".h":
            wrapper = wrap_root / (str(rel).replace("/", "__") + f".{h}.tu.c")
            make_header_wrapper(rel, wrapper)
            parse_input = wrapper
            kind = "header_wrapper"
        else:
            parse_input = p
            kind = "source"

        ok, msg = clang_ast_dump_json(parse_input, out_json, repo_root)

        item = {
            "path": str(rel),
            "hash": h,
            "kind": kind,
            "parse_input": str(parse_input.relative_to(repo_root)) if parse_input.is_relative_to(repo_root) else str(parse_input),
            "ast_json": str(out_json.relative_to(repo_root))
        }

        if ok:
            manifest["targets"].append(item)
        else:
            item["error"] = msg
            manifest["errors"].append(item)

    (Path(OUT_DIR)/"ast_manifest.json").write_text(json.dumps(manifest, indent=2), encoding="utf-8")
    print(f"[OK] AST dump complete. success={len(manifest['targets'])}, failed={len(manifest['errors'])}")
    if manifest["errors"]:
        print("[WARN] Some files failed to parse; see ast_manifest.json")

if __name__ == "__main__":
    main()
'''
        writeFile file: "${OUT_DIR}/build_graphs.py", text: '''\
#!/usr/bin/env python3
import os, json, re
from pathlib import Path
from typing import Dict, Any, List, Set, Tuple

OUT_DIR = os.environ.get("OUT_DIR", "ast_out")

def load_json(p: Path) -> Any:
    return json.loads(p.read_text(encoding="utf-8", errors="ignore"))

def walk(node: Any):
    if isinstance(node, dict):
        yield node
        for k, v in node.items():
            if isinstance(v, (dict, list)):
                yield from walk(v)
    elif isinstance(node, list):
        for x in node:
            yield from walk(x)

def extract_functions(ast: Dict[str, Any]) -> Dict[str, Dict[str, Any]]:
    funcs: Dict[str, Dict[str, Any]] = {}
    for n in walk(ast):
        if not isinstance(n, dict):
            continue
        if n.get("kind") != "FunctionDecl":
            continue
        name = n.get("name")
        if not name:
            continue
        inner = n.get("inner", [])
        has_body = any(isinstance(x, dict) and x.get("kind") in ("CompoundStmt", "CXXTryStmt") for x in inner)

        params = []
        for x in inner:
            if isinstance(x, dict) and x.get("kind") == "ParmVarDecl":
                pn = x.get("name")
                if pn:
                    params.append(pn)

        funcs.setdefault(name, {"name": name, "params": params, "returns": [], "calls": [], "has_body": has_body})

        for x in walk(n):
            if not isinstance(x, dict):
                continue
            if x.get("kind") == "CallExpr":
                callee = None
                args = []
                inn = x.get("inner", [])
                if inn:
                    for y in walk(inn[0]):
                        if isinstance(y, dict) and y.get("kind") == "DeclRefExpr":
                            callee = y.get("referencedDecl", {}).get("name") or y.get("name")
                            if callee:
                                break
                if len(inn) >= 2:
                    for a in inn[1:]:
                        token = extract_expr_token(a)
                        if token:
                            args.append(token)
                if callee:
                    funcs[name]["calls"].append({"callee": callee, "args": args})

            if x.get("kind") == "ReturnStmt":
                inn = x.get("inner", [])
                if not inn:
                    continue
                token = extract_expr_token(inn[0])
                if token:
                    funcs[name]["returns"].append(token)

    for f in funcs.values():
        f["returns"] = sorted(set([r for r in f["returns"] if r]))
    return funcs

def extract_expr_token(expr: Any) -> str:
    if not isinstance(expr, (dict, list)):
        return ""
    for n in walk(expr):
        if not isinstance(n, dict):
            continue
        k = n.get("kind")
        if k == "DeclRefExpr":
            nm = n.get("referencedDecl", {}).get("name") or n.get("name")
            if nm: return nm
        if k in ("IntegerLiteral", "FloatingLiteral"):
            v = n.get("value")
            if v is not None: return str(v)
        if k == "StringLiteral":
            v = n.get("value")
            if v is not None:
                s = str(v)
                if len(s) > 32: s = s[:32] + "…"
                return f"\\\"{s}\\\""
    return ""

def build_call_graph(all_funcs: Dict[str, Dict[str, Any]]) -> List[Tuple[str,str,List[str]]]:
    edges = []
    for caller, info in all_funcs.items():
        for c in info.get("calls", []):
            callee = c.get("callee")
            if not callee:
                continue
            edges.append((caller, callee, c.get("args", [])))
    return edges

def merge_ast_by_inlining(funcs: Dict[str, Dict[str, Any]]) -> Dict[str, Any]:
    MAX_DEPTH = 6
    def inline(fn: str, depth: int, path: Set[str]) -> Dict[str, Any]:
        base = funcs.get(fn, {"name": fn, "params": [], "returns": [], "calls": [], "has_body": False})
        node = {
            "name": fn,
            "params": base.get("params", []),
            "returns": base.get("returns", []),
            "has_body": base.get("has_body", False),
            "inlined_calls": []
        }
        if depth >= MAX_DEPTH:
            return node
        path2 = set(path)
        path2.add(fn)

        for call in base.get("calls", []):
            callee = call.get("callee")
            if not callee:
                continue
            if callee in path2:
                node["inlined_calls"].append({"callee": callee, "skipped": True, "reason": "cycle"})
                continue
            if callee not in funcs:
                node["inlined_calls"].append({"callee": callee, "skipped": True, "reason": "out_of_scope"})
                continue
            child = inline(callee, depth+1, path2)
            child["_callsite_args"] = call.get("args", [])
            node["inlined_calls"].append(child)
        return node

    merged = {}
    for fn in sorted(funcs.keys()):
        merged[fn] = inline(fn, 0, set())
    return merged

def write_csv(nodes: List[Dict[str, Any]], rels: List[Dict[str, Any]], nodes_path: Path, rels_path: Path):
    import csv
    nodes_path.parent.mkdir(parents=True, exist_ok=True)
    with nodes_path.open("w", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        w.writerow([":ID(Function)", "name", "params", "has_body"])
        for n in nodes:
            w.writerow([n["id"], n["name"], n.get("params",""), n.get("has_body", False)])

    with rels_path.open("w", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        w.writerow([":START_ID(Function)", ":END_ID(Function)", ":TYPE", "label"])
        for r in rels:
            w.writerow([r["start"], r["end"], r["type"], r.get("label","")])

def main():
    out = Path(OUT_DIR)
    manifest = load_json(out/"ast_manifest.json")
    repo_root = Path(manifest["repo_root"])
    targets = manifest["targets"]

    all_funcs: Dict[str, Dict[str, Any]] = {}
    for t in targets:
        ast_path = repo_root / t["ast_json"]
        try:
            ast = load_json(ast_path)
        except Exception:
            continue
        funcs = extract_functions(ast)
        for name, info in funcs.items():
            if name not in all_funcs:
                all_funcs[name] = info
            else:
                if (not all_funcs[name].get("has_body")) and info.get("has_body"):
                    all_funcs[name] = info

    merged = merge_ast_by_inlining(all_funcs)
    (out/"merged"/"merged_ast.json").write_text(json.dumps(merged, indent=2), encoding="utf-8")

    fn_id = {name: f"F_{i}" for i, name in enumerate(sorted(all_funcs.keys()), start=1)}

    nodes = []
    for name in sorted(all_funcs.keys()):
        info = all_funcs[name]
        nodes.append({
            "id": fn_id[name],
            "name": name,
            "params": ",".join(info.get("params", [])),
            "has_body": info.get("has_body", False)
        })

    rels = []
    call_edges = build_call_graph(all_funcs)

    for caller, callee, _args in call_edges:
        if caller not in fn_id or callee not in fn_id:
            continue

        callee_params = all_funcs.get(callee, {}).get("params", [])
        label_fwd = ",".join(callee_params) if callee_params else ""
        rels.append({"start": fn_id[caller], "end": fn_id[callee], "type": "CALLS", "label": label_fwd})

        rets = all_funcs.get(callee, {}).get("returns", [])
        if rets:
            label_back = "|".join(rets)
        else:
            label_back = "function return(void)"
        rels.append({"start": fn_id[callee], "end": fn_id[caller], "type": "RETURNS", "label": label_back})

    write_csv(nodes, rels, out/"neo4j"/"functions.nodes.csv", out/"neo4j"/"functions.rels.csv")

    dfd = {"nodes": [{"id": fn_id[n["name"]], "name": n["name"]} for n in nodes], "edges": rels}
    (out/"dfd"/"dfd.json").write_text(json.dumps(dfd, indent=2), encoding="utf-8")

    summary = {
        "function_count": len(nodes),
        "call_edge_count": sum(1 for r in rels if r["type"] == "CALLS"),
        "return_edge_count": sum(1 for r in rels if r["type"] == "RETURNS"),
        "note": "Calls/Returns edges are derived from clang AST dump; out-of-scope callees are skipped in merging."
    }
    (out/"summary.json").write_text(json.dumps(summary, indent=2), encoding="utf-8")
    print("[OK] Built merged_ast.json + Neo4j CSV + DFD")

if __name__ == "__main__":
    main()
'''
        sh """#!/usr/bin/env bash
          set -euxo pipefail
          chmod +x ${OUT_DIR}/extract_ast.py
          chmod +x ${OUT_DIR}/build_graphs.py
        """
      }
    }

    stage('Generate AST (per file)') {
      steps {
        sh """#!/usr/bin/env bash
          set -euxo pipefail
          ${PY} ${OUT_DIR}/extract_ast.py
        """
      }
    }

    stage('Merge AST + Build Neo4j/DFD artifacts') {
      steps {
        sh """#!/usr/bin/env bash
          set -euxo pipefail
          ${PY} ${OUT_DIR}/build_graphs.py
        """
      }
    }

    stage('Archive artifacts') {
      steps {
        archiveArtifacts artifacts: "${OUT_DIR}/**", fingerprint: true
        sh 'echo "Done."'
      }
    }
  }

  post {
    always {
      sh 'ls -R ast_out || true'
    }
  }
}
