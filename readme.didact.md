# Kamelets on Kafka Demo

This tutorial is about using [Kamelets](https://camel.apache.org/camel-kamelets/latest/) (from Apache Camel) to connect systems to Apache Kafka.

## Requirements

1. An OpenShift 4.7 cluster and related `oc` (or `kubectl`) CLI
1. The AMQ Streams 1.7 (stable) operator installed in the cluster (global mode)
1. The Apache Camel K 1.4.0 (stable-1.4) operator installed in the cluster (global mode) and related `kamel` CLI

Make sure that you have all these tools installed and that you're connected to the cluster.

## Checking basic requirements

<a href='didact://?commandId=vscode.didact.validateAllRequirements' title='Validate all requirements!'><button>Validate all Requirements at Once!</button></a>

**OpenShift CLI ("oc")**

The OpenShift CLI tool ("oc") will be used to interact with the OpenShift cluster.

[Check if the OpenShift CLI ("oc") is installed](didact://?commandId=vscode.didact.cliCommandSuccessful&text=oc-requirements-status$$oc%20help&completion=Checked%20oc%20tool%20availability "Tests to see if `oc help` returns a 0 return code"){.didact}

*Status: unknown*{#oc-requirements-status}


**Connection to an OpenShift cluster**

You need to connect to an OpenShift cluster in order to run the examples.

[Check if you're connected to an OpenShift cluster](didact://?commandId=vscode.didact.requirementCheck&text=cluster-requirements-status$$oc%20get%20project$$NAME&completion=OpenShift%20is%20connected. "Tests to see if `kamel version` returns a result"){.didact}

*Status: unknown*{#cluster-requirements-status}

**Apache Camel K CLI ("kamel")**

You also need the Apache Camel K CLI ("kamel") in order to 
access all Camel K features.

[Check if the Apache Camel K CLI ("kamel") is installed](didact://?commandId=vscode.didact.requirementCheck&text=kamel-requirements-status$$kamel%20version$$Camel%20K%20Client&completion=Apache%20Camel%20K%20CLI%20is%20available%20on%20this%20system. "Tests to see if `kamel version` returns a result"){.didact}

*Status: unknown*{#kamel-requirements-status}

## Setting up Kafka

Using the OpenShift console, you can start afresh by **creating a new project** (namespace).

The "Installed Operators" tab will help you creating a new "Kafka" instance using the "AMQ Streams" operator. You can name the Kafka cluster "my-cluster" and use default settings.

Assuming that the project will be called "kafka-demo", you can now switch the CLI to that project:

`oc project kafka-demo` ([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=newTerminal$$oc%20project%20kafka-demo))

## Setting up Kamelets

The Kamelets used in this demo should be applied to the namespace, in order for the demo to use the right set of Kamelets.
To do so, you can apply all kamelets from the "kamelets" directory:

```
kubectl apply -f ./kamelets
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=newTerminal$$kubectl%20apply%20-f%20./kamelets))

## Register with external services

This tutorial makes use of external services. In order to run it, you need to follow these steps to prepare the external services for connection.

### Register a Twitter App

You need to create a standalone app on Twitter from the following link: https://developer.twitter.com/en/portal/projects-and-apps.

Once you've created the app, save the following information in the [secrets/twitter.properties](didact://?commandId=vscode.openFolder&projectFilePath=secrets/twitter.properties&completion=File%20opened "Opens the file"){.didact} file.

You can now create a secret containing those properties:

```
# remember to put your authorization tokens in the file first

kubectl create secret generic twitter-connection --from-file secrets/twitter.properties
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=newTerminal$$kubectl%20create%20secret%20generic%20twitter-connection%20--from-file%20secrets/twitter.properties))


Now that the secret is created, you can mark it with some labels so that it's automatically picked up by Camel K when using the `twitter-search-source` kamelet:

```
kubectl label secrets/twitter-connection camel.apache.org/kamelet=twitter-search-source
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=newTerminal$$kubectl%20label%20secrets/twitter-connection%20camel.apache.org/kamelet=twitter-search-source))


### Register a Telegram Bot

You need to create a telegram bot using [Telegram botfather](https://core.telegram.org/bots).

Once you've created the bot, get the authorization token and save it to [secrets/telegram.properties](didact://?commandId=vscode.openFolder&projectFilePath=secrets/telegram.properties&completion=File%20opened "Opens the file"){.didact} file.

To send message to your specific chat ID, you need to set it in the above file. Since most Telegram clients don't provide information about the chat IDs, we provide an
utility integration that allows you to know the chat ID.

To execute it, copy and paste the following command to the terminal (putting your own token):

```
kamel run utils/get-my-telegram-chat-id.yaml -p authorizationToken=put-your-token-here
```

After the integration is ready, you should be able to write to your bot using your Telegram app and the bot will reply with the ID corresponding to that specific chat room.

Put the chatId in the [secrets/telegram.properties](didact://?commandId=vscode.openFolder&projectFilePath=secrets/telegram.properties&completion=File%20opened "Opens the file"){.didact} file, then register it to the cluster.

You can now create a secret containing the Telegram properties:

```
# remember to put both authorizationToken and chatId in the file first

kubectl create secret generic telegram-connection --from-file secrets/telegram.properties
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=newTerminal$$kubectl%20create%20secret%20generic%20telegram-connection%20--from-file%20secrets/telegram.properties))


Now that the secret is created, you can mark it with some labels so that it's automatically picked up by Camel K when using the `telegram-sink` kamelet:

```
kubectl label secrets/telegram-connection camel.apache.org/kamelet=telegram-sink
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=newTerminal$$kubectl%20label%20secrets/telegram-connection%20camel.apache.org/kamelet=telegram-sink))

### Get your OpenAI token

You need to have access to OpenAI (beta) to run this demo as it is.

We're going to use classification, so you need to upload to OpenAI a file with classified examples, then reference the file in the openai.properties file.

Once you've the token to access the OpenAI APIs and the file ID, you can create a secret containing the OpenAI properties:

```
# remember to overwrite the file with the authorizationToken and the file ID first

kubectl create secret generic openai-connection --from-file secrets/openai.properties
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=newTerminal$$kubectl%20create%20secret%20generic%20openai-connection%20--from-file%20secrets/openai.properties))


Now that the secret is created, you can mark it with some labels so that it's automatically picked up by Camel K when using the `openai-classification-action` kamelet:

```
kubectl label secrets/openai-connection camel.apache.org/kamelet=openai-classification-action
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=newTerminal$$kubectl%20label%20secrets/openai-connection%20camel.apache.org/kamelet=openai-classification-action))

## Stream tweets to Kafka

All you need to do to stream data into Kafka is to create the binding:

```
kubectl apply -f twitter-to-kafka.yaml
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=newTerminal$$kubectl%20apply%20-f%20twitter-to-kafka.yaml))

## Have a look at the data

You can create a logging pod that connects to the cluster and look at the logs:

```
kamel bind kafka.strimzi.io/v1beta2:KafkaTopic:tweets?autoOffsetReset=earliest log:info
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=newTerminal$$kamel%20bind%20kafka.strimzi.io/v1beta2:KafkaTopic:tweets?autoOffsetReset=earliest%20log:info))

## Send sentiments to your Telegram chat

You can process data using OpenAI and send the result to telegram by applying the last binding:

```
kubectl apply -f kafka-to-telegram.yaml
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=newTerminal$$kubectl%20apply%20-f%20kafka-to-telegram.yaml))

