---
title: Working around WebForms Event Validation
layout: post
author: Michael Nguyen
---
WebForms Event Validation feature means that the form will only postback if the controls havent been tampered with (e.g.: someone setting malicious input via JavaScript).

On a more technical level, its basically a hash of all the trusted control values, which is then compared prior to postback.

The problem comes when you change to data after the page load, which is "outside" the values webforms trusts.

## Example
A common use-case is changing dropdown options after page load.

```c#
MyPage.aspx.cs:
Dropdown.DataSource = new List<int> { 1 };
```

```html
<asp:DropDownList ID="Dropdown" runat="server">
</asp:DropDownList>
```

Fine so far. And now lets change the dropdown options.

```javascript
<script>
	var option = $('<option/>', { value: 2 });
	$('#<%: Dropdown.ClientID %>').add(option);
</script>
```

This new selectable value "2" is "outside" the values webforms trusts.

So when you post 1, its all good. But then post back 2, and webforms will shout at the top of its lungs, saying event validation failed.

## Workaround
The main issue is that we dont want event validation.

But WebForms only allows us to change that at the site-wide and page level.

Cant change it at component-level, like for our dropdown (e.g.: some asp web control attribute like EnableEventValidation=False).

## Unofficial workaround?
Well, theres 3 options.

### 1. Flick off page-level event validation
Not recommended, its there to keep us safe

### 2. Add a class
Event validation is declared through SupportsEventValidation class attribute.

The attribute does not apply to children.

So you can just make your own Dropdown class that inherits from ASP's dropdown, then use that.

Creating a class that shouldnt be necessary seems messy.

### 3. Dont postback the control
By this, i mean putting the asp web control OUTSIDE the form that asp renders.

Usually you can shove it between the closing <form/> tag, and the closing <body/> tag.

This effectively mimicks 2, but in a cleaner fashion.