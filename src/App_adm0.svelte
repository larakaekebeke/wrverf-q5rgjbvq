<script>
  import { data_adm0 } from './data_adm0.js';
  import { EU_countries } from './EU_countries.js';
  import { Graphic, PolygonLayer, fitScales } from '@snlab/florence';
  import DataContainer from '@snlab/florence-datacontainer';
  import { scaleOrdinal, scaleLinear, scaleBand } from 'd3-scale';
  import { schemeDark2 } from 'd3-scale-chromatic';
  import { onMount } from 'svelte';
  import { color } from 'd3-color';
  import BarChart from './App_barchart.svelte';


///////////////data manipulation
// Initialize DataContainers
  let polygons_EU = new DataContainer(EU_countries);
  const data_EU = new DataContainer(data_adm0);

  // Summarize totalMgr for each country
  const totalMgr_data = data_EU.groupBy('polygonId').summarise({ totalMgr: { netMgr: 'sum' } }).rows();

  // Filter data for specific years
  const data_2000 = data_EU.filter(row => row.year === 2000).rows();
  const data_2019 = data_EU.filter(row => row.year === 2019).rows();
  

  //Calculate total population change  
  const popChange_data = data_2019.map(row2019 => {
    const row2000 = data_2000.find(row2000 => row2000.polygonId === row2019.polygonId);
    return row2000
      ? {
          polygonId: row2019.polygonId,
          adminName: row2019.adminName,
          totalPopChange: row2019.popChange - row2000.popChange,
        }
      : {
          polygonId: row2019.polygonId,
          adminName: row2019.adminName,
          totalPopChange: null,
        };
  });

  // calculate natural population change (without migration)
  const popChange_woMgr_data = popChange_data.map(popChange => {
    const mgrData = totalMgr_data.find(mgr => mgr.polygonId === popChange.polygonId);
    const totalMgr = mgrData ? mgrData.totalMgr : 0;
    return {
      polygonId: popChange.polygonId,
      adminName: popChange.adminName,
      totalPopChange: popChange.totalPopChange,
      totalMgr: totalMgr, 
      popChange_woMgr: totalMgr === 0 ? 0 : popChange.totalPopChange - totalMgr,
    };
});

  //establish categories 
  const category_data = popChange_woMgr_data.map(row => {
      let category = '';
      if (row.popChange_woMgr === 0 || row.totalPopChange === 0 || row.totalMgr === 0) {
          category = 'cat7';
      } else if (row.popChange_woMgr > 0 && row.totalMgr > 0) {
          category = 'cat1';
      } else if (row.totalPopChange < 0 && row.totalMgr > 0) {
          category = 'cat2';
      } else if (row.popChange_woMgr < 0 && row.totalPopChange > 0) {
          category = 'cat3';
      } else if (row.popChange_woMgr < 0 && row.totalMgr < 0) {
          category = 'cat4';
      } else if (row.totalPopChange > 0 && row.totalMgr < 0) {
          category = 'cat5';
      } else if (row.popChange_woMgr > 0 && row.totalPopChange < 0) {
          category = 'cat6';
      }
      return {
        polygonId: row.polygonId,
        adminName: row.adminName,
        totalMgr: row.totalMgr,
        totalPopChange: row.totalPopChange,
        popChange_woMgr: row.popChange_woMgr,
        category: category
      };
  });

   const categoryLabels = {
    cat1: "Migration increases natural population growth",
    cat2: "Migration slows population decline",
    cat3: "Migration turns declining population to growth",
    cat4: "Migration increases natural population decline",
    cat5: "Migration slows population growth",
    cat6: "Migration turns growing population to decline",
    cat7: "No Data"
  };

  //add columns to polygon data
  const category_data_container = new DataContainer(category_data);
  category_data_container.setKey('polygonId');  
  polygons_EU.setKey('polygonId') 

  const addColumnFromData = (columnName, dataKey) => {
    polygons_EU.addColumn(columnName, polygons_EU.map('polygonId', polygonId => {
      const row = category_data_container.row({ key: polygonId });
      return row ? row[dataKey] : null;
    }));
  };

  addColumnFromData('adminName', 'adminName');
  addColumnFromData('totalMgr', 'totalMgr');
  addColumnFromData('totalPopChange', 'totalPopChange');
  addColumnFromData('popChange_woMgr', 'popChange_woMgr');
  addColumnFromData('category', 'category');


///////////////////////////map
  // set scales
  const myGeoScale = fitScales(polygons_EU.domain('$geometry'));
  
  const myColorScale = scaleOrdinal()
    .domain(["cat6", "cat5", "cat4", "cat1", "cat2", "cat3", "cat7"])
    .range(schemeDark2)

  // hovering over countries
  let activeCountry = null;
  let activeLabel = null;
  let clickedCategory = null;
  let tooltipElement;
  let isHovering = false;
  let isClicking = false;


  $: activeCat = activeCountry
  ? polygons_EU.row({key: activeCountry})?.category
  : null;


