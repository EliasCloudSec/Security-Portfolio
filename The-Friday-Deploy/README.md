# The Friday Deploy: Shift-Left Security Review Report


I reviewed a newly deployed notification workload against an established production baseline. This review covers region placement, workload identity, storage exposure, and inbound network access. The assessment resulted in a prioritized remediation plan that addressed demonstrated anonymous data access before lower-risk configuration improvements.  


>This pre-production security review simulated a DevSecOps validation process within a CI/CD workflow, using the lab's production-reference stack as a configuration baseline. No live production environments were modified. Sensitive deployment details, resource identifiers, endpoints, timestamps, and challenge answers have been withheld.  



---

 
ROLE: Cloud Platform Engineer  
TICKET: Review Deployment before prod promotion  
DEPLOYED: Friday, 4:57pm  
DEPLOYED BY: The intern. They said "it works."  
CLEARANCE: Reader on [REDACTED]  
GOAL: Compare against the production baseline. Log every finding. Block the "ship to prod" if needed.  

## Context and Scope
**Environment:** Live, multi-user Azure training tenant  
**Access:** Reader for the deployment under review  
**Review disposition:** Hold production promotion pending corrective work and validation


## Method 

I compared the new deployment against an existing, healthy production-reference workload within the shared lab resource group. Since no formal security baseline was provided, I established one by examining the configurations of the existing workload.  

I structured the assessment around five guiding questions:

- The Where: Is the workload deployed in the correct region?

- The Who: How does the workload authenticate to dependent services?

- The Leak: Where is the data stored, and is it exposed?

- The Door: Who can access the workload?

- The Call: Which security issues require remediation first?






## The Priority Call 


**Principle: EXPOSURE BEATS HYGIENE. ACTIVE BEATS POTENTIAL**

>[!IMPORTANT]  
>Close the container to stop anonymous access to the exposed file immediately, then verify that it no longer loads without authorization. Hold the release until you correct that and test the remaining deployment gaps.  
>> Data exposure takes priority, the file is already accessible from the internet. Very bad.

| Priority | Finding | Why | Recommended timing | Class |
| --- | --- | --- | --- | --- |
| 1 | Anonymous access to internal data | A file was already readable without sign-in and accessible from the internet | Contain immediately; repeat the unauthenticated check | Exposure |
| 2 | Unrestricted public application ingress | A scheduled workload had an unnecessary public entry point | Correct promptly in the same change cycle| Hygiene |
| 3 | No managed identity attached | Workload authentication differed from the reference design | Resolve authentication and permission design | Hygiene |
| 4 | Region placement differed from the baseline | Cross-region dependencies can add latency and transfer cost | Correct placement in the next deployment | Hygiene |


## Findings  

### THE WHERE
- **What I observed:** The function app was deployed in Australia East, while the production stack and the data source are in Central US. Different country and region.
- **How I verified it:** In the overview page I compared the Location fields of the new app and the production function app. Production runs in Central US; the new app's App Service plan and storage account were also in Australia East.
- **Why it matters:** Every call between this app and the rest of the platform has to travel between two locations thousands of miles apart, which adds latency to every request. It also costs money: Azure doesn't charge for data coming into a region (ingress), but it does charge for data leaving one (egress), so every response is billed.
- **Fix:** Redeploy the app, its plan, and its storage into the platform's standard region, Central US.

![a](thefridaydeploy1.png)
![a](thefridaydeploy2.png)
![a](thefridaydeploy3.png)
> note the regions.


### THE WHO
- **What I observed:** The intern function app had no managed identity, no system assigned, or user assigned, while baseline uses a user-assigned identity.
- **How I verified it:** I looked in the Identity blade of the intern app and checked both the System Assigned and User Assigned tabs and found no managed identity attached in either of those locations. The User Assigned tab of the baseline app had a User Assigned managed identity attached.
- **Why it matters:** Stored credentials can leak into config files, code, log files, repos, etc. Because a stored credential never expires or rotates on its own, it keeps working until someone notices the unauthorized access. A managed identity has no secrets to leak.
- **Fix:** Attach a user-assigned managed identity with a scoped role on the storage account, then remove the stored credential.




### THE LEAK
- **What I observed:** A storage container was configured for anonymous access.
- **How I verified it:** I opened the blob (file) inside the container and copied its URL. I opened that URL in a private browser window, and the file loaded with no sign-in required.
- **Why it matters:** This is a confirmed, public exposure. Anyone with the URL can read the data in this container. The exposed deployment notes referenced a connection string shared through an internal chat, highlighting unsafe credential handling.
- **Fix:** Make the affected container private immediately. Disallow anonymous blob access at the storage-account level. Retest the same unsigned URL from a clean session and confirm the data is unavailable. Review available access logs and the exposed content to assess possible impact; rotate secrets only if they were actually present or otherwise exposed.





### THE DOOR
- **What I observed:** The intern app allows public access from all networks with no access restrictions, and unmatched traffic is allowed.
- **How I verified it:** I checked Access restrictions under the Networking blade on the Main site tab of both apps. The intern app has no rules, and its catch-all allows all traffic. Production has one deny rule blocking all addresses, and its catch-all denies unmatched traffic.
- **Why it matters:** Anyone on the internet can reach this app's endpoint, even though it only runs scheduled jobs and never needs inbound traffic.
- **Fix:** Add a single deny rule, matching production. Once any rule exists, App Service treats unmatched traffic as an implicit deny.


### THE CALL : RECOMMENDATIONS
Close the public container immediately. Attach a user-assigned managed identity with a scoped role on the storage account and remove the stored credential. Redeploy into the platform's standard region. Add a single deny rule to close inbound, since one rule flips everything unmatched to implicit deny. And one worth adding yourself: a deploy checklist so the next Friday afternoon does not need a human to catch four things.

