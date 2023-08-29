<script>
  import {scaleOrdinal, scaleLinear, scaleBand, scaleTime,scaleThreshold,scaleSequential } from 'd3-scale'
  import {timeseries} from './timeseries_data.js'
  import {april2022} from './april2022_data.js'
  import {countries_english} from './countries_english_data.js'
  import { schemePaired,schemeBlues } from 'd3-scale-chromatic'
  import { hoods } from './hoods.js'
  import { Eu } from './Eu.js'
  import { Graphic, PolygonLayer, fitScales,Section,createZoomHandler,createPanHandler, RectangleLayer, PointLayer, Line, XAxis, YAxis } from '@snlab/florence'
  import DataContainer from '@snlab/florence-datacontainer'

  // constants
  let selectedIndex = null
  let description=null
  let selectedCountries = null
  let textXPos = 100
  let textYPos = 200
  let selectedQuarter = '2022-04-01'
  let zoomIdentity = { x: 0, y: 0, kx: 1, ky: 1 }
  let blockReindexing = false;

  // data formatted from YYYY-MM-DD to Month YYYY
  $: quarter = new Date (selectedQuarter)
  const options = {
    month: 'long',
    year: 'numeric', 
    }
  $: formattedDate = quarter.toLocaleDateString('en-US', options);

  // load data
  const neighbourhoods = new DataContainer(Eu)
  console.log(neighbourhoods)
  // entire dataset
  const april2022_data = new DataContainer (april2022)
  let data_selectedQuarter = april2022_data
  const timeseries_data = new DataContainer (timeseries) 


  // data for menu
  const time = timeseries_data.domain('quarter').sort()
  $: data_selectedQuarter = timeseries_data.filter(row => row.quarter === selectedQuarter)
  // console.log('')
  const setZoomIdentity = zoomId => { zoomIdentity = zoomId }
  const setBlockReindexing = bool =>{blockReindexing = bool}
  const zoom = createZoomHandler(zoomIdentity, {
    setZoomIdentity,
    minZoom: 0.2,
    maxZoom: 3
  })

  const pan = createPanHandler(zoomIdentity,{
    setZoomIdentity,
    setBlockReindexing,
    extentX:[-500,500],
    extentY:[-500,500]

  })
  const padding = { left: 80, bottom: 40, top: 10, right: 5 }

  // set up scales

  // 1. position
  const myGeoScale = fitScales(neighbourhoods.domain('$geometry'))

 $: speed_distribution = data_selectedQuarter
    .bin({ 
      column: 'avg_d', 
      method: 'Manual', // method: customBinning,
      manualClasses: [0, 20, 40, 60, 80, 100, 120, 140, 160, 180, 200] , // numClasses: binRanges.length - 1
    })
    .summarise({ total_count: { id: 'count' }})


  // BARS AND COLORS
      // colors of bars
      $: fillRectangles = speed_distribution.map('bins', bin => {
          if(speed_distribution.data().bins[selectedIndex] == bin && selectedIndex != null) return 'rgb(255,100,180)'; //selected bar gets barbie hue (gotta stay trendy)
          const speed = (bin[0] + bin[1]) / 2;
          if (speed > 120) return 'rgb(0,255,0)'; // I set color to green for speeds above 120 Mbps
          // I'm calculating gradient colors from blue to tourquize based on speed value
          const hue = 255 - (speed / 120) * 120; // 0 Mbps: blue (hue = 255), 100 Mbps: tourquize (hue = 120)
          return `hsl(${hue}, 100%, 50%)`;
        });
      
      // i wanna make barbie shine
      $: fillOpacityRectangles = speed_distribution.map('bins', bin => {
          if (speed_distribution.data().bins[selectedIndex] != bin && selectedIndex != null) return 0.3; else {return 1}
          });


  // TEXT DESCRIPTION: i want to show which countries are in which 'bin' in the text below the graph
  $: {
      if(selectedIndex != null){
      selectedCountries = data_selectedQuarter.filter(row => row.avg_d >= speed_distribution.data().bins[selectedIndex][0] && row.avg_d <= speed_distribution.data().bins[selectedIndex][1]);
     // textXPos = selectedIndex * 40 + 100;
     // textYPos = 450 - selectedCountries.column('name').length * 45;
      } else {
        selectedCountries = null;
      }
    }  

  $: classThresholds = binsToThreshold(speed_distribution._data['bins'])
  
  $: myColorScale = scaleThreshold()
    .domain(classThresholds)
    .range(fillRectangles)

  function binsToThreshold (bins) {
    const thresholds = []
    for (let index = 1; index < bins.length; index++) {
      const bin = bins[index]
      thresholds.push(bin[0])
    }
    return thresholds
  }
 
