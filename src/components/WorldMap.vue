<template>
  <div>
    <div ref="mapContainer" class="map-container"></div>
    <div ref="zoomControls" class="zoom-controls">
      <button @click="zoomIn">+</button>
      <button @click="zoomOut">-</button>
    </div>
    <div ref="legendContainer" class="legend-container"></div>
  </div>
</template>

<script>
import * as d3 from 'd3';

export default {
  name: 'WorldMapWithColoredRegionsAndZoom',
  mounted() {
    this.drawMap();
    this.drawLegend();
  },
  methods: {
    async drawMap() {
      const width = 800;
      const height = 600;

      // Create an SVG container
      this.svg = d3.select(this.$refs.mapContainer)
        .append('svg')
        .attr('width', width)
        .attr('height', height);

      // Create a group to contain all the map elements
      this.g = this.svg.append('g');

      const projection = d3.geoMercator()
        .scale(130)
        .translate([width / 2, height / 1.5]);

      const path = d3.geoPath().projection(projection);

      // Load GeoJSON and DOP data
      const [geojson, dopData] = await Promise.all([
        d3.json('/custom.geo.json'),
        this.loadDopData('/dop_output.txt'),
      ]);

      const gridWidth = 100;
      const gridHeight = 100;
      const dopGrid = this.createDopGrid(dopData, gridWidth, gridHeight);

      const gridCellWidth = width / gridWidth;
      const gridCellHeight = height / gridHeight;

      // Draw each cell based on DOP value
      this.g.append('g')
        .selectAll('rect')
        .data(dopGrid)
        .enter()
        .append('rect')
        .attr('x', d => d.x * gridCellWidth)
        .attr('y', d => d.y * gridCellHeight)
        .attr('width', gridCellWidth)
        .attr('height', gridCellHeight)
        .attr('fill', d => this.getColorBasedOnDop(d.PDOP))
        .attr('stroke', 'none')
        .attr('opacity', 0.4);

      // Draw country boundaries on top of the grid
      this.g.selectAll('path.country')
        .data(geojson.features)
        .enter()
        .append('path')
        .attr('class', 'country')
        .attr('d', path)
        .attr('stroke', '#333')
        .attr('stroke-width', 0.5)
        .attr('fill', 'none');

      this.zoom = d3.zoom()
        .scaleExtent([1, 8])
        .translateExtent([[0, 0], [width, height]])
        .on('zoom', (event) => {
          this.g.attr('transform', event.transform);
        });

      this.svg.call(this.zoom);
    },

    zoomIn() {
      this.svg.transition().call(this.zoom.scaleBy, 1.2);
    },

    zoomOut() {
      this.svg.transition().call(this.zoom.scaleBy, 0.8);
    },

    async loadDopData(filePath) {
      const response = await fetch(filePath);
      const text = await response.text();
      return text.split('\n').slice(1).map(line => {
        const [LatitudeDegrees, LongitudeDegrees, GDOP, PDOP, HDOP, VDOP, TDOP, NumInView] = line.split(/\s+/);
        return {
          LatitudeDegrees: parseFloat(LatitudeDegrees),
          LongitudeDegrees: parseFloat(LongitudeDegrees),
          GDOP: parseFloat(GDOP),
          PDOP: PDOP === 'NaN' ? null : parseFloat(PDOP),
          HDOP: parseFloat(HDOP),
          VDOP: parseFloat(VDOP),
          TDOP: parseFloat(TDOP),
          NumInView: parseInt(NumInView, 10),
        };
      });
    },

    createDopGrid(dopData, width, height) {
      const grid = [];
      dopData.forEach(row => {
        const { LatitudeDegrees, LongitudeDegrees, PDOP } = row;
        const x = Math.floor(((LongitudeDegrees + 180) / 360) * width);
        const y = Math.floor(((90 - LatitudeDegrees) / 180) * height);

        if (x >= 0 && x < width && y >= 0 && y < height) {
          grid.push({
            x: x,
            y: y,
            PDOP: PDOP !== null ? PDOP : -1
          });
        }
      });
      return grid;
    },

    getColorBasedOnDop(pdopValue) {
      const colors = [
        '#00FF00', '#66FF00', '#CCFF00', 
        '#FFFF00', '#FF9900', '#FF3300', '#FF0000'
      ];

      if (pdopValue === -1) return '#e0e0e0';
      if (pdopValue < 0 || pdopValue > 3) return '#e0e0e0';

      const index = Math.round(pdopValue * 2);
      return colors[index];
    },

    drawLegend() {
      const colors = [
        '#00FF00', '#66FF00', '#CCFF00', 
        '#FFFF00', '#FF9900', '#FF3300', '#FF0000'
      ];
      const values = [0, 0.5, 1, 1.5, 2, 2.5, 3];

      const legend = d3.select(this.$refs.legendContainer)
        .append('svg')
        .attr('width', 200)
        .attr('height', 300);

      const legendScale = d3.scaleBand()
        .domain(values)
        .range([20, 280]);

      const legendAxis = d3.axisRight(legendScale)
        .tickFormat(d => d.toFixed(1));

      legend.append('g')
        .attr('transform', 'translate(40, 0)')
        .call(legendAxis);

      colors.forEach((color, index) => {
        legend.append('rect')
          .attr('x', 10)
          .attr('y', legendScale(values[index]))
          .attr('width', 20)
          .attr('height', legendScale.bandwidth())
          .attr('fill', color);
      });

      legend.append('rect')
        .attr('x', 10)
        .attr('y', 300)
        .attr('width', 20)
        .attr('height', 20)
        .attr('fill', '#e0e0e0');

      legend.append('text')
        .attr('x', 40)
        .attr('y', 315)
        .text('No Data')
        .style('font-size', '12px')
        .attr('alignment-baseline', 'middle');
    },
  },
};
</script>

<style>
.map-container {
  width: 80%;
  height: 100%;
  float: left;
}

.legend-container {
  width: 20%;
  height: 100%;
  float: left;
  display: flex;
  justify-content: center;
  align-items: center;
}

.zoom-controls {
  position: absolute;
  top: 20px;
  right: 20px;
  display: flex;
  flex-direction: column;
}

.zoom-controls button {
  margin: 5px 0;
  width: 40px;
  height: 40px;
  font-size: 24px;
  cursor: pointer;
}
</style>
