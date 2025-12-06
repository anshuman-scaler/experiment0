This is a significant pivot in requirements. You are moving from "Generating Tasks" to **"Procedural Generation of Verified Workflows."**

To meet the constraint of "5-15 minute human completion time" while guaranteeing 100% feasibility, we cannot generate atomic actions ("Update Issue"). We must generate **Narratives** that thread through multiple valid states.

Here is the **Master Plan** for an environment-agnostic, rigorous Task Generation Pipeline.

---

### Phase 1: Context-Aware Introspection (The "World Builder")
**Input:** `App Name`, `DB Creds`, `Codebase Path`, `Features Doc`.

We don't just dump the schema. We must map the **Business Logic Constraints** that exist in the real world but not explicitly in the schema.

#### Step 1.1: Context-Dependent Value Extraction
*Problem:* Statuses are not global. Marketing Team might use `["Ideation", "Live"]`, Engineering uses `["Backlog", "Deployed"]`.
*   **The Fix:** We do not just run `SELECT DISTINCT`. We run **Grouped Discovery**.
    *   **Heuristic:** Identify "Partition Columns" (e.g., `team_id`, `project_id`, `department_id`).
    *   **Query:** `SELECT team_id, array_agg(DISTINCT status) FROM issues GROUP BY team_id`.
    *   **Output:** A `state_matrix.json` that maps `{ "Team_Engineering_ID": ["Backlog", "Done"], "Team_Marketing_ID": ["Ideation", "Live"] }`.
    *   **Usage:** If the LLM generates a task for the Marketing team, it is *forbidden* from using "Backlog".

#### Step 1.2: The Shadow Permission Graph
*Problem:* Who can see what?
*   **The Fix:** We parse the codebase (specifically looking for "Policy", "Permission", or "Middleware" files) to extract the **Visibility Rules**.
    *   *Rule Extraction:* If the code says `if (team.private && !user.isMember) return 403`, we translate this into a Python function `can_access(user, team)`.
    *   **Output:** `shadow_permissions.py` (Dynamically generated logic module).

---

### Phase 2: Definition of Complexity (The "Time" Constraint)
To hit your 5/10/15 minute targets, we classify tasks by **Search Space** and **Cognitive Load**, not just steps.

*   **Level 1: Easy (5+ mins, "The Investigator")**
    *   *Pattern:* **Search $\rightarrow$ Filter $\rightarrow$ Bulk Action**.
    *   *Example:* "Find all tickets submitted by 'Acme Corp' in Q1 that are still 'Open'. Mark them as 'High Priority' and add a comment asking for updates."
    *   *Why it takes time:* Human must build the filter, verify dates, select all, and act.
*   **Level 2: Medium (10+ mins, "The Coordinator")**
    *   *Pattern:* **Cross-Reference $\rightarrow$ Recursive Logic $\rightarrow$ Multi-Entity Update**.
    *   *Example:* "Check the 'Launch' project. For every incomplete task, find the assignee. If the assignee is out of office (check their status), reassign to their manager."
    *   *Why it takes time:* Requires navigating between Project $\rightarrow$ Task $\rightarrow$ User $\rightarrow$ User Profile $\rightarrow$ Task.
*   **Level 3: Hard (15+ mins, "The Cleaner")**
    *   *Pattern:* **Unstructured Data $\rightarrow$ Structured Action $\rightarrow$ Verification**.
    *   *Example:* "Read the latest PDF spec attached to the 'Epic'. Create sub-tasks for every requirement listed in the PDF that isn't yet tracked. Link them to the Epic."
    *   *Why it takes time:* Requires reading, comprehending, cross-checking existing data, and creating new entities.

---

### Phase 3: The Generation Pipeline (The Code)

This script implements the rigorous checks you requested: `verification.sql`, `manifest.json`, and the `Shadow Validator`.

#### The Architecture

