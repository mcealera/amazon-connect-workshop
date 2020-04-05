# Intro to Amazon Connect

## What are we going to build?
1. A working instance of Connect
2. A voiced IVR flow that will recognize incoming calls, offer personalized greetings and adapt based on user interaction history.

## What services are we going to use?
1. Connect as the Contact Center solution and backbone for the other services
2. A DynamoDB table to store user information 
3. Lambda functions to store and retrieve user information
4. S3 to store conversation data and call metadata
5. Cloudwatch to monitor and debug the solution
6. Polly to voice IVR dialogue
7. (optional) Lex for basic chatbot functionality

## Prepare the Environment (if not already done)

### Launch an Amazon Connect instance
1. Log into the console.
2. Navigate to the Amazon Connect service page.
3. Select Add an instance
4. For identity management, choose Store users within Amazon Connect and enter a unique url to serve as an access point for your Contact center. Click Next.
   1. For production applications, it may make more sense to integrate with an existing directory, but for the purposes of today, let's use the built in user management capabilities.
5. For Create an Administrator page, add an admin identity.  You can use your IAM user to authenticate as an admin into the instance if you do not create a separate entity. Click Next.
6. For Telephony Options, make sure both incoming and outbound calls are selected. Click Next.
7. For Data storage, leave the default settings. Click Next.
8. Review and Create the instance.
9. Once the instance is created, select Get Started.

![](images/success.png)


# Getting Started with Amazon Connect

1. Access the instance.
   1. Navigate to the Amazon Connect access URL
   2. Connect with the Username and Password for the admin account.
2. Once you&#39;ve entered the Amazon Connect application, select &quot;Let&#39;s go&quot;.
3. Claim your phone number. The country should be the same as where your billing address is. Select Direct Dial number and click Next.

![](images/phone_number.png)
5. Wait a minute or so.  
6. Give yourself a call! Amazon Connect comes with a number of example contact flows to demonstrate different functionalities.
(Optional) Choose 1 > 1 > 1 in the IVR to get transfered to an agent and make the web client ring!

![](images/makeacall.png)

### Create your first Contact Flow

![](images/contactflow.png)

1. Under Routing, select Contact Flows.
2. Select create contact flow.
3. Enter the name TransferToQueue.
4. Under Interact, add a Play prompt module and link it to Entry point.
5. Click into the module, select Text to speech and enter the text &quot;I&#39;m putting you in the queue&quot;.
6. Under Set, add a Set working queue module and link it to the Play prompt module.
7. Click into the module and select the BasicQueue.

![](images/1_TransferToQueue.png)

8. Under Terminate/Transfer add a Transfer to Queue module and link it to the Success option of the Set working queue module.
9. Add two more Play prompt modules. Make one say &quot;Something went wrong. Try again later&quot; and the other &quot;I&#39;m sorry. It seems our queue is at capacity. Try again later&quot;.
10. Link the error message to the Error options on set working queue, Transfer to Queue, and the at capacity message.
11. Under Terminate/Transfer, add a Disconnect/Hang up module and link your final messages to it.
12. Save and then publish.

### Test it out!

1. Under phone numbers, select the number you&#39;ve claimed.
2. Under Contact flow / IVR, select the TransferToQueue contact flow you just created and save.
3. Open up the CCP by clicking the phone icon in the upper right hand corner.

![](images/2_CCP.png)

1. Wait a few moments and give yourself a call.
2. Notice that when a customer is put into a queue, they are really put into the Default customer queue.  If you want to change the experience, you can.  You can also build things like interruptible queue flows.  Similarly, the agent (you) heard the Default agent whisper.  Whispers are used to share information to only one side of the conversation.

### Creating a Callback Contact Flow – Import a Contact Flow

![](images/3_TransferToCallbackQueue.png)

1. Under Routing, select Contact Flows.
2. Select create contact flow.
3. In the upper right next to the Save button, select Import flow (beta).
4. Upload the TransferToCallbackQueue file from the contact-flows folder in the repository.
5. Modify the Set working queue module to select the BasicQueue and save.
6. Click through the modules to understand how the pieces work together.
7. Save, Publish, and test the TransferToCallbackQueue contact flow like your TransferToQueue contact flow.  Notice that when connected using the callback queue, the caller heard the Default outbound contact flow.

### Using Hours of Operation

![](images/4_InboundCallRouter.png)

