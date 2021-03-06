To help you to get started with creating Custom Widgets using BizTalk360 API, you can use below template. <br />
The following custom widget shows web endpoint status details in BizTalk360.

![WebEndpointWidget](https://github.com/biztalk360/Custom-Widgets/blob/master/Kovai.BizTalk360.CustomWidgets/Images/WebEndpointWidget.png)

```
<html>
<body>
    <div id="WidgetScroll" style="top:30px;" data-bind="addScrollBar: WidgetScroll, scrollCallback: 'false'">
        <div class="col-md-12">
            <div class="grid-large-normal" id="appsList">
                <div id="basic" class="alarm-section">
                    <h4>Web Endpoints </h4>
                </div>
                <table class="table table-lists">
                    <thead>
                        <tr>
                            <th>Name</th>
                            <th>Endpoint</th>
                            <th>Status</th>
                        </tr>
                    </thead>
                    <tbody data-bind="foreach: endpointLists">
                        <tr>
                            <td><span data-bind="text:name()"></span></td>
                            <td><span data-bind="text:endpointUrl()"></span></td>
                            <td>
                                <!--ko if: $data.status() == -1 -->
                                <span class="success-tag">
                                    <i class="fa fa-check"></i>
                                    <span>HEALTHY</span>
                                </span>
                                <!--/ko-->

                                <!--ko if: $data.status() == 0 -->
                                <span class="warning-tag">
                                    <i class="fa fa-exclamation-triangle"></i>
                                    <span>WARNING</span>
                                </span>
                                <!--/ko-->

                                <!--ko if: $data.status() == 1 -->
                                <span class="failure-tag">
                                    <i class="fa fa-times"></i>
                                    <span>ERROR</span>
                                </span>
                                <!--/ko-->

                                <!--ko if: $data.status() == 2 -->
                                <span class="info-tag">
                                    <i class="fa fa-question" aria-hidden="true"></i>
                                    <span>NOT VALIDATED</span>
                                </span>
                                <!--/ko-->
                            </td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>
    </div>
    <script>
        username = ""; //Create a placeholder in biztalk360 application for the username and password
        password = "";
        environmentId = "";
        alarmId = "";
        monitorGroupType = "";
        monitorGroupName = "";
        monitorName = "";
        endpointStatusLists = ko.observableArray([]);
        endpointLists = ko.observableArray([]);
        var count = 0;
        //URLs     
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
                    if (callback) {

                        _this.endpointStatusLists([]);
                        ko.utils.arrayForEach(JSON.parse(data.serializedMonitorConfig), function (endpointData) {
                            _this.endpointStatusLists.push(
                                new endpointStatusFunc(endpointData.name, endpointData.endPoint, endpointData.issue.IssueType)
                            );
                        }, this);

                        var tmpArray = _this.endpointStatusLists.sort(function (left, right) { return left.name() == right.name() ? 0 : (left.name() < right.name() ? -1 : 1) });
                        _this.endpointLists([]);
                        ko.utils.arrayForEach(tmpArray, function (data) {

                            _this.endpointLists.push(data);
                        });
                        callback(data);
                    }
                },
                error: function (xhr, ajaxOptions, thrownError) { //Add these parameters to display the required response
                    alert(xhr.status);
                    alert(xhr.responseText);
                },
            });

        };

        function endpointStatusFunc(name, endpointUrl, status) {
            var self = this;
            self.name = ko.observable(name);
            self.endpointUrl = ko.observable(endpointUrl);
            self.status = ko.observable(status);
        }

        getWebEdnpointStatus = function () {
            var _this = this;
            _this.getWebEndpointMonitoringConfiguration(function (data) {
                this.endpointStatusLists([]);
                ko.utils.arrayForEach(JSON.parse(data.serializedMonitorConfig), function (endpointData) {
                    this.endpointStatusLists.push(
                        new endpointStatusFunc(endpointData.name, endpointData.endPoint, endpointData.issue.IssueType)
                    );
                }, this);
            })

            var tmpArray = this.endpointStatusLists.sort(function (left, right) { return left.name() == right.name() ? 0 : (left.name() < right.name() ? -1 : 1) });
            this.endpointLists([]);
            ko.utils.arrayForEach(tmpArray, function (data) {

                this.endpointLists.push(data);
            });
        }

        getWebEndpointDetails = function () {
            $.when(this.getWebEndpointMonitoringConfiguration()).then(this.getWebEdnpointStatus());
        };

        getWebEndpointDetails();
        setInterval(getWebEndpointDetails, 60 * 1000);  //Interval time for refresh, as of now it has been set to 1 min

    </script>
</body>
</html>
```
