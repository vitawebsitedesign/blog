---
title: Button.OnClientClick vs ScriptManager.RegisterOnSubmitStatement
layout: post
author: Michael Nguyen
---
When you need to run client-side JS in a filthy WebForms page, we got 2 primary paths - ScriptManager or OnClientClick.

There seems to be some confusion though, that Button.OnClientClick can be used in place of ScriptManager.RegisterOnSubmitStatement & vice versa.

People looking at these kinds of StackOverflow posts are in either 1 of 3 situations:
* They need to run code before form submission
* They need to run code after form submission
* They dont care about the timing

For the last situation, `ScriptManager.RegisterStartupScript` is the most simplest & should be used.

But theres a thin line between other 2 scenarios and what is called for. And i think that explains why the StackOverflow posts often have comments from people sitting on both sides of the fence.

Brace yourselfves - let us dive deep into the exciting world of WebForms server-side html rendering.

## Button.OnClientClick() or ScriptManager.RegisterOnSubmitStatement()?
One popular option is to use ScriptManager.RegisterOnSubmitStatement.

The difference is that OnClientClick runs BEFORE validation, and RegisterOnSubmitStatement runs AFTER validation and BEFORE form submission.

Lets have a look at the rendered html that WebForms sends back.

### Button.OnClientClick
```asp
<asp:Button OnClientClick="return someField.value > 1" />
<asp:Button OnClick="MyButton_Click" />
```

generates to

```asp
<asp:Button onclick="DoPostBackWithOptions(...);return someField.value > 1" />
```

If we need to validate a form before posting it back, this aint too useful since the check runs after the postback.

But if we wanted to show some message like "Processing" after the form submits, OnClientClick is a good option.

### ScriptManager.RegisterOnSubmitStatement
Another popular option is ScriptManager.RegisterOnSubmitStatement, which may imply that it can be used in-place of Button.OnClientClick (and it can), but not in all situations:

```csharp
var script = "return someField.value > 1";
Page.ClientScript.RegisterOnSubmitStatement(this.GetType(), nameof(MyFunction), script);
```

```asp
<asp:Button OnClick="MyButton_Click" />
```

generates to

```asp
<form onSubmit="return someField.value > 1">
	<asp:Button onclick="DoPostBackWithOptions(...);" />
</form>
```

So instead of running after a postback, it hooks into `onSubmit` and we can terminate the continue to postback with whatever boolean is evaluated.

It's important to note that the button handler `MyButton_Click` will still fire after `Page_Load`, we are only controlling the form submission behaviour, not the `__doPostBack` or `DoPostBackWithOptions` logic.

## Which one to choose?
Instead of hitting up StackOverflow, firstly think of when your code needs to run.

If theres no specific timing needed, just run the script on page load.

If a specific timing is needed, and your situation has extra complexity for a number of reasons, i'd recommend reading the [ScriptManager API](https://docs.microsoft.com/en-us/dotnet/api/system.web.ui.scriptmanager?view=netframework-4.8) and see what it has on offer.