function handleMouseover(event) {
  isHovering = true;
  activeCountry = event.key || null; // Set activeCountry to the key of the hovered country
  if (activeCountry) {
    const countryData = polygons_EU.row({ key: activeCountry });
    if (countryData) {
      const countryName = countryData.adminName;
      const infoText = `<div><strong>Country:</strong> ${countryName}</div>`;

      if (tooltipElement) {
        tooltipElement.innerHTML = infoText;
        const offset = 10;
        tooltipElement.style.left = event.clientX + offset + 'px'; 
        tooltipElement.style.top = event.clientY - tooltipElement.offsetHeight - offset + 'px'; 
        tooltipElement.style.display = 'block';
      }
    }
  }
}

function handleMouseout() {
  isHovering = false;
  activeCountry = null; 
  if (tooltipElement) {
    tooltipElement.style.display = 'none';
    tooltipElement.innerHTML = '';
  }
}

onMount(() => {
  tooltipElement = document.createElement('div');
  tooltipElement.id = 'countryTooltip';
  tooltipElement.style.position = 'absolute';
  tooltipElement.style.display = 'none'; 
  document.body.appendChild(tooltipElement);
});


  // clicking on legendlabel or legendcolorbox
  function onClick(event) {
    clickedCategory = event.target.dataset.category || null;
  if (clickedCategory === activeLabel) {
    activeLabel = null;
    activeCountry = null;
    }else{
      activeLabel = clickedCategory;
       const countriesInCategory = polygons_EU.filter(row => row.category === activeLabel);
      activeCountry = countriesInCategory.length > 0 ? countriesInCategory[0] : null;
    }
  }


////////////////barchart
let selectedCountry = '';
let selectedData = {totalMgr: 0, totalPopChange: 0, popChange_woMgr: 0};
let scaleX = '';
let scaleY = '';
let values = [];
let adjustedValues = [];
let minY = '';
let maxY = '';
let maxAbsValue = '';
const offset = 0.3;

const parameters = ["totalMgr", "popChange_woMgr", "totPopChange"];
const colors = ["#1f77b4", "#ff7f0e", "#2ca02c"];
const labels = ["Total Migration", "Natural Population Change", "Population Change"];


$: {if (selectedCountry) {
  console.log("Selected country:", selectedCountry);
  selectedData = polygons_EU
    .filter((row) => row.adminName === selectedCountry)
    .rows()[0]||{ totalMgr: 0, totalPopChange: 0, popChange_woMgr: 0 };

}

 values = [
  parseFloat(selectedData.totalMgr.toFixed(2)),
  parseFloat(selectedData.popChange_woMgr.toFixed(2)),
  parseFloat(selectedData.totalPopChange.toFixed(2))
];

scaleX = scaleBand().domain(parameters).padding(0.5);

minY = Math.min(0,...values);
maxY = Math.max(0,...values);

maxAbsValue = Math.max(Math.abs(minY), Math.abs(maxY));

scaleY = scaleLinear().domain([-maxAbsValue - (maxAbsValue * offset), maxAbsValue + (maxAbsValue * offset)]).nice();

adjustedValues = values.map(value => ({
  y1: value > 0 ? scaleY(0) : scaleY(value), // Start from zero for positive, from value for negative
  y2: value > 0 ? scaleY(value) : scaleY(0), // End at value for positive, at zero for negative
}));
}

//click on country, graph appears
function handleClick(event) {
  const countryKey = event.key || null;
  if (countryKey) {
    const countryData = polygons_EU.row({key: countryKey});
      if (countryData)
      {
        selectedCountry = countryData.adminName;
      }
  }
}

</script>



<div class="whole">
<section>
<div class="chart-box">
  <div class="main-chart">
    <Graphic width="100%" height="100%"  
        {...myGeoScale} 
        flipY 
        zoomable
        pannable
        zoomPanSettings>
          <PolygonLayer
          geometry={polygons_EU.column('$geometry')}
          stroke={polygons_EU.map('polygonId', polygonId => {
            return polygonId === activeCountry ? 'red' : 'black';  
          })} 
          strokeWidth={polygons_EU.map('polygonId', polygonId => {
            return polygonId === activeCountry ? '3' : '0.5';  
          })} 
          fill={polygons_EU.map('polygonId', polygonId => {
            const countryCategory = polygons_EU.row({ key: polygonId }).category;
            const baseColor = color(myColorScale(countryCategory));
            if (activeLabel) {
              if (countryCategory === activeLabel) {
                return baseColor.formatRgb()
              } else {
                 return `rgba(${baseColor.r}, ${baseColor.g}, ${baseColor.b}, 0.2)`;
              }
            }
            return baseColor.formatRgb(); 
          })}
          keys={polygons_EU.column('polygonId')}
          onMouseover={handleMouseover}
          onMouseout={handleMouseout}
          onClick={handleClick}
        />
    </Graphic>
  </div>

