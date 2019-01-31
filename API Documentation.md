# How to find BizTalk360 APIs and how to call API within custom widget?

The architecture of BizTalk360 has four main components
 
*	Web Interface Layer
*	REST API Layer
*	BizTalk360 Monitoring Service
*	BizTalk360 Database

All communications between the web interface layer and the BizTalk environment take place through the REST API layer.

There may be occasions when organizations face the need to show information from the BizTalk environment on external applications such as SharePoint, internal applications, internal dashboards, and so on. To achieve this, organizations are forced to write their own piece of code that will fetch the information from the BizTalk environment and display them in the external applications.

With API Documentation capability of BizTalk360, there are over 350 REST services (APIs) have exposed that have the capability to retrieve the requested information from the BizTalk server environment.
Retrieve and modify information BizTalk Server or BizTalk360 done using one of the following two methods

*	GET
*	POST

# Retrieve information about the BizTalk environment (Using HTTP GET service)
The following API call get the web endpoint details using GET method. 
```
webEndpointURL = 'http://localhost/BizTalk360.Dev/Services.REST/AlertService.svc/GetAlertMonitorSerializedConfig';
getWebEndpointMonitoringConfiguration = function (callback) {
            var _this = this;
            $.ajax({
                dataType: "json",
                url: webEndpointURL,
                type: "GET",
                username: _this.username,
                password: _this.password,
                data: {
  environmentId: _this.environmentId, alarmId: _this.alarmId, monitorGroupType: _this.monitorGroupType, monitorGroupName: _this.monitorGroupName, monitorName: _this.monitorName
                },
                cache: false,
                success: function (data) {
                  if(callback)
                    callback(data);
                },
                error: function (xhr, ajaxOptions, thrownError) { 
                    alert(xhr.status);
                    alert(xhr.responseText);
                },
            });
        };
```
# Modify information in the BizTalk environment or BizTalk360 database (using HTTP POST service)
The following API uses POST method to execute the custom SQL query on BizTalk360 database.
```
url = 'http://' + bt360server + '/BizTalk360/Services.REST/BizTalkGroupService.svc/ExecuteCustomSQLQuery';
getbizTalkSendPorts = function (callback) {
      var _this = this;			      
      $.ajax({
         dataType: "json",
         url: _this.url,
         type: "POST",
         contentType: "application/json",
         username: _this.username,
         password: _this.password,
         data: '{"context":{"environmentSettings":{"id":"'+ _this.environmentId +'","licenseEdition":0},"callerReference":"REST-SAMPLE"},"query":{"id":"'+ _this.queryId + '","name":"' + _this.queryName + '","sqlInstance":"' + _this.sqlInstance + '","database":"'+ _this.database +'","sqlQuery":"' + _this.sqlQuery + '","isGlobal":false}}',
         cache: false,
         success: function (data) {
            callback(data);
         },
         error: function (xhr, ajaxOptions, thrownError) {
            alert(xhr.status);
            alert(xhr.responseText);
         },
      });
   };
```
