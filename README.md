# DotNetUpdater
Archive of MS example component for auto-updating .NET apps

ref: http://windowsclient.net/articles/appupdater.aspx

This repo file contains the .NET Application Updater component sample.  For more information on this component see the ".NET Application Updater.doc" file in the Docs directory.

## CONTENTS:

- AppStart - THe AppStart.exe project.
- AppUpdater - The .NET Application Updater component project.
- AppUpdaterKeys - The assembly used to hold valid application keys.
- Docs - Documenation for the .NET Application Updater component.
- Samples - Files for the SampleApp walkthrough.
  - SampleApp - Sample described in detail in ".Net Application Updater.doc".
  - AdvancedSample - Illustrates advances uses of the .Net Application Updater component.
  - SimpleSample - Illustrates the most basic use of the .NET Application Updater component.



# .NET Client Applications: .NET Application Updater Component

I recently received a mail from the Microsoft IT team notifying me that they had detected several applications on my desktop computer that did not have the latest patches installed and instructed me to install the latest updates. Ill be the first to admit that I dont update the applications I run as much as I should; either on my home machine or on my work machines. It usually takes a problem like a broken feature in an application or an email (or sometimes several) from the IT department, to get me to install updates. Unfortunately Im more of the rule than the exception when it comes to users updating their applications.

This requirement of needing a user or admin to manually install an update is why rolling out client updates has traditionally been such a huge problem and expense. One solution is to move the responsibility of updating the application from the user to the application itself. Instead of the user obtaining and installing a software update, the client application itself is responsible for downloading and installing updates from a well known server. The only user interaction necessary is whether or not they want to install the new updates now or later. You can see this type of approach to updating applications in action today with products like Windows XP and Microsoft Money.

In this article we will talk about an approach to building .NET client applications that are able to automatically update themselves.

# The .NET Application Updater Component

Included with this whitepaper is a component for enabling .NET client applications to automatically update themselves. The component was written using the .NET Framework and enables you to make your application auto-updatable simply by dropping the component into your existing .NET applications and setting a few properties (ex. where to get updates from). Download the .NET Application Updater Component.

This component is not a Microsoft product. It is intended as a sample to get you started and as such the source is also included with this whitepaper. However, it is worth mentioning that this component has gotten a fair amount of real world use already. It has been used internally in Microsoft to enable auto-updatability in the .NET Terrarium game. Terrarium has been installed and used by over 10,000 individuals since it was first unveiled as a beta product in October, 2001.

This component will be the basis for the discussion of what it takes to make an application auto-updatable. This paper will focus on how the .NET Application Updater component works and how you can use it in your own application.

# Checking for Updates

The first thing an application needs to be able to do in order to update itself is figure out when a new update is available. In order to do this an application needs to know three things 1) where to check for updates, 2) when to check for updates and 3) how to check for updates. The application update component uses HTTP for all network communication. This allows it to update applications over the intranet or extranet. Thus the where to check for updates is simply a URL to a well known Web server.

To address the when to check for updates, the .NET Application Updater component creates a thread on component creation which is responsible for checking for updates. This thread will sleep for most of the time, but will wake up at the configured interval to perform an update check. How often the application checks for new updates is dependent on the individual application, but common values range from one hour to one day between update checks. This polling based approach is not appropriate for all applications, for example Microsoft Money only checks for updates when the user tells them to. In this case, the update poller thread can be disabled and update checks performed on-demand by calling the CheckForUpdate() method on the updater component.

There are several ways to go about the how to check for updates:

## Method #1: Direct File Check
The simplest way to check for updates is to use HTTP to compare the last modified date/time stamp of the application files on the server with that on the client. If the server has newer files, the client knows its time to update itself. This is the same way a Web browser knows if it needs to re-download an html page or image or whether it can just re-use the one previously downloaded. This is certainly the simplest to administrate. When a new version of the app is available, the administrator simply copies the newer version over the older version on the Web server. The problem with this is that the update is not atomic and thus there are potential windows of failure. For example, if an administrator updates the version of the app on the Web server while a client was in the middle of downloading the previous update, that client may be left with some files from the previous update and some files from new update. For this reason, using a direct file check is not recommended for any non-trivial applications.

## Method #2: Manifest Check
To solve the atomicity problem with direct file checks a level of indirection is needed. To create a level of indirection a manifest file on the server is used. A valid server manifest file for use with the .NET Application Updater component looks like this:

    <VersionConfig>
        <AvailableVersion>1.0.0.0</AvailableVersion>
        <ApplicationUrl>http://localhost/demos/selfupdate/V1/</ApplicationUrl>
    </VersionConfig>

