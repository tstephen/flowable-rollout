# HOWTO rollout to Flowable process application with forms

This HOWTO assumes you have a development environment and a separate staging environment and need to perform the rollout from one the next.

For development it is convenient to use the [flowable/all-in-one](https://hub.docker.com/r/flowable/all-in-one) container because it has all the modeling tools as well as the runtime engines. Everything is stored in the embedded H2 database, which is fine for development needs.

However for staging, production etc. we will use the [flowable/flowable-rest](https://hub.docker.com/r/flowable/flowable-rest) container. Obviously in real-life we would externalise the database but for convenience here we'll continue to use the H2 embedded one.

We need a process and its associated forms to demonstrate how to perform the rollout. It doesn't matter what the process andd form actually do so in time-honoured tradition we'll use a very simple hello world app that captures the user's name in the process starter form and then echos it back in the user task form.

The process looks like this:
  ![start event -> user task -> end event](src/images/HelloWorldProcess.png "Hello world process")
The start event has a form named 'StartHelloWorld' to capture the user's name and the user task displays it in a form named 'SayHello'.

## Download the BPMN file

Unlike to forms it _is_ possible to do this through the UI but for completeness we'll do it at the command line because we're likely to want to automate it anyway.

First let's list the process definitions in the development environment:
```
curl --user admin:test http://localhost:8080/flowable-task/process-api/repository/process-definitions/ | python3 -m json.tool | grep HelloWorld
```

You'll see I'm piping the output through a little python script to format it more readably and then grepping for the HelloWorld process. If you do the same you should see something like this:
```
    "id": "HelloWorld:1:b33721df-6cc6-11e9-b2c1-0242ac110002",
    "url": "http://localhost:8080/flowable-task/process-api/repository/process-definitions/HelloWorld:1:b33721df-6cc6-11e9-b2c1-0242ac110002",
    "key": "HelloWorld",
    "resource": "http://localhost:8080/flowable-task/process-api/repository/deployments/b23e910c-6cc6-11e9-b2c1-0242ac110002/resources/HelloWorld.bpmn",
    "diagramResource": "http://localhost:8080/flowable-task/process-api/repository/deployments/b23e910c-6cc6-11e9-b2c1-0242ac110002/resources/HelloWorld.HelloWorld.png",
```

The thing to note is that the resource url: `http://localhost:8080/flowable-task/process-api/repository/deployments/b23e910c-6cc6-11e9-b2c1-0242ac110002/resources/HelloWorld.bpmn`

Changing `resources` path to `resourcedata` allows us to get the BPMN definition like this:
```
curl --user admin:teshost:8080/flowable-task/process-api/repository/deployments/b23e910c-6cc6-11e9-b2c1-0242ac110002/resourcedata/HelloWorld.bpmn > HelloWorld.bpmn
```

You can now place this under source control and then we'll look at how to deploy it to the staging environment.

## Import the process into the staging environment

Deploying the process is a simple matter of:
```
curl -v --user rest-admin:test -F "file=@HelloWorld.bpmn" http://localhost:8090/flowable-rest/service/repository/deployments/
```

Now we're ready to move on to the forms.

## Download the forms

Let's start by listing the form definitions in our engine:

```
curl --user admin:test http://localhost:8080/flowable-task/form-api/form-repository/form-definitions/ | python3 -m json.tool
```

This gives us the following JSON and you can see our forms `SayHello` and `StartHelloWorld` listed. However, there a no form fields here!:
```json
{
"data": [
    {
        "id": "b38b5c63-6cc6-11e9-b2c1-0242ac110002",
        "url": "http://localhost:8080/flowable-task/form-api/form-repository/form-definitions/b38b5c63-6cc6-11e9-b2c1-0242ac110002",
        "category": null,
        "name": "SayHello",
        "key": "SayHello",
        "description": null,
        "version": 1,
        "resourceName": "form-SayHello.form",
        "deploymentId": "b3413400-6cc6-11e9-b2c1-0242ac110002",
        "tenantId": ""
    },
    {
        "id": "b38b5c64-6cc6-11e9-b2c1-0242ac110002",
        "url": "http://localhost:8080/flowable-task/form-api/form-repository/form-definitions/b38b5c64-6cc6-11e9-b2c1-0242ac110002",
        "category": null,
        "name": "StartHelloWorld",
        "key": "StartHelloWorld",
        "description": null,
        "version": 1,
        "resourceName": "form-StartHelloWorld.form",
        "deploymentId": "b3413400-6cc6-11e9-b2c1-0242ac110002",
        "tenantId": ""
    }
],
"total": 2,
"start": 0,
"sort": "name",
"order": "asc",
"size": 2
}
```
    
However, now that we have the form id we can get the form definition _AND_ model. I'll just show you the first here:
```
curl --user admin:test http://localhost:8080/flowable-task/form-api/form-repository/form-definitions/b38b5c64-6cc6-11e9-b2c1-0242ac110002/model | python3 -m json.tool > StartHelloWorld.form
```

Note in the results we have internal information such as the id and url as well as the field list we're looking for: 
  ```json
  {
    "id": "b38b5c64-6cc6-11e9-b2c1-0242ac110002",
    "name": "StartHelloWorld",
    "key": "StartHelloWorld",
    "version": 1,
    "url": "http://localhost:8080/flowable-task/form-api/form/model",
    "fields": [
        {
            "fieldType": "FormField",
            "id": "name",
            "name": "Name",
            "type": "text",
            "value": null,
            "required": false,
            "readOnly": false,
            "overrideId": false,
            "placeholder": "Please enter the name we should call you by",
            "layout": null
        }
    ],
    "outcomes": []
  }
```

**If you try to deploy the above JSON unchanged you will get an error!**
```
Caused by: com.fasterxml.jackson.databind.exc.UnrecognizedPropertyException: Unrecognized field "category" (class org.flowable.form.model.SimpleFormModel), not marked as ignorable (7 known properties: "outcomes", "version", "fields", "name", "description", "key", "outcomeVariableName"])
```

Basically, Jackson is telling us to remove these attributes:
- category
- id
- url
    
Having done so we can deploy the form definition to our staging environment like this:
```
curl -v --user rest-admin:test -F "file=@StartHelloWorld.form" http://localhost:8090/flowable-rest/form-api/form-repository/deployments/
```

## Introspect this form and start a process instance  

If you've followed this far then you know the form key, but lets assume you only know the process key and need to introspect the form definition. 

In the staging environment, use the `/process-definitions` endpoint to find the latest `processDefinitionId` as we did previously in the development environment. 

Get the form key to use to start the process from this process definition id:
```
curl --user rest-admin:test http://localhost:8090/flowable-rest/service/form/form-data/?processDefinitionId=HelloWorld:1:db82ba53-6cc7-11e9-a68d-0242ac110003
```
The result should look something like this:
```json
{"formKey":"StartHelloWorld","deploymentId":"db7f5eea-6cc7-11e9-a68d-0242ac110003","processDefinitionId":"HelloWorld:1:db82ba53-6cc7-11e9-a68d-0242ac110003","processDefinitionUrl":"http://localhost:8090/flowable-rest/service/repository/process-definitions/HelloWorld:1:db82ba53-6cc7-11e9-a68d-0242ac110003","taskId":null,"taskUrl":null,"formProperties":[]}
```
  
Now we have the form key, we need to find the full definition. Note the double-quotes around the GET URL to ensure both params are used:
```
curl -v --user rest-admin:test "http://localhost:8090/flowable-rest/form-api/form-repository/form-definitions/?keyLike=StartHelloWorld&latest=true" | python3 -m json.tool
```

The result will be something like this:
```json
{
"data": [
    {
        "id": "b340f78b-6ccf-11e9-a68d-0242ac110003",
        "url": "http://localhost:8090/flowable-rest/form-api/form-repository/form-definitions/b340f78b-6ccf-11e9-a68d-0242ac110003",
        "category": null,
        "name": "StartHelloWorld",
        "key": "StartHelloWorld",
        "description": null,
        "version": 2,
        "resourceName": "StartHelloWorld.form",
        "deploymentId": "b33fbf09-6ccf-11e9-a68d-0242ac110003",
        "tenantId": ""
    }
],
"total": 1,
"start": 0,
"sort": "name",
"order": "asc",
"size": 1
}
```

Finally we can get form _model_ containing the fields needed to render the form:
```
curl --user rest-admin:test http://localhost:8090/flowable-rest/form-api/form-repository/form-definitions/b340f78b-6ccf-11e9-a68d-0242ac110003/model | python3 -m json.tool 
```

Since we've seen what that looks like above I'll not repeat it again.

So to recap, we now have a *scriptable* way to extract the latest models from a development environment and inject them into a fresh environment.