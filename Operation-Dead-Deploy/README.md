Operation Dead Deploy - Governance forensics, deployment audit trail

## Scenario 
An intern with temporary Contributor access deployed a "test environment" over a weekend, cut every corner, and left. I was tasked with finding out why the governance policy didn't prevent it. I was given Reader access to reconstruct how this occured.

## Environment
live multi-user Azure training tenant, Reader access.

## Investigation
I first checked the client's subscription, didn't notice anything out of the ordinary. Then proceeded to examine the subscription groups and tags.
I found a subscription group that didn't follow the naming convention [testdeploy123], which lead me to the intern's storage account. Offender located.
I also checked the tags of client's subcription to see if saw anything also out of the naming convention there as well or anything key terms like "test","intern", or "deploy".
I found tags that labeled "unspecified" and "unkown" which lead to the same subscription group and storage account that belonged to the intern. Offender confirmed.
Looked at interns deployments and time stamps to confirm. Then looked at the subscription group's [testdeploy123] policies.
After reviewing the policies and seeing a naming convention policy existed, I investigated further to find a error in the policy effect was set to "audit" instead of "block".
This was how the intern was able to deploy a resource group outside the naming convention.

## What broke / what surprised me
Some dead ends with searching for keyword "intern". Which was what lead me to just looking for things out of place in the resource groups. Also when I was looking at tags,
the terms "unspecified" and "unknown" raised suspicion as those terms wouldn't make sense to have in a finished project. Luckily those tags connected to the intern's account and deployment.

## Findings and recommendations
Because the policy for the naming convention was set to "audit" instead of "block"; the intern was able to deploy a resource group outside of the naming convention.
Audit simply alerts or notifies non compliance, changing it to block would prevent it.

## What I learned
- root cause analysis in an azure environment
- how policy and compliance audits prevent situations like this
- recontructing a timeline of events