AvailableVersion specifies the assembly version number of the latest available version. The ApplicationURL property specifies the URL where that version of the application resides. When the administrator wants to update the client applications, they would copy the new version of the app up to the Web server and modify the server manifest file appropriately. The client itself will then detect that the server manifest file has changed and download the manifest file. The client then compares the assembly version number specified in the manifest with the assembly version number of the applications .exe file. If the server manifest file AvailableVersion is newer, the application knows its time to perform an update. This approach has none of the atomicity problems of the previous approach and is the recommended way to check for updates for most applications.

## Method #3: XML Web Service Check XML
Web services provide a way to perform more advanced update checks. For example, suppose you wanted to roll out an update to a set of early adopters before you rolled it out to the rest of your users. If the client application calls an XML Web service to check if an update is available, that XML Web service could then look up that user in a database and determine if the user is an early adopter or not. If they are, the XML Web service could return a value indicating that an update is available. If not, the Web service could return a value indicating that a new update is not available. Because the Web service used to check for updates could take many forms depending on what custom functionality is desired, the .NET Application Updater component does not provide direct XML Web service support. To use an XML Web service to check for updates, first build the XML Web service and then hook the OnCheckForUpdate event. This allows the poller threads update check to be replaced with your own custom check. The OnCheckForUpdate event has a return value to indicate whether an update was detected or not.

# Downloading Updates

Once the application determines a new update is available, the update needs to be downloaded. By default, when the .NET Application Updater component detects that a new update is available it will automatically kick off another thread and start an asynchronous background download of the update. The download is done using HTTP-DAV. DAV is an extension to HTTP that provides functionality such as directory and file enumeration. A complete and deep download is done starting with the specified URL. The URL used to download depends on the type of update check being done. For example, if a server manifest is used, the URL to download the update from is specified by the ApplicationURL property in the server manifest.

The download of the update obviously needs to be robust. Leaving the client application in any kind of bad state after the download and update is unacceptable. Any number of problems could occur during the download: The Web server where the update resides could go down, the client machine could crash, or for that matter the user could simply shut down the application itself. Since the application is what is doing the download, if its shut down, the download will stop. An alternative design would have been to use a separate system service to do the download and update of the application. With a system service, the download of the update could continue even when the application itself is not running. In fact, Windows XP has a built-in download service called BITS for just this purpose. BITS is what Windows XP uses to download updates to Windows itself. For more information on BITS, see http://msdn.microsoft.com/library/en-us/dnwxp/html/WinXP_BITS.asp. A service was not used in the .NET Application Updater component so that it would work on the Windows 9x operating systems which do not support system services.

The .NET Application Updater component achieves robustness by breaking the download and update process into separate stages. As each stage is complete, it is recorded in an updater manifest file which resides in the application directory on the client. If the download or update process is interrupted at any stage, it will simply resume from the point of the last completed phase the next time the app is started. Each stage is re-run able so that if a failure occurs in the middle of a stage, re-running the stage from start will succeed. If an error occurs, such as the loss of connectivity with the Web server during a download, the .NET Application Updater component will simply try again later. If enough failures are reported (ex. The Web server never came back online), the download and update will be aborted and an error reported.

# Performing updates

Once the new update has been downloaded, the last step is to apply the update. This last step is by far the most challenging. The fundamental problem is that the application is trying to update its own files while running. Thus the files are locked. The only way to unlock the files is to stop the application, but if you stop the application, how can it update its files? Its a catch 22. We went through several designs before we came up with our final approach. Before I describe the final design, lets look at our first approach and describe its shortcomings.

Our first approach was to simply kick off a separate process to perform the update. This separate process would first shut down the application process, perform the update (since the files were now unlocked), restart the application process and then shut itself down when complete. However, there were three fundamental problems with this design:

It simply did not work in some scenarios. For example, Windows runs screen savers in what is called a Job object. The defining characteristic of a Job is that when the root process of a Job is shut down, all processes it created are also shut down. This makes sense when you think about it. When a screen saver goes away, you want to make sure it all goes away. However, in the application update case, when the updater process shut down the original application process, the updater process itself would also be shut down and thus no update performed.
We wanted to be able to automatically update all of the code that performed the update. In the course of use, we would find bugs in the .NET Application Updater component, (ex. A timing window that would effect 1% of clients), and we wanted the ability to automatically roll out fixes of not just the application, but the .NET Application Updater component as well. With this model, we could not update the process that performed the update.
Forcing the user to shut down the application and wait in the middle of use was undesirable.

