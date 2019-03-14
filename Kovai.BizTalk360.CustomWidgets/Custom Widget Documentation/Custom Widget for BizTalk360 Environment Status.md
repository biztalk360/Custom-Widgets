# Custom Widget for BizTalk Environment Status

The following custom widget shows BizTalk Environment status of Host instance,Host Throttling,WebEndpoint, Database Query and BizTalk Health Monitor Status, which are mapped with BizTalk360 Alarms. We firstly need to create the Secure SQL Query, we would like to have as a widget, in the Secure SQL Queries feature. You can find this feature under Operations and then Data Access. To create custom widget using Secure SQL Query refer [this](https://docs.biztalk360.com/docs/creating-a-custom-widget-for-executing-secure-sql-queries) article.<br /><br />
In the below template secure SQL query is executed using POST method. Commonly payload for API call accepts only JSON data, so here payload object is converted to JSON data and binding with GOJs Graph for graphical representation.

![SampleCustomTemplateWidget](https://github.com/biztalk360/Custom-Widgets/blob/master/Kovai.BizTalk360.CustomWidgets/Images/BizTalkEnvironmentStatus.png)

```
<div id="WidgetScroll" style="top:30px;">
    <div class="query-builder margin-t">
        <a data-toggle="collapse" data-bind="attr: { href: '#graph-BizTalkEnv' }" class="collapse-trigger collapsed">Filters</a>
        <div data-bind="attr: { id: 'graph-BizTalkEnv' }" class="collapse">             
            <div class="row" style="padding-top: 12px;">
                <div class="col-md-9">
                    <div class="form-group row">
                        <label class="col-md-2" style="font-size: 15px;" for="alarm">Alarm</label>
                        <div class="col-md-8">
                            <select
                                data-bind="kendoMultiSelect: { data:AlarmDetailsForBizTalkEnv,dataTextField:'AlarmName',dataValueField:'AlarmName',value:selectedAlarmForBizTalkEnv}"
                                multiple="multiple"></select>
                        </div>
                        <div>
                            <button class="btn" id="select" data-bind="click:selectAllAlarmsForBizTalkEnv">
                                <a data-bind="bsToolTip: { placement: 'top' }" data-container="body"
                                    data-toggle="tooltip" data-original-title="Select All">
                                    <i class="fa fa-check-square-o"></i>
                                </a>
                            </button>
                            <button class="btn" id="deselect" data-bind="click:deSelectAllAlarmsForBizTalkEnv">
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
                                    data-bind="kendoMobileSwitch: { checked: includeBizTalkEnvHealthyStatus, onLabel: 'YES', offLabel: 'NO' }" />
                                <span class="caption">Include Healthy BizTalk Environments</span>
                            </label>
                        </div>
                    </div>
                </div>
                <div class="col-md-3">
                    <button class="btn btn-primary btn-medium" data-bind="click:filterBizTalkEnv" id="select"><i
                            class="fa fa-sliders" aria-hidden="true"></i> Search
                    </button>
                </div>
            </div>
        </div>
    </div>
    <div id="bizTalkEnvironmentDiagram" style="border: solid 1px white; height: 600px"></div>
</div>
<script>
    // BEGIN User variables
    biztalkEnvRefresh = 10;
    username = "BizTalk360"; // BizTalk360 service account
    password = "Rajsree9656624199"; // Password of BizTalk360 service account
    environmentId = "a3d01fb3-0f1b-46e5-9ec0-2dbb9a3f2513"; // BizTalk360 Environment ID  (take from SSMS or API Documentation)
    BizTalkEnvQueryId = "05888511-a8ad-4f5b-a803-cde6db753f79"; // Id of the Secure SQL Query (take from SSMS)
    BizTalkEnvQueryName = "BizTalk Environment Status"; // Name of the Secure SQL Query as it is stored under Operations/Secure SQL Query
    sqlInstance = "BT360DEV22"; // SQL Instance against which the SQL Query must be executed
    database = "BizTalk360"; // Database against which the SQL Query must be executed
    BizTalkEnvStatusQuery = "Select AA.Id, AA.[Name],AME.MonitorStatus,AME.ExecutionResult from [dbo].[b360_alert_MonitorExecution] AME Inner Join[dbo].[b360_alert_Alarm] AA ON AME.AlarmId = AA.Id and AME.MonitorGroupType = 'BizTalkEnvironment' WHERE AME.LastExecutionTime = (SELECT MAX(LastExecutionTime) from [dbo].[b360_alert_MonitorExecution] WHERE MonitorGroupType = 'BizTalkEnvironment' and AA.IsAlertDisabled='false' and AlarmId = AA.Id and LastExecutionTime >= DATEADD(MINUTE,-60,GETUTCDATE())) AND AME.EnvironmentId='a3d01fb3-0f1b-46e5-9ec0-2dbb9a3f2513'"// The SQL Query which needs to be executed from the custom Widget
    bt360server = "localhost"; // Name of the BizTalk360 server (needed to do an API call to execute the SQL query
    
    //Mention the created alarm details and the respective partner name to display in the graph.
    AlarmDetailsForBizTalkEnv = [
        { AlarmName: "Threshold Alarm-1", PartnerName: "Air India" },
        { AlarmName: "Threshold Alarm-2", PartnerName: "Qatar Airways" },
        { AlarmName: "Threshold Alarm-3", PartnerName: "Jet Airlines" },
        { AlarmName: "Threshold Alarm-4", PartnerName: "" }
    ];
    // END User variables

    executeBizTalkEnvStatusUrl =
        "http://" +
        bt360server +
        "/BizTalk360/Services.REST/BizTalkGroupService.svc/ExecuteCustomSQLQuery";
    BizTalkEnvMonitorStatusDetails = ko.observableArray();
    artifactNodeArray = [
        {
            key: 1,
            name: "BizTalk Environments",
            overallStatus: "RootNode",
            url: 'biztalkenvironment',
            module: ''
        }
    ];
    selectedAlarmForBizTalkEnv = ko.observableArray();
    x2js = new X2JS({
        attributePrefix: "",
        arrayAccessForm: "property",
        arrayAccessFormPaths: ["root.records.record"]
    });
    graphKey = 1;
    includeBizTalkEnvHealthyStatus = ko.observable(false);
    GetBizTalkEnvMonitorStatus = function () {
        var _thisEnv = this;
        _thisEnv.BizTalkEnvMonitorStatusDetails.removeAll();
        _thisEnv.executeBizTalkEnvCustomSqlQuery(function (data) {
            var bizTalkEnvResults = x2js.xml_str2json(data.queryResult);

            if (bizTalkEnvResults.root.records.record != null) {
                if (Array.isArray(bizTalkEnvResults.root.records.record)) {
                    ko.utils.arrayForEach(
                        bizTalkEnvResults.root.records.record,
                        function (item) {
                            _thisEnv.mappedBizTalkEnvData(item);
                        }
                    );
                } else {
                    _thisEnv.mappedBizTalkEnvData(bizTalkEnvResults.root.records.record);
                }
                _thisEnv.bizTalkEnvGraphData(_thisEnv.BizTalkEnvMonitorStatusDetails());
                _thisEnv.displayBizTalkEnvGraph();

            }
        });
    };
    mappedBizTalkEnvData = function (data) {
        var _thisEnv = this;
        var mappedBizTalkEnvs = {
            "id": "",
            "alarmName": "",
            "overallStatus": "",
            "mappedBizTalkEnv": []
        };
        var result = x2js.xml_str2json(data.ExecutionResult);
        mappedBizTalkEnvs.id = data.Id;
        mappedBizTalkEnvs.alarmName = data.Name;
        mappedBizTalkEnvs.overallStatus = data.MonitorStatus;
        if (Array.isArray(result.MonitorGroupData.monitorGroups.MonitorGroup)) {
            ko.utils.arrayForEach(result.MonitorGroupData.monitorGroups.MonitorGroup, function (app) {
                var monitors = [];
                ko.utils.arrayForEach(app.monitors.Monitor, function (monitor) {
                    if (monitor.monitorStatus != "NotConfigured") {
                        var mappedEnvName = {
                            "envName": "",
                            "monitorStatus": "",
                            "issues": []
                        };
                        mappedEnvName.envName = monitor.name;
                        mappedEnvName.monitorStatus = monitor.monitorStatus;
                        if (Array.isArray(monitor.issues.Issue)) {
                            ko.utils.arrayForEach(monitor.issues.Issue, function (issue) {
                                mappedEnvName.issues.push(issue.description);
                            });
                        } else {
                            if (monitor.issues.Issue != undefined) {
                                mappedEnvName.issues = new Array(monitor.issues.Issue.description);
                            }
                        }
                        monitors.push(mappedEnvName);
                    }
                });
                mappedBizTalkEnvs.mappedBizTalkEnv = monitors;
            });
            _thisEnv.BizTalkEnvMonitorStatusDetails.push(mappedBizTalkEnvs);
        }
        else {
            var monitors = [];
            ko.utils.arrayForEach(result.MonitorGroupData.monitorGroups.MonitorGroup.monitors.Monitor, function (monitor) {
                if (monitor.monitorStatus != "NotConfigured") {
                    var mappedEnvName = {
                        "envName": "",
                        "monitorStatus": "",
                        "issues": []
                    };
                    mappedEnvName.envName = monitor.name;
                    mappedEnvName.monitorStatus = monitor.monitorStatus;
                    if (Array.isArray(monitor.issues.Issue)) {
                        ko.utils.arrayForEach(monitor.issues.Issue, function (issue) {
                            mappedEnvName.issues.push(issue.description);
                        });
                    } else {
                        if (monitor.issues.Issue != undefined) {
                            mappedEnvName.issues = new Array(monitor.issues.Issue.description);
                        }
                    }
                    monitors.push(mappedEnvName);
                }
            });
            mappedBizTalkEnvs.mappedBizTalkEnv = monitors;
            _thisEnv.BizTalkEnvMonitorStatusDetails.push(mappedBizTalkEnvs);
        }
    };
    executeBizTalkEnvCustomSqlQuery = function (callback) {
        var _thisEnv = this;
        $.ajax({
            dataType: "json",
            url: _thisEnv.executeBizTalkEnvStatusUrl,
            type: "POST",
            contentType: "application/json",
            username: _thisEnv.username,
            password: _thisEnv.password,
            data:
                '{"context":{"environmentSettings":{"id":"' +
                _thisEnv.environmentId +
                '","licenseEdition":0},"callerReference":"REST-SAMPLE"},"query":{"id":"' +
                _thisEnv.BizTalkEnvQueryId +
                '","name":"' +
                _thisEnv.BizTalkEnvQueryName +
                '","sqlInstance":"' +
                _thisEnv.sqlInstance +
                '","database":"' +
                _thisEnv.database +
                '","sqlQuery":"' +
                _thisEnv.BizTalkEnvStatusQuery +
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
    bizTalkEnvGraphInit = function () {
        var _thisEnv = this;
        //if (window.goSamples) goSamples(); // init for these samples -- you don't need to call this
        var $ = go.GraphObject.make; // for conciseness in defining templates
        _thisEnv.bizTalkEnvMyDiagram = $(go.Diagram, "bizTalkEnvironmentDiagram", {
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
        _thisEnv.bizTalkEnvMyDiagram.addDiagramListener("InitialLayoutCompleted", function (e) {
        });
        _thisEnv.bizTalkEnvMyDiagram.nodeTemplate = $(go.Node, "Vertical", new go.Binding("text", "name"),
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
                                _thisEnv.redirectToBizTalkEnvPath(obj.part.data);
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
        _thisEnv.bizTalkEnvMyDiagram.linkTemplate = $(go.Link, {
            routing: go.Link.Orthogonal,  // may be either Orthogonal or AvoidsNodes
            corner: 5,
            relinkableFrom: false,
            relinkableTo: false
        }, $(go.Shape, {
            strokeWidth: 3,
            stroke: "#00a4a4"
        }));
    };
    bizTalkEnvGraphData = function (biztalkEnvDetails) {
        var _thisEnv = this;
        _thisEnv.artifactNodeArray = [{
            key: 1,
            name: "BizTalk Environments",
            overallStatus: "RootNode",
            url: 'biztalkenvironment',
            module: ''
        }];
        var data = [];
        if (_thisEnv.selectedAlarmForBizTalkEnv().length != 0) {
            ko.utils.arrayForEach(_thisEnv.selectedAlarmForBizTalkEnv(), function (item) {
                var alrm = _.findWhere(biztalkEnvDetails, {
                    alarmName: item
                });
                if (alrm != undefined) {
                    data.push(alrm);
                }
            });
        } else {
            data = biztalkEnvDetails;
        }
        for (var i = 0; i < data.length; i++) {
            var node;
            if (data[i] != undefined) {
                if (data[i].overallStatus == "Healthy") {
                    if (_thisEnv.includeBizTalkEnvHealthyStatus()) {
                        var partnerDetail = _.findWhere(_thisEnv.AlarmDetailsForBizTalkEnv, {
                            AlarmName: data[i].alarmName
                        });
                        _thisEnv.bizTalkEnvParentNode(
                            partnerDetail.PartnerName,
                            data[i].overallStatus,
                            data[i].id,
                            data[i].alarmName
                        );
                        var envParentKey = _thisEnv.graphKey;
                        ko.utils.arrayForEach(data[i].mappedBizTalkEnv, function (env) {
                            _thisEnv.bizTalkEnvNode(env.envName, env.monitorStatus, envParentKey, data[i].id, data[i].alarmName);
                        });
                    }
                } else {
                    var partnerDetail = _.findWhere(_thisEnv.AlarmDetailsForBizTalkEnv, {
                        AlarmName: data[i].alarmName
                    });
                    _thisEnv.bizTalkEnvParentNode(
                        partnerDetail.PartnerName,
                        data[i].overallStatus,
                        data[i].id,
                        data[i].alarmName
                    );
                    var envParentKey = _thisEnv.graphKey;
                    ko.utils.arrayForEach(data[i].mappedBizTalkEnv, function (env) {
                        if (env.monitorStatus == "Healthy") {
                            if (_thisEnv.includeBizTalkEnvHealthyStatus()) {
                                _thisEnv.bizTalkEnvNode(env.envName, env.monitorStatus, envParentKey, data[i].id, data[i].alarmName);
                            }
                        } else {
                            _thisEnv.bizTalkEnvNode(env.envName, env.monitorStatus, envParentKey, data[i].id, data[i].alarmName);
                            if (env.issues.length > 0) {
                                _thisEnv.bizTalkEnvChildNode(
                                    env.issues,
                                    env.monitorStatus
                                );
                            }
                        }
                    });

                }
            }

        }
    };
    bizTalkEnvParentNode = function (value, status, alarmId, alarmName) {
        var _thisEnv = this;
        var node = {
            key: ++_thisEnv.graphKey,
            name: value,
            alarmName: alarmName,
            overallStatus: status,
            url: 'biztalkenvironment',
            alarmId: alarmId,
            parent: 1
        };
        _thisEnv.artifactNodeArray.push(node);
    };
    bizTalkEnvNode = function (value, status, parentKey, alarmId, alarmName) {
        var _thisEnv = this;
        var node = {
            key: ++_thisEnv.graphKey,
            name: value,
            alarmName: alarmName,
            overallStatus: status,
            url: 'biztalkenvironment',
            module: value,
            alarmId: alarmId,
            parent: parentKey
        }
        _thisEnv.artifactNodeArray.push(node);
    }

    bizTalkEnvChildNode = function (value, status) {
        var _thisEnv = this;
        var node = {
            key: ++_thisEnv.graphKey,
            items: value,
            overallStatus: status,
            parent: _thisEnv.graphKey - 1
        };
        _thisEnv.artifactNodeArray.push(node);
    };
    displayBizTalkEnvGraph = function () {
        var _thisEnv = this;
        var data = {
            class: "go.TreeModel",
            nodeDataArray: _thisEnv.artifactNodeArray
        };
        _thisEnv.bizTalkEnvMyDiagram.div = null;
        _thisEnv.bizTalkEnvGraphInit();
        _thisEnv.bizTalkEnvMyDiagram.model = go.Model.fromJson(data);
        _thisEnv.bizTalkEnvMyDiagram.findTreeRoots().each(function (n) { n.collapseTree(2); });

    };
    selectAllAlarmsForBizTalkEnv = function () {
        selectedAlarmForBizTalkEnv.removeAll();
        ko.utils.arrayForEach(AlarmDetailsForBizTalkEnv, function (item) {
            selectedAlarmForBizTalkEnv.push(item.AlarmName);
        });
    };
    deSelectAllAlarmsForBizTalkEnv = function () {
        selectedAlarmForBizTalkEnv.removeAll();
    };
    filterBizTalkEnv = function () {
        var selectedAlarmData = [];
        if (selectedAlarmForBizTalkEnv().length != 0) {
            ko.utils.arrayForEach(selectedAlarmForBizTalkEnv(), function (item) {
                var alrm = _.findWhere(BizTalkEnvMonitorStatusDetails(), {
                    alarmName: item
                });
                if (alrm != undefined) {
                    selectedAlarmData.push(alrm);
                }
            });
            if (selectedAlarmData.length > 0) {
                bizTalkEnvGraphData(selectedAlarmData);
                bizTalkEnvMyDiagram.div = null;
                displayBizTalkEnvGraph();
            }
        } else {
            bizTalkEnvGraphData(BizTalkEnvMonitorStatusDetails());
            bizTalkEnvMyDiagram.div = null;
            displayBizTalkEnvGraph();
        }
    };
    redirectToBizTalkEnvPath = function (node) {
        var config = require("biztalk360/framework/core/config");
        var router = require("plugins/router");
        var url = node.url;
        var module = "";
        if (node.module != undefined) {
            switch (node.module) {
                case "BizTalkHealthMonitor":
                    module = "BiztalkHealthMonitor";
                    break;
                case "HostThrottling":
                    module = "HostThrottling"
                    break;
                case "HostInstances":
                    module = "HostInstances"
                    break;
                case "WebEndpoints":
                    module = "WebEndpoints"
                    break;
                case "Database Query":
                    module = "DatabaseQuery"
                    break;
            }
        }
        var link = '#/' + url + '/' + module;
        if (node.alarmId != undefined && node.alarmId != '') {
            alarm = {
                alarmId: ko.observable(node.alarmId),
                name: node.alarmName
            };
            config.currentAlarm(alarm);
        }
        router.navigate(link, true);
    };
    GetBizTalkEnvMonitorStatus();
    bizTalkEnvGraphInit();
    setInterval(GetBizTalkEnvMonitorStatus, biztalkEnvRefresh * 2000);    
</script>
```