```python
"""
Enterprise RL Task Generator - Feasibility First Architecture.
"""
import sqlite3
import json
import os
from typing import List, Dict, Any

class TaskPackage:
    def __init__(self):
        self.task_description = ""
        self.manifest = {}
        self.setup_script = ""      # Python script to seed DB
        self.verification_sql = ""  # SQL to check pre-reqs
        self.grader_script = ""     # Python grader
        self.complexity = ""

class ShadowValidator:
    """
    Simulates the Application Layer (UI/API) Logic.
    This is where we catch things the DB allows but the App forbids.
    """
    def __init__(self, db_conn, permission_logic_module):
        self.conn = db_conn
        self.perms = permission_logic_module # Injected module based on codebase analysis

    def check_feasibility(self, manifest: Dict, user_id: str) -> List[str]:
        errors = []
        cursor = self.conn.cursor()

        # 1. VISIBILITY CHECK (Can the user see the entities?)
        for entity in manifest['entities']:
            # Call the extracted permission logic
            # e.g. perms.can_view(user_id, entity_type, entity_id, cursor)
            if not self.perms.check_access(user_id, entity['type'], entity['id'], cursor):
                errors.append(f"Visibility Error: User {user_id} cannot access {entity['type']} {entity['id']}")

        # 2. STATE TRANSITION CHECK (Is the move from A to B valid?)
        for trans in manifest.get('transitions', []):
            # Fetch current state
            curr = cursor.execute(f"SELECT status FROM {trans['table']} WHERE id=?", (trans['id'],)).fetchone()[0]
            
            # Check against the Context-Aware State Matrix (generated in Phase 1)
            # e.g. Is 'Done' in the allowed states for this specific team?
            allowed_states = self.perms.get_allowed_states(trans['context_id'], cursor)
            if trans['to_state'] not in allowed_states:
                errors.append(f"State Error: '{trans['to_state']}' is not a valid state for this context. Valid: {allowed_states}")

        # 3. FIELD IMMUTABILITY CHECK
        for update in manifest.get('updates', []):
            if update['field'] in ['created_at', 'id']: # Example immutable fields
                errors.append(f"Immutable Error: UI does not allow changing '{update['field']}'")

        return errors

class TaskGenerator:
    def generate(self, complexity: str, theme: str, db_context: Dict) -> TaskPackage:
        # Prompt construction is critical here.
        # We enforce the "Recursive" requirement via the prompt.
        
        prompt = f"""
        Generate a {complexity} task for the theme '{theme}'.
        
        **CRITICAL REQUIREMENTS:**
        1. **Recursive Logic:** If the task involves "instructions", you MUST create a `setup.py` that inserts a comment/document containing those instructions.
        2. **Manifest:** You must output a JSON manifest listing every entity read or written.
        3. **SQL Verification:** Output a SQL query that returns TRUE only if all pre-requisites exist.
        
        **CONTEXT (Use ONLY these IDs):**
        {json.dumps(db_context)}
        
        **VALID STATES FOR THIS CONTEXT:**
        {json.dumps(db_context['valid_states'])}
        """
        
        # ... Call LLM ...
        # ... Parse Output ...
        return parsed_package

class FeasibilityEngine:
    """
    The Master Validator.
    """
    def __init__(self, db_path):
        self.db_path = db_path

    def validate_task(self, package: TaskPackage, user_id: str) -> bool:
        # 1. CLONE DB (Sandbox)
        sandbox_db = "/tmp/sandbox.db"
        shutil.copy(self.db_path, sandbox_db)
        conn = sqlite3.connect(sandbox_db)
        
        try:
            # 2. RUN SETUP SCRIPT
            # This seeds the recursive data (e.g., the comment saying "Delete Issue X")
            exec(package.setup_script, {'db_conn': conn})
            
            # 3. EXECUTE VERIFICATION SQL
            # This ensures the setup script actually worked
            cursor = conn.cursor()
            cursor.execute(package.verification_sql)
            if cursor.fetchone()[0] == 0:
                print("SQL Verification Failed: Pre-requisite data missing.")
                return False

            # 4. RUN SHADOW VALIDATOR (UI Constraints)
            # This checks permissions and transitions on the NOW SEEDED data
            validator = ShadowValidator(conn, permission_logic)
            errors = validator.check_feasibility(package.manifest, user_id)
            if errors:
                print("Shadow Validation Failed:", errors)
                return False

            # 5. ORACLE GRADER TEST (The "Doability" Check)
            # We do NOT just update the DB. We check if the *Final State* is reachable.
            # We assume the user performs the action successfully.
            # We simulate the DB update to the 'After' state defined in manifest.
            # Then we run the grader.
            
            self._apply_manifest_updates(conn, package.manifest)
            
            # Run the User's Grader
            score = self._run_grader(package.grader_script, conn)
            if score != 1.0:
                print(f"Grader Logic Error: Even with perfect state, grader returned {score}")
                return False

            return True

        finally:
            os.remove(sandbox_db)

    def _apply_manifest_updates(self, conn, manifest):
        # Applies the 'expected_after_state' to the DB
        # This is purely to test if the GRADER code is valid
        for change in manifest['changes']:
            conn.execute(f"UPDATE {change['table']} SET {change['field']} = ? WHERE id = ?", (change['value'], change['id']))
```