The final approach used to perform the application update was inspired by the side-by-side assembly model of the .NET Framework. Instead of trying to update the application itself, create a new upto-date version of the application next to the existing version. The new version can be created by merging the existing application directory with the downloaded update. When the new version is complete, the user will automatically use that version the next time they launch the application. The original copy of the application can then be removed. This tricky part is figuring out which version to launch at any given time. To do that, a program named Appstart is introduced. Appstart is the entry point into your application. With this model, your application directory will look something like this:

 - Program Files
    - MyApp
      - Appstart.exe
      - Appstart.config
      - V1 Folder
        - MyApp.exe
      - V1.1 Folder
        - MyApp.exe

To run your application, you always run Appstart.exe. If you want a shortcut on the desktop, the shortcut should point at Appstart, not at the application directly (Note that you can rename AppStart.exe to be whatever you want, ex. YourApp.exe). Appstart.exe is a very simple program that reads the Appstart.config file and launches the specified application. A valid Appstart.config file looks like the following:

    <Config>
        <AppFolderName>V1 Folder</AppFolderName>
        <AppExeName>MyApp.exe</AppExeName>
        <AppLaunchMode>appdomain</AppLaunchMode>
    </Config>

The AppFolderName specifies the subfolder that contains the current version of the app to run. The AppExeName contains the name of the exe within that folder to launch. When an application update is complete, the final step is to change the AppFolderName value to point at the new version of the application. Thus the next time the user runs the application, they will run the new updated version of the application. The AppLaunchMode specifies how to launch the application. There are two ways to launch the application. This first approach is with AppDomains. AppDomains are a feature of the .NET Framework common language runtime and the logical unit of isolation and administration of objects. The common language runtime allows multiple application domains per process. Thus Appstart.exe can launch your application in a separate AppDomain but within the same AppStart.exe process. Thus, despite the fact that two different exes are running (in this case Appstart.exe and MyApp.exe), only one process is used. For most applications AppDomains will work well, however there are a few minor differences with running in a separate AppDomain vs. a separate process. For that case, the AppLaunchMode can be set to process which will cause the application to launch in a separate process.

Once Appstart has started the application, it will go to sleep and wait for the application to terminate. Once the application terminates, Appstart will also shut down. However, if the application shuts down with a well known return code, the Appstart process will re-read the Appstart.config file and re-launch the application. This is useful if the user wants to begin using the new version of the application immediately after it is available.

# Using the .NET Application Updater Component

Now that weve discussed how the .NET Application Updater works, lets put it in action. First we will build a very simple Windows Forms application. Then we will enable it to auto-update itself using the .NET Application Updater component. Finally well deploy the application and see the auto-update feature in action.

Included with this paper is a zip file named DotNetUpdater.zip that contains the .NET Application Updater component. The zip file also contains a Sample directory which contains a number of files and folders we will be using in this walkthrough.

## Step 1: Build the application to update

In this step we will build the application to auto-update. If you want, you can substitute in your own application here. You can also use the pre-built sample application included in the Samples\SampleApp\SampleApp directory of the zip file. However for the purpose of proving that there isnt anything special about SampleApp, well walk through its creation.

1. Use Visual Studio .NET to create a new Windows Application project, name it SampleApp.
2. Give the form an interesting background color of your choice. We will be using background color to differentiate between versions later.
3. Now lets add a tiny bit of functionality to this application in the form of a button that opens a form residing in a separate assembly. First add a button to your form. The zip file contains an assembly with a simple Windows Form in it. Add a reference to the Samples\SampleApp\SimpleForm assembly in the zip. Then add two lines of code to your button event handler:

    SimpleForm.Form1 F = new SimpleForm.Form1();
    F.Show();

4. Switch your build flag to build RELEASE instead of debug. This will allow us to avoid pdb file locking problems later when we build a new version of the application while the original copy is still running.

5. Build and test your application. It should look similar to the Samples\SampleApp\SampleApp in the zip file.

## Step 2: Add the .NET Application Updater Component

In this step we will add the .NET Application Updater component to SampleApp.

