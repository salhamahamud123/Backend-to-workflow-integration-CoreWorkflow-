---
Title: "Backend-to-workflow-integration-CoreWorkflow"
Description: Example of Web Application for Submit Contact Form using using Workflow Core
Date: 15/11/2021
---

# Backend-to-workflow-integration-CoreWorkflow

A small example of using Workflow Core as part of an ASP.NET Core 5 web application.

In this document, you will learn how to:

- Install NuGet package "WorkflowCore"
- use Fluent API
- Passing data between steps
- Injecting dependencies into steps
- Do Error handling
- Do Parallel Paths Control Structures

### Install NuGet package "WorkflowCore"

##### Using nuget

      PM> Install-Package WorkflowCore
      
##### Using .net cli
 
      dotnet add package WorkflowCore
      
##### Manage Nuget Package from Visual Studio

      Go to Project > Manage Nuget Packages...

![Manage Nuget Package](https://user-images.githubusercontent.com/94332683/141721270-baf4fafd-c085-464f-877d-451e758e21e1.png)

      Browse "WorkflowCore", then install it

![Brose WorkflowCore](https://user-images.githubusercontent.com/94332683/141721408-ab174761-8a88-4055-a2ea-8d13941f4c35.PNG)

### Fluent API

Define workflows with the fluent API
 
```
public class ContactFormWorkflow : IWorkflow<ContactForm
{
      public void Build(IWorkflowBuilder<ContactForm> builder)
            {    
                builder
                   .StartWith<Task1>()
                   .Then<Task2>()
                   .Then<Task3>;
            }
}
```

### Passing data between steps

*Our workflow step with inputs and outputs*

```
public class EmailAdmins : StepBody
{
        public string From { get; set; }

        public string Subject { get; set; }

        public string Message { get; set; }

        public Guid Key { get; internal set; }
  
       public override ExecutionResult Run(IStepExecutionContext context)
        {
            using (var client = new SmtpClient()
            {
                DeliveryMethod = options.DeliveryMethod,
                PickupDirectoryLocation = options.PickupDirectoryLocation,
            })
            {
                var fullMessage = HttpUtility.HtmlEncode(Message) + $@"<br><br><a href=""{string.Format(options.AckUrlFormat, Key)}"">Click here to acknowledge the Submission</a>";

                client.Send(new MailMessage(From, options.AdminEmails, Subject, fullMessage) { IsBodyHtml = true });
                
            }

            return ExecutionResult.Next();
        }
}
```

*Our workflow definition with strongly typed internal data and mapped inputs & outputs*

```
    public class ContactFormWorkflow : IWorkflow<ContactForm>
    {
        public void Build(IWorkflowBuilder<ContactForm> builder)
        {
            // Email admins every 30 seconds until receipt is acknowledged.
            builder
                .Start()
                .Parallel()
                    .Do(then => then
                        .StartWith(context => ExecutionResult.Next())
                        .WaitFor("ContactFormAcknowledged", (data, step) => data.Key.ToString())
                        .Output(step => step.Acknowledged, data => true)
                    )
                    .Do(then => then
                        .StartWith(context => ExecutionResult.Next())
                        .Recur(data => TimeSpan.FromSeconds(30), data => data.Acknowledged)
                        .Do(innerThen => innerThen
                            .StartWith(context => ExecutionResult.Next())
                            .Then<EmailAdmins>()
                                .Input(step => step.Key, data => data.Key)
                                .Input(step => step.From, data => data.From)
                                .Input(step => step.Subject, data => data.Subject)
                                .Input(step => step.Message, data => data.Message)
                        )
                    )
                .Join();
        }
    }

```
### Injecting dependencies into steps

Add both service and the workflow step as transients to the service collection when setting up your IoC container. (Avoid registering steps as singletons, since multiple concurrent workflows may need to use them at once.)

```
public void ConfigureServices(IServiceCollection services)
{
      services.AddWorkflow();
      services.AddControllersWithViews();
}
```

```
 host = app.ApplicationServices.GetService<IWorkflowHost>();
            host.RegisterWorkflow<ContactFormWorkflow, ContactForm>();
            host.Start();
```
### Parallel Paths Control Structures

Use the .Parallel() method to branch parallel tasks
```
            .Parallel()
                    .Do(then => then
                        .StartWith(context => ExecutionResult.Next())
                        .WaitFor("ContactFormAcknowledged", (data, step) => data.Key.ToString())
                        .Output(step => step.Acknowledged, data => true)
                    )
                    .Do(then => then
                        .StartWith(context => ExecutionResult.Next())
                        .Recur(data => TimeSpan.FromSeconds(30), data => data.Acknowledged)
                        .Do(innerThen => innerThen
                            .StartWith(context => ExecutionResult.Next())
                            .Then<EmailAdmins>()
                                .Input(step => step.Key, data => data.Key)
                                .Input(step => step.From, data => data.From)
                                .Input(step => step.Subject, data => data.Subject)
                                .Input(step => step.Message, data => data.Message)
                    )
```

### Do Error handling

Each step can be configured with it's own error handling behavior, it can be retried at a later time, suspend the workflow or terminate the workflow.
