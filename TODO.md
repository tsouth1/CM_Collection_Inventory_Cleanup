#Add exclusions to collection inventory output csv
- [x] 1. exclude collections in the object paths:
\Assets and Compliance\Overview\Device Collections\Deployment\Software Center Deployments\Office 365*
\Assets and Compliance\Overview\Device Collections\Deployment\Security Patches*
- [x] 2. create several 'ReferencedBy' columns instead of just one to break out the different referencing classes - one for each reference, such as 'ReferencedBy-Query', 'ReferencedBy-Limiting', 'ReferencedBy-Include'. Move these columns near the CollectionName column.
- [x] 3. Add column for 'Custom Client Settings' deployments
- [x] 4. Create separate Deployments columns - 'Deployments - Application', Deployments - Configuration Baseline', 'Deployments - Task Sequence', Deployments - Custom Client Settings', etc.
