# KnockoutJS - Data Binding
Knockoutjs v3.3.0 is used for the data binding. The below code snippet is used to bind data for web end point status
Following custom widget uses knockout to bind the data to the table.

![WebEndpointWidget](https://github.com/biztalk360/Custom-Widgets/blob/master/Kovai.BizTalk360.SampleCustomWidgets/Images/WebEndpointWidget.png)

 ```
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
```
