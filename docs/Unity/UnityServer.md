# Working with Unity server and client structure

Within this document there will be some explanation of how to work with Unity's build in messaging system.
We'll create a client and a server with both a GUI.

## Client

For our client we want to both receive and send messages.
We want to do the receiving of messages with delegates and as that's more complicated it will be done later.
This client will work on voting as I've done that before and don't want to make up a scenario.

### Sending messages

First of all to keep things organized, let's create a message class for sending a vote.

#### Message class

```cs
[Serializable]
public class VotingMessage
{
    public VotingMessage(int choice, int question)
    {
        Choice = choice;
    }

    public int Choice { get; private set; }

    public int Question { get; private set; }
}
```

---
NOTES: All messages have to be `Serializable`, this means they can be converted to a string and send to the server.

So far so good, `choice` is the decision our player made and `question` is which question they did that on.

You could also use strings if it's an answer with text or an open question.

#### Setting up your protocol

To make sure that all communication is safe and proper we want to work on a decent setup.

First of all we are going to use a `ClientScript.cs` file to send data to our server.

Our server needs to have an IP and Port so lets put that in the constructor.

```cs
public class ClientScript : MonoBehaviour
{
    [SerializedField]
    private string _ip;

    [SerializedField]
    private int _port;

    private NetworkClient _client;
}
```

`SerializedField` makes it editable in Unity so you can change the server port and IP however you want.

#### Connecting

Let's start off by connecting to the server.
This will be done in the start method

```cs
public void Start()
{
    _client = new NetworkClient();
    _client.Connect(_ip, _port);
}
```

Yes this is actually all that connecting is... for now.

#### Message ids

We are going to introduce an `enum` to keep track of what numbers we are using for what messages.

```cs
public enum MessageType
{
    VotingMessage = 1331
}
```

This could be any number you like, I just work down from `1331`.

So now to send a message with this, we will make a simple method.
This is done in `ClientScript.cs`

```cs
private void SendMessage(MessageType messageType, object payload)
{
    _client.Send(Convert.ToInt16(messageType), payload)
}
```

The convert to `Int16` aka `short` is needed as this is the only accepted type.

We can use this to send our message using one simple method.

```cs
public void SendVote(int choice, int question)
{
    SendMessage(MessageType.VotingMessage, new VotingMessage(choice, 
                question));
}
```

Aaaand that's sending a message.

### Server Receiving

Let's start receiving the messages on the server side.

#### Copying the classes

We create a new Unity project and grab both the `VotingMessage.cs` class and the `MessageType` enum.
These NEED to be the same between the Client and Server.

#### ServerScript base

Let's start off by creating a new script called `ServerScript.cs`.

```cs
public class ServerScript : MonoBehavior
{
    [SerializedField]
    public int _port;

    public void Start()
    {
        // Registers that when a message comes in, it's handled by the proper method.
        NetworkServer.RegisterHandler(Convert.ToInt16
            (MessageType.VotingMessage), OnVotingReceived);
        NetworkServer.Listen(_port);
    }

    // Handles the incoming message.
    private void OnVotingReceived(NetworkMessage networkMessage)
    {
        var votingMessage = networkMessage.ReadMessage<VotingMessage>();
        // Do something with the voting message.
    }
}
```

`NetworkServer.RegisterHandler()` registers a method to call when a message is received of a particular type.
So when Unity receives a message with the number `1331` it will call the `OnVotingReceived()` method.

**All of these messages need to have a parameter `NetworkMessage`**.

`ReadMessage<Type>()` converts the gained message into the type you want.
For us this is the `VotingMessage.cs` class.