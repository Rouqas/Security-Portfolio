# Opeation Dead Deploy
Investigating a unusually named resource group 

## Scenario
An intern with temporary Contributor access deployed a "test environment" taking short route and left. An audit took place to reconstruct the incident and establish the reason of this incident.

## Environment
Live multi-user Azure training tenant, Reader access


## Investigation

### Stage 1: Identify the non-compilant resource group
 Under our designated subscription while scanning the resource groups, i found a resource group which didn't follow the Microsoft's naming convention. The resource group name was: **testdeploy123**. The naming format of this resource group didn't match with other existing resource groups in our subscription.
![naming_discrepency](/evidence/w1_naming_discrepency.png)

### Stage 2: Inspect the payload
 Under **testdeploy123** resource group, only one resource was found. I found Several tags under this resource: owner, cost-center and intern-flag and environment. These tags told me this resource was deployed by a company intern.
![found_resource](/evidence/w1_found_resource.png)
![resource_tags](/evidence/w1_resource_tags.png)

### Stage 3: Trace the deployment
Every azure resource is created by an IaC deployment. I looked at the creation story, under this(**testdeploy123**) resource group's deployment blade. I found one deployment:**interntestdeploy2026** with status "Succeeded". Start time: *24/05/2026, 22:42:25*, which is the start of any incident timeline. The name of this deployment told me who ran it. The input blade of this deployment, also showed which parameters were used. This tells that the resource was deployed by a intern which matches the finding of my previous step.
![deployment](/evidence/w1_deployment.png)
![deployment_inputs](/evidence/w1_deployment_inputs.png)

### Stage 4: Examine the policy
A naming policy existed on the subscription and was active, yet the resource got created anyway. The **Effect** parameter was set to **Audit**, which logs violations, instead of **Deny**, which blocks them.
![policy_exists](/evidence/w1_policy_exists.png)
![policy_parameter](/evidence/w1_policy_in_audit.png)

## What surprised me
Navigating the live azure environment and understanding different components(eg. subscriptions, resource groups, resource, deployments, tags etc) took me some time. Doing so helped me understand hierarcy, infrastruce and how resources are organised in the environment. It was nice that this simple things enabled me investigate the creator/ owner of a resource, the time of creation, tags and policies assigned to it. Initially, I thought there was no naming policy, but looking at the policies blade under that resource group, I found there was a naming policy. The logs also showed that the name was non-compilant, but still it didn't stop the creation. 

## Findings and recommendations

This non-compliant resource was the result of a legitimate deployment created without following compliant naming conventions, because the naming policy effect was not set to block the request if it was non-compliant. I recommend determining whether the Audit effect is intentional, and if not, adjusting the policy parameters to enforce the expected policy outcome.

## What I learned
- I would follow microsoft's or, the company's naming convention while naming any resources. It helps to identify and give context about the resource before opening, this simple practice will make work efficient for everyone.
- I would assign resource specific tags and correct flags for that resource during creation
- Every resource has an audit trail in the form of logs. These can be correlated with security alerts to build an incident timeline and create a holistic picture during incident investigation.
- Policy effect has a crucial and critical role, if not assigned accurately according to the weight of a resource, system may behave unexpectedly and will require time and effort to investigate and fix the issue. 
