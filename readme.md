We will be using Azure Portal online tools for this workshop. 

## Pre-requisites

1. Azure account to login to Azure [Portal](http://portal.azure.com)
2. Make sure you have access to [LUIS](https://www.luis.ai) using the same account as you used to log into Azure portal.


## General Notes

1.  We will use Node SDK for Bot Framework and Azure runtime in Azure to run the bot service. 
2.  Your code changes go live as the code changes are saved in online tools.


## Step 1: Create a Language Understanding bot with Bot Service

1.  In the Azure portal, select Create new resource in the menu blade and click See all.
2.  In the search box, search for Web App Bot.
3.  In the Bot Service blade, provide the required information. Make sure to click on Bot Template blade
    ,choose "Node.js" and "Language Understanding" Bot. click Create. 
    This creates and deploys the bot service and LUIS app to Azure.
4.  Set App name to your bot’s name. The name is used as the subdomain when your bot is deployed 
    to the cloud (for example, mynotesbot.azurewebsites.net).
    This name is also used as the name of the LUIS app associated with your bot. Copy it to use later, 
    to find the LUIS app associated with the bot.
5.  Select the subscription, resource group, App service plan, and location.
6.  Select the Language understanding (Node.js) template for the Bot template field.
7.  Check the box to confirm to the terms of service.
8.  Confirm that the bot service has been deployed.
9.  Click Notifications (the bell icon that is located along the top edge of the Azure portal). 
    The notification will change from Deployment started to Deployment succeeded.
10. After the notification changes to Deployment succeeded, click Go to resource on that notification.

## Step 2: Try the bot

1. Confirm that the bot has been deployed by checking the Notifications. The notifications will change from 
    Deployment in progress...to Deployment succeeded. Click Go to resource button to open the bot's resources blade.
2. Once the bot is registered, click Test in Web Chat to open the Web Chat pane. Type "hello" in Web Chat.

## Step 3: Modify the LUIS app

1. Log in to https://www.luis.ai using the same account you use to log in to Azure.
2. Click on My apps. 
3. In the list of apps, find the app that begins with the name specified in App name in the Bot 
Service blade when you created the Bot Service.
4. The LUIS app starts with 4 intents: Cancel: Greeting, Help, and None.
5. The following steps add the Note.Create, Note.ReadAloud, and Note.Delete intents:
6. Click on Prebuit Domains in the lower left of the page. Find the Note domain and click Add domain.

    This tutorial doesn't use all of the intents included in the Note prebuilt domain. In the Intents page, click on
    each of the following intent names and then click the Delete Intent button.
    
    **Note.ShowNext,
    Note.DeleteNoteItem,
    Note.Confirm,
    Note.Clear,
    Note.CheckOffItem,
    Note.AddToNote**
    
    The only intents that should remain in the LUIS app are the following:
    
    **Note.ReadAloud,
    Note.Create,
    Note.Delete,
    None,
    Help,
    Greeting,
    Cancel**

6.  Click the Train button in the upper right to train your app.
7.  Click PUBLISH in the top navigation bar to open the Publish page. Click the Publish to production slot button. 
    After successful publish, a LUIS app is deployed to the URL displayed in the Endpoint column in the Publish App 
    page, in the row that starts with the Resource Name Starter_Key. 
    
    The URL has a format similar to this example: 
    https://westus.api.cognitive.microsoft.com/luis/v2.0/apps/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx?subscription-key=
    xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx&timezoneOffset=0&verbose=true&q=. 
    
    The app ID and subscription key in this URL 
    are the same as LuisAppId and LuisAPIKey in ** App Service Settings > ApplicationSettings > App settings **

## Step 4: Modify the bot code

1. Click Build and then click Open online code editor.
2. In the code editor, open app.js. and review the code.

## Step 5: Edit the default message handler

1. The bot has a default message handler. Edit it to match the following:
````JavaScript
    // Create your bot with a function to receive messages from the user.
    // This default message handler is invoked if the user's utterance doesn't
    // match any intents handled by other dialogs.
    var bot = new builder.UniversalBot(connector, function (session, args) {
        session.send("Hi... I'm the note bot sample. I can create new notes, read saved notes to you and 
        delete notes.");
    
       // If the object for storing notes in session.userData doesn't exist yet, initialize it
       if (!session.userData.notes) {
           session.userData.notes = {};
           console.log("initializing userData.notes in default message handler");
       }
    });
````
## Handle the Note.Create intent
````JavaScript
// CreateNote dialog
bot.dialog('CreateNote', [
    function (session, args, next) {
        // Resolve and store any Note.Title entity passed from LUIS.
        var intent = args.intent;
        var title = builder.EntityRecognizer.findEntity(intent.entities, 'Note.Title');

        var note = session.dialogData.note = {
          title: title ? title.entity : null,
        };
        
        // Prompt for title
        if (!note.title) {
            builder.Prompts.text(session, 'What would you like to call your note?');
        } else {
            next();
        }
    },
    function (session, results, next) {
        var note = session.dialogData.note;
        if (results.response) {
            note.title = results.response;
        }

        // Prompt for the text of the note
        if (!note.text) {
            builder.Prompts.text(session, 'What would you like to say in your note?');
        } else {
            next();
        }
    },
    function (session, results) {
        var note = session.dialogData.note;
        if (results.response) {
            note.text = results.response;
        }
        
        // If the object for storing notes in session.userData doesn't exist yet, initialize it
        if (!session.userData.notes) {
            session.userData.notes = {};
            console.log("initializing session.userData.notes in CreateNote dialog");
        }
        // Save notes in the notes object
        session.userData.notes[note.title] = note;

        // Send confirmation to user
        session.endDialog('Creating note named "%s" with text "%s"',
            note.title, note.text);
    }
]).triggerAction({ 
    matches: 'Note.Create',
    confirmPrompt: "This will cancel the creation of the note you started. Are you sure?" 
}).cancelAction('cancelCreateNote', "Note canceled.", {
    matches: /^(cancel|nevermind)/i,
    confirmPrompt: "Are you sure?"
});
````

Any entities in the utterance are passed to the dialog using the args parameter. The first step of the 
waterfall calls EntityRecognizer.findEntity to get the title of the note from any Note.Title entities in
the LUIS response. If the LUIS app didn't detect a Note.Title entity, the bot prompts the user for the 
name of the note. 

The second step of the waterfall prompts for the text to include in the note. Once the bot has the text 
of the note, the third step uses session.userData to save the note in a notes object, using the title as
the key. For more information on session.UserData see Manage state data.

## Step 6: Handle the Note.Delete intent

1.  Just as for the Note.Create intent, the bot examines the args parameter for a title. If no title is detected, 
    the bot prompts the user. 
        
    The title is used to look up the note to delete from session.userData.notes.

    Copy the following code and paste it at the end of app.js:

````JavaScript
// Delete note dialog
   bot.dialog('DeleteNote', [
       function (session, args, next) {
           if (noteCount(session.userData.notes) > 0) {
               // Resolve and store any Note.Title entity passed from LUIS.
               var title;
               var intent = args.intent;
               var entity = builder.EntityRecognizer.findEntity(intent.entities, 'Note.Title');
               if (entity) {
                   // Verify that the title is in our set of notes.
                   title = builder.EntityRecognizer.findBestMatch(session.userData.notes, entity.entity);
               }
               
               // Prompt for note name
               if (!title) {
                   builder.Prompts.choice(session, 'Which note would you like to delete?', 
                   session.userData.notes);
               } else {
                   next({ response: title });
               }
           } else {
               session.endDialog("No notes to delete.");
           }
       },
       function (session, results) {
           delete session.userData.notes[results.response.entity];        
           session.endDialog("Deleted the '%s' note.", results.response.entity);
       }
   ]).triggerAction({
       matches: 'Note.Delete'
   }).cancelAction('cancelDeleteNote', "Ok - canceled note deletion.", {
       matches: /^(cancel|nevermind)/i
   });
````
   The code that handles Note.Delete uses the noteCount function to determine whether the notes object contains 
   notes.

2. Paste the noteCount helper function at the end of app.js.
````JavaScript
    // Helper function to count the number of notes stored in session.userData.notes
    function noteCount(notes) {
    
        var i = 0;
        for (var name in notes) {
            i++;
        }
        return i;
    }
````
## Step 7: Handle the Note.ReadAloud intent
Copy the following code and paste it in app.js after the handler for Note.Delete:

````JavaScript
    // Read note dialog
    bot.dialog('ReadNote', [
        function (session, args, next) {
            if (noteCount(session.userData.notes) > 0) {
               
                // Resolve and store any Note.Title entity passed from LUIS.
                var title;
                var intent = args.intent;
                var entity = builder.EntityRecognizer.findEntity(intent.entities, 'Note.Title');
                if (entity) {
                    // Verify it's in our set of notes.
                    title = builder.EntityRecognizer.findBestMatch(session.userData.notes, entity.entity);
                }
                
                // Prompt for note name
                if (!title) {
                    builder.Prompts.choice(session, 'Which note would you like to read?', 
                    session.userData.notes);
                } else {
                    next({ response: title });
                }
            } else {
                session.endDialog("No notes to read.");
            }
        },
        function (session, results) {        
            session.endDialog("Here's the '%s' note: '%s'.", results.response.entity, session.userData.notes
            [results.response.entity].text);
        }
    ]).triggerAction({
        matches: 'Note.ReadAloud'
    }).cancelAction('cancelReadNote', "Ok.", {
        matches: /^(cancel|nevermind)/i
    });
````
        The session.userData.notes object is passed as the third argument to builder.Prompts.choice, so that the prompt 
        displays a list of notes to the user.

## Step 8: Review the Code

    Now that you've added handlers for the new intents, the full code for app.js contains the following:
````JavaScript
    var restify = require('restify');
    var builder = require('botbuilder');
    var botbuilder_azure = require("botbuilder-azure");
    
    // Setup Restify Server
    var server = restify.createServer();
    server.listen(process.env.port || process.env.PORT || 3978, function () {
       console.log('%s listening to %s', server.name, server.url); 
    });
      
    // Create chat connector for communicating with the Bot Framework Service
    var connector = new builder.ChatConnector({
        appId: process.env.MicrosoftAppId,
        appPassword: process.env.MicrosoftAppPassword,
        openIdMetadata: process.env.BotOpenIdMetadata 
    });
    
    // Listen for messages from users 
    server.post('/api/messages', connector.listen());
    
    /*----------------------------------------------------------------------------------------
    * Bot Storage: This is a great spot to register the private state storage for your bot. 
    * We provide adapters for Azure Table, CosmosDb, SQL Azure, or you can implement your own!
    * For samples and documentation, see: https://github.com/Microsoft/BotBuilder-Azure
    * ---------------------------------------------------------------------------------------- */
    
    var tableName = 'botdata';
    var azureTableClient = new botbuilder_azure.AzureTableClient(tableName, 
                           process.env['AzureWebJobsStorage']);
    var tableStorage = new botbuilder_azure.AzureBotStorage({ gzipData: false }, azureTableClient);
    
    // Create your bot with a function to receive messages from the user.
    // This default message handler is invoked if the user's utterance doesn't
    // match any intents handled by other dialogs.
    var bot = new builder.UniversalBot(connector, function (session, args) {
        session.send("Hi... I'm the note bot sample. I can create new notes, 
        read saved notes to you and delete notes.");
    
       // If the object for storing notes in session.userData doesn't exist yet, initialize it
       if (!session.userData.notes) {
           session.userData.notes = {};
           console.log("initializing userData.notes in default message handler");
       }
    });
    
    bot.set('storage', tableStorage);
    
    // Make sure you add code to validate these fields
    var luisAppId = process.env.LuisAppId;
    var luisAPIKey = process.env.LuisAPIKey;
    var luisAPIHostName = process.env.LuisAPIHostName || 'westus.api.cognitive.microsoft.com';
    
    const LuisModelUrl = 'https://' + luisAPIHostName + '/luis/v2.0/apps/' + luisAppId + '?subscription-key=
                         ' + luisAPIKey;
    
    // Create a recognizer that gets intents from LUIS, and add it to the bot
    var recognizer = new builder.LuisRecognizer(LuisModelUrl);
    bot.recognizer(recognizer);
    
    // CreateNote dialog
    bot.dialog('CreateNote', [
        function (session, args, next) {
            // Resolve and store any Note.Title entity passed from LUIS.
            var intent = args.intent;
            var title = builder.EntityRecognizer.findEntity(intent.entities, 'Note.Title');
    
            var note = session.dialogData.note = {
              title: title ? title.entity : null,
            };
            
            // Prompt for title
            if (!note.title) {
                builder.Prompts.text(session, 'What would you like to call your note?');
            } else {
                next();
            }
        },
        function (session, results, next) {
            var note = session.dialogData.note;
            if (results.response) {
                note.title = results.response;
            }
    
            // Prompt for the text of the note
            if (!note.text) {
                builder.Prompts.text(session, 'What would you like to say in your note?');
            } else {
                next();
            }
        },
        function (session, results) {
            var note = session.dialogData.note;
            if (results.response) {
                note.text = results.response;
            }
            
            // If the object for storing notes in session.userData doesn't exist yet, initialize it
            if (!session.userData.notes) {
                session.userData.notes = {};
                console.log("initializing session.userData.notes in CreateNote dialog");
            }
            // Save notes in the notes object
            session.userData.notes[note.title] = note;
    
            // Send confirmation to user
            session.endDialog('Creating note named "%s" with text "%s"',
                note.title, note.text);
        }
    ]).triggerAction({ 
        matches: 'Note.Create',
        confirmPrompt: "This will cancel the creation of the note you started. Are you sure?" 
    }).cancelAction('cancelCreateNote', "Note canceled.", {
        matches: /^(cancel|nevermind)/i,
        confirmPrompt: "Are you sure?"
    });
    
    // Delete note dialog
    bot.dialog('DeleteNote', [
        function (session, args, next) {
            if (noteCount(session.userData.notes) > 0) {
                // Resolve and store any Note.Title entity passed from LUIS.
                var title;
                var intent = args.intent;
                var entity = builder.EntityRecognizer.findEntity(intent.entities, 'Note.Title');
                if (entity) {
                    // Verify that the title is in our set of notes.
                    title = builder.EntityRecognizer.findBestMatch(session.userData.notes, entity.entity);
                }
                
                // Prompt for note name
                if (!title) {
                    builder.Prompts.choice(session, 'Which note would you like to delete?'
                                           , session.userData.notes);
                } else {
                    next({ response: title });
                }
            } else {
                session.endDialog("No notes to delete.");
            }
        },
        function (session, results) {
            delete session.userData.notes[results.response.entity];        
            session.endDialog("Deleted the '%s' note.", results.response.entity);
        }
    ]).triggerAction({
        matches: 'Note.Delete'
    }).cancelAction('cancelDeleteNote', "Ok - canceled note deletion.", {
        matches: /^(cancel|nevermind)/i
    });
    
    
    // Read note dialog
    bot.dialog('ReadNote', [
        function (session, args, next) {
            if (noteCount(session.userData.notes) > 0) {
               
                // Resolve and store any Note.Title entity passed from LUIS.
                var title;
                var intent = args.intent;
                var entity = builder.EntityRecognizer.findEntity(intent.entities, 'Note.Title');
                if (entity) {
                    // Verify it's in our set of notes.
                    title = builder.EntityRecognizer.findBestMatch(session.userData.notes, entity.entity);
                }
                
                // Prompt for note name
                if (!title) {
                    builder.Prompts.choice(session, 'Which note would you like to read?'
                    , session.userData.notes);
                } else {
                    next({ response: title });
                }
            } else {
                session.endDialog("No notes to read.");
            }
        },
        function (session, results) {        
            session.endDialog("Here's the '%s' note: '%s'.", results.response.entity
            , session.userData.notes[results.response.entity].text);
        }
    ]).triggerAction({
        matches: 'Note.ReadAloud'
    }).cancelAction('cancelReadNote', "Ok.", {
        matches: /^(cancel|nevermind)/i
    });
    
    
    // Helper function to count the number of notes stored in session.userData.notes
    function noteCount(notes) {
    
        var i = 0;
        for (var name in notes) {
            i++;
        }
        return i;
    }
````
## Step 9: Test the bot

In the Azure Portal, click on Test in Web Chat to test the bot. Try type messages like "Create a note", 
"read my notes", and "delete notes" to invoke the intents that you added to it. 

## Next steps
From trying the bot, you can see that the recognizer can trigger interruption of the currently active dialog. 
Allowing and handling interruptions is a flexible design that accounts for what users really do. 


## Step 10: Handle user Actions

Users commonly attempt to access certain functionality within a bot by using keywords like "help", "cancel", 
or "start over." Users do this in the middle of a conversation, when the bot is expecting a different response. 

By implementing actions, you can design your bot to handle such requests more gracefully. The handlers will examine
user input for the keywords that you specify, such as "help", "cancel", or "start over," and respond appropriately.

## Step 11: Bind actions to dialog

Either user utterances or button clicks can trigger an action, which is associated with a dialog. If matches is 
specified, the action will listen for the user to say a word or a phrase that triggers the action. The matches 
option can take a regular expression or the name of a recognizer. To bind the action to a button click, use 
CardAction.dialogAction() to trigger the action.

Actions are chainable, which allows you to bind as many actions to a dialog as you want.

## Step 12: Bind a triggerAction

To bind a triggerAction to a dialog, do the following:

````JavaScript              
// Order dinner.
    bot.dialog('ReadAloud', [
        //...waterfall steps...
    ])
    // Once triggered, will clear the dialog stack and pushes
    // the 'orderDinner' dialog onto the bottom of stack.
    .triggerAction({
        matches: /^nevermind$/i
    });
````

## Publish your Bot

A channel is the connection between the Bot Framework and communication apps. You configure a bot to connect to the 
channels you want it to be available on. For example, a bot connected to the Skype channel can be added to a contact 
list and people can interact with it in Skype.

The publishing process is different for each channel.By default your Bot comes with Web Chat and Skype channels 
built into it. 

For this workshop, we will test it with Skype channel.

1. In the Bot Service blade, click Channels under Bot Management.
2. You will see that Skype and Web Chat are already connected.
3. Click on 'Skype' which will take you to a page to login to skype app. 
4. Use your personal credentials to connect. 
5. Once connected, you will see the bot in your contact list. Start conversing and have fun!


## How to run this locally on your machines 

If you want to run this in your IDE then you can download the source code from Azure Portal.


## Build and debug
1. In the Azure Portal open your Web App Bot and Click on Build under 'BOT MANAGEMENT'.
2. Click on 'Download zip file' to download the source code.
3. download source code zip and extract source in local folder.
4. open the source folder in  your IDE.
5. make code changes.
6. download and run [botframework-emulator](https://emulator.botframework.com/)
7. connect the emulator to http://localhost:3987

