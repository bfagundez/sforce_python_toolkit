=======
Salesforce Python toolkit
=============

My fork to this great Salesforce SOAP wrapper.

*Modified the partner class to return inner query results, the original version returned only the first element of the inner query.*

Original code at: https://code.google.com/p/salesforce-python-toolkit/ 

The original README:


Mission: To provide a thin layer around the raw SOAP interaction that consumes Salesforce's
Enterprise and Partner WSDLs and handles the nitty-gritty details of the SOAP interaction.


The Salesforce Python Toolkit features the following:
  - Supports both the Enterprise and Partner WSDL
  - Unifies the interface to Enterprise and Partner objects.  There should be no such thing as an
    'Enterprise' example or a 'Partner' example, outside of specifying the correct WSDL and 
    instantiating SforceEnterpriseClient or SforcePartnerClient
  - Handles rewriting the SOAP endpoint after the connection is established
  - Stores the session ID, and attaches it in a SOAP header in each subsequent call
  - Manages SOAP headers for you, particularly which headers get attached when
      - For example, CallOptions only applies to the Partner WSDL
      - AssignmentRuleHeader only applies to create(), merge(), update(), and upsert()
      - Check out _setHeaders() in SforceBaseClient for more details :)
  - Manages object/field ens:sobject.{partner,enterprise}.soap... namespace vs. 
    {partner,enterprise}.soap namespace
    - Suds doesn't natively do this, in either the Partner WSDL or the Enterprise WSDL.
  - Allows you to pass a relative path, absolute page, or local or remote URL pointing to your WSDL


But what about those other libraries?  And why Suds and not SOAPpy or ZSI?  And why not just Suds?
  - First and foremost, SOAPpy and ZSI dead projects.
  - Why not SOAPpy?   http://bugs.debian.org/cgi-bin/bugreport.cgi?bug=523083
    - You can't use the Enterprise WSDL because SOAPpy doesn't support manually setting namespace 
      prefixes for fields.
  - Why not Beatbox?  No WSDL support.
  - Honestly?  Suds is pretty damn wonderful, and it has an active community supporting active 
    development.
    - If there weren't so many Salesforce-specific implementation details, building directly on top
      of Suds would make perfect sense.  Unfortunately, this just isn't the case.


Dependencies:
  - Suds 0.3.6+
    - You can install suds by issuing `easy_install suds --install-dir=<install directory>` 
      - This method requires setuptools.
    - If you have a previous version of Suds installed, or don't want to install setuptools, you 
      can simply unpack 0.3.6+ and add the path to PYTHONPATH.


Tested with:
  - OS X 10.5
  - Python 2.5.1
  - suds 0.3.6


Examples:
  - You can find examples for every method call in the EXAMPLES file.


Caching:
  - The Toolkit supports caching of remote WSDL files for a defineable amount of time (in seconds):

h = SforceEnterpriseClient('https://example.com/enterprise.wsdl.xml', cacheDuration = 90) 

h = SforcePartnerClient('https://example.com/partner.wsdl.xml', cacheDuration = 86400) # 1 day


Proxies:
  - The toolkit supports HTTP proxies, but not HTTPS.  This is due to a limitation in the underlying
    urllib2 library that ships with Python.  Details can be found at the bottom of this document.

    It should be noted that all traffic between the proxy and Salesforce will be sent over HTTPS.

    The constructors take an argument 'proxy', e.g.

h = SforceEnterpriseClient('enterprise.wsdl.xml', proxy = {'http': 'proxy.example.com:8888'}) 

h = SforcePartnerClient('partner.wsdl.xml', proxy = {'http': 'proxy.example.com:8888'}) 

    Any attempt to pass an https proxy will raise a NotImplementedError.


Inspecting your data:
  - It's quite simple to see the structure of your objects.  For instance:

lead = h.generateObject('Lead')
lead.FirstName = 'Joe'
lead.LastName = 'Moke'
lead.Company = 'Jamoke, Inc.'

