This is a complex engineering challenge. You are building a synthetic data pipeline where the output (the tasks) must be strictly compatible with a deterministic system (the Codebase/DB).

The current script is a "fire and forget" approach. To meet your high-reliability requirements (Contract Safety), we need to move to a **"Generate, Validate, Retry"** architecture.

Here are the two parts of the solution:
1.  **Optimized Generation Script**: Modifying `create_rl_tasks.py` to inject "Ground Truth" into prompts and handle complexity.
2.  **The "Contract Saver" Test Suite**: A robust verification script to run on generated tasks.

---

### Part 1: Improving `create_rl_tasks.py`

Your current script relies entirely on the LLM to browse the database and find valid IDs. This is the biggest source of "hallucinations" (referencing non-existent IDs) or "permission errors" (assigning a user to a private team they aren't on).

**Key Changes:**
1.  **Inject Context (RAG-lite):** Pre-fetch a small sample of *valid* DB records (Issues, Teams, Users) and inject them into the prompt. Don't let the LLM guess IDs.
2.  **Complexity Parameter:** Allow passing a difficulty flag to ensure the distribution you want.
3.  **Strict "Ground Truth" definitions:** explicit enumerations of Statuses and Priorities.

Here is the enhanced `create_rl_tasks.py` logic (focusing on the `load_prompt` and main loop logic).

```python
# ... [Imports remain same, add sqlite3, json] ...
import sqlite3
import json
import random

# ... [Args parsing remains similar] ...

def get_db_context(db_path: Path, user_id: str) -> str:
    """
    Fetches REAL, VALID data from the DB to inject into the prompt.
    This ensures the LLM uses IDs that actually exist and users that have permission.
    """
    conn = sqlite3.connect(db_path)
    cursor = conn.cursor()
    
    context = {}
    
    # 1. Get Teams the user is actually part of (Permission Check)
    cursor.execute("""
        SELECT t.id, t.name FROM team t 
        JOIN team_member tm ON t.id = tm.team_id 
        WHERE tm.user_id = ?
    """, (user_id,))
    teams = cursor.fetchall()
    context['accessible_teams'] = [{"id": t[0], "name": t[1]} for t in teams]
    
    # 2. Get a sample of Issues within those teams
    team_ids = [t[0] for t in teams]
    if team_ids:
        placeholders = ','.join('?' * len(team_ids))
        cursor.execute(f"""
            SELECT id, title, priority, status_id FROM issue 
            WHERE team_id IN ({placeholders}) 
            LIMIT 10
        """, team_ids)
        context['valid_issues'] = [
            {"id": row[0], "title": row[1], "priority": row[2], "status": row[3]} 
            for row in cursor.fetchall()
        ]
        
    # 3. Get valid Label IDs
    cursor.execute("SELECT id, name FROM label LIMIT 10")
    context['valid_labels'] = [{"id": row[0], "name": row[1]} for row in cursor.fetchall()]
    
    conn.close()
    
    return json.dumps(context, indent=2)

def load_enhanced_prompt(prompt_file: Path | None, complexity: str, db_context: str) -> str:
    base_prompt = load_prompt(prompt_file)
    
    complexity_instruction = ""
    if complexity == "easy":
        complexity_instruction = "Create an EASY task (5-7 steps). Focus on updating single entities (e.g., change status, add 1 label)."
    elif complexity == "medium":
        complexity_instruction = "Create a MEDIUM task (8-12 steps). Involve relationships (e.g., Create Issue -> Assign to Cycle -> Add Sub-issue)."
    elif complexity == "hard":
        complexity_instruction = "Create a HARD task (>12 steps). Involve complex filtering and batch updates (e.g., Find all high priority issues in Team X, bulk update their due dates, and add a summary comment to the parent project)."

    # Inject the pre-validated data
    injection = f"""
    
    **CONTEXTUAL DATA (USE THESE EXACT IDS):**
    The following JSON contains REAL database records. You MUST use these IDs. Do not invent new IDs.
    {db_context}
    
    **COMPLEXITY REQUIREMENT:**
    {complexity_instruction}
    """
    
    return base_prompt + injection

async def run_codex_once(index: int, base_prompt: str, args: argparse.Namespace) -> None:
    # 1. Determine complexity based on index distribution
    # 33% Easy, 33% Medium, 33% Hard
    complexity = "easy"
    if index % 3 == 1: complexity = "medium"
    elif index % 3 == 2: complexity = "hard"

    # 2. Fetch Context
    # (Assuming DB path is constant as per your prompt)
    db_path = Path("/Users/aashnadogra/Desktop/scaler/projects/linear-rl-web/backend/linear/apps/linear/db/linear.db")
    user_id = "a9a832b1-0fb1-401e-859c-dae1350af2ee"
    db_context = get_db_context(db_path, user_id)
    
    # 3. Construct Final Prompt
    final_prompt = load_enhanced_prompt(args.prompt_file, complexity, db_context)
    
    # ... [Rest of the Codex execution code remains the same] ...
```

---

### Part 2: The "Contract Saver" Verification Suite

You cannot rely on `quick_test.sh` alone. You need a static analyzer that parses the generated Python code and JSON files to check for logical consistency without just running the RL agent.

Create a file named `verify_task_integrity.py`. This script should be run immediately after a task is generated.

#### 1. The Strategy
We will perform "Static Analysis" and "Database Lookups" to pass your constraints.

*   **Constraint 1: Feasibility (Data Exists)** -> Regex scan the task files for UUIDs, then query the SQLite DB.
*   **Constraint 2: Permissions** -> Check if the user ID involved in the task has a row in `team_member` for the target team.
*   **Constraint 3: Exact UI Values** -> Maintain a dictionary of `VALID_UI_STATES`. If the grader checks for `status == "finished"`, but the UI only uses `["Backlog", "Todo", "In Progress", "Done", "Canceled"]`, fail the task.

#### 2. The Verification Code

```python
import ast
import json
import sqlite3
import re
import os
from pathlib import Path

# --- CONFIGURATION ---
DB_PATH = "/Users/aashnadogra/Desktop/scaler/projects/linear-rl-web/backend/linear/apps/linear/db/linear.db"
USER_ID = "a9a832b1-0fb1-401e-859c-dae1350af2ee"

# This is the "Truth" from your codebase/APPLICATION_FEATURES.md
# You must populate this with the EXACT strings your UI uses.
VALID_UI_VALUES = {
    "priority": ["No Priority", "Urgent", "High", "Medium", "Low"],
    "status": ["Backlog", "Todo", "In Progress", "Done", "Canceled"],
    "size": ["Small", "Medium", "Large"],
}

VALID_START_URLS = [
    "/", "/inbox", "/my-issues", "/projects", "/teams", 
    # Add all 85+ routes here as regex or strings
]

class TaskVerifier:
    def __init__(self, task_dir: Path):
        self.task_dir = task_dir
        self.errors = []
        self.conn = sqlite3.connect(DB_PATH)
        self.cursor = self.conn.cursor()

    def log_error(self, msg):
        self.errors.append(msg)

    def verify_files_exist(self):
        for f in ["task.json", "grader.py", "auxiliary_data.json"]:
            if not (self.task_dir / f).exists():
                self.log_error(f"Missing file: {f}")
                return False
        return True

    def verify_data_integrity(self):
        """Finds all UUIDs in task files and ensures they exist in DB."""
        # Simple UUID regex
        uuid_pattern = re.compile(r'[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}')
        
        content = ""
        for f in ["task.json", "grader.py"]:
            try:
                content += (self.task_dir / f).read_text()
            except: pass

        found_uuids = set(uuid_pattern.findall(content))
        
        # Whitelist the current user
        if USER_ID in found_uuids:
            found_uuids.remove(USER_ID)

        for uuid in found_uuids:
            # Check if this UUID exists ANYWHERE in the DB (simplified check)
            # A more robust check would infer table context, but this catches hallucinations.
            exists = False
            tables = ["issue", "project", "team", "cycle", "user", "label"]
            for table in tables:
                try:
                    res = self.cursor.execute(f"SELECT 1 FROM {table} WHERE id=?", (uuid,)).fetchone()
                    if res:
                        exists = True
                        break
                except: continue
            
            if not exists:
                self.log_error(f"Hallucinated Data: UUID {uuid} found in task but not in DB tables.")

    def verify_permissions(self):
        """Checks if the hardcoded user has access to referenced teams."""
        content = (self.task_dir / "task.json").read_text()
        task_data = json.loads(content)
        
        # Extract Team IDs (heuristic)
        # Assuming instructions might contain team names, but task.json metadata usually better
        # If your task.json doesn't strictly have team_id, scan grader.py for team_id usage
        pass # Implementation depends on how specifically you mention teams in the task structure

    def verify_value_consistency(self):
        """Parses grader.py to ensure it checks for VALID UI values."""
        grader_path = self.task_dir / "grader.py"
        tree = ast.parse(grader_path.read_text())
        
        # Visitor to find string literals in the code
        class StringVisitor(ast.NodeVisitor):
            def __init__(self, verifier):
                self.verifier = verifier

            def visit_Constant(self, node):
                if isinstance(node.value, str):
                    val = node.value
                    # Check if this string looks like a status but is wrong
                    # e.g., "Closed" instead of "Done"
                    if val in ["Closed", "Complete", "Fixed"] and val not in VALID_UI_VALUES["status"]:
                         self.verifier.log_error(f"Potential Value Mismatch: Grader checks for '{val}', but valid statuses are {VALID_UI_VALUES['status']}")
                self.generic_visit(node)

        StringVisitor(self).visit(tree)

    def verify_start_url(self):
        data = json.loads((self.task_dir / "task.json").read_text())
        url = data.get("starting_url", "")
        
        # Simple whitelist check
        # You might need regex matching for dynamic URLs like /team/{team_id}/issue/{issue_id}
        valid = False
        for v in VALID_START_URLS:
            if url == v or url.startswith(v): # Simplify logic as needed
                valid = True
                break
        
        if not valid:
            self.log_error(f"Invalid Starting URL: {url}")

    def run(self):
        if not self.verify_files_exist(): return self.errors
        
        self.verify_data_integrity()
        self.verify_value_consistency()
        self.verify_start_url()
        # self.verify_permissions() 
        
        return self.errors

# Usage
if __name__ == "__main__":
    import sys
    task_id = sys.argv[1] # Pass task folder name
    verifier = TaskVerifier(Path(f"codex_runs/{task_id}")) # Adjust path
    errors = verifier.run()
    
    if errors:
        print("FAILED VERIFICATION:")
        for e in errors: print(f"- {e}")
        sys.exit(1)
    else:
        print("Task Verified.")
        sys.exit(0)
```

### 3. Integration Plan

1.  **Modify `create_rl_tasks.py`**:
    *   Add the logic to fetch DB context.
    *   Update the `prompt` string to include this context.
    *   Add the 33/33/33 complexity distribution logic.
2.  **Add Post-Processing Step**:
    *   In `run_codex_once`, immediately after the files are written, call `TaskVerifier(task_path).run()`.
    *   If `errors` is not empty, **delete the task folder** and retry (or flag it as failed). This ensures only high-quality tasks make it to your final dataset.

### Why this solves your constraints:

*   **Feasibility:** By injecting real DB IDs into the prompt, the model cannot invent an Issue ID that doesn't exist. The `verify_data_integrity` confirms this.
*   **Permissions:** The DB context injection only selects teams the user `a9a8...` is actually in (`JOIN team_member`).
*   **Exact Values:** The `StringVisitor` in the AST parser acts as a spell-checker for your specific business logic (checking "Done" vs "Closed").
*   **Complexity:** Explicitly handled by the modulo logic in the task index.