1. In the components tab of the Visual Studio .NET toolbox, right click and select Customize Toolbox. Select the .NET Framework Components tab. Browse and select the AppUpdater.dll in the AppUpdater project included in the zip. Click OK.

2. An AppUpdater icon should now show up at the bottom of the list of components in the toolbox. Drag and drop the AppUpdater component onto the SampleApp Form. An appUpdater1 instance of the .NET Application Updater component should be instantiated and appear below the form.

## Step 3: Configure the .NET Application Updater Component

In this step we will configure the .NET Application Updater component. To do this, select the appUpdater1 component and open its properties. The following section contains a description of each property and what value to set it to. Note that you only need to change the first four properties for this example, for the rest, the defaults are adequate.

### AppUpdater Properties 
These are the core properties of the .NET Application Updater and will need to be set for this application as follows:

|Property Name | Description |
|--------------|-------------|
| AutoFileLoad | This controls the on-demand download feature described later, for now set this to true. |
| ChangeDetectionMode | This enum determines how to check for updates. In this example, we will use a server manifest check, so set this value to ServerManifestCheck. |
| ShowDefaultUI | The .NET Application Updater component has a set of simple UI for notifying the user of events such as a new update becoming available or errors during updates. This UI can be replaced with custom application specific UI by disabling the default UI, hooking the appropriate events (ex. OnUpdateComplete) and popping up the custom UI. For this example we will use the default UI, so set this value to true. |
| UpdateUrl     | The UpdateUrl is what determines where the updater looks for updates. In this case we are using a server manifest to check for updates, so this property should be set to the URL of the server manifest. For this example, set it to http://yourWebserver/SampleApp_ServerSetup/UpdateVersion.xml. Replace yourWebserver with the name of your Web server. |


### Downloader Properties
 The AppUpdater component has two sub-components. The first is called the Downloader and controls the download and installation of the component. Below is a description of the properties, but all defaults will work fine for our SampleApp.

| Property Name | Description |
|---------------|-------------|
| DownloadRetryAttempts | If a failure occurs during download (ex the Web server goes down) the downloader will try again a little later. This property controls the number of times the downloader will retry the network request before treating it as a complete application update failure. |
| SecondsBeteweenDownloadRety | The number of seconds before retrying the network request. |
| UpdateRetryAttempts | If a serious error occurs during the update process, (ex. The downloader has exceeded the DownloadRetryAttempts), an application update error is generated. By default, the update attempt will stop, but will attempt to resume the next time the application is started (ex maybe the update Web server was just down for day). This property controls how many times an update will be attempted. If this value is exceeded, the updater aborts the update, resets its state and goes back to checking for updates. |
| ValidateAssemblies | This controls the level of validation done on downloaded assemblies. See the security section of this paper for more info. |

###Poller Properties 
The second sub-component of the AppUpdater is the Poller. The Poller controls the update checks. Below is a description of the properties, but all defaults will work fine for our SampleApp.

| Property Name | Description |
|---------------|-------------|
| AutoStart     | A Boolean that controls whether or not the Poller should begin polling for updates on application startup or whether it should wait until it is explicitly started programmatically. |
| DownloadOnDetection | A Boolean that controls whether or not the Poller starts a download of an update immediately after a new update is found, or whether the download must be started explicitly by a call to the DownloadUdpate() method. |
| InitialPollInterval | The number of seconds after application startup before the first update check is performed. |
| PollInterval | After the first update check, the PollInterval controls the number of seconds between each subsequent update check. Note: By default this checks every 30 seconds; clearly you will want to reduce the frequency for your application. |

When all is said and done, your property grid should look the following:
IMAGE

The Samples\SampleApp\SampleApp_Complete directory contains a version of the application correctly setup.

## Step 4: Build & Deploy V1 of the application to the client

In this step we will build the V1 version of the application and deploy it to the client. The deployment is essentially simulating what the install program for your application will do.

1. In the SampleApp project, open the AssemblyInfo.cs file. Change the AssemblyVersion value from "1.0.*" to 1.0.0.0. This will cause the built assembly to get marked with a value of 1.0.0.0 instead of the ever increasing value Visual Studio normally assigns.

2. Build the application.

3. Copy the Samples\SampleApp\SampleApp_ClientSetup directory from the zip onto your local machine. It doesnt matter where you copy it, however the program files directory is the most realistic place to put it since that is where most applications get installed. Youll notice that SampleApp_ClientSetup directory already has AppStart.exe included. AppStart.config is already set to point into the 1.0.0.0 directory and run SampleApp.exe.

