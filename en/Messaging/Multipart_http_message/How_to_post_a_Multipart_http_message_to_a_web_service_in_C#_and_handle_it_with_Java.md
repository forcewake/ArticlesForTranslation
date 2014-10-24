[Source](http://dotnetcodr.com/2013/01/10/how-to-post-a-multipart-http-message-to-a-web-service-in-c-and-handle-it-with-java/ "Permalink to How to post a Multipart http message to a web service in C# and handle it with Java")

# How to post a Multipart http message to a web service in C# and handle it with Java

This post is going to handle the following scenario:

* Upload a file to a server by sending an HTTP POST to a web service in a multipart form data content type with C#
* Accept and handle the message in the web service side using Java

I admit this may not be the typical scenario you encounter in your job as a .NET developer; however, as I need to switch between the .NET and the Java world relatively frequently in my job this just happened to be a problem I had to solve recently.

Let's start with the .NET side of the problem: upload the byte array contents of a file to a server using a web service. We'll take the following steps:

* Read in the byte array contents of the file
* Construct the HttpRequestMessage object
* Set up the request message headers
* Set the Multipart content of the request
* Send the request to the web service
* Await the response

Start Visual Studio 2012 – the below code samples should work in VS2010 as well – and create a new Console application. We will only work within Program.cs for simplicity.

**Step 1**: read the file contents, this should be straightforward



    private static void SendFileToServer(string fileFullPath)
            {
                FileInfo fi = new FileInfo(fileFullPath);
                string fileName = fi.Name;
                byte[] fileContents = File.ReadAllBytes(fi.FullName);
            }


**Step2**: Construct the HttpRequestMessage object

The HttpRequestMessage within the System.Net.Http namespace represents exactly what it says: a HTTP request. It is a very flexible object that allows you to specify the web method, the contents, the headers and much more properties of the HTTP message. Add the following code to SendFileToServer(string fileFullPath):



    Uri webService = new Uri(@"http://avalidwebservice.com");
    HttpRequestMessage requestMessage = new HttpRequestMessage(HttpMethod.Post, webService);
    requestMessage.Headers.ExpectContinue = false;


The last piece of code, i.e. the one that sets ExpectContinue to false means that the Expect header of the message will not contain Continue. This property is set to true by default. However, a number of servers don't know how to handle the 'Continue' value and they will throw an exception. I ran into this problem when I was working on this scenario so I'll set it to false. This does not mean that you have to turn off this property every time you call a web service with HttpRequestMessage, but in my case it solved an apparently inexplicable problem.

You'll obviously need to replace the fictional web service address with a real one.

**Step 3**: set the multipart content of the http request

You should specify the boundary string of the multipart message in the constructor of the MultipartFormDataContent object. This will set the boundary of the individual parts within the multipart message. We'll then add a byte array content to the message passing in the bytes of the file to be uploaded. Note that we can add the following parameters to the to individual multipart messages:

* The content itself, e.g. the byte array content
* A name for that content: this is ideal if the receiving party needs to search for a specific name
* A filename that will be added to the content-disposition header of the message: this is a name by which the web service can save the file contents

We also specify that the content type header should be of application/octet-stream for obvious reasons.

Add the following code to SendFileToServer(string fileFullPath):



    MultipartFormDataContent multiPartContent = new MultipartFormDataContent("----MyGreatBoundary");
    ByteArrayContent byteArrayContent = new ByteArrayContent(fileContents);
    byteArrayContent.Headers.Add("Content-Type", "application/octet-stream");
    multiPartContent.Add(byteArrayContent, "this is the name of the content", fileName);
    requestMessage.Content = multiPartContent;


**Step 4**: send the message to the web service and get the response

We're now ready to send the message to the server by using the HttpClient object in the System.Net.Http namespace. We'll also get the response from the server.



    HttpClient httpClient = new HttpClient();
    Task<HttpResponseMessage> httpRequest = httpClient.SendAsync(requestMessage,    HttpCompletionOption.ResponseContentRead, CancellationToken.None);
    HttpResponseMessage httpResponse = httpRequest.Result;


We can send the message using the SendAsync method of the HttpClient object. It returns a Task of type HttpResponseMessage which represents a Task that will be carried out in the future. Note that this call will NOT actually send the message to the service, this is only a preparatory phase. If you are familiar with the Task Parallel Library then this should be no surprise to you – the call to the service will be made upon calling the Result property of the Task object.

This post is not about the TPL so I will not go into any details here – if you are not familiar with the TPL but would like to learn about multipart messaging then read on and please just accept the provided code sample 'as is'. Otherwise there are a great number of sites on the net discussing the Task object and its workings.

**
Step 5**: read the response from the server

Using the HttpResponseMessage object we can analyse the service response in great detail: status code, response content, headers etc. The response content can be of different types: byte array, form data, string, multipart, stream. In this example we will read the string contents of the message, again using the TPL. Add the following code to SendFileToServer(string fileFullPath):



    HttpStatusCode statusCode = httpResponse.StatusCode;
                    HttpContent responseContent = httpResponse.Content;

                    if (responseContent != null)
                    {
                        Task<String> stringContentsTask = responseContent.ReadAsStringAsync();
                        String stringContents = stringContentsTask.Result;
                    }


It is up to you of course what you do with the string contents.

Ideally we should include the web service call in a try-catch as service calls can throw all sorts of exceptions. Here the final version of the method:



    private static void SendFileToServer(string fileFullPath)
            {
                FileInfo fi = new FileInfo(fileFullPath);
                string fileName = fi.Name;
                byte[] fileContents = File.ReadAllBytes(fi.FullName);
                Uri webService = new Uri(@"http://avalidwebservice.com");
                HttpRequestMessage requestMessage = new HttpRequestMessage(HttpMethod.Post, webService);
                requestMessage.Headers.ExpectContinue = false;

                MultipartFormDataContent multiPartContent = new MultipartFormDataContent("----MyGreatBoundary");
                ByteArrayContent byteArrayContent = new ByteArrayContent(fileContents);
                byteArrayContent.Headers.Add("Content-Type", "application/octet-stream");
                multiPartContent.Add(byteArrayContent, "this is the name of the content", fileName);
                requestMessage.Content = multiPartContent;

                HttpClient httpClient = new HttpClient();
                try
                {
                    Task<HttpResponseMessage> httpRequest = httpClient.SendAsync(requestMessage, HttpCompletionOption.ResponseContentRead, CancellationToken.None);
                    HttpResponseMessage httpResponse = httpRequest.Result;
                    HttpStatusCode statusCode = httpResponse.StatusCode;
                    HttpContent responseContent = httpResponse.Content;

                    if (responseContent != null)
                    {
                        Task<String> stringContentsTask = responseContent.ReadAsStringAsync();
                        String stringContents = stringContentsTask.Result;
                    }
                }
                catch (Exception ex)
                {
                    Console.WriteLine(ex.Message);
                }
            }


This concludes the .NET portion of our problem. Let's now see how the incoming message can be handled in a Java web service.

So you have a Java web service which received the above multipart message. The solution presented below is based on a Servlet with the standard doPost method.

The HttpServletRequest in the signature of the doPost method can be used to inspect the individual parts of the incoming message. This yields a collection which we can iterate through:



    @Override
        public void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException
        {
            Collection<Part> requestParts = request.getParts();
            Iterator<Part> partIterator = requestParts.iterator();
            while (partIterator.hasNext())
            {
            }
        }


If the message is not of type MultipartFormData then the collection of messages will be zero length.

The Part object in the java.servlet.http namespace represents a section in the multipart message delimited by some string token, which we provided in the MultipartFormDataContent constructor. Now our goal is to specifically find the byte array message we named "this is the name of the content" in the .NET code. This name can be extracted using the getName() getter of the Part object. Add the following code to the while loop:



    Part actualPart = partIterator.next();
    if (actualPart.getName().equals("this is the name of the content"))
    {
    }


The Part object also offers a getInputStream() method that can be used later to save the byte array in a file. The file name we provided in the C# code will be added to the content-disposition header of the multipart message – or to be exact to the header of the PART of the message. Keep in mind that each individual message within the multipart message can have its own headers. We will need to iterate through the headers of the byte array message to locate the content-disposition header. Add the following to the if clause:



    InputStream is = actualPart.getInputStream();
    String fileName = "";
    Collection<String> headerNames = actualPart.getHeaderNames();
    Iterator<String> headerNamesIterator = headerNames.iterator();
    while (headerNamesIterator.hasNext())
    {
        String headerName = headerNamesIterator.next();
        String headerValue = actualPart.getHeader(headerName);
        if (headerName.equals("content-disposition"))
        {
        }
    }


The last step of the problem is to find the file name within the header. The value of the content-disposition header is a collection of comma separated key-value pairs. Within it you will find "filename=myfile.txt" or whatever file name was provided in the C# code. I have not actually found any ready-to-use method to extract exactly the filename so my solution is very a very basic one based on searching the full string. Add the below code within "if (headerName.equals("content-disposition"))":



    String searchTerm = "filename=";
    int startIndex = headerValue.indexOf(searchTerm);
    int endIndex = headerValue.indexOf(";", startIndex);
    fileName = headerValue.substring(startIndex + searchTerm.length(), endIndex);


So now you have access to all three ingredients of the message:

* The byte array in form of an InputStream object
* The name of the byte array contents
* The file name

The next step would be to save the message in the file system, but that should be straightforward using the 'read' method if the InputStream:



    OutputStream out = new FileOutputStream(f);
    byte buf[] = new byte[1024];
    int len;
    while ((len = is.read(buf)) > 0)
    {
        out.write(buf, 0, len);
    }


…where 'is' is the InputStream presented above and 'f' is a File object where the bytes will be saved.

View the list of posts on Messaging [here][1].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/messaging/ "Messaging"
