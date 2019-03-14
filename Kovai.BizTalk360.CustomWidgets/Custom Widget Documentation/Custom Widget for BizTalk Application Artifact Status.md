# Custom Widget for BizTalk Application Artifact Status

The following custom widget shows BizTalk Application Artifact Status details, which are mapped with BizTalk360 Alarms. We firstly need to create the Secure SQL Query, we would like to have as a widget, in the Secure SQL Queries feature. You can find this feature under Operations and then Data Access. To create custom widget using Secure SQL Query refer [this](https://docs.biztalk360.com/docs/creating-a-custom-widget-for-executing-secure-sql-queries) article.<br /><br />
In the below template secure SQL query is executed using POST method. Commonly payload for API call accepts only JSON data, so here payload object is converted to JSON data and binding with GOJs Graph for graphical representation.

![SampleCustomTemplateWidget](https://github.com/biztalk360/Custom-Widgets/blob/master/Kovai.BizTalk360.CustomWidgets/Images/ApplicationArtifactStatus.png)


```
<div id="WidgetScroll" style="top:30px;">   
    <div class="query-builder margin-t">
        <a data-toggle="collapse" data-bind="attr: { href: '#graph-artifact' }" class="collapse-trigger collapsed">Filters</a>
        <div data-bind="attr: { id: 'graph-artifact' }" class="collapse">          
                <div class="row" style="padding-top: 12px;">
                    <div class="col-md-9">
                        <div class="form-group row">
                            <label class="col-md-2" style="font-size: 15px;" for="alarm">Alarm</label>
                            <div class="col-md-8">
                                <select
                                    data-bind="kendoMultiSelect: { data:AlarmDetailsForApplicationArtifact,dataTextField:'AlarmName',dataValueField:'AlarmName',value:selectedAlarmForArtifact}"
                                    multiple="multiple"></select>
                            </div>
                            <div>
                                <button class="btn" id="select" data-bind="click:selectAllAlarmsForArtifact">
                                    <a data-bind="bsToolTip: { placement: 'top' }" data-container="body"
                                        data-toggle="tooltip" data-original-title="Select All">
                                        <i class="fa fa-check-square-o"></i>
                                    </a>
                                </button>
                                <button class="btn" id="deselect" data-bind="click:deSelectAllAlarmsForArtifact">
                                    <a data-bind="bsToolTip: { placement: 'top' }" data-container="body"
                                        data-toggle="tooltip" data-original-title="Deselect All">
                                        <i class="fa fa-square-o"></i>
                                    </a>
                                </button>
                            </div>
                        </div>
                        <div class="form-group row">
                            <label class="col-md-2 control-label" style="font-size: 15px;" for="status">Status</label>
                            <div class="col-md-10">
                                <label class="swich-container">
                                    <input id="chbHighImportance" type="checkbox"
                                        data-bind="kendoMobileSwitch: { checked: includeArtifactHealthyStatus, onLabel: 'YES', offLabel: 'NO' }" />
                                    <span class="caption">Include Healthy Artifacts</span>
                                </label>
                            </div>
                        </div>
                    </div>
                    <div class="col-md-3">
                        <button class="btn btn-primary btn-medium" data-bind="click:filterArtifact" id="select"><i
                                class="fa fa-sliders" aria-hidden="true"></i> Search
                        </button>
                    </div>
                </div>           
        </div>
    </div>
    <div id="applicationArtifactGrid" style="border: solid 1px white; height: 600px"></div>
</div>
<script>
    // BEGIN User variables
    applicationArtifactRefresh = 10;
    username = "BizTalk360"; // BizTalk360 service account
    password = "Rajsree9656624199"; // Password of BizTalk360 service account
    environmentId = "a3d01fb3-0f1b-46e5-9ec0-2dbb9a3f2513"; // BizTalk360 Environment ID  (take from SSMS or API Documentation)
    applicationStatusQueryId = "a1b1c020-cc02-4e67-b785-23510275fa15"; // Id of the Secure SQL Query (take from SSMS)
    applicationQueryName = "Application Artifact Status"; // Name of the Secure SQL Query as it is stored under Operations/Secure SQL Query
    sqlInstance = "BT360DEV22"; // SQL Instance against which the SQL Query must be executed
    database = "BizTalk360"; // Database against which the SQL Query must be executed
    applicationStatusQuery = "Select AA.Id,AA.[Name],AME.MonitorStatus,AME.ExecutionResult from [dbo].[b360_alert_MonitorExecution] AME Inner Join[dbo].[b360_alert_Alarm] AA ON AME.AlarmId = AA.Id and AME.MonitorGroupType = 'Application' WHERE AME.LastExecutionTime = (SELECT MAX(LastExecutionTime) from [dbo].[b360_alert_MonitorExecution] WHERE MonitorGroupType = 'Application' and AA.IsAlertDisabled='false' and AlarmId = AA.Id and LastExecutionTime >= DATEADD(MINUTE,-60,GETUTCDATE()))AND AME.EnvironmentId='a3d01fb3-0f1b-46e5-9ec0-2dbb9a3f2513'"// The SQL Query which needs to be executed from the custom Widget
    bt360server = "localhost"; // Name of the BizTalk360 server (needed to do an API call to execute the SQL query
    //Mention the created alarm details and the respective partner name to display in the graph.
    AlarmDetailsForApplicationArtifact = [
        { AlarmName: "Threshold Alarm-1", PartnerName: "Partner-1" },
        { AlarmName: "Threshold Alarm-2", PartnerName: "Partner-2" },
        { AlarmName: "Threshold Alarm-3", PartnerName: "Partner-3" },
        { AlarmName: "Threshold Alarm-4", PartnerName: "Partner-4" }
    ];
    // END User variables

    executeApplicationStatusUrl =
        "http://" +
        bt360server +
        "/BizTalk360/Services.REST/BizTalkGroupService.svc/ExecuteCustomSQLQuery";
    applicationMonitorStatusDetails = ko.observableArray();
    artifactNodeArray = [
        {
            key: 1,
            name: "Application Artifacts",
            overallStatus: "RootNode",
            url: 'alrmapplications',
            module: ''
        }
    ];
    selectedAlarmForArtifact = ko.observableArray();

    x2js = new X2JS({
        attributePrefix: "",
        arrayAccessForm: "property",
        arrayAccessFormPaths: ["root.records.record"]
    });
    graphKey = 1;
    includeArtifactHealthyStatus = ko.observable(false);
    GetArtifactMonitorStatus = function () {
        var _this = this;
        _this.applicationMonitorStatusDetails.removeAll();
        _this.executeArtifactStatusCustomSqlQuery(function (data) {
            var applicationResults = x2js.xml_str2json(data.queryResult);
            if (applicationResults.root.records.record != null) {
                if (Array.isArray(applicationResults.root.records.record)) {
                    ko.utils.arrayForEach(
                        applicationResults.root.records.record,
                        function (item) {
                            _this.mappedApplicationArtifactData(item);
                        }
                    );
                } else {
                    _this.mappedApplicationArtifactData(applicationResults.root.records.record);
                }
                _this.artifactGraphData(_this.applicationMonitorStatusDetails());
                _this.artifactMyDiagram.div = null;
                _this.displayArtifactGraph();

            }
        });
    };
    mappedApplicationArtifactData = function (data) {
        var _this = this;
        var applicationDetails =
        {
            "id": "",
            "alarmName": "",
            "overallStatus": "",
            "mappedApplications": []
        };
        var result = x2js.xml_str2json(data.ExecutionResult);
        applicationDetails.id = data.Id;
        applicationDetails.alarmName = data.Name;
        applicationDetails.overallStatus = result.MonitorGroupData.monitorStatus;
        if (Array.isArray(result.MonitorGroupData.monitorGroups.MonitorGroup)) {
            ko.utils.arrayForEach(result.MonitorGroupData.monitorGroups.MonitorGroup, function (app) {
                var mappedApplication = {
                    "applicationName": "",
                    "monitorStatus": "",
                    "monitors": []
                }
                mappedApplication.applicationName = app.name;
                mappedApplication.monitorStatus = app.monitorStatus;
                ko.utils.arrayForEach(app.monitors.Monitor, function (monitor) {
                    if (monitor.monitorStatus != "NotConfigured") {
                        var mappedArtifact = {
                            "artifactTypeName": "",
                            "monitorStatus": "",
                            "issues": []
                        };
                        mappedArtifact.artifactTypeName = monitor.name;
                        mappedArtifact.monitorStatus = monitor.monitorStatus;
                        if (Array.isArray(monitor.issues.Issue)) {
                            ko.utils.arrayForEach(monitor.issues.Issue, function (issue) {
                                mappedArtifact.issues.push(issue.description);
                            });
                        } else {
                            if (monitor.issues.Issue != undefined) {
                                mappedArtifact.issues = new Array(monitor.issues.Issue.description);
                            }
                        }
                        mappedApplication.monitors.push(mappedArtifact);
                    }
                });
                applicationDetails.mappedApplications.push(mappedApplication);
            });
            _this.applicationMonitorStatusDetails.push(applicationDetails);
        }
        else {
            var mappedApplication = {
                "applicationName": "",
                "monitorStatus": "",
                "monitors": []
            };
            mappedApplication.applicationName = result.MonitorGroupData.monitorGroups.MonitorGroup.name;
            mappedApplication.monitorStatus = result.MonitorGroupData.monitorGroups.MonitorGroup.monitorStatus;
            ko.utils.arrayForEach(result.MonitorGroupData.monitorGroups.MonitorGroup.monitors.Monitor, function (monitor) {
                if (monitor.monitorStatus != "NotConfigured") {
                    var mappedArtifact = {
                        "artifactTypeName": "",
                        "monitorStatus": "",
                        "issues": []
                    };
                    mappedArtifact.artifactTypeName = monitor.name;
                    mappedArtifact.monitorStatus = monitor.monitorStatus;
                    if (Array.isArray(monitor.issues.Issue)) {
                        ko.utils.arrayForEach(monitor.issues.Issue, function (issue) {
                            mappedArtifact.issues.push(issue.description);
                        });
                    } else {
                        if (monitor.issues.Issue != undefined) {
                            mappedArtifact.issues = new Array(monitor.issues.Issue.description);
                        }
                    }
                    mappedApplication.monitors.push(mappedArtifact);
                }
            });
            applicationDetails.mappedApplications.push(mappedApplication);
            _this.applicationMonitorStatusDetails.push(applicationDetails);
        }

    };
    executeArtifactStatusCustomSqlQuery = function (callback) {
        var _this = this;
        $.ajax({
            dataType: "json",
            url: _this.executeApplicationStatusUrl,
            type: "POST",
            contentType: "application/json",
            username: _this.username,
            password: _this.password,
            data:
                '{"context":{"environmentSettings":{"id":"' +
                _this.environmentId +
                '","licenseEdition":0},"callerReference":"REST-SAMPLE"},"query":{"id":"' +
                _this.applicationStatusQueryId +
                '","name":"' +
                _this.applicationQueryName +
                '","sqlInstance":"' +
                _this.sqlInstance +
                '","database":"' +
                _this.database +
                '","sqlQuery":"' +
                _this.applicationStatusQuery +
                '","isGlobal":false}}',
            cache: false,
            success: function (data) {
                callback(data);
            },
            error: function (xhr, ajaxOptions, thrownError) {
                alert(xhr.status);
                alert(xhr.responseText);
            }
        });
    };
    artifactGraphInit = function () {
        var _this = this;
        //if (window.goSamples) goSamples(); // init for these samples -- you don't need to call this
        var $ = go.GraphObject.make; // for conciseness in defining templates
        _this.artifactMyDiagram = $(go.Diagram, "applicationArtifactGrid", {
            // make sure users can only create trees
            "animationManager.isEnabled": false,
            validCycle: go.Diagram.CycleDestinationTree,
            // users can select only one part at a time
            maxSelectionCount: 0,
            // mouse wheel zooms instead of scrolls
            "toolManager.mouseWheelBehavior": go.ToolManager.WheelZoom,
            layout: $(go.TreeLayout, {
                treeStyle: go.TreeLayout.StyleLastParents,
                arrangement: go.TreeLayout.ArrangementHorizontal,
                // properties for most of the tree:
                angle: 90,
                layerSpacing: 35,
                // properties for the "last parents":
                alternateAngle: 90,
                alternateLayerSpacing: 35,
                alternateNodeSpacing: 20,
                alternateAlignment: go.TreeLayout.AlignmentBus
            })
        });
        _this.artifactMyDiagram.addDiagramListener("InitialLayoutCompleted", function (e) {
        });
        _this.artifactMyDiagram.nodeTemplate = $(go.Node, "Vertical", new go.Binding("text", "name"),
            $(go.Panel, "Auto",
                $(go.Shape, "RoundedRectangle", {
                    fill: 'gray',
                    stroke: null,
                    cursor: "pointer"
                }, new go.Binding("fill", "overallStatus", function (v) {
                    if (v == "Critical") return "rgb(231, 76, 60)";
                    if (v == "Warning") return "rgb(241, 196, 15)";
                    if (v == "Monitor Error") return "rgb(64,64,64)";
                    if (v == "RootNode") return "#2672EC";
                    if (v == "Healthy") return "rgb(39, 174, 96)";
                })
                ),
                $(go.Panel, "Vertical", new go.Binding("itemArray", "items"),
                    {
                        itemTemplate: $(
                            go.Panel,
                            "Auto",
                            { margin: 1 },
                            $(go.TextBlock, new go.Binding("text", ""), {
                                font: "9.5pt sans-serif",
                                background: "rgb(237, 239, 242)",
                                overflow: go.TextBlock.OverflowEllipsis,
                                maxLines: 3,
                                width: 200
                            })
                        )
                    }
                ),
                $(go.Panel, "Horizontal",
                    $(go.Shape,
                        {
                            geometryStretch: go.GraphObject.Uniform,
                            fill: "white", strokeWidth: 0, height: 30,
                            margin: new go.Margin(6, 8, 6, 10),
                        }, new go.Binding("", "", function (properties, node) {
                            var value = go.Geometry.fillPath("M384 1184v64q0 13-9.5 22.5t-22.5 9.5h-64q-13 0-22.5-9.5t-9.5-22.5v-64q0-13 9.5-22.5t22.5-9.5h64q13 0 22.5 9.5t9.5 22.5zm0-256v64q0 13-9.5 22.5t-22.5 9.5h-64q-13 0-22.5-9.5t-9.5-22.5v-64q0-13 9.5-22.5t22.5-9.5h64q13 0 22.5 9.5t9.5 22.5zm0-256v64q0 13-9.5 22.5t-22.5 9.5h-64q-13 0-22.5-9.5t-9.5-22.5v-64q0-13 9.5-22.5t22.5-9.5h64q13 0 22.5 9.5t9.5 22.5zm1152 512v64q0 13-9.5 22.5t-22.5 9.5h-960q-13 0-22.5-9.5t-9.5-22.5v-64q0-13 9.5-22.5t22.5-9.5h960q13 0 22.5 9.5t9.5 22.5zm0-256v64q0 13-9.5 22.5t-22.5 9.5h-960q-13 0-22.5-9.5t-9.5-22.5v-64q0-13 9.5-22.5t22.5-9.5h960q13 0 22.5 9.5t9.5 22.5zm0-256v64q0 13-9.5 22.5t-22.5 9.5h-960q-13 0-22.5-9.5t-9.5-22.5v-64q0-13 9.5-22.5t22.5-9.5h960q13 0 22.5 9.5t9.5 22.5zm128 704v-832q0-13-9.5-22.5t-22.5-9.5h-1472q-13 0-22.5 9.5t-9.5 22.5v832q0 13 9.5 22.5t22.5 9.5h1472q13 0 22.5-9.5t9.5-22.5zm128-1088v1088q0 66-47 113t-113 47h-1472q-66 0-113-47t-47-113v-1088q0-66 47-113t113-47h1472q66 0 113 47t47 113z");
                            if (value === null) {
                                node.visible = false;
                            }
                            else {
                                node.geometryString = value;
                                node.visible = false;
                            }
                        })),
                    $(go.Panel, "Table", {
                        maxSize: new go.Size(150, 999),
                        margin: new go.Margin(3, 3, 0, 3),
                        defaultAlignment: go.Spot.Left
                    }, $(go.RowColumnDefinition, {
                        column: 2,
                        width: 4
                    }), $(go.TextBlock, {
                        name: 'LablePart',
                        row: 0,
                        column: 0,
                        columnSpan: 5,
                        editable: false,
                        isMultiline: false,
                        stroke: "white",
                        minSize: new go.Size(10, 14),
                        cursor: "pointer"
                    }, new go.Binding("text", "name").makeTwoWay(), {
                            click: function (e, obj) {
                                _this.redirectToApplicationPath(obj.part.data);
                            },
                            mouseEnter: function (e, obj, prev) {
                                var shape = obj.part.findObject("LablePart");
                                if (shape)
                                    shape.isUnderline = true;
                            },
                            mouseLeave: function (e, obj, next) {
                                var shape = obj.part.findObject("LablePart");
                                if (shape)
                                    shape.isUnderline = false;
                            }
                        }))
                )//End of Horizontal Panel
            ),//end of Auto Panel
            $(go.Panel, { height: 15 },
                $("TreeExpanderButton"),
                {
                    width: 14,
                    "ButtonIcon.stroke": "white",
                    "ButtonBorder.fill": "darkblue",
                    "ButtonBorder.stroke": "transparent",
                    "_buttonFillOver": "darkred",
                    "_buttonStrokeOver": "red",
                }, new go.Binding("visible", "", function (properties) {
                })
            )
        );
        _this.artifactMyDiagram.linkTemplate = $(go.Link, {
            routing: go.Link.Orthogonal,  // may be either Orthogonal or AvoidsNodes
            corner: 5,
            relinkableFrom: false,
            relinkableTo: false
        }, $(go.Shape, {
            strokeWidth: 3,
            stroke: "#00a4a4"
        }));
    };
    artifactGraphData = function (artifactData) {
        var _this = this;
        var data = [];
        _this.artifactNodeArray = [{
            key: 1,
            name: "Application Artifacts",
            overallStatus: "RootNode",
            url: 'alrmapplications',
            module: ''
        }];
        if (_this.selectedAlarmForArtifact().length != 0) {
            ko.utils.arrayForEach(_this.selectedAlarmForArtifact(), function (item) {
                var item = _.findWhere(artifactData, {
                    alarmName: item
                });
                if (item != undefined) {
                    data.push(item);
                }
            });

        } else {
            data = artifactData;
        }
        for (var i = 0; i < data.length; i++) {
            var node;
            if (data[i] != undefined) {
                if (data[i].overallStatus == "Healthy") {
                    if (_this.includeArtifactHealthyStatus()) {
                        var partnerDetail = _.findWhere(_this.AlarmDetailsForApplicationArtifact, {
                            AlarmName: data[i].alarmName
                        });
                        _this.artifactParentNode(
                            partnerDetail.PartnerName,
                            data[i].overallStatus
                        );
                        var appParentKey = _this.graphKey;
                        ko.utils.arrayForEach(data[i].mappedApplications, function (app) {
                            _this.artifactApplicationNode(app.applicationName, app.monitorStatus, appParentKey, data[i].id, data[i].alarmName);
                            var appArtifactKey = _this.graphKey;
                            var appName = app.applicationName;
                            ko.utils.arrayForEach(app.monitors, function (artifact) {
                                _this.artifactNodeType(artifact.artifactTypeName, artifact.monitorStatus, appArtifactKey, data[i].id, data[i].alarmName, appName);
                            });
                        });
                    }
                } else {
                    var partnerDetail = _.findWhere(_this.AlarmDetailsForApplicationArtifact, {
                        AlarmName: data[i].alarmName
                    });
                    _this.artifactParentNode(
                        partnerDetail.PartnerName,
                        data[i].overallStatus
                    );
                    var appParentKey = _this.graphKey;
                    ko.utils.arrayForEach(data[i].mappedApplications, function (app) {
                        if (app.monitorStatus == "Healthy") {
                            if (_this.includeArtifactHealthyStatus()) {
                                _this.artifactApplicationNode(app.applicationName, app.monitorStatus, appParentKey, data[i].id, data[i].alarmName);
                                var appArtifactKey = _this.graphKey;
                                var appName = app.applicationName;
                                ko.utils.arrayForEach(app.monitors, function (artifact) {
                                    if (artifact.monitorStatus == "Healthy") {
                                        if (_this.includeArtifactHealthyStatus()) {
                                            _this.artifactNodeType(artifact.artifactTypeName, artifact.monitorStatus, appArtifactKey, data[i].id, data[i].alarmName, appName);
                                            if (artifact.issues.length > 0) {
                                                _this.artifactChildNode(
                                                    artifact.issues,
                                                    artifact.monitorStatus
                                                );
                                            }
                                        }
                                    } else {
                                        _this.artifactNodeType(artifact.artifactTypeName, artifact.monitorStatus, appArtifactKey, data[i].id, data[i].alarmName, appName);
                                        if (artifact.issues.length > 0) {
                                            _this.artifactChildNode(
                                                artifact.issues,
                                                artifact.monitorStatus
                                            );
                                        }
                                    }
                                });
                            }
                        }
                        else {
                            _this.artifactApplicationNode(app.applicationName, app.monitorStatus, appParentKey, data[i].id, data[i].alarmName);
                            var appArtifactKey = _this.graphKey;
                            var appName = app.applicationName;
                            ko.utils.arrayForEach(app.monitors, function (artifact) {
                                if (artifact.monitorStatus == "Healthy") {
                                    if (_this.includeArtifactHealthyStatus()) {
                                        _this.artifactNodeType(artifact.artifactTypeName, artifact.monitorStatus, appArtifactKey, data[i].id, data[i].alarmName, appName);
                                        if (artifact.issues.length > 0) {
                                            _this.artifactChildNode(
                                                artifact.issues,
                                                artifact.monitorStatus
                                            );
                                        }
                                    }
                                } else {
                                    _this.artifactNodeType(artifact.artifactTypeName, artifact.monitorStatus, appArtifactKey, data[i].id, data[i].alarmName, appName);
                                    if (artifact.issues.length > 0) {
                                        _this.artifactChildNode(
                                            artifact.issues,
                                            artifact.monitorStatus
                                        );
                                    }
                                }
                            });
                        }
                    });
                }
            }
        }
    };
    artifactParentNode = function (value, status) {
        var _this = this;
        var node = {
            key: ++_this.graphKey,
            name: value,
            url: "manage",
            overallStatus: status,
            parent: 1
        };
        _this.artifactNodeArray.push(node);
    };
    artifactApplicationNode = function (value, status, parentKey, alarmId, alarmName) {
        var _this = this;
        var node = {
            key: ++_this.graphKey,
            name: value,
            alarmId: alarmId,
            alarmName: alarmName,
            url: "alrmapplications",
            overallStatus: status,
            appName: value,
            parent: parentKey
        }
        _this.artifactNodeArray.push(node);
    }
    artifactNodeType = function (value, status, childKey, alarmId, alarmName, appName) {
        var _this = this;
        var node = {
            key: ++_this.graphKey,
            name: value,
            alarmId: alarmId,
            alarmName: alarmName,
            appName: appName,
            url: "alrmapplications",
            module: value,
            overallStatus: status,
            parent: childKey
        }
        _this.artifactNodeArray.push(node);
    };
    artifactChildNode = function (value, status) {
        var _this = this;
        var node = {
            key: ++_this.graphKey,
            items: value,
            overallStatus: status,
            parent: _this.graphKey - 1
        };
        _this.artifactNodeArray.push(node);
    };
    displayArtifactGraph = function () {
        var _this = this;
        var data = {
            class: "go.TreeModel",
            nodeDataArray: _this.artifactNodeArray
        };
        _this.artifactMyDiagram.div = null;
        _this.artifactGraphInit();
        _this.artifactMyDiagram.model = go.Model.fromJson(data);
        _this.artifactMyDiagram.findTreeRoots().each(function (n) { n.collapseTree(2); });

    };
    selectAllAlarmsForArtifact = function () {
        selectedAlarmForArtifact.removeAll();
        ko.utils.arrayForEach(AlarmDetailsForApplicationArtifact, function (item) {
            selectedAlarmForArtifact.push(item.AlarmName);
        });
    };
    deSelectAllAlarmsForArtifact = function () {
        selectedAlarmForArtifact.removeAll();
    };
    filterArtifact = function () {
        var selectedAlarmData = [];
        if (selectedAlarmForArtifact().length != 0) {
            ko.utils.arrayForEach(selectedAlarmForArtifact(), function (item) {
                var item = _.findWhere(applicationMonitorStatusDetails(), {
                    alarmName: item
                });
                if (item != undefined) {
                    selectedAlarmData.push(item);
                }
            });
            artifactGraphData(selectedAlarmData);
            artifactMyDiagram.div = null;
            displayArtifactGraph();
        } else {
            artifactGraphData(applicationMonitorStatusDetails());
            artifactMyDiagram.div = null;
            displayArtifactGraph();
        }

    };
    redirectToApplicationPath = function (node) {
        var config = require("biztalk360/framework/core/config");
        var router = require("plugins/router");
        var url = node.url;
        var appName = "";
        var link;
        if (node.appName != undefined) {
            appName = node.appName;
        }
        var module = "";
        if (node.module != undefined) {
            switch (node.module) {
                case "Orchestrations":
                    module = "Orchestrations";
                    break;
                case "ReceiveLocations":
                    module = "ReceiveLocations"
                    break;
                case "SendPorts":
                    module = "SendPorts"
                    break;
                case "Service Instances":
                    module = "ServiceInstances"
                    break;
                case "SendPortGroups":
                    module = "SendPortGroups"
                    break;
            }
        }
        if (appName != "" || module != "") {
            link = '#/' + url + '/' + appName + '/' + module;
        } else {
            link = '#/' + url;
        }
        if (node.alarmId != undefined && node.alarmId != "") {
            alarm = {
                alarmId: ko.observable(node.alarmId),
                name: node.alarmName
            };
            config.currentAlarm(alarm);
        }
        router.navigate(link, true);
    };
    GetArtifactMonitorStatus();
    artifactGraphInit();
    setInterval(GetArtifactMonitorStatus, applicationArtifactRefresh * 4500);    
</script>
```