**New Relic:**
 New Relic provides a platform where you can integrate other apps like Grafana dashboard, OpenTelemetry and you can get logs, events, alerts, full software stack information on a single platform. You can analyze, your full stack from one place.
 - They have APM(application performance management) section, Browser section(which gives Full-stack visibility, identifies latency from backend services to end-user experiences ), Synthetics(where you can simulate user traffic), infrastrcuture( where we can integrate AWS, GCP and other accounts) and others.
 
 **Bug Foraging of reports from Jon on New Relic** (https://hackerone.com/jon_bottarini)
 
**Bug 1:** https://hackerone.com/reports/197436
 Description: Restricted User is able to edit Alert Conditions of Synthetics Monitors even if Synthetics Permissions is enabled by an admin
 
 
 Admin   ->  User1
 
 
 **Synthetics** has **monitors**
 Synthetics -> Permissions -> group1
 
 Add User1 -> group1 
 
 
**Monitor Permissions for Group1**

View all monitors = checked

Edit all monitors = unchecked

Private locations = checked

Edit all private locations = unchecked


Add Admin -> Group2

**Monitor Permissions for Group2**

View all monitors = checked

Edit all monitors = checked

Private locations = checked

Edit all private locations = checked

- Admin account creates two monitors. The monitors are called: Monitor1 and Monitor2.

Monitor 1 : Do not notify
Monitor2: Get Notified with an email filled


User1 is part of Group1 and he does not have permission to edit any monitor

- User1 navigates to Synthetics and clicks on “Monitor1”. The URL should look something like this: https://synthetics.newrelic.com/accounts/1523936/monitors/5b8d01f9-389e-4120-aa83-20baa98e086c/
- User1 appends /conditions at the end of the URL. The URL now looks like this: https://synthetics.newrelic.com/accounts/1523936/monitors/8f4a3a49-6355-4455-8480-79d93acdfda7/conditions
- User1 should not be able to be here! He isn’t authorized to make edits to Monitors


**What do we learn?**

- Always check the requests of admin user for the privileged endpoint.
- Append the same endpoint to the less privileged user and try to use it.

**Bug 2:** https://hackerone.com/reports/271393
**Description**: NR Internal_API call allows me to read the events/violations/policies/messages of ANY New Relic account (AND pull data from infrastructure)


Normal Rest API: https://api.newrelic.com/v2/applications/{application_id}/hosts.json

- It uses api.newrelic.com for api requests and `X-api-key` as header for authentication

- But in some of the .js file, they had reference to "/internal_api"
- You can also found the same by spidering/crawling *infrastructure.newrelic.com* and* alerts.newrelic.com domains.*
- So internal api looks like this:
`
https://alerts.newrelic.com/internal_api/1/accounts/{ACCOUNT NUMBER}/incidents
`

- This "/internal_api/" has the similar endpoint as the main api endpoint.
- By manipulating the ids on this request, any use can see other user account events, messages, polcies etc. 

**What do we learn?**
Check Js files and other subdomains for unique internal endpoints.

**Bug 3: (Combining 3 reports exposing same confidential info from different endpoints)** https://hackerone.com/reports/267636
https://hackerone.com/reports/320173
https://hackerone.com/reports/271861

**Description**: Ability to see full name associated with other New Relic accounts

- Found a issue in previous report that full names of a user are coming in the user management section.
- But there is another way to get full names:
	- Add a new user to the account  (https://account.newrelic.com/accounts/1523936/users/new)
	- Ener some name and emailid of the user USER1 and add this user.
	- Now go to permission section ( https://synthetics.newrelic.com/accounts/1523936/permissions), as we know from Bug1 that we need to make a group and add permissions to a group.
	- Then, we can add user to the permission group, bug was that when a admin user(let's say) Click on the plus button next to "People" to add USER1 in the permission group, it shows fullname of any email on any account.

**What do we learn?**
- Understand the bussiness logic, if one endpoint shows some info then look for other endpoints too, it might be the case that you find other endpoint which is leaking confidential info.

**Bug 4:** https://hackerone.com/reports/520623

**Description:** Giving myself access to NR1 UI / one.newrelic.com without the proper feature flags on my account


- Use the Burp Suite match and replace rule to change the feature flag from false to true.
https://www.jonbottarini.com/2019/06/17/using-burp-suite-match-and-replace-settings-to-escalate-your-user-privileges-and-find-hidden-features/

**What do we learn?**
- There might be some hiddne feature or functionality in website, try looking at all request in Burp and use match and replace if you see values like "USER", "BASIC", "TRUE", "FALSE".


**Bug 5 :** https://hackerone.com/reports/397137
**Description:** Pull any Insights/NRQL data from any NR account
https://www.jonbottarini.com/2018/10/09/get-as-image-function-pulls-any-insights-nrql-data-from-any-new-relic-account-idor/

- As I told before,  NewRelic allow us to integrate other apps.
- Common integrations include AWS, Azure,  Google Cloud Platform (GCP). In Google Cloud Platform there is also the ability to create dashboards.
- Apart from other apps, GCP dashboard shows some unique chart options, one of them is "Get as image"
-  This option converts the New Relic Query Language (NRQL) query that generates the dashboard into an image – and this is where the vulnerability lies. 
-  The normal POST request to generate the dashboard image is as follows:

{"query":{"account_id":1523936,"nrql":"SELECT count(*) FROM IntegrationError FACET dataSourceName SINCE 1 DAY AGO"},"account_id":1523936,"endpoint":"/v2/nrql","title":"Authentication Errors"}

- The application failed to check and see if the account_id parameter belonged to the user making the request. The account number 1523936 belongs to me, but if I changed it to another number, I could pull data from another account.
- In NR, all acountids are incremental, so you can pass this request to burp intruder and fuzz it to get data from other users account.
- Infact, in this case, we can modify the NRQL query as well to get more data than above query will give. NRQL queries are public: https://docs.newrelic.com/docs/insights/nrql-new-relic-query-language/nrql-resources/nrql-syntax-components-functions

**What do we learn?**
- Go through all the functionality of the app, look for unique feature in any section of the app, try to manipulate requests if you see any id or query.

**Bug 6 :** https://hackerone.com/reports/202501
**Description:** Restricted user is able to delete filter sets of admin users in https://infrastructure.newrelic.com/accounts/{{ACC#}}/settings/filterSets

- Admin  invites -> User2 as a restricted user (as described in above bugs with less/no privileges by putting User2 in a less privilege permission group)
- he admin creates a filter by navigating to Infrastructure > Inventory > Add Filter (https://infrastructure.newrelic.com/accounts/\<account-id\>/inventory)
- User2 joins the team, then clicks on "Infrastructure" tab
- User2 clicks on "Settings", within Infrastructure.
- At this URL, User2 can click on the trash can icon next to the filter that the admin made, and then refresh the page.
The admin's filter is deleted

**What do we learn?**
- Make something which only admin or high privilege user can do and then try the deleting privileged action done by admin from restricted/low privileged account. 

**Bug 7:** https://hackerone.com/reports/447975
**Description: ** Upgrade menu exposes the mobile application token meant to only be visible to administrators


- As an admin user, create a new mobile app (https://rpm.newrelic.com/accounts/1523936/mobile/setup)
- Once you're done setting it up, create a new user with restricted read-only settings for mobile, and log in as that user.
- As the restricted user, attempt to navigate to the settings page (https://rpm.newrelic.com/accounts/1523936/mobile/{MOBILE_ID}/edit) - you will be denied, and you won't be able to view the page with the application token
- Now, navigate to the upgrade page for the same mobile app (https://rpm.newrelic.com/accounts/1523936/mobile/{MOBILE_ID}/upgrade
and you will get the token for the app.
- This token for the app, is similar to the license token of the NR web, so this can be use to authenticate to app.

**What do we learn?**
- Look at all the requests that went through the burp when you are serving the application as a privieleged user.
- Send the same request as low privielged user to expose funtionality.

**Bug 8:** https://hackerone.com/reports/479139
**Description**: Bypass of #447975 - view mobile application token though "Application Information" sidebar on Installation page

- All steps are same as above but this time, token is exposed on app installation page.
- Navigate to the "Deploy" page which follows the following URL structure with low privileged user:
https://rpm.newrelic.com/accounts/1523936/mobile/{MOBILE_ID}/deploy
The mobile application token will be on the right hand side of the page:

**What do we learn?**
- As told in above bugs, if one endpoint is exposing confidential info, there can be other endpoints also which are leaking the same info.

- I will just put out comment here from the report which describe the feeling:

*Total RIPage, dude! You aint been slacking on yer hacking. Our fix was lacking some backing, so we'll get cracking! Otherwise I better start packing.*

**Bug 9: ** https://hackerone.com/reports/476958
**Description**: IDOR allows accounts to view full name of other accounts based on email through share notes feature

- This is the same issue as Bug3, but on a different endpoint.
- Admin add new user to the account  (https://account.newrelic.com/accounts/1523936/users/new)
- From your admin account, go and create a note and view the note:
https://rpm.newrelic.com/accounts/1523936/notes/6610

- From here, click on the "Share this note" button:
	- Now this section will show full usernames of other users.

**What do we learn?**
 - Look at all the feature which can expose confidential info.

**Bug 10**: https://hackerone.com/reports/634692
**Description**: Stored XSS Via NRQL chartbuilder JSON view
- In the Chart Builder, we can put NRQL queries to create the chart by querying the info.
- Within the chart builder, perform the following NRQL query:
```
SELECT `"><img src=x onerror=alert(document.domain)> "' Style=position` FROM SyntheticCheck
```

- This was the stored XSS, as all the recent queries are saved and showed to all the users of acount.

**What do we learn?**
- Try XSS payloads in all input fields.

**Bug 11: ** https://hackerone.com/reports/255685
**Description:** Restricted User can still integrate with AWS via forced browsing

 - Admin add -> User1 with restricted role(read only access to infrastructure)
 -  Now, as a restricted User1 if you go to infrastructure integration page(https://infrastructure.newrelic.com/accounts/%7B%7BACC#%7D%7D/integrations), you will see the message that integration is disabled at this time, contact admin.
 -  But, you can force browse  to the setup page (https://infrastructure.newrelic.com/accounts/%7B%7BACC#%7D%7D/integrations/accounts/wizard/1/step/1) and integrate your AWS account.


**What do we learn?**
- Look at the requests which high privileged account is doing, repeat same requests with low privileged account(force browsing).

**Bug 12:**  https://hackerone.com/reports/344309
**Description:** Adding a new user discloses their full name in the "Users" section of NR Alerts notification channels page
- Each user of NR can add himself to a notification under the notification channel.
- As an admin, add a new user USER1  to your NR account.
- Navigate to https://alerts.newrelic.com/accounts/ACCOUNT_ID/channels
- Observe that the full name of the user is disclosed in the "Users" section on this page. 
- Optionally, you can directly navigate to the user channel you added as well and the full name is not hidden:
https://alerts.newrelic.com/accounts/ACCOUNT_ID/channels/1081322
^ 1081322 is the User notification channel for the user USER1.

**What do we learn?**
 Here devs have forgotten to conceal the name of new users added to the account via the "channels" endpoint.
 
If one endpoint is fixed, look for same info in other endpoints.

**Bug 13**: https://hackerone.com/reports/344468

**Description**: User is able to access and create private synthetics locations without upgrading

- In a normal/trial plan, if a admin goes to Synthetics->Private Location(  https://synthetics.newrelic.com/accounts/1523936/private-locations), he is treated with a message to "contact sales".
- But, by force browsing to:  https://synthetics.newrelic.com/accounts/1523936/private-locations/new, you can make a private location synthetics.

**What do we learn?**
 - If you can't upgrade your account and see privielege request which you can force browse as non privilege user then try guessing the enndpoint and force browse it.

**Bug 14**: https://hackerone.com/reports/200576

**Description**: Logic flaw enables restricted account to access account license key

- Admin invites -> User1 as a restricted user to join the team
- User1 joins the team, then clicks on "Infrastructure" tab
- User1 clicks on "Start my 30 day free trial", then they are directed to this url: https://infrastructure.newrelic.com/accounts/1523936/install
- At this URL, User2 then clicks on "Windows" platform
License key(License key is applicable to full account) is in plain view on this page.

**What do we learn?**

 - Follow the same thing from a restricted account which you can follow with a high privilege account and keep an eye on response for confidential info.

**Bug 15**: https://hackerone.com/reports/320200
**Description**:  User with no Synthetics permissions can view synthetic monitor details through /internal_api/ endpoint

- Check **Bug 2** to know how to get info on /internal_api/
- Admin adds -> User1 with restricted/no permissions to view synthetics.
- When a user with no permissions try to view synthetics by navigating directly to the Synthetics monitor list (https://synthetics.newrelic.com/accounts/1523936/monitors), he gets an error "Permission required to view monitors"
- However, the restricted user can bypass this completely by sending this GET request to the internal API at this endpoint `/internal_api/1/accounts/<accountnumber>/conditions/load_synthetics_monitors`

**What do we learn?**
- Once you get a internal_api or some api endpoint , try your normal reuests through that internal_api endpoint.

Bug 16: https://hackerone.com/reports/520518

**Description: **Full name of other accounts exposed through NR API Explorer
- Admin adds -> User1
- Now using the new api(https://api.newrelic.com/v2/users.json) endpoint, full names of all the users are getting exposed.

**What do we learn?**
- Monitor new endpoints, or keep an eye on program updates for new api info.

**Bug 17**: https://hackerone.com/reports/347665
**Description**: Permissions leaks the full name of other NR accounts

- This is exactly same as **Bug 3** above, and due to the incorrect fix.
- Here the full name will appear in the add user in the group permission list when that particular use has logged in anytime in the past on the NR platform.

**What do we learn?**
- Bussiness logic :)

**Bug 18**: https://hackerone.com/reports/334143

**Descriptions**: Restricted User can add/modify alert conditions on monitors without any synthetics privileges

- Login as an admin and navigate to Synthetics. Make sure that Synthetics privileges are turned ON and the Restricted User is not given any privileges.
- Create a new monitor
- Navigate to the alert settings for the monitor (https://synthetics.newrelic.com/accounts/1523936/monitors/99657e19-ace3-483d-a5d4-d199f09e177b/conditions)
- Click on the "Add alert condition" button
Choose any alert condition and turn intercept on in Burp Suite before you click "Save".
There will be a POST request, now send this POST request with a restricted user cookies and CSRF token.
- You will see 200 Ok with an alert created. you can check this by logging back to admin account and check on the UI.

**What do we learn?**
- That's how IDOR can be found, by doing the action from a restriced/low privileged user.

**Bug 19**: https://hackerone.com/reports/332381

**Description**: Internal API endpoint discloses full account name of email address associated with unconfirmed user

- Admin add new user -> User1 (https://account.newrelic.com/accounts/1523936/users/new), notice that in UI, full names are masked.
- But, using /internal_api/ (https://alerts.newrelic.com/internal_api/1/accounts/%7BYOURACCOUNTNUMBER%7D/users/), it is exposing all names of users. 

**What do we learn?**
- User internal apis/api endpoints which you find during crawling or js analysis to find confidential info.


**Bug 20**: https://hackerone.com/reports/349291
**Description**: IDOR via internal_api "users" endpoint

- Again using /internal_api/( https://alerts.newrelic.com/internal_api/1/accounts/1523936/users/517408) endpoint, full name is getting exposed

**What do we learn?**
- Use internal apis/api endpoints which you find during crawling or js analysis to find confidential info.

**Bug 21: ** https://hackerone.com/reports/479135\

**Description**:  GET request to accounts.json on support site leaks the root account license key and the browser license key to a restricted user

- Admin User creates -> User1 (restricted User)
- Login as User1 and go to  https://support.newrelic.com/
- When User1 clicks on create ticket(/customer/accounts.json), license key is exposed.

**What do we learn?**
Look at each endpoint in burp, and check their response.

**Bug 22: **  https://hackerone.com/reports/459443

**Description**: Modify the filter settings for any NR Insights dashboard through internal_api endpoint

- An IDOR exists that allows to change the filter settings of any account on New Relic through the  PUT request to /internal_api/1/accounts/ACCOUNTID/dashboards/DASHBOARDNUMBER/filter
- dashboard_id within the request body is not being validated in the request body, so it can be changed to victim account.


**What do we learn?**
Simple IDOR where we manipulate the id to the victime account.

**Bug 23 ** https://hackerone.com/reports/320689

**Description** Restricted user can view synthetics monitors and user permissions through .json endpoint at /permissions/securablemetadata/{GROUP ID}

- When a restricted user with no permissions to view synthetics monitors tries to navigate to the permissions settings within Synthetics (https://synthetics.newrelic.com/accounts/1523936/permissions). he does not see anything.
- But, there is a request being made which can be seen using proxy to endpoint: https://synthetics.newrelic.com/accounts/accounid/permissions/securablemetadata/groupid.json
- This was exposing all the monitor info.

**What do we learn?**
- Check all the requests that are happening in background. sometimes you will see request to api/other endpoint which is to query if the user has permission or not but in response it leaks info.


**Bug 24:** https://hackerone.com/reports/386556

**Description** Internal API exposes Synthetics monitor details to a restricted user without view monitor permissions

- Admin -> User 1(restricted with no permission to synthetics)
- From admin create a Synthetics monitor to fail each time, example: create a monitor with a URL that checks for the ping response from a website that doesn't exist, such as "lakfnas8df8hasfd.com".
- Add an alert monitor that checks for a failure of this monitor.
-  once you have this setup, you're going to get a lot of failures in NR Alerts, which will appear in the Events > Violations section.
-  Now as a restricted user, if you go this Events > Violations section, you will not see much of the info about monitor. You can see the name of the monitor, and that the monitor had an alert that failed, but you can't tell why or what asset is being monitored.
-  However - there is an internal API JSON endpoint that leaks all this hidden info to the restricted user:

https://alerts.newrelic.com/internal_api/1/accounts/1523936/monitor_failures/MonitorID?limit=1&reverse=true&start_timestamp=1

- You can get the monitorID as restricted user by just hovering over the monitor on the incident section.

**What do we learn?**
Use internal apis/api endpoints which you find during crawling or js analysis to find confidential info.

**Bug 25: ** https://hackerone.com/reports/397483

**Description** Restricted user can update integration provider account name via integrations API

- NR Infra allows you to create a GCP integration and set a custom name.
- By default, the restricted user account can't change anything associated associated with this type of infra integration
- But there is a missing check on changing the name of the integration through the integrations API (https://integrations-api.service.newrelic.com/api/v1/accounts/AcountID/provider_accounts/GCPIntegrationID)
- From the restricted user account, send the PUT request to above endpoint and change the name to something else via the {"name":"CHANGED"} parameter within the request.

**What do we learn?**
- Try to use the api to do some privilege tasks from the low level account.

**Bug 26: **  https://hackerone.com/reports/462321

**Description**: Ability to view monitor names of other NR accounts through internal API (v3) via "monitor_id" parameter.

- In NR, we can see alert policy on a synthetic monitor, to get the alerts on synthetic moitor failure or some condition.
- Create a synthetic monitor from victim's id and note down its monitor id.
- Now, login as attacker, and create a new alery policy.
- Create a condition "Synthetics" and name anything.
- Capture the save request and change the current monitor id to the one which we have taken from victim.
- Send the request through and you'll see that the monitor name has been changed to reflect the monitor belonging to the other account, indicating the IDOR was successful


**What do we learn?**
Change the ID in request to victim account to find IDORs.

**Bug 27: **  https://hackerone.com/reports/380413

**Description**: Restricted user can bypass permissions restriction to create NR Alert policies

- When a restricted user navigates to NR Alerts, they don't have the option to create a new alert policy. 
- This can be bypassed by sending a custom POST request to infrastructure-data.service.newrelic.com(/accounts/{ACCOUNT_ID}/policies) endpoint

**What do we learn?**
Have a look at other subdomain requests also, sometimes depending on the funtionality, different endpoints of different subdomains get called.

**Bug 28: **  https://hackerone.com/reports/388743

**Description**: Data app permissions setting does not fully prevent other users from modifying/changing data related to your data app

- Create "data app" as the admin(https://insights.newrelic.com/apps/accounts/account ID)
- Make permissions setting as "Visible to others in my account" and fill any details as title.
- Now, from the "restricted" user who shouldn't have the ability to edit this data app, login to New Relic and head back to the apps section (https://insights.newrelic.com/apps/accounts/account ID)
- Now, In order to set this up correctly, go within Burp Suite Professional, then go to Proxy > Options > scroll down to "Match and Replace" and click on "Add" and fill

Match: "editable":false
Replace: "editable":true

- Now, when you visit a data app that is set to "visible to others on my account" made by another user, you get the nice edit options in the upper right hand corner and api is able to accept our change

**What do we learn?**
Always check response of all requests for keywords like "editable", acceptable, true, false etc.


**Bug 29:**:  https://hackerone.com/reports/501672

**Descriptions:** Restricted user can view all account invoices, payment method details, PII of account owner through zoura_api endpoints

- NR switched to using Zoura (https://www.zuora.com/) to handle your New Relic customer subscriptions. 
- A restricted user without administrative privileges cannot view and data associated with the billing page (https://rpm.newrelic.com/accounts/1523936/payments) and the link returns a 403. 
- The Zoura API endpoints, however, are available for the restricted user to access, and leads to the exposure of the following sensitive information:

Last four digits, cardholder name, card type, and expiration date of credit card on file of the account owner or billing contact
https://self-serve.service.newrelic.com/accounts/ACCOUNT_ID/zuora_api/payment_methods

All invoice amounts processed after November 2018
https://self-serve.service.newrelic.com/accounts/ACCOUNT_ID/zuora_api/invoices

An account summary showing non-sensitive information about the default payment method and whether the account has future subscriptions
https://self-serve.service.newrelic.com/accounts/ACCOUNT_ID/zuora_api/account_summary

**What do we learn?**
Look for changes in the app, keep monitoring the js file for endpoint changes. In this case, on seeing the requests to the zuora endpoint during the payment, anyone can get access to confidential info.


**Bug 30:**:  https://hackerone.com/reports/268541

**Descriptions:**  Individual account permissions are not properly managed and inherited on sub accounts.

- Since a user management overhaul happened few permissions issues came in the app.
- IN NR, When you have a sub account on your account, you get this message:
`Anybody with permission to access this account will also have the same permissions on these sub-accounts: {{name of the sub account}}`

- But this seems to be broken.
- Now, if Admin adds a User1 as admin privileges on main account as well as adming privileges on sub account.
- But lets say user1 leave the org and in that case admin removes the User1 from main account but User1 will still remain admin of sub account.

This is breaking the above statement where a user of sub account inherits the permission of main account, but with the current scenario, a sub account user priviliges always overwrites the main account pivileges.

**What do we learn?**
- This is purely a bussiness logic error which can be find out by reading docs related to privielges.