1. Under Routing, select Contact Flows.
2. Select create contact flow.
3. Enter the name InboundCallRouter.
4. Under Branch, select Check hours of operation. Select Basic Hours.
5. Build out the Error flow with error message and termination.
6. Under Branch, select Check staffing. Under Status to check, select Available. Optionally select the basic queue. Link this module to Check hours of operation&#39;s In Hours module. Link the error option.
7. Under Terminate/Transfer, select Transfer to flow. Select your TransferToQueue contact flow and save. Link this to the True output of Check Staffing and the error path.
8. Add a Play prompt module that says &quot;There is currently no one ready to accept your call.  Let me put you in the callback queue so you don&#39;t have to wait on hold&quot; and link this to the False output of Check Staffing.
9. Under Terminate/Transfer, select Transfer to flow. Select your TransferToCallbackQueue contact flow and save. Link this to your prompt and the error path.
10. Add a Play prompt module that says &quot;We are currently closed.  Please call again later.&quot;  Link this to the Out of Hours option and terminate.
11. Save, Publish, and Update your phone number&#39;s Contact Flow.
12. Now you can test how the caller is routed when you are Available or Unavailable in the CCP. Similarly, if you can change the Basic Hours of operation to see how users are routed.

# Building On the Basics

## Integrating AWS Lambda and External Transfers

### Building an On Call Contact Flow

![](images/5_OnCallFlow.png)

1. Create a contact flow and Import the OnCallFlow from the contact-flows folder.
2. Update the Invoke AWS Lambda function module to call a lambda function that looks like ContactRouterLambda and save the module
3. Update the Get customer input module to select the Lex bot in the account and save.
4. Save, Publish, and test the OnCallFlow contact flow like your TransferToQueue contact flow.  The Lambda function we&#39;re invoking is picking up two contacts from a backend system and using that to dynamically route to another individual.  Look at the ContactRouterLambda to see how we do that.

### Using Amazon Lex as a Conversational Router

![](images/6_InboundLexRouter.png)

1. Create a new contact flow called InboundLexRouter.
2. Under Interact, add a Get customer input module. Add a Text to speech prompt. &quot;Would you like to wait on hold or be called back later when we are ready to serve you?&quot; Select Amazon Lex and the ConnectBot in the account. Add WaitOnHold, CallBack, and Emergency as Intents.
3. Create an error flow.
4. Under Terminate/Transfer, select Transfer to flow. Select your InboundCallRouter contact flow and save. Link this to the WaitOnHold output and the error path.
5. Under Terminate/Transfer, select Transfer to flow. Select your TransferToCallbackQueue contact flow and save. Link this to the Callback output and the error path.
6. Under Terminate/Transfer, select Transfer to flow. Select your OnCallFlow contact flow and save. Link this to the Emergency output and the error path.
7. Save, Publish, and Test by pointing your phone number to the InboundLexRouter contact flow.

### Using Amazon Connect and Amazon Lex for Outbound Surveys

![](images/7_CallerSurvey.png)

1. Create a new contact flow and import the CallerSurvey contact flow.
2. Modify the Get customer input module to point to the ConnectBot in the account.
3. Save and Publish (test if you&#39;d like).
4. Take a note of the Contact Flow ID.  It is the last 36 character string in the URL.  The URL is formatted like so: https://[CONNECTINSTANCENAME].awsapps.com/connect/contact-flows/edit?id=arn:aws:connect::[ACCOUNTNUMBER]:instance/[INSTANCEID]/contact-flow/[CONTACTFLOWID]

## Putting it all Together

![](images/8_EntryPoint.png)
1. Create a new contact flow and import the EntryPoint contact flow.
2. Modify the first Invoke AWS Lambda function to ensure it is calling a lambda function that looks like GetHotMessageLambda.
3. Modify the second Invoke AWS Lambda function to ensure it is calling a lambda function that looks like ContactLookupLambda.
4. Modify the Get customer input module to point to the ConnectBot in the account.
5. Modify the second Invoke AWS Lambda function to ensure it is calling a lambda function that looks like PutContactinQueueLambda.  Modify the OutboundContactFlowId parameter to be the contact flow ID of the survey contact flow.
6. Modify the Transfer to flow module at the end of the contact flow to point to the InboundLexRouter contact flow.
7. Save, Publish, and Test.

# License
This library is licensed under the MIT-0 License. See the LICENSE file.
