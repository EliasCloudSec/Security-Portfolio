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

Close the container to stop anonymous access to the exposed file immediately, then verify that it no longer loads without authorization. Hold the release until you correct that and test the remaining deployment gaps.
> Data exposure takes priority, the file is already accessible from the internet. Very bad.


## Findings  

### THE WHERE


