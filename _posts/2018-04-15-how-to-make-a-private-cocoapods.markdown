---
layout: post
title: "How to Make a Private CocoaPods"
date: 2018-04-15T15:35:29+08:00
---

# Overview #

CocoaPods is a great tool not only for adding open source code to your project, but also for sharing components across projects.

This post will describe how to make a private CocoaPods for your application.
You will need a private spec repository, letting CocoaPods know where to find it and adding the PodSpecs file to the repo.

Also, you will need a private repository that stores the source code of your pod library and an example project (if you have).

# Pod Creation #

We use `pod lib create YOUR_PROJECT_NAME` command to bootstrap the process.
That will automatic generate an entire project by asking you few questions.

For example,

1. What platform do you want to use? [iOS/macOS] -> iOS
2. What language do you want to use? [Swift/ObjC] -> ObjC
3. Would you like to inculde a demo application with your library? [Yes/No] -> Yes
4. Which testing frameworks will you use? [Specta/Kiwi/None] -> None
5. Would you like to view based testing? [Yes/No] -> No
6. What is your class prefix? -> FF

# Directory of the Pod project #

If you finish the few questions, running `pod install` on the example folder.
The basic pod library is born.

You can add your dependency libraries you need like AFNetworking on Podfile or some static frameworks.

The folder structure will look like this:

	TestPro 
	├── Example 
	│   ├── Podfile 
	│   ├── Podfile.lock 
	│   ├── Pods 
	│   ├── TestPro 
	│   ├── TestPro.xcodeproj 
	│   ├── TestPro.xcworkspace 
	│   └── Tests 
	├── LICENSE 
	├── README.md 
	├── TestPro 
	│   ├── Assets 
	│   └── Classes 
	├── TestPro.podspec 
	└── _Pods.xcodeproj -> Example/Pods/Pods.xcodeproj
	
* _Pods.xcodeproj: A symlink to your Pod's project for Carthage support.
* TestPro.podspec: The PodSpec of your library, you can add some formulas such as spec name, spec version or the path of source files, and so on. You can find more syntax information on Reference section below.
* Assets & Classes: That contains all .swift/.h/.m files and images you will need.
* Pods folder: That stores the external pod libaries you need, including a dynamic framework be installed from Podfile or a static framework you added manually.
* Example folder: That stores the example project using the Pod library.

# Add custom files as you need #


#### Dynamic frameworks ####

You can use some dependency frameworks through Podfile,
for example,

	source 'https://github.com/CocoaPods/Specs.git' 
	
	platform :ios, '8.0' 
	use_frameworks! 
	
	inhibit_all_warnings! 
	
	target 'TestPro_Example' do 
	
		pod 'TestPro', :path => '../' 
		pod 'AFNetworking' 
		
		target 'TestPro_Tests' do 
			inherit! :search_paths 
		end 
		
	end

then run `pod install`, that will fetch the AFNetworking library into the Pods folder.

#### Static frameworks ####

If there are no Pod resource support, you can add the library into the Pods folder by manually.

Then modify the PodSpec file by adding static_framework and dependency.

For example,

	s.static_framework = true 
	s.dependency 'AliPay_SDK'
	
then run `pod install`, that will generate the AliPay_SDK library dependency on Xcode.

#### Wrappers files ####

You can new some wrapper files on Classes folder that will handle those libraries for usage more easier.


# Push the Pod source code #

After you finished features development and write some descriptions on README.md file.

You need to commit and tag a version number, then push the source code to the remote repository.

We suggest that the tag number should be equal to the PodSpec version number and follow [Semantic version define] (https://semver.org/).

For example,

	git add . 
	git commit -m 'First release' 
	git tag '1.0.0' 
	git push origin master 
	git push --tags

# Deploy the Pod library #

Using the URL of your private spec repository, adding your repo using:

`pod repo add POD_NAME THE_URL_OF_SPEC_REPOSITORY`.

Save your PodSpec file and add to the repo:

`pod repo push POD_NAME YOUR_PODSPEC_FILE`.

# Pod update #

Once we modified the Pod library, we need to change the version number on PodSpec file.

Before you deploy the Pod library, you need to commit, bump up tag version number and push source code to the remote repo under source control..

If you facing a problem about _The TestPro.podspec specification does not validate._ when running _pod repo push POD_NAME YOUR_PODSPEC_FILE_.

You can run _pod repo push POD_NAME YOUR_PODSPEC_FILE --verbose --use-libraries --allow-warnings_ to resolve it.

### Reference ###

* [Podspec Syntax Reference](https://guides.cocoapods.org/syntax/podspec.html#specification)
* [Using Pod Lib Create](https://guides.cocoapods.org/making/using-pod-lib-create)
* [Private Pods](https://guides.cocoapods.org/making/private-cocoapods)
* [Semantic Versioning 2.0.0](https://semver.org/)