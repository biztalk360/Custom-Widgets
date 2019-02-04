# Styles
The styles of BizTalk360 can be used in custom widgets by inspecting the element in the BizTalk360 Application. Take advantage of the listed styles and controls while creating the custom widgets.
*	Boot Strap
*	Font Awesome Icons
*	KendoUI Controls
*	KnockoutJS
# BizTalk360 Styles
HTML v5 and above version is used to design custom widget templates in BizTalk360. 
WidgetScroll and table table-lists are styles of BizTalk360.

<div id=”WidgetScroll” style=”top:30px;” data-bind=”addScrollBar: WidgetScroll, scrollCallback: ‘false’”>
<table class=”table table-lists”>
</tbody>
</table>
</div>

**Example:** <br />
To do list widget contains list of To do details designed with table and widget scroll styles.
 
![TodolistCustomWidget](https://github.com/biztalk360/Custom-Widgets/blob/master/Kovai.BizTalk360.CustomWidgets/Images/TodolistCustomWidget.png)

# Custom Styles
There is possibility to write your own CSS styles like follows,
```
<style>
.legend-item {
font-family: “Lucida Grande”, “Lucida sans Unicode”, Arial, Helvetica, sans-serif;
font-size: 12px;
position: absolute;
white-space: nowrap;
color: rgb(51, 51, 51);
font-weight: bold;
text-overflow: ellipsis;
cursor: pointer;
 overflow: hidden;
margin-left: -39px;
margin-top: 0px;
left: 21px;
fill: rgb(51, 51, 51);
}
</style>
```
The below widget uses custom styles for fill high chart color and align the send port’s status and Host instance status.
 
![SendPortWidget](https://github.com/biztalk360/Custom-Widgets/blob/master/Kovai.BizTalk360.SampleCustomWidgets/Images/SendPortWidget.png)

![ArtifactsCountWidget](https://github.com/biztalk360/Custom-Widgets/blob/master/Kovai.BizTalk360.SampleCustomWidgets/Images/ArtifactsCountWidget.png)
 
In the below widget CSS styles are applied to the font awesome icon.
```
<style>
.widget-success-color {
color:#1caf9a;
}
.widget-BTstatus.fa {
display:block;
font-size:130px;
text-align:center;
}
</style>
```
 
 

# Bootstrap style
BizTalk360 uses Bootstrap v3.3.2, which is used to design CSS based design templates like forms, buttons, glyph icons etc.
Following custom widget uses bootstrap to design the form, input fields and button.

![BootstrapStyleWidget](https://github.com/biztalk360/Custom-Widgets/blob/master/Kovai.BizTalk360.SampleCustomWidgets/Images/BootstrapStyleWidget.png)
 
