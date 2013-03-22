Update the sources by running:

	cd vendor/
	rm -rf browserid-ios
	git clone git://github.com/couchbaselabs/browserid-ios.git
	cd browserid-ios
	rm -rf .git/

I'd use git submodules but I'm not sure how the plugin installer feels about them.
