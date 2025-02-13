<template>
    <VContainer align="center" fluid color="surface">
        <VRow max-width="1200px">
            <!-- Row 1-->
            <VCol cols="9">
                <!-- Row 1 column 1-->
                <figure class="highcharts-fugure">
                    <div id="container"></div>
                </figure>
            </VCol>


            <VCol cols="3">
                <!-- Row 1 column 2-->
                <v-card class="mb-5" max-width="500" color="primaryContainer" subtitle="Temperature">
                    <!-- Card 1-->
                    <VCardItem>
                        <span class="text-h3 text-onPrimaryContainer">{{ temperature }}</span>
                    </VCardItem>
        
                </v-card>
                <v-card class="mb-5" max-width="500" color="tertiaryContainer" subtitle="Heat Index (Feels like)">
                    <!-- Card 2-->
                    <VCardItem>
                        <span class="text-h3 text-onTertiaryContainer">{{ heatindex }}</span>
                    </VCardItem>
                </v-card>
                <v-card class="mb-5" max-width="500" color="secondaryContainer" subtitle="Humidity">
                    <!-- Card 3-->
                    <VCardItem>
                        <span class="text-h3 text-onSecondaryContainer">{{ humidity }}</span>
                    </VCardItem>
                </v-card>
                
            </VCol>
        </VRow>
        <VRow max-width="1200px" justify="start">
            <!-- Row 2-->
            <VCol cols="9">
                <!-- Row 2 column 1-->
                <figure class="highcharts-fugure">
                    <div id="container1"></div>
                </figure>
            </VCol>
        </VRow>
    </VContainer>
</template>

<script setup>
/** JAVASCRIPT HERE */

// IMPORTS
import { ref,reactive,watch ,onMounted,onBeforeUnmount,computed } from "vue";  
import { useRoute ,useRouter } from "vue-router";


import { useMqttStore } from '@/store/mqttStore'; // Import Mqtt Store 
import { storeToRefs } from "pinia"; 

import { useAppStore } from "@/store/appStore"; 



// Highcharts, Load the exporting module and Initialize exporting module. 
import Highcharts from 'highcharts'; 
import more from 'highcharts/highcharts-more'; 
import Exporting from 'highcharts/modules/exporting'; 
Exporting(Highcharts);  
more(Highcharts);




// VARIABLES 
const Mqtt     = useMqttStore(); 
const { payload, payloadTopic } = storeToRefs(Mqtt);
 
// VARIABLES
const router      = useRouter();
const route       = useRoute();  

const AppStore    = useAppStore(); 
const tempHiChart     = ref(null); // Chart object 
const humidHiChart = ref(null);

const points  = ref(10); // Specify the quantity of points to be shown on the live graph simultaneously. 
const shift = ref(false); // Delete a point from the left side and append a new point to the right side of the graph.

// COMPUTED PROPERTIES 
const temperature = computed(()=>{ 
    if(!!payload.value){ 
        return `${payload.value.temperature.toFixed(2)} °C`; 
    } 
});

const humidity = computed(()=>{ 
    if(!!payload.value){ 
        return `${payload.value.humidity.toFixed(2)}%`; 
    } 
});

const heatindex = computed(()=>{ 
    if(!!payload.value){ 
        return `${payload.value.heatindex.toFixed(2)} °C`; 
    } 
});

// FUNCTIONS
onMounted(()=>{
    // THIS FUNCTION IS CALLED AFTER THIS COMPONENT HAS BEEN MOUNTED
     // THIS FUNCTION IS CALLED AFTER THIS COMPONENT HAS BEEN MOUNTED
     CreateCharts(); 
    
    Mqtt.connect(); // Connect to Broker located on the backend 
    
    setTimeout( ()=>{ 
        // Subscribe to each topic 
        Mqtt.subscribe("620165845");
        //Mqtt.subscribe("620165845_pub"); 
    },3000);
});


onBeforeUnmount(()=>{
    // THIS FUNCTION IS CALLED RIGHT BEFORE THIS COMPONENT IS UNMOUNTED
    Mqtt.unsubcribeAll();
});


const CreateCharts = async () => { 
    // TEMPERATURE CHART 
    
    tempHiChart.value = Highcharts.chart('container', { 
        chart: { zoomType: 'x', backgroundColor: '#1e1e1e'   }, 
        title: { text: 'Temperature Analysis (Live)', align: 'left', style: { color: '#FFFFFF' } }, 
       
        yAxis: { 
            title: { text: '°C' , style:{color:'#FFFFFF'}}, 
            labels: { format: '{value} °C', style: { color: '#FFFFFF' } } ,
            gridLineColor: '#FFFFFF'
        }, 
        
        xAxis: { 
            type: 'datetime', 
            title: { text: 'Time', style:{color:'#FFFFFF'}},
            labels: { style: { color: '#FFFFFF' } },
            gridLineColor: '#FFFFFF'
         
        }, 
        
        tooltip: { shared:true, }, 
        legend: {
            itemStyle: { color: '#FFFFFF' } 
        },
        
        series: [ 
            { 
                name: 'Temperature', 
                type: 'spline', 
                data: [], 
                turboThreshold: 0, 
                color: Highcharts.getOptions().colors[0] 
            }, 
            
            { 
                name: 'Heat Index', 
                type: 'spline', 
                data: [], 
                turboThreshold: 0, 
                color: Highcharts.getOptions().colors[1] 
            } 
        ], 
    }); 

    // HUMIDITY CHART 
    
    humidHiChart.value = Highcharts.chart('container1', { 
        chart: { zoomType: 'x', backgroundColor: '#1e1e1e' }, 
        title: { text: 'Humidity Analysis (Live)', align: 'left', style: { color: '#FFFFFF' } }, 
        
        yAxis: { 
            title: { text: '%' , style:{color:'#FFFFFF'}}, 
            labels: { format: '{value} %', style:{color:'#FFFFFF'} },
            gridLineColor: '#FFFFFF' 
        }, 
        
        xAxis: { 
            type: 'datetime', 
            title: { text: 'Time', style:{color:'#FFFFFF'} }, 
            labels: { style: { color: '#FFFFFF' } },
             gridLineColor: '#FFFFFF'
        }, 
        
        tooltip: { shared:true, }, 
        legend: {itemStyle: { color: '#FFFFFF' } },
        
        series: [ 
            { 
                name: 'Humidity', 
                type: 'spline', 
                data: [], 
                turboThreshold: 0, 
                color: Highcharts.getOptions().colors[0] 
            }, 
            
           
        ], 
    }); 
};


// WATCHERS 
watch(payload,(data)=> { 
    if(points.value > 0){ points.value -- ; } 
    else{ shift.value = true; } 
    
    tempHiChart.value.series[0].addPoint({y:parseFloat(data.temperature.toFixed(2)) ,x: data.timestamp * 1000 }, true, shift.value); 
    tempHiChart.value.series[1].addPoint({y:parseFloat(data.heatindex.toFixed(2)) ,x: data.timestamp * 1000 }, true, shift.value); 
    
    humidHiChart.value.series[0].addPoint({y:parseFloat(data.humidity.toFixed(2)), x: data.timestamp * 1000}, true, shift.value);
})



</script>


<style scoped>
/** CSS STYLE HERE */
Figure { 
border: 2px solid black; 
}

</style>
  