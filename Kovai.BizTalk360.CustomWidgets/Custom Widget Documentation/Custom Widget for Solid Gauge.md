# Custom Widget for Azure Service Bus Queue Status

The following custom widget shows how to use any kind of data in solid guage in BizTalk360. You can use below template for graphical representation <br />

![SampleCustomTemplateWidget](https://github.com/biztalk360/Custom-Widgets/blob/master/Kovai.BizTalk360.CustomWidgets/Images/SolidGauge.png)

```
<head>
<style>
.outer {
	width: 600px;
	height: 200px;
	margin: 0 auto;
}
.container {
	width: 300px;
	float: left;
	height: 200px;
}
.highcharts-yaxis-grid .highcharts-grid-line {
	display: none;
}

@media (max-width: 600px) {
	.outer {
		width: 100%;
		height: 400px;
	}
	.container {
		width: 300px;
		float: none;
		margin: 0 auto;
	}
}
</style>	
	
</head>
<body>
<div class="outer">
    <div id="container-speed" class="container"></div>
    <div id="container-rpm" class="container"></div>
</div>
	<script>	 

function injectScript(src) {
    return new Promise((resolve, reject) => {       
		for (var i = 0; i < src.length; i++) {
    var script = document.createElement('script');
    script.src = src[i];
    script.async = false; // This is required for synchronous execution
    document.head.appendChild(script);
  }        
        script.addEventListener('load', resolve);
        script.addEventListener('error', () => reject('Error loading script.'));
        script.addEventListener('abort', () => reject('Script loading aborted.'));
        document.head.appendChild(script);
    });
}

injectScript(['https://cdnjs.cloudflare.com/ajax/libs/highcharts/5.0.14/highcharts-more.js','https://cdnjs.cloudflare.com/ajax/libs/highcharts/5.0.14/js/modules/solid-gauge.js'])
    .then(() => {
        console.log('Script loaded!');
		   console.log('1. Antes de var gaugeOptionsHours...');
	
    var gaugeOptions = {

    chart: {
        type: 'solidgauge'
    },

    title: null,

    pane: {
        center: ['50%', '85%'],
        size: '140%',
        startAngle: -90,
        endAngle: 90,
        background: {
            backgroundColor: (Highcharts.theme && Highcharts.theme.background2) || '#EEE',
            innerRadius: '60%',
            outerRadius: '100%',
            shape: 'arc'
        }
    },

    tooltip: {
        enabled: false
    },

    // the value axis
    yAxis: {
        stops: [
            [0.1, '#55BF3B'], // green
            [0.5, '#DDDF0D'], // yellow
            [0.9, '#DF5353'] // red
        ],
        lineWidth: 0,
        minorTickInterval: null,
        tickAmount: 2,
        title: {
            y: -70
        },
        labels: {
            y: 16
        }
    },

    plotOptions: {
        solidgauge: {
            dataLabels: {
                y: 5,
                borderWidth: 0,
                useHTML: true
            }
        }
    }
};

// The speed gauge
var chartSpeed = Highcharts.chart('container-speed', Highcharts.merge(gaugeOptions, {
    yAxis: {
        min: 0,
        max: 200,
        title: {
            text: 'Speed'
        }
    },

    credits: {
        enabled: false
    },

    series: [{
        name: 'Speed',
        data: [80],
        dataLabels: {
            format: '<div style="text-align:center"><span style="font-size:25px;color:' +
                ((Highcharts.theme && Highcharts.theme.contrastTextColor) || 'black') + '">{y}</span><br/>' +
                   '<span style="font-size:12px;color:silver">km/h</span></div>'
        },
        tooltip: {
            valueSuffix: ' km/h'
        }
    }]

}));

// The RPM gauge
var chartRpm = Highcharts.chart('container-rpm', Highcharts.merge(gaugeOptions, {
    yAxis: {
        min: 0,
        max: 5,
        title: {
            text: 'RPM'
        }
    },

    series: [{
        name: 'RPM',
        data: [1],
        dataLabels: {
            format: '<div style="text-align:center"><span style="font-size:25px;color:' +
                ((Highcharts.theme && Highcharts.theme.contrastTextColor) || 'black') + '">{y:.1f}</span><br/>' +
                   '<span style="font-size:12px;color:silver">* 1000 / min</span></div>'
        },
        tooltip: {
            valueSuffix: ' revolutions/min'
        }
    }]

}));

// Bring life to the dials
    // Speed
    var point,
        newVal,
        inc;

    if (chartSpeed) {
        point = chartSpeed.series[0].points[0];
        inc = Math.round((Math.random() - 0.5) * 100);
        newVal = point.y + inc;

        if (newVal < 0 || newVal > 200) {
            newVal = point.y - inc;
        }

        point.update(newVal);
    }

    // RPM
    if (chartRpm) {
        point = chartRpm.series[0].points[0];
        inc = Math.random() - 0.5;
        newVal = point.y + inc;

        if (newVal < 0 || newVal > 5) {
            newVal = point.y - inc;
        }

        point.update(newVal);
    }
}, 2000);
	</script>
</body>
```