print lead

outputs this:

(sObject){
   fieldsToNull[] = <empty>
   Id = None
   type = "Lead"
   FirstName = "Joe"
   LastName = "Moke"
   Company = "Acme, Inc."
 }


Strict- and non-strict modes:
  - Set with the method call setStrictResultTyping() (takes a bool)
  - If any of the following calls return a single result, the result object will not be wrapped in a
    list (i.e. returns sObject instead of [sObject]):
convertLead()
create()
delete()
emptyRecycleBin()
invalidateSessions()
merge()
process()
undelete()
update()
upsert()
describeSObjects()
sendEmail()
  - This is to facilitate things for folks who nearly always create/update/delete a single record at
    a time.
  - Note that calls that return an indeterminate number of results (e.g. query()) will always return
    a list of results, even if a single record is returned.
  - We accept in strict- and non-strict mode, for parameters, any of
    - '00Q.....'    (value)
    - ['00Q.....']  (value(s) wrapped in list)
    - ('00Q.....')  (value(s) wrapped in tuple)
      - so, simply call crmHandle.emptyRecycleBin('001x00000000JerAAE')
        instead of emptyRecycleBin(['001x00000000JerAAE'])


Logging:
  - If you're curious what's happening at the SOAP level, Suds offers you a great deal of 
    information.  Have a look at https://fedorahosted.org/suds/wiki/Documentation#LOGGING for more
    information.


Conventions:
  - Fields are named internally exactly as they are specified in the documentation and the XML,
    except where they differ (userID vs. userId is one example), in which case the XML spec wins.


Caveats (short form):
  - gzip/deflate not implemented
  - HTTP 1.1 persistent connections not supported
  - Supports HTTP proxies, not HTTPS (traffic between the proxy and Salesforce still sent HTTPS)
  - na0.salesforce.com (a.k.a ssl.salesforce.com) probably doesn't work


Caveats (long form):
  - gzip/deflate not implemented
    - Suds doesn't handle HTTP headers in SOAP response such as 
      Content-Encoding: gzip
  - HTTP 1.1 persistent connections are not supported because of a limitation in the urllib2, which
    Suds sits on top of
    - urllib2 secretly sets a header in do_open():
      headers["Connection"] = "close"
      which does bubble up to Suds, and consequently is not part of our header dict.  This means 
      that logging suds.transport will not show the Connection: close header, even though it's sent
      - This is still an issue in Python 3.1, though now there is the following comment:
        # TODO(jhylton): Should this be redesigned to handle
        # persistent connections?
  - Connections to a proxy must use HTTP and not HTTPS due to a limitation in urllib2, where it
    does not implement the HTTP CONNECT method.  All traffic between the proxy and Salesforce will
    of course be sent over HTTPS.
    - http://code.activestate.com/recipes/456195/ illustrates a possible workaround
  - na0.salesforce.com (a.k.a ssl.salesforce.com) probably doesn't work
    - This affects customers that signed up from the US web site prior to roughly June 2002
      - http://forums.sforce.com/sforce/board/message?board.id=PerlDevelopment&message.id=401
    - According to the Salesforce docs, this particular instance requires ISO-8859-1 instead of 
      UTF-8, and UTF-8 is hard-coded into Suds
      - http://www.salesforce.com/us/developer/docs/api/Content/implementation_considerations.htm
    - I don't have access to an account on this cluster, so it may or may not take the UTF-8 data
      and cram it into ISO-8859-1 (it may also throw a SoapFault).  I really have no idea.
    - If anyone has access to it and wants to get it working, one possibility _might_ be to add
      the HTTP header 'Content-Type: text/xml; charset=iso-8859-1'.  You still wouldn't be able
      to use any non-8859-1 characters, but Salesforce _might_ be willing to interact using that
      header.  Needless to say, this would be a horrible hack at best.
  - For caveats that don't affect the end-user, but do concern implementation details, see the 
    HACKS file
