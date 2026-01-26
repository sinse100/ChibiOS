pipeline {
  agent any

  triggers {
    githubPush()
  }

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
  }

  environment {
    SRC_DIR  = "os/rt/src"
    INC_DIR  = "os/rt/include"
    OUT_DIR  = "artifacts"
    PY       = "python3"
    CLANG    = "clang"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Prepare workspace') {
      steps {
        sh '''
          set -eux
          mkdir -p "${OUT_DIR}"/{ast_json,neo4j_ast,neo4j_dfd,logs}
        '''
      }
    }

    stage('Generate AST JSON per C file') {
      steps {
        // Python script that:
        // - finds .c/.h in SRC_DIR/INC_DIR
        // - runs clang AST dump json for each .c (and optionally headers if you want)
        // - outputs JSON files under OUT_DIR/ast_json
        writeFile file: 'gen_ast_json.py', text: '''
#!/usr/bin/env python3
import os, sys, json, subprocess, hashlib
from pathlib import Path

SRC_DIR = os.environ.get("SRC_DIR", "os/rt/src")
INC_DIR = os.environ.get("INC_DIR", "os/rt/include")
OUT_DIR = os.environ.get("OUT_DIR", "artifacts")
CLANG   = os.environ.get("CLANG", "clang")

AST_OUT = Path(OUT_DIR) / "ast_json"
LOG_OUT = Path(OUT_DIR) / "logs"
AST_OUT.mkdir(parents=True, exist_ok=True)
LOG_OUT.mkdir(parents=True, exist_ok=True)

def sha1(s: str) -> str:
    return hashlib.sha1(s.encode("utf-8", errors="ignore")).hexdigest()

def run_clang_ast_dump_json(c_path: Path) -> Path:
    out_json = AST_OUT / f"{c_path.name}.{sha1(str(c_path))}.ast.json"
    out_log  = LOG_OUT / f"{c_path.name}.{sha1(str(c_path))}.clang.log"

    # NOTE:
    # - We intentionally add include paths for both SRC_DIR and INC_DIR.
    # - If your project needs additional -D or -I, extend EXTRA_ARGS below.
    extra_args = []
    cmd = [
        CLANG,
        "-Xclang", "-ast-dump=json",
        "-fsyntax-only",
        "-I", INC_DIR,
        "-I", SRC_DIR,
        *extra_args,
        str(c_path)
    ]

    with out_log.open("w", encoding="utf-8") as lf:
        p = subprocess.run(cmd, stdout=subprocess.PIPE, stderr=lf)
    if p.returncode != 0:
        # If clang fails, we still keep the log and skip that file.
        return None

    out_json.write_bytes(p.stdout)
    return out_json

def iter_files(root: Path, exts):
    for p in root.rglob("*"):
        if p.is_file() and p.suffix.lower() in exts:
            yield p

def main():
    src_root = Path(SRC_DIR)
    inc_root = Path(INC_DIR)

    c_files = list(iter_files(src_root, {".c"}))
    if not c_files:
      print(f"[WARN] No .c files under {SRC_DIR}", file=sys.stderr)

    ok = 0
    for c in c_files:
        j = run_clang_ast_dump_json(c)
        if j:
            ok += 1
            print(f"[OK] AST JSON: {c} -> {j}")
        else:
            print(f"[SKIP] clang failed: {c} (see logs)", file=sys.stderr)

    print(f"[DONE] generated {ok}/{len(c_files)} AST json files")

if __name__ == "__main__":
    main()
'''
        sh '''
          set -eux
          chmod +x gen_ast_json.py
          "${PY}" gen_ast_json.py
        '''
      }
    }

    stage('Merge ASTs (caller <- callee) & Build DFD') {
      steps {
        // This script:
        // - parses clang AST json
        // - extracts function defs, builds call graph inside (SRC_DIR, INC_DIR)
        // - merges: expand callee AST nodes into caller merged tree (best-effort)
        // - skips callees not found within dirs (per your current algorithm constraint)
        // - produces Neo4J CSV for:
        //   (a) AST graph representation
        //   (b) DFD representation (nodes=functions, flows=call/return)
        writeFile file: 'merge_and_export_neo4j.py', text: '''
#!/usr/bin/env python3
import os, json, re, hashlib
from pathlib import Path
from typing import Any, Dict, List, Tuple, Set, Optional

SRC_DIR = os.environ.get("SRC_DIR", "os/rt/src")
INC_DIR = os.environ.get("INC_DIR", "os/rt/include")
OUT_DIR = os.environ.get("OUT_DIR", "artifacts")

AST_JSON_DIR = Path(OUT_DIR) / "ast_json"
NEO4J_AST_DIR = Path(OUT_DIR) / "neo4j_ast"
NEO4J_DFD_DIR = Path(OUT_DIR) / "neo4j_dfd"
NEO4J_AST_DIR.mkdir(parents=True, exist_ok=True)
NEO4J_DFD_DIR.mkdir(parents=True, exist_ok=True)

def sha1(s: str) -> str:
    return hashlib.sha1(s.encode("utf-8", errors="ignore")).hexdigest()

def stable_id(*parts: str) -> str:
    raw = "::".join(parts)
    return sha1(raw)

def is_under_dirs(path: str) -> bool:
    # clang nodes may include absolute paths; we match substrings.
    return (f"/{SRC_DIR}/" in path) or (f"/{INC_DIR}/" in path) or path.endswith(f"/{SRC_DIR}") or path.endswith(f"/{INC_DIR}")

def walk(node: Any):
    if isinstance(node, dict):
        yield node
        for k, v in node.items():
            if isinstance(v, (dict, list)):
                yield from walk(v)
    elif isinstance(node, list):
        for it in node:
            yield from walk(it)

def get_loc_file(n: Dict[str, Any]) -> Optional[str]:
    loc = n.get("loc") or {}
    file = loc.get("file")
    if isinstance(file, str):
        return file
    return None

def extract_functions(ast_root: Dict[str, Any]) -> List[Dict[str, Any]]:
    funcs = []
    for n in walk(ast_root):
        if not isinstance(n, dict): 
            continue
        if n.get("kind") == "FunctionDecl":
            # In clang AST json, function definitions often have "isImplicit" false and have "inner" etc.
            # We prefer nodes that appear to be definitions (have body in 'inner' with CompoundStmt)
            name = n.get("name")
            if not name:
                continue
            funcs.append(n)
    return funcs

def is_function_definition(fn: Dict[str, Any]) -> bool:
    # heuristic: has 'inner' containing 'CompoundStmt'
    inn = fn.get("inner", [])
    if not isinstance(inn, list): 
        return False
    for c in inn:
        if isinstance(c, dict) and c.get("kind") == "CompoundStmt":
            return True
    return False

def extract_calls(fn: Dict[str, Any]) -> List[Dict[str, Any]]:
    calls = []
    for n in walk(fn):
        if not isinstance(n, dict): 
            continue
        if n.get("kind") == "CallExpr":
            calls.append(n)
    return calls

def callee_name_from_call(call: Dict[str, Any]) -> Optional[str]:
    # CallExpr typically has 'inner' with 'DeclRefExpr' or 'MemberExpr' etc.
    inn = call.get("inner", [])
    if not isinstance(inn, list): 
        return None
    # find first DeclRefExpr with referencedDecl
    for x in inn:
        if isinstance(x, dict) and x.get("kind") == "DeclRefExpr":
            ref = x.get("referencedDecl", {})
            if isinstance(ref, dict):
                n = ref.get("name")
                if isinstance(n, str):
                    return n
    # sometimes callee is a MemberExpr (obj->method). For now skip.
    return None

def callee_args_from_call(call: Dict[str, Any]) -> List[str]:
    # We want callee parameter *names* for caller->callee flow label,
    # but we may not have the declaration here. So we fallback to argument tokens.
    # Later we will use the callee FunctionDecl's param names if available.
    args = []
    inn = call.get("inner", [])
    if not isinstance(inn, list):
        return args
    # In clang json, first inner is callee, following inners are args
    for x in inn[1:]:
        if not isinstance(x, dict): 
            continue
        # try to capture identifiers/literals
        k = x.get("kind")
        if k == "DeclRefExpr":
            nm = x.get("name") or (x.get("referencedDecl", {}) or {}).get("name")
            if isinstance(nm, str):
                args.append(nm)
        elif k in ("IntegerLiteral", "FloatingLiteral", "StringLiteral", "CharacterLiteral"):
            val = x.get("value")
            if val is None:
                val = k
            args.append(str(val))
        else:
            # fallback: kind name
            args.append(k)
    return args

def function_params(fn: Dict[str, Any]) -> List[str]:
    params = []
    for n in fn.get("inner", []) or []:
        if isinstance(n, dict) and n.get("kind") == "ParmVarDecl":
            nm = n.get("name")
            if isinstance(nm, str):
                params.append(nm)
    return params

def function_return_tokens(fn: Dict[str, Any]) -> List[str]:
    # Find ReturnStmt and capture simple return expr identifiers/literals; otherwise "function return(void)" if no return.
    returns = []
    has_return_stmt = False
    for n in walk(fn):
        if not isinstance(n, dict):
            continue
        if n.get("kind") == "ReturnStmt":
            has_return_stmt = True
            inn = n.get("inner", [])
            if isinstance(inn, list) and inn:
                x = inn[0]
                if isinstance(x, dict):
                    k = x.get("kind")
                    if k == "DeclRefExpr":
                        nm = x.get("name") or (x.get("referencedDecl", {}) or {}).get("name")
                        returns.append(str(nm) if nm else "return")
                    elif k in ("IntegerLiteral","FloatingLiteral","StringLiteral","CharacterLiteral"):
                        returns.append(str(x.get("value", k)))
                    else:
                        returns.append(k)
            else:
                # return;
                returns.append("function return(void)")
    if not has_return_stmt:
        # likely void or no explicit return
        returns.append("function return(void)")
    return returns[:1]  # keep one representative token

class FuncIndex:
    def __init__(self):
        # name -> list of defs (overloads are rare in C; but static funcs with same name could exist)
        self.defs: Dict[str, List[Dict[str, Any]]] = {}
        self.meta: Dict[str, Dict[str, Any]] = {}  # func_uid -> {name,file,is_def}
    def add(self, fn: Dict[str, Any], origin_file: str):
        name = fn.get("name")
        if not isinstance(name, str): 
            return
        uid = stable_id(origin_file, name, str(fn.get("id","")))
        self.meta[uid] = {"name": name, "file": origin_file, "is_def": is_function_definition(fn)}
        self.defs.setdefault(name, []).append({"uid": uid, "node": fn, "file": origin_file})

    def get_best_def(self, name: str) -> Optional[Dict[str, Any]]:
        cands = self.defs.get(name, [])
        if not cands:
            return None
        # prefer definitions under target dirs
        defs = [c for c in cands if self.meta[c["uid"]]["is_def"]]
        if defs:
            return defs[0]
        # else fallback to any decl
        return cands[0]

def load_all_asts() -> List[Tuple[str, Dict[str, Any]]]:
    asts = []
    for p in AST_JSON_DIR.glob("*.ast.json"):
        try:
            data = json.loads(p.read_text(encoding="utf-8", errors="ignore"))
            asts.append((str(p), data))
        except Exception as e:
            print(f"[WARN] Failed to parse {p}: {e}")
    return asts

def build_index(asts: List[Tuple[str, Dict[str, Any]]]) -> FuncIndex:
    idx = FuncIndex()
    for src, root in asts:
        # try to find the original file path from the AST (translation unit)
        # fallback: src filename
        origin = None
        for n in walk(root):
            if isinstance(n, dict) and "loc" in n and isinstance(n["loc"], dict):
                f = n["loc"].get("file")
                if isinstance(f, str):
                    origin = f
                    break
        if origin is None:
            origin = src
        for fn in extract_functions(root):
            of = get_loc_file(fn) or origin
            idx.add(fn, of)
    return idx

def merge_ast(caller_fn: Dict[str, Any], idx: FuncIndex, visited: Set[str]) -> Dict[str, Any]:
    # Deep-ish copy via json roundtrip (simple)
    merged = json.loads(json.dumps(caller_fn))

    calls = extract_calls(merged)
    for call in calls:
        callee = callee_name_from_call(call)
        if not callee:
            continue
        key = f"{merged.get('name','?')}->{callee}"
        if key in visited:
            continue
        visited.add(key)

        cand = idx.get_best_def(callee)
        if not cand:
            # current version constraint: cannot resolve -> skip
            continue

        callee_node = cand["node"]
        callee_file = cand["file"]

        # Only merge if callee appears to be within our target dirs (best-effort)
        if isinstance(callee_file, str) and (SRC_DIR in callee_file or INC_DIR in callee_file or is_under_dirs(callee_file)):
            # recurse
            callee_merged = merge_ast(callee_node, idx, visited)

            # Attach to call node for traceability
            call.setdefault("mergedCallee", {})
            call["mergedCallee"] = {
                "name": callee,
                "file": callee_file,
                "ast": callee_merged
            }
        else:
            # out-of-scope -> skip
            continue

    return merged

def export_neo4j_ast_graph(merged_funcs: List[Dict[str, Any]]):
    # Graph model:
    # - (:ASTNode {id, kind, name, file}) for each node in merged trees
    # - (parent)-[:HAS_CHILD]->(child)
    # - (func)-[:CONTAINS]->(node)
    nodes_csv = NEO4J_AST_DIR / "ast_nodes.csv"
    rels_csv  = NEO4J_AST_DIR / "ast_rels.csv"

    # headers suitable for neo4j-admin import or LOAD CSV
    nodes_csv.write_text(":ID,:LABEL,kind,name,file\\n", encoding="utf-8")
    rels_csv.write_text(":START_ID,:END_ID,:TYPE\\n", encoding="utf-8")

    seen: Set[str] = set()

    def node_id(n: Dict[str, Any]) -> str:
        kind = str(n.get("kind",""))
        name = str(n.get("name",""))
        file = str(get_loc_file(n) or "")
        # include clang internal id if present for stability
        cid = str(n.get("id",""))
        return stable_id(kind, name, file, cid, json.dumps(n.get("range",{}), sort_keys=True))

    def add_node(n: Dict[str, Any], label: str="ASTNode"):
        nid = node_id(n)
        if nid in seen:
            return nid
        seen.add(nid)
        kind = str(n.get("kind","")).replace(",", " ")
        name = str(n.get("name","")).replace(",", " ")
        file = str(get_loc_file(n) or "").replace(",", " ")
        nodes_csv.write_text("", encoding="utf-8")  # ensure file exists
        with nodes_csv.open("a", encoding="utf-8") as f:
            f.write(f"{nid},{label},{kind},{name},{file}\\n")
        return nid

    def add_rel(a: str, b: str, typ: str):
        with rels_csv.open("a", encoding="utf-8") as f:
            f.write(f"{a},{b},{typ}\\n")

    for mf in merged_funcs:
        func_label = "Function"
        fid = add_node(mf, func_label)

        # walk tree and build edges
        def rec(parent: Optional[Dict[str, Any]], cur: Any):
            if isinstance(cur, dict):
                cid = add_node(cur, "ASTNode")
                if parent is not None:
                    pid = add_node(parent, "ASTNode")
                    add_rel(pid, cid, "HAS_CHILD")
                else:
                    add_rel(fid, cid, "CONTAINS")

                for k, v in cur.items():
                    if isinstance(v, (dict, list)):
                        rec(cur, v)
            elif isinstance(cur, list):
                for it in cur:
                    rec(parent, it)

        rec(None, mf)

    print(f"[OK] Neo4J AST CSV: {nodes_csv}, {rels_csv}")

def export_neo4j_dfd(merged_funcs: List[Dict[str, Any]], idx: FuncIndex):
    # DFD model:
    # - (:Function {id, name, file})
    # - (:Function)-[:CALL {flow}]->(:Function)
    # - (:Function)-[:RETURN {flow}]->(:Function)
    #
    # flow naming rules per your spec:
    # - caller -> callee flow name: callee parameter names (if available); else fallback to call arg tokens
    # - callee -> caller flow name: return value token; if none -> "function return(void)"
    nodes_csv = NEO4J_DFD_DIR / "dfd_nodes.csv"
    rels_csv  = NEO4J_DFD_DIR / "dfd_rels.csv"
    nodes_csv.write_text(":ID,:LABEL,name,file\\n", encoding="utf-8")
    rels_csv.write_text(":START_ID,:END_ID,:TYPE,flow\\n", encoding="utf-8")

    func_ids: Dict[str, str] = {}  # name->id (best-effort)

    def upsert_func(fn: Dict[str, Any]) -> str:
        name = str(fn.get("name",""))
        file = str(get_loc_file(fn) or "")
        fid = stable_id("func", name, file)
        key = f"{name}@@{file}"
        if key in func_ids:
            return func_ids[key]
        func_ids[key] = fid
        with nodes_csv.open("a", encoding="utf-8") as f:
            f.write(f"{fid},Function,{name.replace(',', ' ')},{file.replace(',', ' ')}\\n")
        return fid

    def add_rel(a: str, b: str, typ: str, flow: str):
        flow = (flow or "").replace(",", " ")
        with rels_csv.open("a", encoding="utf-8") as f:
            f.write(f"{a},{b},{typ},{flow}\\n")

    # Build DFD edges from merged AST call sites
    for caller in merged_funcs:
        caller_id = upsert_func(caller)
        caller_name = str(caller.get("name",""))

        for call in extract_calls(caller):
            callee = callee_name_from_call(call)
            if not callee:
                continue

            cand = idx.get_best_def(callee)
            if not cand:
                # unresolved -> skip
                continue

            callee_node = cand["node"]
            callee_file = cand["file"]
            # enforce current constraint: only merge/dfd edges for in-scope targets
            if not (isinstance(callee_file, str) and (SRC_DIR in callee_file or INC_DIR in callee_file or is_under_dirs(callee_file))):
                continue

            callee_id = upsert_func(callee_node)

            # caller -> callee: use callee param names if possible
            params = function_params(callee_node)
            if params:
                flow_call = " ".join(params)
            else:
                flow_call = " ".join(callee_args_from_call(call))
            if not flow_call.strip():
                flow_call = "(no-params)"
            add_rel(caller_id, callee_id, "CALL", flow_call)

            # callee -> caller: return token
            ret_tokens = function_return_tokens(callee_node)
            flow_ret = ret_tokens[0] if ret_tokens else "function return(void)"
            add_rel(callee_id, caller_id, "RETURN", flow_ret)

    print(f"[OK] Neo4J DFD CSV: {nodes_csv}, {rels_csv}")

def main():
    asts = load_all_asts()
    if not asts:
        print("[ERROR] No AST JSON files found. Run gen_ast_json stage first.")
        return

    idx = build_index(asts)

    # Choose "root functions" = all function definitions found in-scope,
    # then merge each one (caller-expansion).
    merged_funcs = []
    for name, cands in idx.defs.items():
        # pick best definition as a root if it is a definition and in-scope
        best = idx.get_best_def(name)
        if not best:
            continue
        fn = best["node"]
        file = best["file"]
        if not is_function_definition(fn):
            continue
        if not (isinstance(file, str) and (SRC_DIR in file or INC_DIR in file or is_under_dirs(file))):
            continue

        visited = set()
        mf = merge_ast(fn, idx, visited)
        merged_funcs.append(mf)

    # Export Neo4J formats
    export_neo4j_ast_graph(merged_funcs)
    export_neo4j_dfd(merged_funcs, idx)

    # Also store merged AST json (optional, but useful)
    merged_out = Path(OUT_DIR) / "merged_ast.json"
    merged_out.write_text(json.dumps(merged_funcs, ensure_ascii=False, indent=2), encoding="utf-8")
    print(f"[OK] merged AST JSON saved: {merged_out}")

if __name__ == "__main__":
    main()
'''
        sh '''
          set -eux
          chmod +x merge_and_export_neo4j.py
          "${PY}" merge_and_export_neo4j.py
        '''
      }
    }

    stage('Archive artifacts') {
      steps {
        archiveArtifacts artifacts: 'artifacts/**', fingerprint: true
      }
    }
  }

  post {
    always {
      sh 'ls -R artifacts || true'
    }
    failure {
      echo "Build failed. Check artifacts/logs/*.clang.log for clang failures."
    }
  }
}
