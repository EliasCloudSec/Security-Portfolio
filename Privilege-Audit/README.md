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




1. ACCESS CONTROL
role assignments
>
2. COMMAND LINE
CLI holds it all
>
3. RESOURCE GRAPH
KQL is powerful
>
4. PIM
eligible vs active
>
5. THE HUNT
find the culprit



[ KQL QUERY ]
authorizationresources
| where type =~ 'microsoft.authorization/roleassignments'
| extend principalId = tostring(properties.principalId)
| extend description = properties.description
| where principalId == '<insert orphaned principal ID>'
| project name, principalId, principalType = properties.principalType, scope = properties.scope, description






# [Title, outcome-flavored: "Investigating an Identity Attack in Entra ID"]

## Scenario
2 to 3 sentences. What was the situation and what question did the investigation answer? Frame it like a work ticket, not homework.

## Environment
One list: platform, services, tools, access level. Honest framing: "live multi-user Azure training tenant, Reader access."

## Investigation
The core. Numbered steps IN YOUR OWN WORDS: what you looked at, what you found, what you concluded at each step. 6 to 12 screenshots of meaningful moments (portal views, query results, before/after).

## What broke / what surprised me
The most credible section in the document. Dead ends, wrong guesses, the thing that took an hour. Employers know real work is messy. This section separates you from certificate collectors.

## Findings and recommendations
What you determined, plus 2 or 3 recommendations as if you were reporting to the resource owner.

## What I learned
3 to 5 bullets. At least one technical, one "what I'd do differently."
