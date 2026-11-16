---
lab:
  title: 'Design stateful agentic loops with Microsoft Foundry Agent Service'
  description: 'Implement a bounded Agents v2 reflection loop, persistent conversation, and resumable branch for investment research.'
  duration: 45
  level: 400
  islab: true
  status: 'released'
---

# Design stateful agentic loops with Microsoft Foundry Agent Service

## Customer scenario

Contoso Capital needs an investment-research agent that retains multi-turn context, performs bounded self-review, reports completion explicitly, and supports alternate research branches without logging confidential conversation text.

## Lab scenario

You will migrate a supplied Agents v1 pattern to Agents v2, deploy the required Foundry resources, and capture conversation, reflection, response, and branch evidence.

By the end of this exercise, you will be able to:

- Map run-status handling to bounded response handling and exceptions.
- Implement visible planning and reflection summaries without exposing hidden reasoning.
- Maintain multi-turn state in a conversation.
- Fork a research path with `previous_response_id`.
- Explain agents, conversations, responses, and typed output items.
- Migrate v1 thread/run/tool concepts to `azure-ai-projects` 2.x.

> **Important:** The model deployment, Application Insights ingestion, and Log Analytics retention are billable. Use only the synthetic request in `assets`.

## Task 1: Prepare the lab

You need:

