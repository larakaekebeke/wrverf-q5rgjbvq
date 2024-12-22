<script>
  import { Graphic, RectangleLayer, XAxis, YAxis, LabelLayer, x2s } from '@snlab/florence';
  export let scaleX;
  export let scaleY;
  export let selectedCountry;
  export let parameters;
  export let values;
  export let colors;
  export let maxAbsValue;
  export let labels;
</script>


<div class="graph">
 <div class="title">
    <h4>Population change dynamics of {selectedCountry} (2000-2019)</h4>
  </div>
  <div class="main-chart">
    <Graphic width={100} height={100} {scaleX} {scaleY} flipY padding={50}>
              <RectangleLayer
              x1={parameters}
              x2={x2s(parameters)}
              y1={values.map(value => value > 0 ? 0 : value)} 
              y2={values.map(value => value > 0 ? value : 0)} 
              fill={colors}
              stroke="black"
          />
      <XAxis baseLineColor="black" baseLineWidth={1} vjust=center 
       ticks={false}/>
      <YAxis baseLineWidth={1} title="Per 1000 habitants" titleFontSize={14} titleFontWeight={600} titleXOffset={-40} labelFontSize={12}/>
       <LabelLayer
        x={parameters.map (parameter => parameter)}
        y={values.map(value => (value > 0 ? value + (maxAbsValue/5) : value - (maxAbsValue/5)))}
        text={values}
        fontSize={12}
        anchorPoint="l"
      />
    </Graphic>
   </div>
       <div class="legend">
      {#each colors as color, index}
        <div class="legend-item">
          <div class="legend-color-box" style="background-color: {color};"></div>
          <span class="legend-label">{labels[index]}</span>
        </div>
      {/each}
    </div>
  </div>


<style>
.graph {
  display: flex;
  justify-content: center;
  align-items: center;
}

.main-chart {
  position: relative;;
}

.title {
  font-size: 12px; 
  font-family: 'Calibri', serif
  
}

.labelLayer {
  padding: 2px;

}

.legend-item {
    display: flex;
    align-items: center; 
    gap: 5px; 
    padding: 5px;
  }

.legend-color-box {
    width: 20px;
    height: 10px;
    border-width: 1.5px;
    border-style: solid;
  }


.legend-label {
    font-size: 12px;
  }

</style>