4. Copy the complete SampleApp (Appupdater.dll, SimpleForm.dll & SampleApp.exe) from the release build directory of SampleApp to the SampleApp_ClientSetup\1.0.0.0 directory on your client.

At this point a fully functional version of the application should be installed on the client and executable by running AppStart.exe

## Step 5: Setup the Web server

In this step we will setup the Web server for use in rolling out application updates. The .NET Application Updater component uses HTTP-DAV to download the application update and thus requires a Web server that supports HTTP-DAV. IIS 5.0 that comes with Windows 2000 and newer operating systems support HTTP-DAV.

1. Copy the Samples/SampleApp_ServerSetup directory from the zip into the wwwroot directory of your Web server. Note: Any location on your Web server can be used, but be sure to update the UpdateUrl property accordingly. Notice that UpdateVersion.xml is already setup for use as the server manifest file.

2. For completeness, copy the V1 version of SampleApp into the 1.0.0.0 folder of the Web server.

3. Enable IIS Directory Browsing for the SampleApp_ServerSetup directory on your Web server. Since the .NET Application Updater component enumerates the contents of directories during download, Directory Browsing must be enabled. To do this, open the Internet Information Services manager in the control panel. Select the SampleApp_ServerSetup folder and open its properties. Select the Directory Tab. Check Directory Browsing, and click OK.

## Step 6: Automatically update the application

OK, now its time to see the results of all this hard work by automatically rolling out a new version.

1. If the SampleApp version you deployed to the client isnt running, launch it and leave it running. Remember to use AppStart.exe.
2. Go back to Visual Studio and change something noticeable on the SampleApp form (ex change the background color).
3. Change the VersionInfo in AssemblyInfo.cs to be 2.0.0.0.
4. Rebuild.
5. Go to the Web server and create a 2.0.0.0 directory as a peer to the 1.0.0.0 directory. Copy the new version of the application from the release build directory into the new 2.0.0.0 directory on the Web server.
6. Open UpdateVersion.xml and change AvailableVersion to be 2.0.0.0. Change the ApplicationURL to point at the new 2.0.0.0 directory.
7. Save the changes to UpdateVersion.xml.

As soon as you save the new UpdateVersion.xml, within 30 seconds the running copy of SampleApp should detect the newly available version. SampleApp will then download the new version, apply the update, and pop up the default UI asking the user if they want to restart and start using the new version immediately. Click Yes in response to this dialog. SampleApp should restart and be running the new version. If you look at the client deployment of SampleApp you will notice there is now a 2.0.0.0 directory next to the original 1.0.0.0 directory. The 1.0.0.0 directory will be cleaned up the next time an update occurs.

# On-Demand Install

By exploiting the extensible nature of the .NET Framework, the .NET Application Updater component is able to enable another feature, On-Demand installation. By enabling On-Demand installation, only the main exe needs to be explicitly installed on the client. The remainder of the application can be automatically downloaded and installed on an as needed basis.

You can see this feature in action with the SampleApp you just finished building and deploying. In SampleApp we enabled the On-Demand install feature by setting the AutoFileLoad property to true. First close SampleApp if it is still running. Next, in the 2.0.0.0 folder of the SampleApp client deployment, delete SimpleForm.dll. Youll recall that SimpleForm.dll is the assembly that contains the form that is displayed when the button on SampleApp is clicked. With SimpleForm.dll deleted, you would expect the application to throw an exception when the button is clicked. Go ahead and try it by running SampleApp (remember to use AppStart.exe) and click the button in SampleApp.

What happened? It worked. In fact if you look in the 2.0.0.0 folder youll see that SimpleForm.dll has reappeared. So how did this happen? The CLR is fundamentally programmable and extensible. When the CLR could not find the SimpleForm.dll assembly, instead of simply failing, it raised the AppDomain.AssemblyResolve event. The .NET Application Updater component hooks that event. The .NET Application Updater component knows where a valid copy of SimpleForm.dll resides; on the deployment Web server. It then downloads the assembly to the local application directory and tells the CLR to try to load the assembly again. The second load attempt then succeeds.

On-Demand installation is enabled and disabled via the AutoFileLoad property on the .NET Application Updater component. It is a powerful feature that can make the initial download and install of your application tiny, but can be difficult to use correctly. You must think hard about where the assembly boundaries reside in your application and what actions will cause an assembly to be downloaded. Because the assembly download involves network I/O, it will take variable lengths of time to download. During the assembly download, the application is frozen waiting for the assembly download to complete.

