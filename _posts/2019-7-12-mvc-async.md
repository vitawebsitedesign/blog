Converting a MVC page to load content asynchronously
Using jQuery.load(...) and PartialView, we can load stuff async.

Parallelism options
Since jQuery.load is XHR via jQuery.Deferred(), we can run multiple calls.
If you dont need to support browsers, you can also use promises or async/await pattern

Sync MVC example (w/ existing partial)
MyController.cs:
[HttpGet]
public async Task<ActionView> Index() {
  return View("Index", await GetModel());
}

private async Task<MyModel> GetModel() {
  await Task.Delay(1000);
  return new MyModel() {
    apple = 1,
    taco = 2
  };
}

Index.cshtml:
@{ Html.RenderAction("_MyPartial1"); }
@{ Html.RenderAction("_MyPartial2"); }

_MyPartial1.cshtml:
@Model.Apple

_MyPartial2.cshtml:
@Model.Taco

Conversion process
1. Split GetModel code into PartialView routes
The View(...) overload we'll use is string, object.
PartialView inherits from ActionView. Always use the most base return signature possible.
If you need state between form submits, you'll also need an model param in Index route.

[HttpGet]
public async Task<ActionView> Index() {
  return View("Index", await GetModel());
}

[HttpGet]
public async Task<ActionResult> GetAppleModel() {
  await Task.Delay(1000);
  return new AppleModel() {
    Apple = 1
  };
}

[HttpGet]
public async Task<ActionResult> GetTacoModel() {
  await Task.Delay(1000);
  return new TacoModel() {
    Taco = 2
  };
}

Lifecycle:
initial page load
index route
ajax calls routes
routes fetch data asynchronously in parallel
jQuery.deferred done
partial html rendered
form submit, which may start process from beginning again (index route), or just a route if using partial update

2. Call routes via jQuery.load
It's important to note that $.ready cant be relied on as a hook for partials. Usually the document loads BEFORE partials load.
This means we need to use <script></script> inside partial (instead of @section scripts { }), where $.ready is still used (with the callback firing immediately)
This applies to any JS that loads stuff at runtime

Example (pre ES6):
@section scripts {
<script>
	$(document).ready(function() {
		$('.content-apple').load('GetAppleModel', 'MyController');
		$('.content-taco').load('GetTacoModel', 'MyController');
	});
</script>
}

We can make this more strongly typed via razor engine + c# nameof:
@section scripts {
<script>
	$(document).ready(function() {
		$('.content-apple').load(@nameof(MyController.GetAppleModel), @nameof(MyController));
		$('.content-taco').load(@nameof(MyController.GetTacoModel), @nameof(MyController));
	});
</script>
}

This way, if we rename controller or method, razor is also updated.
Since code is always growing due to businesses forcing new requirements to out-compete other businesses for profit, code maintainability and productivity are important factors. We can improve those code-ilities with code like this.

How far can i take auto-properties after converting?
Once routes with parameters come into play, MVC cant auto-map URL parameters to route parameter objects, if the model property does not have private setter.
Model properties should always have getters AND setters where required. Otherwise remove property setters.

Should i make query params strongly typed too?
Consider:
$('.content').load(@nameof(MyController.GetModel)?Model.Apple=1, @nameof(MyController));

Assuming we've used a Model predirective in our cshtml.
Here, Apple is a property in a POCO 

But models are usually POCO's when content is loaded synchronously.
Once code is changed to load content asynchronously, we have more routes.
And this means those big monolithic models now become "complex" & nested models.

Example:
public class IndexModel {
	public int Apple {get;set;}
}


Now consider this:
public class AppleModel {
	public int Apple {get;set;}
}
public class IndexModel {
	public AppleModel AppleModel {get;set;}
}

MVC auto-maps complex model properties to URL params by using dot syntax. For a GET route, this would be:
/root/MyController/GetAppleModel?AppleModel.Apple=1

Making this strongly typed means making the $.load like:
$.load(@nameof(MyController.GetAppleModel)?@(nameof(IndexModel.AppleModel)).@nameof(AppleModel.Apple)=@Model.AppleModel.Apple

Here, strongly-typed code has actually degraded maintainability & productivity due to lower readability.
Sticking to non-strongly-typed strings are probably better for complex POCOs:
$.load(@nameof(MyController.GetAppleModel)?AppleModel.Apple=@Model.AppleModel.Apple

And only use strongly typed JS URL parameters for POCO's.
