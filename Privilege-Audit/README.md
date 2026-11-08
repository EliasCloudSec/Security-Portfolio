[ CLASSIFIED // AUDIT BRIEFING ]
CASE: THE PRIVILEGE AUDIT
TENANT: Mad Hat Labs
STATUS: FIELD AUDIT IN PROGRESS
CLEARANCE: OPERATIVE (Reader)
OBJECTIVE: Walk through the privilege access audit. Recover the security team's finding at each stage. Confirm the culprit.





| Method               | Sees                                        | Misses                              |
|----------------------|---------------------------------------------|-------------------------------------|
| IAM blade / export   | Active assignments at scope, inherited      | Group members, orphaned principals  |
| Azure CLI            | Same, plus null principalName (orphans)     | One scope per run                   |
| Resource Graph (KQL) | Whole tenant in one query                   | Eligible assignments                |
| PIM export           | Eligible vs active, activation history      | Assignments outside PIM             |