# Deployment Security

While the ability to roll out application updates automatically has great benefits, it also comes with an inherent danger. When you make it easier to roll out updates, if not careful, you could also make it easier to roll out malicious code. There are two dangers. The first danger is that someone will spoof the Web server used to deploy updates with their own Web server. They could then use that Web server to roll out a virus in place of your application. The easiest way to prevent spoofing or any other tampering over the wire is to use HTTPS. To use HTTPS with the .NET Application Updater component, simply use HTTPS URLs in place of HTTP URLs.

However, HTTPS is not a silver bullet. There are two problems with HTTPS. The first is scalability. Using HTTPS requires the server to encrypt all of the files that are downloaded from the Web server. If an application update is large, the cost of encrypting the update can overburden the server. The other issue with HTTPS is that it does nothing to address the second security danger. The second danger has to do with your Web server being compromised, either from an insider or from an outsider hacking your server. If your Web server is compromised, its bad enough, but if it also means that a thousand clients are also compromised via automatic updates, it could be disastrous.

To solve this problem, the .NET Application Updater component uses the ability to strong name a .NET assembly to verify the authenticity of the assembly upon download. If the .NET Application Updater detects that an assembly is not signed with your applications key during download, the update is aborted. This means that only someone that has the private key of your application can build updates that can be automatically deployed. .NET security is based on keeping your private key secret. Even inside your own organization, only the few select people should have access to your private key.

To validate assemblies, the .NET Application Updater component verifies that the public key on the exe of your current installed application matches the public key on downloaded updates. The embedded public key can only be the same if the two assemblies were signed with the same secret private key. Because the assembly is loaded by the CLR in order to verify its public key, the CLR does its normal hash value check to ensure the assembly is in fact a real assembly and has not been tampered with. If a verification check fails, it is treated just like any other update failure. To enable verification on download, simply strong name all of your application assemblies and set the ValidateAssemblies property on the .NET Application Updater component to True.

Assembly validation on download works great, but in practice, applications will often have components signed with different private keys. For example, your application may have two files: The exe assembly signed with your private key and a dll assembly that contains a 3rd party charting control you purchased to use in your application. This 3rd party assembly may be signed with the 3rd partys private key and not yours. What makes it even more complicated is that the set of valid private keys that can be used to sign assemblies in your application could change from version to version. How do you auto-update these types of applications? To solve this problem, you create an assembly that contains a list of valid public keys for your application. Sign this assembly with the applications main private key (the key that the exe of the application is signed with) and place this assembly in the directory with the application update on the Web server. Before the update download process begins, the .NET Application Updater will check for a well known assembly AppUpdaterKeys.dll in the application update directory on the server. If present, the assembly will be downloaded. The assembly will be verified against the main application public key. If the signature is valid, the list of keys will be extracted. From that point on, any of the keys in that list will be treated as valid signatures for the files in the update. The zip file that comes with this paper contains an AppUpdaterKeys project that can be used to build your own AppUpdaterKeys assembly. The AppUpdaterKeys.dll also allows you to specify a set of file exceptions. Files specified as exceptions can be downloaded and updated without passing the validation check. This can be useful when your application contains non-assembly files, such as pdb files for debugging.

The recommended approach to security is to use HTTPS URLs to perform update checks. This provides a first level of spoofing protection. For update download, its best not to use HTTPS URLs to avoid the overhead on your Web server. Instead, strong name your application assemblies and enable the Validate Assemblies feature.

# Extensibility and Terrarium

