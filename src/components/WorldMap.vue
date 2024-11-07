<template>
  <div>
    <div ref="mapContainer" class="map-container"></div>
    <div ref="legendContainer" class="legend-container"></div>
  </div>
</template>

<script>
import * as d3 from 'd3';

export default {
  name: 'WorldMapWithLegendAndContours',
  mounted() {
    this.drawMap();
    this.drawLegend();
  },
  methods: {
    drawMap() {
      const width = 800;
      const height = 600;
      const svg = d3.select(this.$refs.mapContainer)
        .append('svg')
        .attr('width', width)
        .attr('height', height);

      const projection = d3.geoMercator()
        .scale(130)
        .translate([width / 2, height / 1.5]); 

      const path = d3.geoPath().projection(projection);

      // Load the custom GeoJSON data for the world map
      d3.json('/custom.geo.json').then((geojson) => {
        // Draw each country
        svg.selectAll('path')
          .data(geojson.features)
          .enter()
          .append('path')
          .attr('d', path)
          .attr('fill', () => {
            const dopValue = Math.random() * 10;
            return this.getColorBasedOnDop(dopValue);})
          .attr('stroke', '#333');
      });

      // Mock DOP data
      const dopData = [];
      for (let y = 0; y < height; y += 10) {
        for (let x = 0; x < width; x += 10) {
          const dopValue = Math.random() * 10;
          dopData.push({ x, y, value: dopValue });
        }
      }

      // Create a contour generator
      const contours = d3.contours()
        .size([width, height])
        .thresholds(d3.range(2, 10, 2));

      // Generate the contour data
      const contourData = contours(dopData.map(d => d.value));

      const colorScale = d3.scaleSequential(d3.interpolateViridis)
        .domain([0, 10]);

      // Draw the contours on the map
      svg.selectAll('path.contour')
        .data(contourData)
        .enter()
        .append('path')
        .attr('class', 'contour')
        .attr('d', d3.geoPath(d3.geoIdentity().scale(1)))
        .attr('fill', d => colorScale(d.value))
        .attr('stroke', '#000')
        .attr('stroke-width', 0.5);
    },

    drawLegend() {
      const legendHeight = 220;
      const legendWidth = 20;

      const colorScale = d3.scaleThreshold()
        .domain([2, 4, 6, 8]) 
        .range(['#B2F6AF', '#87F683', '#4EF148', '#20CE1B', '#157F11']); 

      const legendSvg = d3.select(this.$refs.legendContainer)
        .append('svg')
        .attr('width', 100)
        .attr('height', legendHeight);

      legendSvg.selectAll('rect')
        .data(colorScale.range().map((color, i) => {
          const d = colorScale.domain();
          return {
            color: color,
            value: d[i - 1] || 0,
            nextValue: d[i] || 10,
          };
        }))
        .enter()
        .append('rect')
        .attr('x', 0)
        .attr('y', (d, i) => i * (legendHeight / colorScale.range().length))
        .attr('width', legendWidth)
        .attr('height', legendHeight / colorScale.range().length - 5)
        .attr('fill', d => d.color);

      legendSvg.selectAll('text')
        .data(colorScale.domain())
        .enter()
        .append('text')
        .attr('x', legendWidth + 10)
        .attr('y', (d, i) => i * (legendHeight / colorScale.range().length) + (legendHeight / colorScale.range().length) / 2)
        .attr('dy', '0.35em')
        .text(d => `DOP < ${d}`)
        .style('font-size', '12px');
      legendSvg.append('text')
        .attr('x', legendWidth + 10)
        .attr('y', legendHeight - 20)
        .text('DOP >= 8')
        .style('font-size', '12px');
    },

    getColorBasedOnDop(dopValue) {
      if (dopValue < 2) return '#B2F6AF';
      if (dopValue < 4) return '#87F683';
      if (dopValue < 6) return '#4EF148';
      if (dopValue < 8) return '#20CE1B';
      return '#157F11';
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
</style>