<div class=info-box>
  <div class="map-title">
    The impact of migration on the population in Europe over 2000-2019
  </div>

<div class="legend">
  {#each myColorScale.domain() as category}
    <div class="legend-item" on:click={onClick}>
      <div
        class="legend-color-box"
        style="
          background-color: {myColorScale(category)};
          border-color: {activeCat === category ? 'red' : 'white'};"
          data-category={category}
      ></div>
      <span
        class="legend-label"
        style="color: {activeCat === category ? 'red' : 'black'};"
        data-category={category}
      >
        {categoryLabels[category]}
      </span>
    </div>
  {/each}
</div>

<div class="barchart">
{#if selectedCountry}
  <BarChart 
    {selectedCountry} 
    {scaleX} 
    {scaleY} 
    {parameters} 
    {values} 
    {labels} 
    {colors} 
    {maxAbsValue} 
  />
{:else}
  <div class="caption-placeholder">
    <p>Click on a country to see its population change dynamics!</p>
  </div>
{/if}
</div>
</div>
</div>
<section/>

<div id="descrip_project">

<p>
This visualization project aims to create an interactive map based on the dataset from Niva et al. 2023, which offers global annual net migration and population change data at the country, provincial, and communal levels from 2000 to 2019. The researchers identified six categories of migration effects on population using cumulative migration rates, population change, and natural population change excluding migration [1]. While the paper includes an interactive map, it omits migration impacts, and this project seeks to address that gap by building a foundation for a new interactive visualization that incorporates these categorical data. Initially, the focus will be on Europe at the country level, with plans for expansion to provincial levels. The visualization includes several key design features: a detail plane that appears when hovering over a country to display relevant information and highlight the corresponding category in the legend, a hover interaction where countries turn red and the matching category label is highlighted in red, a click feature allowing users to grey out all countries except those in the selected category to aid pattern recognition, and a zoom functionality to enhance the view of smaller countries. The map is designed for optimal viewing in full screen within a separate tab.
</p>
<p>
[1] Niva, V., Horton, A., Virkki, V., Heino, M., Koseno, M., Kallio, M., Kinnunen, P., Abel, G. J., Muttarak, R., Taka, M., Varis, O., & Kummu, M. (2023). World's human migration patterns in 2000–2019 unveiled by high-resolution data. <i>Nature Human Behaviour, 7</i>(11), 2023–2037. <a href= "https://www.nature.com/articles/s41562-023-01689-4"> https://doi.org/10.1038/s41562-023-01689-4 </a>

</p>

</div>
</div>


<style>
  .chart-box {
    display: flex;
    width: 900px;
    height: 500px;
    margin-left: 14%;
  }

  .main-chart {
    padding: 0.5%;
    border-width: 1px; 
    border-style: solid;
    border-color: #000;
    margin-bottom: 20px; 
    display: flex;
    justify-content: center;
    width: 70%; 
  }

  .info-box {
    position: relative;
    display: flex;
    flex-direction: column;
    width: 30%;
    border-width: 1px; 
    border-style: solid;
    border-color: #000;
    margin-bottom: 20px; 
    align-items: center; 
  }

  .map-title {
    position: absolute; 
    margin-top: 10px;
    display: flex;
    font-size: 20px;
    font-weight: bold;   
    text-align: center;
    padding: 8px;
    font-family: 'Teko', serif
  }

  .legend {
    position: absolute;
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    justify-content: center; 
    margin-top: 125px; 
    width: auto;
    font-family: 'Calibri', serif
  }

  .legend-item {
    display: flex;
    align-items: center; 
    gap: 5px; 
    padding: 5px;
  }

  .legend-color-box {
    width: 12px;
    height: 12px;
    border-width: 2px;
    border-style: solid;
  }

  .legend-label {
    font-size: 12px;
  }

  .barchart {
    position: absolute;
    display: flex;
    flex-direction: column;
    gap: 5px;
    align-items: flex-start;
    justify-content: center; 
    margin-top: 300px; 
    width: auto;
  }

  .caption-placeholder {
    font-size: 12px; 
    font-family: 'Calibri', serif
  }

 :global(#countryTooltip) {
  position: absolute;
  font-family: "Spectral", serif;
  font-size: 12px;
  fill: rgb(0, 0, 0);
  background-color: rgba(250, 250, 250, 0.95);
  padding: 5px; 
  border-color: black;
  border-width: 1px;
  border-style: solid;
  border-radius: 5px; 
  display: none; 
  z-index: 1000;
}


  p{
    font-family: 'Calibri', serif;
    font-size: 12px;
    width: 90%;
    margin-left: 5%;
    margin-right: 5%;
    text-align: justify;
  }

</style>