In the sample we walked through earlier in this paper we enabled auto-deployment in an application simply by dropping a component into an application and setting some properties. While this works well for many applications, some applications require a higher degree of control that can only be achieved by writing code. The .NET Application Updater component has an API that can be used to customize and replace application update behavior. The .NET Terrarium game (http://www.gotdotnet.com/terrarium/) I mentioned earlier is a heavy user of these extensibility hooks. Lets take a look at how Terrarium uses the .NET Application Updater components extensibility mechanisms.

Terrarium uses a custom XML Web service to check for updates. Terrarium hooks the OnCheckForUpdate event of the .NET Application Updater component and calls the XML Web service within this event handler. OnCheckForUpdate fires on the poller thread. Thus the XML Web service request is performed on a separate thread from the UI thread and does not lock the UI. The call to the XML Web service passes the assembly version of the clients current Terrarium install. The XML Web service then decides whether or not a newer update is available for that client. Two values are returned. The first indicates whether or not an update is available. The second is the URL where the update resides. For scalability reasons, in practice, one server is used to check for updates and a separate server is used to download updates. On the return from the XML Web service call, if a new update is available, Terrarium sets the UpdateUrl property of the .NET Application Updater component to the URL returned from the Web service. This instructs the .NET Application Updater to use that URL for the next update download. Finally, Terrarium returns true in the OnCheckForUpdate event handler if an update is available.

Terrarium also uses the custom XML Web service to increase scalability. Updates to Terrarium exceed 10 MB in size. Terrarium also checks for updates very frequently. Thus, in the beginning, when a new update was rolled out, it wasnt uncommon to have hundreds of clients asking for updates at once. This resulted in Gigabytes worth of download requests and the server couldnt handle the load. Moving away from using HTTPS and decreasing the frequency of update checks helped the situation, but the ultimate solution was to build some intelligence into the XML Web service. When the server is under load, the XML Web service only tells the first X number of clients that an update is available. All of the rest are told that no new update is available. The next time the clients check for updates, the next X set of clients are told an update is available. In this way, the update is rolled out in waves and the server is not overloaded.

Terrarium puts up its own custom UI that integrates better with the overall application. Thus Terrarium disables the default .NET Application Updater UI. Terrarium hooks the OnUpdateComplete event. The OnUpdateComplete event includes a set of information, such as whether or not the update was successful, what, if anything, caused failure, and an English string with an explanation of the problem that can be presented to the user. The OnUpdateComplete event fires on the main UI thread of the application. Thus Terrarium raises UI to the user, informing them of update success or failure, directly in the OnUpdateComplete event handler.

The .NET Application Updater component also includes additional APIs that allow explicit control over when actions like update checks and updates are performed. These actions are controllable via the CheckForUpdate () and ApplyUpdate() methods respectively.

# Debugging

The source code for the .NET Application Updater component is provided so that you can debug problems and add functionality as needed. However, before you dive too far into the source code, this section will spell out some first level debugging options, as well as describe the most frequently encountered problems by consumers of this component.

The .NET Application Updater creates a hidden log file named AppUpdate.log that resides in the same directory as AppStart.exe. All successful and failed updates are recorded in this log. The log file is especially useful when there is a particular client machine that wont update successfully. You can use the log to determine when and how the update failed. In addition, the .NET Application Updater component uses the Debug class of the .NET Framework to write out a lot of useful information. If you run your application in the debugger, you will see this information in the output window. You can essentially watch the .NET Application Updater go through its update steps and see where the problem is occurring.

If for whatever reason, you just cant get the .NET Application Updater to work, make sure of the following before you debug too far, more than likely one of these is the problem you are encountering:

- Do you have IIS Directory Browsing turned on? If not, the downloader will see an empty directory on the Web server where the update resides. The updater will then download and install an update of zero files, leaving you with the original unchanged application. See the Setting up the Web server section earlier in this paper for instructions on how to enable Directory Browsing.
- Do you have everything deployed correctly and the correct URLs set? Its easy to make a mistake here. Look at the URLs being output by the .NET Application Updater when running in the debugger. Navigate to these URLs in the browser and make sure they are valid.
- If your app is installed in the program files directory, are you an admin or power user on the box? If not, you wont have the write access necessary to update the application.
- Did you create the AppUpdater object on the main UI thread of the application? If not, the updater will not be able to present UI and will fail when firing events back to the UI thread.
- Is the update succeeding, but the app failing to automatically restart to pick up new updates? The .NET Application Updater component attempts to restart the application by calling the Application.Exit method. However, that method is not guaranteed to shut down an application. If you spawn & leave separate threads running, it will not close the process. The solution is to ensure that all threads are terminated via the Application.OnExit event, or hook the .NET Application Updaters OnUpdateComplete event and handle the shutdown yourself.

# Summary

Ease of deployment for client applications was a major goal for the first version of the .NET Framework. There is no other technology for building client applications that addresses the deployment problems as well as the .NET Framework. Ease of deployment will continue to be a major goal for future versions of the .NET Framework. The .NET Application Updater component described here represents some of our thinking in terms of scenarios we would like to enable directly in future versions of the .NET Framework. However, until that time the .NET Application Updater component is a great way to get started building auto-updating applications. 