- [Python 3.11 or later](https://www.python.org/downloads/)
- [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) with [Bicep](https://learn.microsoft.com/azure/azure-resource-manager/bicep/install)
- [Azure Developer CLI](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd)
- [Visual Studio Code](https://code.visualstudio.com/download)
- An Azure subscription with model quota in a supported region
- Contributor and User Access Administrator on the target resource group, or equivalent preassigned Foundry data-plane access

**Clone and open the repository**

1. Clone the [lab source repository](https://github.com/MicrosoftLearning/mslearn-ai-multi-agents), or clone your fork:

```console
git clone https://github.com/MicrosoftLearning/mslearn-ai-multi-agents.git
```

2. Open the cloned repository in Visual Studio Code.

**Verify tools and authentication**

3. Open a PowerShell terminal in Visual Studio Code and run:

```powershell
cd Allfiles\01-contoso-capital-stateful-agentic-loops
az version
azd version
python --version
az account show --output table
```

4. Confirm that each command succeeds and that `az account show` displays the subscription you intend to use.
5. If needed, authenticate with `az login` and `azd auth login`, and then repeat the checks.

**Architecture checkpoint**

Review these components before editing:

| Component | What to locate |
|---|---|
| `infra/main.bicep` | Foundry resources, model deployment, and project connection for server-side traces |
| `src/main.py` | `build_agent_definition`, `run_reflection_cycle`, and `fork_with_previous_response` |
| `assets/agents-v1-loop.py.txt` | Legacy v1 comparison evidence; do not execute this file |

Before continuing, confirm that you can locate the bounded reflection loop, the alternate response branch, and the legacy pattern used only for comparison.

## Task 2: Build the virtual environment

1. From the lab directory, create and activate the virtual environment:

```powershell
./scripts/setup.ps1
. ./.venv/Scripts/Activate.ps1
```

> On macOS/Linux, run `bash scripts/setup.sh` and `source .venv/bin/activate` instead.

2. Confirm that `(.venv)` appears in the terminal prompt.

## Task 3: Deploy the Azure resources

**Set the deployment values**

1. Set `$azureRegion` to a region where the selected model is available.
> **Resource group:** If your lab environment provides a precreated resource group, set `$resourceGroupName` to its name. Otherwise, leave `$resourceGroupName` empty so the script creates a unique resource group in your subscription.

```powershell
$azureRegion = 'eastus2'
$resourceGroupName = ''
if ([string]::IsNullOrWhiteSpace($resourceGroupName)) {
  $resourceGroupName = "rg-lab01-$((New-Guid).Guid.Substring(0, 8))"
  az group create --name $resourceGroupName --location $azureRegion | Out-Null
}
```

**Validate and provision the infrastructure**

2. Validate Bicep, provision the resources, and export the environment:

```powershell
az bicep build --file infra/main.bicep
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'
azd env new lab01
azd env set AZURE_LOCATION $azureRegion
azd env set AZURE_RESOURCE_GROUP $resourceGroupName
azd env set FOUNDRY_MODEL_NAME gpt-5.4-mini
azd env set FOUNDRY_MODEL_CATALOG_NAME gpt-5.4-mini
azd env set FOUNDRY_MODEL_VERSION 2026-03-17
azd provision
azd env get-values | Out-File .env -Encoding utf8
Remove-Item Env:AZURE_DEV_USER_AGENT
```

3. If provisioning fails, inspect the first deployment error. For model or region availability errors, update the relevant environment value and rerun `azd provision`.

**Verify the generated environment**

4. Open `.env` and confirm that it contains:

- `FOUNDRY_PROJECT_ENDPOINT`
- `FOUNDRY_MODEL_NAME`
- `FOUNDRY_PROJECT_ID`
- `APPLICATIONINSIGHTS_RESOURCE_ID`
- `LOG_ANALYTICS_WORKSPACE_ID`

The Application Insights connection string is stored in the Foundry project connection and is not written to `.env`.

> **Network access for this lab:** The Bicep template enables the Foundry account's native public network access and sets the default network action to **Allow** so the local application can reach the project endpoint. Microsoft Entra authentication and Azure RBAC are still required. After deployment, confirm these settings on the Foundry account **Networking** page. Production environments should use an approved selected-network or private-endpoint design.

5. Do not add tokens or keys to `.env`.

## Task 4: Implement the solution

Each placeholder marks incomplete code. Copy each supplied snippet into its placeholder location, keep the `LAB PLACEHOLDER` comment, replace only the indicated incomplete line or block, and preserve the surrounding indentation.

**Define the prompt agent**

1. In `src/main.py`, find the placeholder in `build_agent_definition`:

```python
# LAB PLACEHOLDER 1: Replace this line with the Task 1 sample.
raise NotImplementedError("Complete build_agent_definition in Task 1")
```

```python
return PromptAgentDefinition(
  model=config.model_deployment,
  instructions=(
    "You are Contoso Capital's investment research agent. Use only the "
    "evidence provided in the request and conversation. Do not invent facts, "
    "recommend trades, or reveal hidden chain-of-thought. Return concise, "
    "visible planning and review summaries using exactly these headings:\n"
    "PLAN: <brief approach>\n"
    "REFLECTION: <evidence gaps or checks performed>\n"
    "STATUS: <COMPLETE or CONTINUE>\n"
    "ANSWER: <evidence-based answer or the next information needed>\n"
    "Use COMPLETE only when the request is fully answered from supplied "
    "evidence. Otherwise use CONTINUE."
  ),
)
```

This definition binds the agent to the deployed model and requests visible summaries plus a stable completion signal. Application code still controls the iteration limit.

**Implement the bounded reflection cycle**

2. In `run_reflection_cycle`, find:

```python
# LAB PLACEHOLDER 2: Replace this line with the Task 2 sample.
raise NotImplementedError("Complete run_reflection_cycle in Task 2")
```

```python
if max_iterations <= 0:
  raise ValueError("max_iterations must be greater than zero")

iterations: list[dict[str, Any]] = []
next_input = request
last_response_id = ""

for iteration_number in range(1, max_iterations + 1):
  response = openai_client.responses.create(
    conversation=conversation_id,
    input=next_input,
  )
print("\n========== AGENT RESPONSE ==========")
print(repr(response.output_text))
print("====================================\n")
visible_text = extract_text(response).strip()
status = "MISSING"

  for line in visible_text.splitlines():
    label, separator, value = line.partition(":")
    if separator and label.strip().strip("*").upper() == "STATUS":
      status = value.strip().strip("* .").upper()
      break

  last_response_id = response.id
  iterations.append(
    {
      "iteration": iteration_number,
      "response_id": response.id,
      "status": status,
      "visible_summary": visible_text,
    }
  )
  LOGGER.info(
    "Iteration completed iteration=%d response_id=%s status=%s",
    iteration_number,
    response.id,
    status,
  )

  if status == "COMPLETE":
    return {
      "iterations": iterations,
      "last_response_id": last_response_id,
    }

  next_input = (
    "Review the previous visible answer for unsupported claims, missing "
    "evidence, and incomplete coverage. Return a revised response with the "
    "required PLAN, REFLECTION, STATUS, and ANSWER headings."
  )

raise RuntimeError(
  f"Reflection cycle exhausted its {max_iterations}-iteration budget "
  "without STATUS: COMPLETE"
)
```

The loop keeps all iterations in one conversation, records response IDs and visible output, and stops after `max_iterations`. Logging is limited to iteration number, response ID, and status.

**Create an alternate response branch**

3. In `fork_with_previous_response`, find:

```python
# LAB PLACEHOLDER 3: Replace this line with the Task 3 sample.
raise NotImplementedError("Complete fork_with_previous_response in Task 3")
```

```python
if not previous_response_id:
  raise ValueError("previous_response_id is required")

return openai_client.responses.create(
  previous_response_id=previous_response_id,
  input=branch_request,
)
```

Because the call omits `conversation`, `previous_response_id` creates a separate branch without changing the original conversation.

**Record the v1-to-v2 migration**

4. Create `migration-notes.md` in the lab folder with this comparison:

```markdown
# Agents v1 to v2 migration notes

| Agents v1 construct | Agents v2 replacement |
|---|---|
| `AgentsClient` | `AIProjectClient` plus `get_openai_client(agent_name=agent.name)` |
| Shared project OpenAI endpoint plus `extra_body.agent_reference` | Agent-bound OpenAI client using the agent's dedicated endpoint |
| Thread | A project OpenAI `conversation` |
| Run creation and status polling | Synchronous `responses.create()` calls bounded by application code |
| `requires_action` run state | Typed response output items and tool-call handling when tools are configured |
| GUID agent ID | Stable agent name plus an immutable agent version |
| New thread for an alternate path | `previous_response_id` without a conversation ID |

The migration changes both the client API and the state model. Conversations retain
multi-turn history, while response chaining creates an auditable alternate branch.
Application code, rather than the model, owns the maximum iteration count.
```

5. Compare the table with `assets/agents-v1-loop.py.txt`; do not execute the legacy file.

**Check the completed code**

6. Save `src/main.py` and run the local checks before making a billable model call:

```powershell
python -m py_compile src/main.py scripts/preflight.py
python scripts/preflight.py
```

7. Confirm that all three markers remain, no `NotImplementedError` remains, and preflight reports `READY (local)`.

## Task 5: Run the solution

1. Run the application once and save its console output:

```powershell
python -m src.main --input assets/research-request.json --retain-resources 2>&1 | Tee-Object -FilePath artifacts-lab01.txt
```

The supplied synthetic scenario contains revenue, earnings before interest, taxes, depreciation, and amortization (EBITDA), debt, liquidity, maturity, contracted-revenue, and permitting facts plus explicit evidence gaps. The branch request adds a 250-basis-point refinancing-cost assumption so you can observe whether the alternate path reuses the original evidence without mutating its conversation history.

Authentication messages can show unavailable credential types before `DefaultAzureCredential acquired a token from AzureCliCredential`. That sequence is expected when the application uses your Azure CLI sign-in.

By default, the application retains the conversation and exact agent version so that you can inspect them in the Foundry portal. The explicit `--retain-resources` option in the command makes that intent visible and sets `resources_retained` to `true`. If you do not need portal inspection, use `--cleanup-resources`; the application then deletes the conversation and exact agent version it created in a `finally` block, including when a response call fails.

## Task 6: Validate the implementation

**Validate the captured output**

1. Locate the identifiers and iteration records:

```powershell
Select-String -Path artifacts-lab01.txt -Pattern 'agent_version|conversation_id|branch_response_id|iterations'
```

2. Confirm that all four fields are present, then open `artifacts-lab01.txt` and check:

| Field | Acceptance criteria |
|---|---|
| `agent_name` and `agent_version` | Identify the named agent and its immutable Foundry version. |
| `conversation_id` | Starts with `conv_`; all reflection iterations use this conversation. |
| `resources_retained` | Is `true` for the inspection run. |
| `iterations` | Contains one to three entries, each with a unique `resp_` response ID, a top-level status, and a nonempty summary containing `PLAN:`, `REFLECTION:`, `STATUS:`, and `ANSWER:`. The final status is `COMPLETE`. |
| `branch_response_id` | Starts with `resp_` and differs from every iteration response ID; the branch is outside the original conversation. |
| `branch_text` | Addresses the 250-basis-point refinancing assumption without inventing missing debt or cash-flow data. |

`STATUS: COMPLETE` means the bounded review finished; it does not prove that the investment question had sufficient evidence. The application-owned iteration limit prevents an unbounded loop.

**Validate agent-version lifecycle in Foundry**

3. Open the [Foundry portal](https://ai.azure.com) and select the project identified by `FOUNDRY_PROJECT_ENDPOINT` in `.env`.
4. Select **Agents** > **contoso-investment-researcher** and open its version history.
5. Confirm that the recorded `agent_version` is present and inspect its definition.

After inspection, delete the retained conversation and agent version in the Foundry portal. For future runs that do not require inspection, pass `--cleanup-resources`. That option cleans up only the resources created by that run; it does not delete versions retained by earlier runs. Version numbers can be greater than `1` after repeated runs.

**Validate the response traces**

6. Select **Agents** > **Traces** and set the time range to include the run.

Trace ingestion can take several minutes.

7. Search for each iteration `response_id` and the `branch_response_id`.
8. Confirm that:

- Each trace shows the expected agent name, agent version, response ID, timestamp, and successful response operation.
- Iteration responses use the recorded `conversation_id` where conversation metadata is available.
- The branch response chains from the final response without joining the conversation.

The traces prove the Foundry calls. `artifacts-lab01.txt` proves the application-owned iteration limit and branch decision.

## Optional challenge: Verify branch isolation

Create a second fork from the same parent response, give the two forks different synthetic follow-up facts, and compare their final state summaries.

**Expected output:** Each fork reports its own fact, neither fork reports the other fork''s fact, and the parent conversation remains unchanged. Record the parent and fork response IDs as evidence.

**Failure investigation:** Remove one persisted state field from a local copy of the session record, rerun the continuation, and identify the first missing continuity signal. Restore the field before cleanup.
## Task 7: Review the design

1. Answer these questions:

- When is a conversation preferable to `previous_response_id` chaining?
- Which completion signal belongs in model instructions, and which limit must remain application-owned?
- How would you migrate historical v1 thread state when the migration tool moves code but not data?
- Which branch metadata should be retained for audit without retaining sensitive prompt text?

## Task 8: Clean up

**Remove the Azure resources**

1. Run the following commands:

```powershell
$env:AZURE_DEV_USER_AGENT='microsoft_foundry_skill'
azd down --purge --force
Remove-Item Env:AZURE_DEV_USER_AGENT
az group show --name (azd env get-value AZURE_RESOURCE_GROUP) --output none
```

2. Confirm that the final command reports that the resource group no longer exists.

**Remove local generated files**

3. Run the following command:

```powershell
Remove-Item .env, artifacts-lab01.txt -ErrorAction SilentlyContinue
```

**Deactivate the virtual environment**

4. In every terminal where `(.venv)` appears, run:

```powershell
deactivate
```

## Summary

You migrated a stateful loop to Agents v2, implemented bounded reflection, retained context in a conversation, created a response branch, and validated the behavior against a live Foundry project.
