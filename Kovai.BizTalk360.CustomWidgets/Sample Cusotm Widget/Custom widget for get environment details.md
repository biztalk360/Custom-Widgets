# Custom Widget using Secure SQL Query

To help you get started with creating Custom Widgets using Secure SQL Query, you can use below template.<br /><br />
The following custom widget shows BizTalk360 environment details. We firstly need to create the Secure SQL Query, we would like to have as a widget, in the Secure SQL Queries feature. You can find this feature under Operations and then Data Access. To create custom widget using Secure SQL Query refer [this](https://docs.biztalk360.com/docs/creating-a-custom-widget-for-executing-secure-sql-queries) article.<br /><br />
In the below template secure SQL query is executed using POST method. Commonly payload for API call accepts only JSON data, so here payload object is converted to JSON data like follows

![SampleCustomTemplateWidget](https://github.com/biztalk360/Custom-Widgets/blob/master/Kovai.BizTalk360.CustomWidgets/Images/SampleCustomTemplateWidget.png)

```
<div id="WidgetScroll" style="top:50px;" data-bind="addScrollBar: WidgetScroll, scrollCallback: 'false'">
    <table class="table table-lists">
        <thead>
            <tr>
                <!-- <th style="width:30%">Environment Id</th> -->
                <th style="width:30%">Environment Name</th>
                <th style="width:30%">Management DB Name</th>
                <th style="width:30%">BizTalk Management Instance Details</th>
            </tr>
        </thead>
        <tbody>
            <!-- ko if: (bizTalkSendPorts()) -->
            <!-- ko foreach: bizTalkSendPorts() -->
            <tr>                
                <!-- <td data-bind="text: EnvironmentId"></td> -->
                <td data-bind="text: Name"></td>
                <td data-bind="text: BizTalkMgmtDb"></td>
                <td data-bind="text: BizTalkMgmtSqlInstance"></td>
            </tr>
            <!-- /ko -->
            <!-- /ko -->
        </tbody>
    </table>
</div>
<script>
    // BEGIN User variables
   username = "";      // BizTalk360 service account
   password =  "";     // Password of BizTalk360 service account
   environmentId = ""; // BizTalk360 Environment ID  (take from SSMS or API Documentation)
   queryId = "";       // Id of the Secure SQL Query (take from SSMS)
   queryName = "";     // Name of the Secure SQL Query as it is stored under Operations/Secure SQL Query
   sqlInstance = "";   // SQL Instance against which the SQL Query must be executed
   database = "";      // Database against which the SQL Query must be executed
   sqlQuery = "";      // The SQL Query which needs to be executed from the custom Widget
   bt360server = "";   // Name of the BizTalk360 server (needed to do an API call to execute the SQL query
   // END User variables

    url = 'http://localhost/BizTalk360/Services.REST/BizTalkGroupService.svc/ExecuteCustomSQLQuery';
    bizTalkSendPorts = ko.observable();

    x2js = new X2JS({ attributePrefix: '', arrayAccessForm: "property", arrayAccessFormPaths: ["root.records.record"] });

    bizTalkSendPortsList = function () {
        var _this = this;
        _this.getbizTalkSendPorts(function (data) {

            var results = x2js.xml_str2json(data.queryResult);
            if (Array.isArray(results.root.records.record))
                _this.bizTalkSendPorts(results.root.records.record);
            else {
                _this.bizTalkSendPorts([results.root.records.record]);
            }
        });
    };
    getbizTalkSendPorts = function (callback) {
        var _this = this;
        var dataObj = {
            "context":
            {
                "environmentSettings": { "id": _this.environmentId, "licenseEdition": 0 },
                "callerReference": "REST-SAMPLE"
            },
            "query": { "id": + _this.queryId, "name": _this.queryName, "sqlInstance": _this.sqlInstance, "database": _this.database, "sqlQuery": _this.sqlQuery, "isGlobal": false }
        };        
        $.ajax({
            dataType: "json",
            url: _this.url,
            type: "POST",
            contentType: "application/json",
            username: _this.username,
            password: _this.password,
            data: ko.toJSON(dataObj),
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
    bizTalkSendPortsList();
</script>
```