---

### The Plan of Action (Step-by-Step)

#### Step 1: Capability Analysis (Codebase Scanning)
We need to know what the UI *can* do.
*   **Action:** Run a script over the user-provided Codebase Path.
*   **Look for:**
    *   Router files (e.g., `routes.tsx`, `urls.py`) $\rightarrow$ Defines "Entry Points".
    *   Dropdown components $\rightarrow$ Defines "Hardcoded Lists".
    *   Middleware/Decorators (e.g., `@login_required`, `has_permission`) $\rightarrow$ Defines "Shadow Logic".
*   **Output:** `capabilities_matrix.json` and `shadow_permissions.py`.

#### Step 2: Contextual Data Sampling
We need data that fits together.
*   **Action:** Write a sampler that pulls **Clusters**.
*   **Logic:** Pick a User $\rightarrow$ Pick their Team $\rightarrow$ Pick that Team's Valid Statuses $\rightarrow$ Pick Issues in that Team.
*   **Why:** This prevents the "Assign User A to Team B's issue" hallucination.

#### Step 3: The Task Generation Loop
Iterate for $N$ tasks.

1.  **Select Complexity:** (e.g., Medium).
2.  **Select Theme:** (e.g., "Sprint Cleanup").
3.  **Fetch Cluster:** Get the User, Team, and Statuses for this specific iteration.
4.  **Prompt LLM:**
    *   Pass the Cluster.
    *   Pass the Constraints (Time > 10 mins).
    *   **Prompt Instruction:** "You must create a recursive task. Create a setup script that inserts a Meeting Note. The Task is to read that Meeting Note and implement the 3 action items listed inside it."
5.  **Receive Output:**
    *   `task.json` (The Prompt).
    *   `setup.py` (Seeds the Meeting Note).
    *   `verification.sql` (Checks: Does Meeting Note exist? Do the 3 referenced items exist?).
    *   `manifest.json` (Lists the 3 items and their permissions).
    *   `grader.py` (Checks if the 3 items are done).

#### Step 4: The Triple-Check Validation
1.  **Run `setup.py`** in Sandbox.
2.  **Run `verification.sql`**. If it returns `FALSE`, the LLM failed to seed the recursive data. **REJECT**.
3.  **Run `ShadowValidator`**. Check if the User has `write_access` to the 3 items listed in the Meeting Note. If not, **REJECT**.
4.  **Run Grader Simulation**. Update DB to finished state. Check if Grader == 1.0.

---

### Addressing Your Specific Concerns

1.  **"Validate unique subset of values":**
    *   Handled in **Phase 1.1**. We query `GROUP BY context_id` to map valid values per team/project. The Shadow Validator uses this specific map, not a global list.
2.  **"Task Hardness (5/10/15 mins)":**
    *   Handled in **Phase 2**. We enforce complexity via "Search Space" and "Indirection" (recursive tasks), requiring the user to read/search before acting.
3.  **"Apply Oracle / Feasibility":**
    *   Handled in **FeasibilityEngine**.
    *   We split the check: `ShadowValidator` proves *Feasibility* (UI allows it). `GraderSimulation` proves *Grader Correctness* (Code isn't buggy).
4.  **"Recursive Checks":**
    *   Handled by `verification.sql`. The LLM is forced to generate SQL that asserts the existence of the *instructions* (e.g., the comment or doc) required to start the task.

### The Manifesto for the LLM Prompt
To ensure the LLM complies, we append this **Manifesto** to the System Prompt:

> "You are an architect of a simulation.
> 1. **No Magic:** The user knows nothing. If they need to 'Close the ticket mentioned in the comment', YOU must write the `setup.py` to CREATE that comment first.
> 2. **Verification is Law:** You must write a SQL query (`verification.sql`) that proves your setup script worked. If this query returns 0 rows, your task fails.
> 3. **The Manifest:** You must list every ID you touch in `manifest.json`. We will verify the user has permission to touch them."

This strategy replaces "hope" with "compiled verification," ensuring the tasks are realistic, challenging, and strictly feasible.
