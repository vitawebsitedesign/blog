---
title: MVC Html.CheckboxFor Shenanigans
layout: post
---

Html.DropDownFor() generates below html:
```html
<select name="some-property" value="taco" />
```

Html.CheckBoxFor()?:
```html
<input name="some-property" type="checkbox" value="false" />
<input name="some-property" type="hidden" value="false" />
```

MVC generates 2 html elements - the actual vanilla HTML checkbox, and a HTML hidden field to support postbacks.

But why 2? Especially considering that they have same name attribute.

## Postbacks
When i say "postback", i mean form submission.

For checkboxes, both fields will be submitted, even though they have the same name.

### Unchecked checkbox + form submit
Generated URL: "SomeProperty=False"

MVC only submits the hidden field and ignores the actual checkbox element.

### Checked checkbox + form submit
Generated URL: "SomeProperty=True&SomeProperty=False"

MVC submits both fields, and MVC (on server-side) receives "true;false" in dictionary.

MVC then extracts the "real" checkbox value from this string. In this case, "true".
