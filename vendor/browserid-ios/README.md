>_Please note that this code/project is work in progress. It currently works good enough for accounts that just go through persona.org but it has problems when alternative Identity Providers are in play._
>_At this point it is not recommended to use this code, except to get an idea of what BrowserID will look like._

## Introduction

This project contains sources to integrate BrowserID in an iOS application.

## Theory of Operation

### In a fully native application

First, implement the BrowserIDViewControllerDelegate methods:

```
#import "BrowserIDController+UIKit.h"

// Called when the user canceled the BrowserID process by hitting the Cancel button

- (void) browserIDViewControllerDidCancel: (BrowserIDViewController*) browserIDViewController
{
    [browserIDViewController dismissModalViewControllerAnimated: YES completion: NULL];
}

// Called when the user successfully went through the BrowserID dialog. The assertion
// should be verified in your server. Do not call the BrowserID hosted verifier from your application
// as that will allow users to spoof a login.

- (void) browserIDViewController: (BrowserIDViewController*) browserIDViewController 
    didSucceedWithAssertion: (NSString*) assertion;
{
    [browserIDViewController dismissModalViewControllerAnimated: YES completion: NULL];
    // Pass the assertion to your server to verify it
}
```

Then start the BrowserID process by creating a BrowserIDViewController and displaying it.

```
    NSURL* origin = [NSURL URLWithString: @"http://my.app/"]; // Your server's root URL
    BrowserIDViewController* browserIDViewController = [BrowserIDViewController new];
    browserIDViewController.origin = origin;
    browserIDViewController.delegate = self;
    [browserIDController presentModalInController: myViewController]
```
You can also let the BrowserIDViewController call your verifier. Simply set the `verifier` property to the location of your own verifier.

```
    browserIDViewController.verifier = [NSURL URLWithString: @"https://my.app/verify"];
```

Then instead of the `browserIDViewController:didSucceedWithAssertion:` delegate method, you implement the `browserIDViewController:didSucceedVerificationWithReceipt:` and `browserIDViewController:didFailVerificationWithError` methods.

The `BrowserIDViewController` will call the verifier endpoint with a `POST` method that has the assertion in the the request body. The verify method is expected to return a JSON structure. The parsed JSON will be given to your in the `browserIDViewController:didSucceedVerificationWithReceipt:` delegate method.

The verify method on your server is also a good place to log the user in. For example by setting a cookie or by generating a session id that you pass back.

Here is a very simple example that is written using the Python Bottle web framework:

```
#
# Called by the native application when it wants to verify the BrowserID token. Will
# return 200 and the user's email address in the body if the verification is succesfull,
# 500 if the request failed or 401 Unauthorized if the verification fails.
#

@route('/verify', method='POST')
def verify():
    data = { 'assertion': request.body.read(), 'audience': "http://localhost:8080"}
    post_response = requests.post('https://browserid.org/verify', data = data, timeout = 5)
    if post_response.status_code != 200:
        abort(500, "BrowserID Error")
    else:
        receipt = json.loads(post_response.content)
        if receipt['status'] == 'okay':
            session_id = create_session(receipt['email'])
            response.set_cookie("browserid_demo_session_id", session_id, max_age = 3600)
        return post_response.content
```

### In a native application that wraps a UIWebView

This is for the case where you are working on a web application that is presented in a `UIWebView` in a native iOS application.

Unfortunately it is not possible to directly use BrowserID from a `UIWebView` because the `UIWebView` has some limitation like for example not being able to deal with popups or inter-webview communication.

Fortunately it is not very difficult to use the `BrowserIDViewController` from a `UIWebView`. What we need to do is bridge the `UIWebView` with native code so that we can present the `BrowserIDViewController` from the native side of the app.

Here is a `UIWebView` delegate method that intercepts a custom `browserid-origin://getVerifiedEmail` URL which will then start the BrowserID process.

```
- (BOOL) webViewShouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
{
    NSURL* url = [request URL];
    if ([[url scheme] isEqualToString: @"browserid-origin"])
    {
        if ([[url host] isEqualToString: @"getVerifiedEmail"])
        {
            [self startBrowserIDWithOrigin: [[url query] substringFromIndex: [@"data=" length]]];
            return NO;
        }
    }
    return YES;
}
```

You will need to implement your own BrowserID 'shim':

```
<script type="text/javascript">
    navigator.id = {};

    navigator.id._callbackToCocoa = function(name, value) {
        window.location = "browserid-origin://" + name + "/callback?data=" + value;
    };

    navigator.id.getVerifiedEmail = function(callback) {
        navigator.id.callback = callback;
        navigator.id._callbackToCocoa("getVerifiedEmail", document.location.origin);
    };
  </script>
```

Then you can call BrowserID in the usual way:

```
// Click handler for the BrowserID Login button
$(".browserid-login").click(function(e){
    navigator.id.getVerifiedEmail(function(assertion) {
		if (assertion) {
			// Call our verifier to make sure this is a valid login
			$.ajax({
                type: "POST",
				url: "/verify",
				data: assertion,
				dataType: "text",
				success: function(data, status, xhr) {
                    // We assume our /verify method returns JSON like { success: true, email: "you@domain" }
					if (data.success) {
						// The verification passed and the user is logged in.
                        alert("Welcome, " + data.email);
					} else {
						// The verification failed
					}
				},
				error: function(xhr, status, error) {
					// Error while verifying
				}
			});
		}
	});
});
```

If your application allows it you can simply redirect away from the login page. You can also call back to the native side to inform it that the BrowserID login and verification were successfull.
