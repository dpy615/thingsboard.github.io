---
layout: docwithnav
assignees:
- vbabak
title: Extensions Development Guide - Rules and Plugins

---

* TOC
{:toc}

#### Introduction

**Thingsboard extensions** are additional modules that could be developed and added to the platform to provide new functionality to cover your use cases.
As an example, you can easily add your custom **extension** that could send messages from **Thingsboard** to any other external system, 
additionally applying your specific business logic calculations before send.
 
#### Step 0. Read the docs
  
We recommend you to attentively review the [**Rule Engine**](/docs/user-guide/rule-engine/) documentation before you proceed. 
A lot of examples in this guide will be based on the existing rule engine components (extensions).
 
#### Step 1. Choose Extension type

We assume you already understand the difference between filter, processor, action and plugin.
If you want to add new plugin, most likely you will also need to add one or more actions to it.

**Example 1**: To add a plugin that sends and SMS using some external services, you will need to add two sibling extensions: SMS Action and Plugin.
SMS Plugin will contain logic for communication with external service. 
SMS Action will contain logic how to map telemetry data and device attributes to the actual message and phone number.

**Example 2**: To add a plugin that aggregates incoming data within certain time-window you need to develop the Plugin, 
but you are able to reuse existing [Telemetry action](/docs/reference/actions/telemetry-plugin-action/) to forward telemetry data to your plugin.
 
#### Step 2. Choose Stateful vs Stateless extensions
 
Extensions that are running inside Thingsboard server may be stateful or stateless. 
There is no differences in the interface itself, but the complexity of implementation may be very different. 

The most important part to understand is about clustering mode. 
Each Thingsboard service launches separate instance of the same plugin and rule. 
If one service goes down due to hardware or network failure there are multiple options how to react. 
But in general case you need the system to be working constantly and without any affect on the ned user.
  
**Example 1**: SMS plugin is stateless. You just receive and send SMS. 
If you decide to add the SMS de-duplication logic it is still stateless, but the duplication check will consume some resources (light-weight transaction in Cassandra DB, etc.)
 
**Example 2**: Data aggregation for multiple devices is stateful. And also required cluster communication between plugin instances.
For example, you can aggregate data on each Thingsboard node separately and periodically send this data to a single node in the cluster to do total aggregation.
But how to avoid data losses when one of the nodes died? There are multiple options and you need to figure out what is the best option in your case.

#### Step 3. Design API

This step is applicable for plugins. Plugin is able to provide following APIs:

 - REST API - to receive API calls from external systems.
 - WebSocket API - to receive API calls from external systems.
 - RPC API - to exchange messages between multiple instances of the same Plugin in the cluster mode.
 
