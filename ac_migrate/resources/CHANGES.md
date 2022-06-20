-----------------------------------
2/25/2022
Lambda Mapping:Updated "ac_diff" script to read lambda ARN replacement from a file.
	Create a mapping of source lamdba name and dest lambda function name and store it in file "resourceMap"
	The "ac_diff" will read and then search/replace the Lambda ARNs in the destination contact flow

-----------------------------------
2/28/2022
Added support to migrate multiple flows by providing pipe (|) delimited string as an argument to "save" script
	e.g. ac_save connect-instance profilename "flow-1|flow-2|flow-3"

-----------------------------------
3/2/2022
Added support to migrate QuickConnects. 
Note: 
Do not use option "-G <flows-to-skip>" with "ac_save" script

-----------------------------------
3/4/2022
Added support to replace Lexbot names and associate it to Connect instance
Added delete script that will delete all newly created resources (helper.del is created during copy).

-----------------------------------
3/6/2022
There are flows that reference the Claimed telephone numbers to set as Caller Id in Outbound whisper and Tranfer to Queue flows.
These flows cannot be migrated as the claimed telephone numbers in source account does not exist in destination account. 
Added functionality to check such flows. If any such flow that references claimed telephone number exists then only a default flow is created 
for such a flow. Such flows then will need to be manually updated in destination account. 
A "flowExceptions" file with a string that is searched in flow JSON is used to find such flows

-----------------------------------
3/8/2022
Updated Routing profiles to migrate more than 10 queue configurations. Updated "create-routing-profile" to make sure it has up to 10 Queue configs and then using "update-routing-profile" to update additional configurations. "create-routing-profile" API will fail if there are more than 10 Queue configurations

-----------------------------------
3/9/2022
Checking queue creation for description and MaxContacts in a queue and updating it accordingly
Added contact flow description while creating contact flows in destination instance if the description is blank in source

-----------------------------------
3/10/2022
Added functionality that will update queue configuration in a routing profile even if there are more than 10 queue configurations.
Added functionality that will update queue configuration in a routing profile even if only Priority and/or Delay are changed.

Exception:
----------
If a routing profile exists in source and destination instances and if Queue configurations are removed from Source instance but exists
in destination instance, then such routing profiles will not be updated. Queue configuration will need to be manually removed from destination instance's routing profile.

-----------------------------------
3/29/2022 - RESOLVED
Amazon Connect resource names (e.g. queue names, flow names, etc.) are not case sensitive. But as we are using bash scripts,
it becomes case sensitive when comparing and identifying resources that exist in source and destination instances.
This will cause an issue if you have flow in source and destination instances with same name but different case.
e.g. source instance has flow name "TestFlow" and destination instance has flow name "testFlow".
The bash scripts are now updated to ignore case when comparing resource names to identify whether resources exists or not in the destination instance

-----------------------------------
4/4/2022
Checking hours of operations for description. If description does not exist then it sets to default
Added flowException (flow to skip) if Queue Arn is that of Agent Queue as agents are not migrated yet but these package

-----------------------------------
4/14/2022
Added handling of associating Lex bot V1 and V2
Added handling of exception for quick connect where type is USER. The quick connect with type=USER will be skipped as users are not migrated
Added quick connect association to the queues. If a quick connect is of type USER and associated with queue in the source instance,
then it is not associated in the destination instance queue.

-----------------------------------
4/18/2022
Merged changes made by Jacky Ko
- Allow slashes in flow names and module names
- Allow special characters in names
- Use encoded flow_Default names
- Handle brackets in AWS CLI parameters
- Fix European charsets
- Added a warning of Extended ASCII compatibility

------------------------------------------------------------
4/29/2022
Fixed path related issues

------------------------------------------------------------
5/16/2022 - VERSION=1.2.2b
Merged Jacky Ko's "dos2unix" related changes (ac_save, ac_diff and ac_copy)

------------------------------------------------------------
5/17/2022 -
updated flowExceptions to add search string to enable skipping flow with HVOC block

------------------------------------------------------------
5/25/2022 -
Script ac_set_env_other to set up Gitbash/Mac Terminal path variables

------------------------------------------------------------
6/13/2022 -
resourceMap now supports phone number mapping between source instance and destination instance
introduced "flowExceptions_phone" file that does not have callerid related search string
If phone mapping is configured in resourceMap then the flows with hard-coded caller-Id settings (e.g. in Transfer to queue or Outbound whisper flows)
are migrated. The flow exceptions file used in this case will be flowExceptions_phone.
If phone mapping is not configured then "flowExceptions" file is used that has search string for caller id, and flows with hard-coded caller-id are skipped
"save" script exports phone numbers
"diff" script has logic for phone mapping

------------------------------------------------------------
6/16/2022 -
"copy" script now updates queue hours of operations if different from source
"copy" script now updates queue outbound configuration whisper flow
"copy" script now updates queue outbound configuration outbound phone number if phone mapping is configured in "resourceMap" file

------------------------------------------------------------
6/17/2022 -
"copy" now updates the quick connect queue configuration. if queue configuration or flow is modified in an existing quick connect then that is updated in destination
