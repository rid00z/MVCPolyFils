MVCPolyfils
=============

MVCPolyfils is a small set of classes that allow you to use views from a MVC project 
in a Xamarin project and have them both compiling and running the same files. 

For the complete guide, see [www.michaelridland.com](http://www.michaelridland.com)

Guide
-----

In this blog I show you how to take an asp.net application and then code share both Razor Views and Business logic with a Xamarin.Forms native application. This integration will allow full sharing between the application, so changes to the Razor views or Business Logic will apply to both projects. The mobile application will also work completely offline in which the Razor Views and Business Logic run completely locally on the device.

And that’s just the start of it! Imagine being able to jump in and out of native/web elements, sharing razor views when it’s a good fit and then jumping into a gorgeous native UI when you like.

### Step 1. Create new solutions
Create a new Xamarin.Forms PCL solution and place it in a directory next to the asp.net MVC project. Because we’re sharing files between projects the directory setup is important. In this case we can call it JellyBeanTracker.Mobile.

#### Step 2. Adding the external files and RazorTemplatePreprocessor
In this example I’ve added the files from the MVC project into the XF PCL project as ‘linked’ files. Create a new directory called ‘External’ and add the following files as per the screenshot. ps, to add files right click the directory and hit add files, Xamarin studio will then ask you if you’d like to link or copy files, in this case you need to link.

![](http://www.michaelridland.com/wp-content/uploads/2014/08/Screen-Shot-2014-08-13-at-8.15.13-am-225x300.png "")

Secondly you need to add the javascript files and css files to the resources directory on each of the platform projects, as per screenshot.

![](http://www.michaelridland.com/wp-content/uploads/2014/08/Screen-Shot-2014-08-13-at-8.20.55-am-250x300.png "")

For each of the razor files you need to right click->properties and set the Custom tool to the RazorTemplatePreprocessor.

![](http://www.michaelridland.com/wp-content/uploads/2014/08/Screen-Shot-2014-08-13-at-8.22.15-am-300x134.png "")

### Step 3. Clone and Add the MVCPolyfils into your Xamarin.Forms Project
Available here: https://github.com/rid00z/MVCPolyFils

### Step 4. Use the Polyfils

On the Razor page remove @model and replace with @inherits System.Web.Mvc.WebViewPage<T> (from MVCPolyfils).

This step is required to have the file work in both MVC and the Xamarin.

![](http://www.michaelridland.com/wp-content/uploads/2014/08/Screen-Shot-2014-08-19-at-9.29.47-pm.png "")

### Step 5. Install Xamarin.Forms.Labs which has a HybridWebView
I’ve done a few modifications to the Xamarin.Forms.Labs project to get this to work and only just put it in my fork, so you’ll need to get the code from my fork until it gets put in the actual project.

https://github.com/rid00z/Xamarin-Forms-Labs

### Step 6. Setup the Xamarin.Forms Project, with the Page and PageModels
This step requires knowledge of Xamarin.Forms, if you’ve not used Xamarin.Forms before then there’s some good videos over here: http://www.michaelridland.com/mobile/xamarin-hack-day-talks-thanks-retrospective/

### Step 7. Sharing Business Logic and Offline
It’s possible to have shared business logic between both of the projects. You can share it via file linking, PCL or a shared project. In my sample application I’ve used file linking so it’s the same as the razor files.

In order to share business logic between asp.net and Xamarin Native you need to code your business logic classes against abstractions/interfaces. If we take a look at the JellyBeanGraph Calculator we can see that it uses a IDataSource for it’s calculations.

`public class JellyBeanGraphCalculator
{
    IDataSource _datasource;
    public JellyBeanGraphCalculator (IDataSource datasource)
    {
        _datasource = datasource;
    }
 
    public IEnumerable<JellyBeanGraphData> GetGraphData()
    {
        return _datasource.GetJellyBeanValues().Result
                    .Select(o => new
                        JellyBeanGraphData 
                        {
                            name = o.Name,
                            color = o.Name,
                            data = new List<decimal> { o.Jan, o.Feb, o.Mar, o.Apr, o.May, o.Jun, o.Jul, o.Aug, o.Sep, o.Oct, o.Nov, o.Dec }
                        });
    }                    
}`


The IDataSource needs to be implemented in two or three places, two places if you’re not supporting offline and three place if you are supporting offline. As you can see in the sample project there’s two sources in the Xamarin project, a LocalDataSource and a RemoteDataSource. These two classes a switched between depending on if it’s online or offline.

#### LocalDataSource

`public class LocalDataSource : IDataSource 
{
    SQLiteConnection _sqliteConnection;
 
    public LocalDataSource ()
    {
        _sqliteConnection = Xamarin.Forms.DependencyService.Get<ISQLiteFactory> ().GetConnection("app.db");
        CreateTable ();
    }
 
    void CreateTable ()
    {
        _sqliteConnection.CreateTable<JellyBeanValue> ();
        _sqliteConnection.CreateTable<MyJellyBean> ();
    }
 
    public Task<IEnumerable<JellyBeanValue>> GetJellyBeanValues ()
    {
        return Task.FromResult((IEnumerable<JellyBeanValue>)_sqliteConnection.Table<JellyBeanValue> ());
    }
 
    public Task<IEnumerable<MyJellyBean>> GetMyJellyBeans ()
    {
        return Task.FromResult((IEnumerable<MyJellyBean>)_sqliteConnection.Table<MyJellyBean> ());
    }
}`

#### RemoteDataSource

`public class RemoteDataSource : IDataSource 
{
    static string HostBase = "http://192.168.56.101:49203";
    public RemoteDataSource ()
    {
    }
 
    public async Task<IEnumerable<JellyBeanValue>> GetJellyBeanValues ()
    {
        return (await GetServerData ()).JellyBeanValues;
    }
 
    public async Task<IEnumerable<MyJellyBean>> GetMyJellyBeans ()
    {
        return (await GetServerData ()).MyJellyBeans;
    }
 
    async Task<SyncContainer> GetServerData()
    {
        HttpClient client = new HttpClient ();
        var result = client.GetStringAsync (HostBase + "/JellyBeans/GetAllData").Result;
 
        var syncContainer = JsonConvert.DeserializeObject<SyncContainer> (result);
 
        //cheap mans offline for this sample, put all into localstoage
        var sql = Xamarin.Forms.DependencyService.Get<ISQLiteFactory> ().GetConnection ("app.db");
 
        sql.CreateTable<MyJellyBean> ();
        sql.CreateTable<JellyBeanValue> ();
 
        if (sql.Table<MyJellyBean>().Count() < 1)
            sql.InsertAll (syncContainer.MyJellyBeans);
 
        if (sql.Table<JellyBeanValue>().Count() < 1)
            sql.InsertAll (syncContainer.JellyBeanValues);
 
        return syncContainer;
    }
}`

### Step 8. Diff Logic in Razor Views using compiler directives
It’s not highly recommended to have logic within your Razor views but it is possible to have different view logic between the platforms. If you’re using file linking the best option is it use #ifdefs, to do this you need to add a compiler Symbol to the native project, as per screen shot.

![](http://www.michaelridland.com/wp-content/uploads/2014/08/Screen-Shot-2014-08-14-at-8.10.36-am-300x163.png "")

Once you’ve setup the compiler Symbol you can then add setup a static IsNative boolean value in a shared file. As per below.

![](http://www.michaelridland.com/wp-content/uploads/2014/08/Screen-Shot-2014-08-20-at-9.32.21-am.png "")

Finally you can add different functionality for the Native applications within your Razor views.

![](http://www.michaelridland.com/wp-content/uploads/2014/08/Screen-Shot-2014-08-20-at-9.32.40-am-1024x102.png "")

### Step 8. Partial Views and Shared Partials

In the open source project I’ve release with this blog post there’s the ability to use Partial views within the Xamarin app. If you provide the classname as a string into the partial the framework will pick it up and run the razor view locally. In order to have the partials work on both the MVC and Xamarin application you need to have the partial views come from a static variable on a class.

### Step 9. Combining the Javascript Logic

It’s also possible to take your javascript files and share them with Xamarin. Mostly they will just function as normal unless you’re doing ajax calls to collect the data. In the case you’re doing ajax calls and want to support offline then you need to communication back and forth between the C#/native and javascript.

There’s three type of communication available in this solution using the HybridWebView (soon to be)in Xamarin.Forms.Labs

a) Injecting javascript from c#

b) Calling a c# method from javascript

c) Calling a c# Func and returning into a closure

#### Step 9a. Injecting Javascript

Using the HybridWebView available in Xamarin.Forms you’re able to execute javascript from outside a webview. This is as easy as calling a method named InjectJavaScript, as per example below.

`string data = JsonConvert.SerializeObject (PageModel.GraphData).Replace ("\r\n", "");
_hybridWebView.InjectJavaScript ("JellyBeanTrackerApp.buildChartStr('" + data + "');");`

#### Step 9b. Calling Native Methods

Also using the HybridWebView you’re able to call Native methods, so this is going from within javascript running in the web view out to a native method written in c#. As a little side note, this works with Mvvm so your business logic is contained within your view model.

##### Registering a method with the HybridWebView.

'
_hybridWebView.RegisterCallback ("EditVisible", (data) => {
  PageModel.EditVisible.Execute(data);
});
'

##### Calling the native method from javascript

`
$('#editVisible').click(function() {
  Native('EditVisible', JellyBeanTrackerApp.currentChartData);
});
`

##### The c# method that’s called from the javascript.

`
public Command<string> EditVisible
{
  get {
    return new Command<string> ((data) => {
      PushViewModel<ManageGraphPageModel>(data, true);
    });
  }
}
`

####  Step 9c. Calling Native Funcs with Closure support

This one I’m a bit proud of as it’s pretty sweet. Using the HybridWebView you now have the ability to call a Native function and the return values of the Native function is injected back into the original closure. Also same as above this works with Mvvm.

##### Registering the Function in the WebView

`
_hybridWebView.RegisterNativeFunction ("GetGraphData", (input) => {
  return PageModel.GetGraphData(input);
});
`

##### Calling the function from javascript with the closure being called on return.

$('#getGraphData').click(function() {
    NativeFunc('GetGraphData', null, function (returnData) {
        JellyBeanTrackerApp.buildChartStr(returnData);
    } );
});

##### The c# function that runs

`
public Func<string, object[]> GetGraphData
{
  get {
    return (data) => {
      GraphData = new JellyBeanGraphCalculator (_dataSourceFactory.GetDataSource ()).GetGraphData();
      return new object[] { GraphData };
    };
  }
}
`

### Step 10. Debugging from Safari

Another thing I should mention is Safari has the ability to debug UIWebViews, the setup details can be found here.

### Just another (awesome)tool for the belt!

This is just one tool for the belt, it’s not the only mobile architecture that you can implement. There’s also the traditional Xamarin approach(Native UI for each platform) and Xamarin.Forms approaches which are still great architectures for mobile applications.

So when should you use this Hybrid type solution?

My personal belief is that using the HybridWebView is acceptable when the view is Not highly interactive as when interacting with a user you want to have everything looking and feeling native. Really it’s up to you to decided and initially it’s good to prototype and test within a device. Of course when you’ve got a large investment in web/mvc assets this is also a good choice.

All the code is available on https://github.com/rid00z/JellyBeanTracker, currently it’s only working with iOS devices. I will attempt add other platforms soon.

Thanks

Michael
