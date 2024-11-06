# MuleSoft IDP Universal 🌐 Service
Scale and Reuse in a Universal IDP Solution

![image](https://github.com/user-attachments/assets/2085844b-7a97-498f-81fb-442428b2f8e2)

# MuleSoft IDP Universal 🌐 Service for Salesforce

The following implementation details were used to create [Salesforce Order Management demonstration](https://www.linkedin.com/posts/georgejeffcock_what-is-idp-intelligent-document-processing-activity-7234212552753184768-8yiO):   
![image](https://github.com/user-attachments/assets/53922dc9-2b7d-46ba-ae97-ac57a6a4fe96)

---

## Approach 1 \- (Business Logic within Salesforce)

FYI: This approach is used in this example.  
![image](https://github.com/user-attachments/assets/e14d0dd7-19f5-4a2b-a807-a60403c02464)


## Approach 2 \- (Business Logic within MuleSoft)

![image](https://github.com/user-attachments/assets/a522f652-834b-4c3b-948e-781abde2ae38)


In both approaches there are considerations when scaling a solution which has dictated the architectural decisions made:

* Content-Type Limitations in Salesforce Apex/Flow REST: Currently, Salesforce Apex/Flow REST [does not support requests with a Content-Type of multipart/form-data](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_rest_methods.htm?q=multipart), which complicates the handling of complex data structures in integrations.  
* Strong Typing in Apex/Flow: [Salesforce’s strongly-typed nature, with strict schema references for objects and fields, creates significant challenges when handling dynamic JSON results from IDP](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_intro_what_is_apex.htm?q=Rigorous), which often require flexible data transformations.  
* Asynchronous Processing of IDP Invocations: Given that IDP operates asynchronously, polling mechanisms for status checks do not scale effectively.  
* Maintaining Context for IDP Submissions: It’s essential to preserve the context of each IDP submission for subsequent processing and tracking.  
* Notifications for Manual IDP Reviews: IDP Action reviewers need timely alerts, such as email notifications, to address pending tasks requiring manual review.  
* Flexibility in Adding IDP Actions or Versions: Adding new IDP Actions or updating to a new version should be achievable without extensive code rewrites.  
* Determining the Location of Business Logic: Deciding where business logic should reside—within Salesforce, MuleSoft, or a dedicated middleware solution—is crucial for a maintainable integration architecture.  
* Middleware Pull vs. Push Model: Choosing whether a middleware layer should actively pull documents from Salesforce or passively receive pushed documents affects both performance and scalability.

---

# Implementation Objects for Approach 1:

| Platform | Name | Role |
| :---- | :---- | :---- |
| Salesforce | MuleSoft Trigger On New Case Origin | Publish New IDP Requests |
| MuleSoft | MuleSoft IDP Universal 🌐 Service \- ListenForNewIDPRequests | Consume New IDP Requests and Invoke Mulesoft IDP service specifying a Callback URL |
| MuleSoft | MuleSoft IDP Universal 🌐 Service \- processCallback | Process the MuleSoft IDP Callbacks and publish updates to Salesforce Platform Events |
| Salesforce | MuleSoft Trigger On Platform Event IDP Status Update | Consume MuleSoft IDP Status Updates and depending on Status process the successful JSON result |
| Salesforce | MuleSoft Trigger On Platform Event myDocActionName | Isolate processing of particular IDP Actions to a separate Salesforce Platform Event for maintainability |

---

## MuleSoft Trigger On New Case Origin

A great example of the use of  MuleSoft IDP is Salesforce [EmailToCase](https://help.salesforce.com/s/articleView?id=service.setting_up_email-to-case.htm&type=5), where we can set up [Trigger Flow](https://help.salesforce.com/s/articleView?id=platform.flow_concepts_trigger_record.htm&type=5) on Case Object   
![image](https://github.com/user-attachments/assets/87f9b3bd-4fdb-4650-b33f-33384c4f9b39)

Note the **Entry Conditions** should restrict on the **Origin Field** value that can automatically populated by the EmailToCase routine  
![image](https://github.com/user-attachments/assets/0672b2e7-a28d-489d-8052-0b2a156598dc)

The role of this triggered flow is to notify any Salesforce Platform Events **consumers** to a **New MuleSoft IDP Request** and therefore we must capture the bare minimum of detail required by the MuleSoft implementation to successfully call [MuleSoft IDP Document Action](https://docs.mulesoft.com/idp/automate-document-processing-with-the-idp-api#execute-the-published-document-actions)  such as:

* MuleSoft\_IDP\_ActionName  
  * In reality both ActionId and ActionVersion are required but you could, like in this case, simplify and give your actions names that are equal to the names of the action in MuleSoft IDP. If you require Versioning Salesforce side you must then capture the Version Id and submit that also but with the Salesforce being **Strongly Typed** breaking changes will be seldom  
![image](https://github.com/user-attachments/assets/24681c69-1895-4ebb-a8b3-b73dc26e4901)

* MuleSoft\_IDP\_ContentVersionId  
  * MuleSoft will pull the document from Salesforce instead of Salesforce pushing the document as a binary object or Base64 which is [problematic](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_rest_methods.htm?q=multipart) for Salesforce Flow architecture  
* MuleSoft\_IDP\_Context  
  * Due to the asynchronous nature of this process we must record what record/object/process wanted the doc in question to be processed so the context field can be [Entity Key](https://help.salesforce.com/s/articleView?id=000385203&type=1) **Polymorphic**

Salesforce Platform Event \- **MuleSoft IDP New**  
![image](https://github.com/user-attachments/assets/4fef9f72-cfa4-4c5c-a27b-19f406f55000)

The logic could consist of 4 parts:

1. Get Content Version Id of Email doc attachment  
2. Determine what Action Name to be used to process the doc. Please note that has been externalized into a subflow as this logic could and will exist elsewhere.   
3. Create the Platform Event  
4. Update Case Comments to give the user an idea of progress

![image](https://github.com/user-attachments/assets/1a231771-57f1-447c-a364-d4c1b6990c19)

Error Handle please but for brevity removed for screenshots

![image](https://github.com/user-attachments/assets/d064f56b-c560-446c-970a-1db20f0d6b7f)

---

Over to [MuleSoft IDP Universal 🌐 Service](https://github.com/composableforce/MuleSoft-IDP-Universal-Service)

## MuleSoft IDP Universal 🌐 Service \- ListenForNewIDPRequests
![image](https://github.com/user-attachments/assets/6981a119-074a-41f2-a0bf-b701043d5f9b)


This flow should not need any changes as it has been built with configuration and not coding in mind. 

1. Subscribes to channel /event/MuleSoft\_IDP\_New\_\_e and as long as you have [defined the event as above](#heading=h.3sceepfa76vf) no changes are required! ![image](https://github.com/user-attachments/assets/4e6b8e7b-a9ef-4ef1-a8ae-06c8deb6584c)


2. ##### Get **Action Keys by Name** uses [MuleSoft IDP Platform API](https://github.com/composableforce/MuleSoft-IDP-Platform-API) which is the current **undocumented** Platform API powering the MuleSoft IDP UI which you are free to use at your own risk or just use a Choice component as hard coded the Action Id and Action Version you want depending on received ActionName

3. Get the Salesforce Doc from Content Version Object

4. #### **Generate the [Callback URL](https://docs.mulesoft.com/idp/automate-document-processing-with-the-idp-api#callback-url)** \- Small dataweave snippet to inform MuleSoft IDP platform to update this MuleSoft Application with any updates and use use an api key in the query parameter for security while also recording the Context of the submission within the query parameters {#generate-the-callback-url---small-dataweave-snippet-to-inform-mulesoft-idp-platform-to-update-this-mulesoft-application-with-any-updates-and-use-use-an-api-key-in-the-query-parameter-for-security-while-also-recording-the-context-of-the-submission-within-the-query-parameters}
![image](https://github.com/user-attachments/assets/d2705ab6-52b3-43da-86a3-d5136ce821bc)


5. [Submits](https://docs.mulesoft.com/idp/automate-document-processing-with-the-idp-api#execute-the-published-document-actions) an action version document execution  
   1. Please note no need to use MultiPart in the dataweave  
   2. Please note this implementation uses [MuleSoft IDP Universal 🌐 REST Smart Connector 🔌](https://github.com/composableforce/MuleSoft-IDP-Universal-REST-Smart-Connector)  which allows us any action to be submitted as the Action keys have been modeled as URI parameters

![][image13]

6. Publish to Salesforce Platform Event **/event/MuleSoft\_IDP\_StatusUpdate\_\_e** of success or failure of submission (Not failure of the doc processing but was the submission accepted or not)

![][image14]

Salesforce Platform Event \- **MuleSoft IDP Status Update**  
![][image15]

---

## MuleSoft IDP Universal 🌐 Service \- processCallback

![][image16]

This flow should not need any changes as it has been built with configuration and not coding in mind. 

1. After we dictated to the submission that a [Callback is to be used](#generate-the-callback-url---small-dataweave-snippet-to-inform-mulesoft-idp-platform-to-update-this-mulesoft-application-with-any-updates-and-use-use-an-api-key-in-the-query-parameter-for-security-while-also-recording-the-context-of-the-submission-within-the-query-parameters), we will receive PATCH updates from MuleSoft IDP Platform. The definition used with this project can be found here [MuleSoft IDP Universal 🌐 Callback API](https://github.com/composableforce/MuleSoft-IDP-Universal-Callback-API)  
2. [Retrieve the Results of the Execution](https://docs.mulesoft.com/idp/automate-document-processing-with-the-idp-api#retrieve-the-results-of-the-execution) an action version document execution  
   1. Please note this implementation uses [MuleSoft IDP Universal 🌐 REST Smart Connector 🔌](https://github.com/composableforce/MuleSoft-IDP-Universal-REST-Smart-Connector)  which allows us to GET any Result as the Action keys have been modeled as URI parameters  
   2. Note this subsection is within an Async Component as MuleSoft IDP does not currently error handle or retry on failed PATCH  
3. This implementation as specified before has dynamic lookup for Action Names as it is the Name of the Action that the consumer (Salesforce) knows the Actions by and not Action Id and Action Version, this needs to be recorded on the Update Platform Event  
4. Publish to Salesforce Platform Event **/event/MuleSoft\_IDP\_StatusUpdate\_\_e** of success or failure of submission processing. In this implementation (no business logic) the MuleSoft Service is there to submit and record status updates for Salesforce then to process accordingly

![][image17]

5. Optional: Return correlationId for future reference

---

Back to Salesforce Platform

## MuleSoft Trigger On Platform Event IDP Status Update

The role of this [Platform Event triggered flow](https://developer.salesforce.com/docs/atlas.en-us.platform_events.meta/platform_events/platform_events_subscribe_flow.htm) is to **orchestrate** the processing of IDP document processing updates:

* Note it is not where we process Successful results, we have separated that out to separate platform events (Flow References not supported yet in Platform Triggered Flows) allowing us to build in isolation and safety of processing each type of doc not affecting another doc processing

![][image18]

1. Determine course of action depending on [Execution Results](https://docs.mulesoft.com/idp/automate-document-processing-with-the-idp-api#execution-status-reference) including bespoke errors you may or not return within the MuleSoft tier like “Failed Submission”  
* ACKNOWLEDGED: The document action execution request was received.  
* IN\_PROGRESS: The execution started.  
* RESULTS\_PENDING: The execution finished and IDP is processing the results.  
* MANUAL\_VALIDATION\_REQUIRED: The execution finished but the results need manual validation.  
* FAILED: The execution request finished unsuccessfully.  
* PARTIAL\_SUCCESS: The execution request finished but some sub-tasks failed.  
* SUCCEEDED: The execution request finished successfully.  
2. A catch all to handle any unrecognized execution status. Raising an Error that stops a flow can be a little hard in flow currently so feel free to use the following apex class:

![][image19]

| public class MuleSoftRaiseErrorClass {     @InvocableMethod(label='Raise Error' description='Raises a custom error based on input' iconName='slds:custom:custom82')     public static void raiseError(List\<Request\> requests) {         for (Request req : requests) {             if (req.raiseError) {                 throw new CustomException(req.errorMessage);             }         }     }      public class CustomException extends Exception {}      public class Request {         @InvocableVariable(required=true description='Indicate if an error should be raised')         public Boolean raiseError;                  @InvocableVariable(required=true description='The error message to raise')         public String errorMessage;     } } |
| :---- |

3. On MANUAL\_VALIDATION\_REQUIRED: we can use an undocumented API to fetch the reviewers for a particular MuleSoft IDP Action. This API has been exposed  in [MuleSoft IDP Universal 🌐 Service](https://github.com/composableforce/MuleSoft-IDP-Universal-Service)  
4. Iterate around the returned list of reviewers and send them an email notifying them of an awaiting review task

![][image20]

5. When SUCCEEDED determine which doc to process based on the Salesforce Platform Event Action Name. We know see the [benefit of Action Names](#bookmark=id.c3ryymd022a6) as we do not need [any fuzzy logic here](#bookmark=id.7f744ocpnjlh)  
6. Publish to Salesforce Platform Event **MuleSoft IDP Result myActionName** of success and therefore record the details for the triggered flow to get the result (Json Result) to process in isolation and dedicated flow for the specific doc  
   ![][image21]

7. Optional \- The Context of the IDP submission will differ between submission requests from the Salesforce Platform so here we determine using the Context field to see if the origin was a Case and therefore 8\. Update the Case Notes

![][image22]

---

Final Step is the Process a Successful IDP Submission

## MuleSoft Trigger On Platform Event myDocActionName

The role of this [Platform Event triggered flow](https://developer.salesforce.com/docs/atlas.en-us.platform_events.meta/platform_events/platform_events_subscribe_flow.htm) is to **process** the successful IDP submission for a particular Action Name in a dedicated flow

Why dedicated flow?: Currently building flows around invocable actions leaves you open to maintenance headaches see the following coming attraction \!

| Dreamforce 2024 (Remember always subject to change) ![][image23] “Allow invocable action developers to create new versions of their actions with breaking changes (behavior, input/output parameter changes) without breaking existing consumers, something that’s not possible today. Deprecate older versions of actions so they are no longer usable in new flows and other consumers. Provide path to update consumers to use newer versions of actions. *Enables invocable action developers to evolve their actions over time in ways that are not possible today without creating entirely new actions. Consumers can then update to newer versions when they are ready.”*  |
| :---- |

Secondly consider this statement:: [Salesforce’s strongly-typed nature, with strict schema references for objects and fields, creates significant challenges when handling dynamic JSON results from IDP](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_intro_what_is_apex.htm?q=Rigorous)

**So in order to limit the effect of these two difficult facts separate the processing into dedicated flows so that each flow can remain at a external service version while you support new docs at different versions of the same external service**

**Secondly part of [MuleSoft IDP Universal 🌐 Service](https://github.com/composableforce/MuleSoft-IDP-Universal-Service) is to merge all your document action json schema into ONE schema which means the Salesforce platform need only consume as an external service ONE service to GET your entire Doc Results and only one APEX generated class is required for you to happily transform the data in a flow\!**

To transform the data you have 3 choices

* Flow Transform Component (chosen for this example and remember in this example we had to create not only a Order Header but Order Item**S)**  
* Pure Apex  
* Dataweave for Apex (super powerful)

![][image24]  
Note: You will see an APEX class called “Get Product and Pricebook Entry” in our flow which executes a SOQL join which will again be supported:

|  Dreamforce 2024 (Remember always subject to change) ![][image25] One Get Records, Multiple Objects Combine multiple related objects (i.e. Opportunities, Products and Pricebooks) into a single variable in your Flow.  |
| :---- |

**The point being build now in Flow, adopt Flow and it will allow greater democratization of who will be able to build and maintain these MuleSoft IDP Json results, if you build in Apex it is harder to change course.**

For your comparison Approach 2 with the business logic inside MuleSoft also requires work, you have to make a decision where and who should do the transformation and importantly who will maintain this over time  
![][image26]

---

## Appendix

## External Service & Named Credentials Screenshots

![][image27]

![][image28]

![][image29]

![][image30]