Please review Plugin [**interface**](https://github.com/thingsboard/thingsboard/blob/master/extensions-api/src/main/java/org/thingsboard/server/extensions/api/plugins/Plugin.java) and implementations for more details.
We recommend to design API calls first and then proceed to implementation.

#### Step 4. Add maven module



Custom extensions are separate *maven* modules in **Thingsboard** [github repository](https://github.com/thingsboard/thingsboard/tree/master/extensions).
New *maven* module must be created inside *extensions* module to add new extension.
Then this new *maven* module should be added as dependency to the *application* module dependencies.

*** TODO - ADD DETAILS REGARDING WHERE INCLUDE THIS  ***

During the start-up **Thingsboard** parses *classes* inside classpath, finds annotated with custom *Thingsboard Plugin and Action Annotations* and adds new extension to the platform.

Once it's done you are able to use your custom extension as any other extension in the system.

#### Step 4. Design and add configuration schema

#### Step 5. Implement corresponding interface

#### Step 6. Deploy and verify



#### API

To properly construct new extension you'll need to develop **Action** and **Plugin** functionality using predefined set of *Classes*, *Interfaces* and *Annotations*.
As well, you'll need to provide JSON descriptor for the UI forms of the **Plugin** and **Action**.

We'll provide details how to create new extensions to the system using as a reference [REST API Call Extension](/docs/reference/plugins/rest/) that is already placed to the system.

Plugin is responsible for sending HTTP requests to specific endpoints and code for this plugin is located [here](https://github.com/thingsboard/thingsboard/tree/master/extensions/extension-rest-api-call/src/main/java/org/thingsboard/server/extensions/rest).

#### Server-side development

Server-side development contains of two parts - **Action** and **Plugin** coding.

![image](/images/user-guide/contribution/action-plugin-communication.png)

##### Action guide development

Action is responsible for processing messages that have been delivered from the devices (telemetry or attribute data) and converting it into **RuleToPluginMsg** object that is going to be processed by **Plugin**.
Here are particular samples:

    - in case of Email extension, action will create email object
    - in case of Kafka extenstion, action will create message object that will be published to Kafka topic
    - in case of REST API Call extenstion, action will create object that contains information reqarding REST request to specific endpoint
    - etc.

First, you'll need to create class that implements **org.thingsboard.server.extensions.api.plugins.PluginAction** interface.

This class is the core of your **Action** and here you should implement logic where you'll create object that is going to be passed to **Plugin** for processing.

It must be annotated with **org.thingsboard.server.extensions.api.component.Action** so **Thingsboard** will correctly process this class during start-up.

In case of **REST API Call** extension sample class is **org.thingsboard.server.extensions.rest.action.RestApiCallPluginAction**.
In this extension **Action** is responsible for creating object that contains information regarding REST request - body, http method, result code, etc.

Object that **Action** creates is a simple POJO class that holds needed information.

In case of **REST API Call** extension sample class is **org.thingsboard.server.extensions.rest.action.RestApiCallActionPayload**.
It holds information regarding REST request. This class must implements **Serializable** interface to be able to be received by **Plugin**.

*** TODO - ADD WHAT IS RestApiCallActionMsg ***

To finish with **Action** implementation you'll need to add POJO class that contains configuration of the *UI Action* form.
It basically contains fields that are mapped to the UI form of **Action**. In this form user will provide details regarding configuration of your **Action**.

In case of **REST API Call** extension sample class is **org.thingsboard.server.extensions.rest.action.RestApiCallPluginActionConfiguration**.
It holds information regarding template of the REST request, expected result code etc.
Please refer to the [UI development part](/docs/user-guide/contribution/extensions-development/#ui-development) to get details regarding how *UI* forms are generated.


##### Plugin guide development

Plugin is responsible for receiving **RuleToPluginMsg** object that was created by **Action** and process it accordingly. This processing depends on your needs and could be anything:

    - send emails
    - send messages to external system
    - send messages to Thingsboard instance
    - etc.

First, you'll need to create class that implements **org.thingsboard.server.extensions.api.plugins.Plugin** interface.

This class is the core of your **Plugin** and here you should implement logic regarding to correctly init plugin.

It must be annotated with **org.thingsboard.server.extensions.api.component.Plugin** so **Thingsboard** will correctly process this class during start-up.

In case of **REST API Call** extension sample class is **org.thingsboard.server.extensions.rest.plugin.RestApiCallPlugin**.
In this extension **Plugin** is responsible for setting URL and headers for the REST request.

Secondly, you need to add class that implements **org.thingsboard.server.extensions.api.plugins.handlers.RuleMsgHandler** interface.

This class actually doing 'real' work - send messages to http endpoints, kafka topics, etc.

In case of **REST API Call** extension sample class is **org.thingsboard.server.extensions.rest.plugin.RestApiCallMsgHandler**.
This message handler is responsible for creating **Spring REST template** and sending HTTP request.

To finish with **Plugin** implementation you'll need to add POJO class that contains configuration of the *UI Plugin* form.
It basically contains fields that are mapped to the UI form of **Plugin**. In this form user will provide details regarding configuration of your **Plugin**.

In case of **REST API Call** extension sample class is **org.thingsboard.server.extensions.rest.plugin.RestApiCallPluginConfiguration**.
It holds information regarding host of the REST endpoint, base path, authentication details, etc.
Please refer to the [UI development part](/docs/user-guide/contribution/extensions-development/#ui-development) to get details regarding how *UI* forms are generated.


#### UI development

**Plugin** and **Action** UI forms are auto-generated using *react-schema-form* [builder](http://networknt.github.io/react-schema-form/).
It allows to build UI forms from pre-defined JSON configuration.
JSON file contains description for the *schema* and *form* components.
*Schema* and *form* components are used to describe what type of elements are going to be on the form and how exactly they are located.

**NOTE**
*For the drop-down box with pre-defined set of options, but without multi-select feature use this [sample configuration](http://networknt.github.io/react-schema-form-rc-select/).*

For the extension you'll need to define two UI configurations - **Plugin** and **Action** JSON configurations.

#### Plugin JSON configuration

This JSON file contains schema and form definition for the **Plugin** form generation. Must be located in the *resources* folder. We'll later refer to this file as **${PLUGIN_FORM_DESCRIPTOR_JSON_FILE}**.

#### Action JSON configuration

This JSON file contains schema and form definition for the **Action** form generation. Must be located in the *resources* folder. We'll later refer to this file as **${ACTION_FORM_DESCRIPTOR_JSON_FILE}**.

Here is sample of the **REST API Call** Plugin and Action form configuration files:

![image](/images/user-guide/contribution/plugin-action-json-location.png)

Once appropriate *JSON* files are created and placed in *resources* project directory they should be provided to the **Plugin** and **Action** annotations accordingly:

```java
@Plugin(name = "${PLUGIN_NAME}", actions = {"${ACTION_CLASS_NAME}".class},
        descriptor = "${PLUGIN_FORM_DESCRIPTOR_JSON_FILE}", configuration = "${PLUGIN_CONFIGURATION_CLASS_NAME}".class)
```

```java
@Action(name = "${ACTION_NAME}",
        descriptor = "${ACTION_FORM_DESCRIPTOR_JSON_FILE}", configuration =  "${ACTION_CONFIGURATION_CLASS_NAME}".class)
```

#### Extension testing and verification

Now it's time to start **Thingsboard** and verify that your **Extension** works as you expected.