</script>

  <div class= "menu" >
    <label  for="quarter-select"
            style="color:#000000; font-size: 20px; font-family: 'Noto Sans'; font-weight: 600;"
            >
            Choose a internet speed:
    </label>
                 
    <select style="color: #0058FF; font-size: 20px; font-family: 'Noto Sans'; font-weight: 600;" 
            bind:value={selectedQuarter} 
            id="quarter-select">
              {#each time as quarter}
                     <option 
                      style="color:#000000; font-size: 20px; font-family: 'Noto Sans'; font-weight: 400;" 
                      value={quarter}>
                     {quarter}
                     </option>
               {/each}
	  </select>
  </div>
<!-- SELECTION MENU -->
<div class= "menu" >
    <label  for="quarter-select"
            style="color:#000000; font-size: 20px; font-family: 'Noto Sans'; font-weight: 600;"
            >
            Choose a quarter of a year to see internet speed:
    </label>
                 
    <select style="color: #0058FF; font-size: 20px; font-family: 'Noto Sans'; font-weight: 600;" 
            bind:value={selectedQuarter} 
            id="quarter-select">
              {#each time as quarter}
                     <option 
                      style="color:#000000; font-size: 20px; font-family: 'Noto Sans'; font-weight: 400;" 
                      value={quarter}>
                     {quarter}
                     </option>
               {/each}
	  </select>
  </div>


<div class="graph">
  <div class="main-chart">
    <Graphic width={825} height={825}>
     <Section
       x1={0.1}
        x2={0.9}
        y1={0}
        y2={0.35}
        padding={50}
        flipY
        scaleX={scaleLinear().domain([0, 200])}
        scaleY={scaleLinear().domain([0, speed_distribution.domain('total_count')[1]])} 
      
      >
        <RectangleLayer 
          x1={speed_distribution.map('bins', bin => bin[0])}
          x2={speed_distribution.map('bins', bin => bin[1])}
          y1={speed_distribution.map('total_count', x => 0)}
          y2={speed_distribution.column('total_count')}
          onMouseover = {({index}) =>{ selectedIndex = index; console.log(selectedIndex);}}
          onMouseout = {() => selectedIndex = null}
          fill={fillRectangles}
          fillOpacity= {fillOpacityRectangles}/>
        <XAxis title={"Download speed in " + formattedDate + " [Mbps]"}
        tickCount={20} 
               titleFont={'Noto Sans'}
               titleFontSize={14}
               titleFontWeight={800}
               labelFont={'Noto Sans'} 
               labelFontSize={10}
               labelFontWeight={800}
               titleColor={'black'}
               titleYOffset = {40}
               />
        <YAxis title="Number of observed European countries"
               titleFont={'Noto Sans'}
               titleFontSize={14}
               titleFontWeight={800}
               labelFont={'Noto Sans'} 
               labelFontSize={10}
               labelFontWeight={800}
               titleColor={'black'}
               titleXOffset = {-40}

               />
      </Section>
      <Section  
       x1={0}
        x2={1}
        y1={0.4}
        y2={1}
      flipY
       {...myGeoScale}
       {zoomIdentity}
      {...zoom.handlers}
      {...pan.handlers}
      >
      >
       <PolygonLayer 
          geometry={neighbourhoods.column('$geometry')}
          stroke={'black'}
          strokeWidth={1} 
          fill={'#F5F5F5'}
        />
      <PolygonLayer 
        geometry={neighbourhoods.column('$geometry')}
        stroke={'lightgray'}
        strokeWidth={0.5}
        fill={timeseries_data.map('avg_d', myColorScale)}
      />
      </Section>
    </Graphic>
  </div>
</div>

<style>
 @import url('https://fonts.googleapis.com/css2?family=Noto+Sans:ital,wght@0,100;0,300;0,400;0,500;0,600;0,700;1,300;1,700&display=swap" rel="stylesheet');
  
  .title-paragraph {
    font-family: 'Noto sans', sans-serif;
    font-size: 18px;
    font-weight: 600;
    color: #0058FF;
    padding: 10px;
    text-decoration-line: overline;
    text-decoration-color: #0058FF;
    text-decoration-style: solid;
  }

  .footnote-paragraph {
    font-family: 'Noto sans', sans-serif;
    font-size: 10px;
    font-weight: 800;
    color: #0058FF;
    padding: 10px;
  }

  .text-paragraph {
    font-family: 'Noto sans', sans-serif;
    font-size: 14px;
    font-weight: 400;
    padding: 10px;
    text-align: justify;
  }

  .thanks-paragraph {
    font-family: 'Noto sans', sans-serif;
    font-size: 14px;
    font-weight: 400;
    padding: 10px;
    text-align: justify;
    border: 4px double;
    border-color: #0058FF;
  }

</style>