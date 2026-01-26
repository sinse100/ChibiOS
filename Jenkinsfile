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
        sh """
          set -euxo pipefail
          mkdir -p ${OUT_DIR}/ast_json
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

def clang_ast_dump_json(src: Path, out_json: Path):
    # -fsyntax-only 로 컴파일 산출물 없이 AST만
    # 헤더는 TU로 parsing하기 까다로워서, 여기서는 .h도 일단 AST dump는 시도하되
    # 실사용은 주로 .c TU 기준(함수 body 포함)으로 병합/DFD를 만들게 됨.
    cmd = [
        CLANG,
        "-fsyntax-only",
        "-Xclang", "-ast-dump=json",
        "-I", str(Path(INC_DIR).resolve()),
        "-I", str(Path(SRC_DIR).resolve()),
        str(src),
    ]
    rc, out, err = run(cmd)
    if rc != 0:
        return False, err.strip()
    out_json.write_text(out, encoding="utf-8")
    return True, ""

def main():
    repo_root = Path(".").resolve()
    out_root = Path(OUT_DIR).resolve() / "ast_json"
    out_root.mkdir(parents=True, exist_ok=True)

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
        ok, msg = clang_ast_dump_json(p, out_json)
        item = {"path": str(rel), "hash": h, "ast_json": str(out_json.relative_to(repo_root))}
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

def get_loc_key(d: Dict[str, Any]) -> str:
    loc = d.get("loc") or d.get("range", {}).get("begin")
    if not loc: return ""
    # loc may contain file/line/col
    f = loc.get("file", "")
    line = loc.get("line", "")
    col = loc.get("col", "")
    return f"{f}:{line}:{col}"

def extract_functions(ast: Dict[str, Any]) -> Dict[str, Dict[str, Any]]:
    """
    반환: { funcName: { 'name', 'params':[...], 'returns':[...], 'calls':[ {callee, args:[...]} ] } }
    - 함수 정의(FunctionDecl with 'inner' including CompoundStmt)가 주 대상
    """
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
        if not has_body:
            # 선언만 있는 경우(특히 헤더)도 params 추출은 가능하지만, return/call은 제한됨
            pass

        # params
        params = []
        for x in inner:
            if isinstance(x, dict) and x.get("kind") == "ParmVarDecl":
                pn = x.get("name")
                if pn:
                    params.append(pn)

        funcs.setdefault(name, {"name": name, "params": params, "returns": [], "calls": [], "has_body": has_body})

        # calls + returns (only meaningful if body exists)
        # calls: CallExpr -> referenced function name is often in 'inner' via DeclRefExpr
        for x in walk(n):
            if not isinstance(x, dict): 
                continue
            if x.get("kind") == "CallExpr":
                callee = None
                args = []
                inn = x.get("inner", [])
                # first inner often is callee expr
                if inn:
                    # find DeclRefExpr or MemberExpr inside first child
                    for y in walk(inn[0]):
                        if isinstance(y, dict) and y.get("kind") == "DeclRefExpr":
                            callee = y.get("referencedDecl", {}).get("name") or y.get("name")
                            if callee:
                                break
                # remaining inners are args (roughly)
                if len(inn) >= 2:
                    for a in inn[1:]:
                        # try extract variable/const tokens
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

    # dedup returns
    for f in funcs.values():
        f["returns"] = sorted(set([r for r in f["returns"] if r]))
    return funcs

def extract_expr_token(expr: Any) -> str:
    """
    반환값/인자 표현을 매우 보수적으로 단순화:
    - DeclRefExpr: 변수명
    - IntegerLiteral/FloatingLiteral/StringLiteral: 리터럴 값(가능하면 value)
    - UnaryOperator/BinaryOperator 등: 내부에서 첫 토큰만 잡거나, repr 대체
    """
    if not isinstance(expr, (dict, list)):
        return ""
    # walk shallowly
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
            # avoid huge strings
            v = n.get("value")
            if v is not None:
                s = str(v)
                if len(s) > 32: s = s[:32] + "…"
                return f"\\\"{s}\\\""
    # fallback: nothing found
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
    """
    '병합된 AST'를 실제 AST 노드 트리로 재구성하는 대신,
    Neo4j에서 시각화하기 쉬운 '함수-호출 트리(inlined)' 형태 JSON으로 산출.
    - 규칙: caller->callee 관계에서 callee가 funcs에 존재할 때만 inline, 아니면 skip
    - 순환 호출은 depth 제한으로 방지
    """
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
    # Neo4j-admin import 친화적으로 header 포함
    # node: :ID(Function),name,params,has_body
    # rel: :START_ID(Function),:END_ID(Function),:TYPE,label
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

    # 파일별 AST에서 함수 정보 수집
    all_funcs: Dict[str, Dict[str, Any]] = {}

    for t in targets:
        ast_path = repo_root / t["ast_json"]
        try:
            ast = load_json(ast_path)
        except Exception:
            continue
        funcs = extract_functions(ast)

        # 동일 이름 함수가 여러 TU에 나올 수 있음.
        # 여기서는 "body가 있는 쪽"을 우선(보수적으로) 채택
        for name, info in funcs.items():
            if name not in all_funcs:
                all_funcs[name] = info
            else:
                # prefer has_body True
                if (not all_funcs[name].get("has_body")) and info.get("has_body"):
                    all_funcs[name] = info

    # 2) AST 병합 (inlined JSON)
    merged = merge_ast_by_inlining(all_funcs)
    (out/"merged"/"merged_ast.json").write_text(json.dumps(merged, indent=2), encoding="utf-8")

    # 3) Neo4j import 형태(함수 노드/호출 엣지)
    # Node IDs
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

    # 4) DFD 생성 규칙 반영
    # - caller -> callee : callee의 파라미터명들(파라미터 명만)
    # - callee -> caller : 반환값(상수/변수명) / 없으면 "function return(void)"
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

    write_csv(
        nodes,
        rels,
        out/"neo4j"/"functions.nodes.csv",
        out/"neo4j"/"functions.rels.csv"
    )

    # 5) DFD JSON도 별도 산출(원하면 시각화 툴로도 가능)
    dfd = {
        "nodes": [{"id": fn_id[n["name"]], "name": n["name"]} for n in nodes],
        "edges": rels
    }
    (out/"dfd"/"dfd.json").write_text(json.dumps(dfd, indent=2), encoding="utf-8")

    # summary
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
        sh """
          chmod +x ${OUT_DIR}/extract_ast.py
          chmod +x ${OUT_DIR}/build_graphs.py
        """
      }
    }

    stage('Generate AST (per file)') {
      steps {
        sh """
          set -euxo pipefail
          ${PY} ${OUT_DIR}/extract_ast.py
        """
      }
    }

    stage('Merge AST + Build Neo4j/DFD artifacts') {
      steps {
        sh """
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
