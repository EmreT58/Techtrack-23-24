<!-- Radarchart.svelte -->
<script>
  import { onMount } from "svelte";
  import * as d3 from 'd3';
  import dataset from "../lib/p4pfighters.json";
  import { storeFighter1, storeFighter2 } from '../lib/selectedFighters.js';

  onMount(() => {
    // Extracting data from the dataset
    // First data extraction for initializing radar chart with all fighters' metrics
    const fightersData = dataset.map(fighter => ({
      id: fighter.id,
      name: fighter.name,
      metrics: {
        significant_striking_accuracy: fighter.significant_striking_accuracy,
        significant_strike_defence: fighter.significant_strike_defence,
        takedown_accuracy: fighter.takedown_accuracy,
        takedown_defense: fighter.takedown_defense,
      },
    }));

    // Set up the dimensions of the radarchart
    const width = 400;
    const height = 400;
    const margin = { top: 50, right: 50, bottom: 50, left: 50 };
    const radius = Math.min(width / 2, height) / 1 - margin.top; // Adjusted radius

    // Create SVG container for the radarchart
    const svg = d3.select("#radarchart")
      .append("svg")
      .attr("width", width)
      .attr("height", height)
      .append("g")
      .attr("transform", `translate(${width / 2},${height / 2})`);

    // List of metric names
    const metrics = Object.keys(fightersData[0].metrics);

    // Scale for the radius
    const rScale = d3.scaleLinear()
      .domain([0, 100]) // Assuming percentages for the metrics
      .range([0, radius]);

    // Add labels for each metric
    const labelMargin = 20; // Adjust this value to increase or decrease the distance from the center
    const labels = svg.selectAll(".label")
      .data(metrics)
      .enter().append("text")
      .attr("x", (d, i) => (rScale.range()[1] + labelMargin) * Math.cos((i * 2 * Math.PI) / metrics.length - Math.PI / 2))
      .attr("y", (d, i) => (rScale.range()[1] + labelMargin) * Math.sin((i * 2 * Math.PI) / metrics.length - Math.PI / 2))
      .text(d => d.replace(/_/g, ' ')) // Label formatting - replacing underscores with a space
      .style("text-anchor", "middle")
      .attr("transform", (d, i) => {
        const angle = (i * 360) / metrics.length; // Calculate the angle for each label
        const x = (rScale.range()[1] + labelMargin) * Math.cos((angle - 90) * (Math.PI / 180));
        const y = (rScale.range()[1] + labelMargin) * Math.sin((angle - 90) * (Math.PI / 180));
        // Check if the label needs special rotation
        const rotation = d === "significant_strike_defence" ? 90 : (d === "takedown_defense" ? -90 : 0);
        return `rotate(${rotation} ${x},${y})`;
      });

      // Create a circular grid
      const circles = svg.selectAll("circle")
        .data(rScale.ticks(5).slice(1))
        .enter().append("circle")
        .attr("r", d => rScale(d))
        .style("stroke", "#888")
        .style("fill", "none");

      // Add labels to the circular grid
      const gridLabels = svg.selectAll(".grid-label")
        .data(rScale.ticks(5).slice(1))
        .enter().append("text")
        .attr("class", "grid-label")
        .attr("x", 4) // Adjust labels position on X-axis
        .attr("y", d => -rScale(d) + 22) // Adjust the offset for better centering on the Y-axis
        .text(d => `${d}%`)
        .style("text-anchor", "middle")
        .style("opacity", "50%") // Label opacity to make it subtle

    // Function to update the radar radarchart based on selected fighters | (bad function name)
    function updateRadarChart(fighter1, fighter2) { // function updateSelectedFighters(fighter1, fighter2)
      // Convert selected fighter IDs to numbers
      const selectedFighter1 = +fighter1;
      const selectedFighter2 = +fighter2;

      console.log("Blauw Fighter1:", selectedFighter1);
      console.log("Bood Fighter2:", selectedFighter2);

      // Extracting data for the selected fighters
      // Second data extraction within update function for selected fighters
      const fightersData = dataset
        .filter(fighter => fighter.id === selectedFighter1 || fighter.id === selectedFighter2)
        .map((fighter, index) => ({
          name: fighter.name,
          metrics: {
            significant_striking_accuracy: fighter.significant_striking_accuracy,
            significant_strike_defence: fighter.significant_strike_defence,
            takedown_accuracy: fighter.takedown_accuracy,
            takedown_defense: fighter.takedown_defense,
          },
          color: fighter.id === selectedFighter1 ? "blue" : "red", // Assign colors dynamically based on order
        }));
      // Update the radarchart based on the selected fighters
      updateChart(svg, fightersData);
    }

    // Function to update the radar radarchart
    function updateChart(svg, fighters) {
      // Create line function
      const line = d3.lineRadial()
        .angle((_, i) => (i * 2 * Math.PI) / metrics.length)
        .radius(d => rScale(d))
        .curve(d3.curveLinearClosed); // Close the path

      // Add a tooltip
      const tooltip = d3.select("#radarchart")
        .append("div")
        .style("position", "absolute")
        .style("visibility", "hidden")
        .style("background-color", "white")
        .style("padding", "5px")
        .style("border-radius", "5px");

      // Update lines
      svg.selectAll(".line")
        .data(fighters)
        .join("path")
        .attr("class", "line")
        .attr("d", fighter => line(metrics.map(metric => fighter.metrics[metric]))) // This translates the metric values of a specific fighter into the corresponding line in the radar chart
        .attr("stroke", fighter => fighter.color)
        .attr("fill", fighter => fighter.color) // Set fill color
        .attr("fill-opacity", 0.1) // Set fill opacity
        .on("mouseover", (event, d) => {
          tooltip.style("visibility", "visible")
            .html(`
              <strong>Name:</strong> ${d.name}<br>
              <strong>striking accuracy:</strong> ${d.metrics.significant_striking_accuracy}%<br>
              <strong>striking defence:</strong> ${d.metrics.significant_strike_defence}%<br>
              <strong>takedown accuracy:</strong> ${d.metrics.takedown_accuracy}%<br>
              <strong>takedown defense:</strong> ${d.metrics.takedown_defense}%
            `)
            .style("top", event.pageY + "px")
            .style("left", event.pageX + "px");
        })
        .on("mouseout", () => {
          tooltip.style("visibility", "hidden");
        });

      // Update data points
      svg.selectAll(".dot")
        .data(fighters.flatMap(d => metrics.map(metric => ({ fighter: d, metric: metric }))))
        .join("circle")
        .attr("class", "dot")
        .attr("cx", d => rScale(d.fighter.metrics[d.metric]) * Math.cos((metrics.indexOf(d.metric) * 2 * Math.PI) / metrics.length - Math.PI / 2))
        .attr("cy", d => rScale(d.fighter.metrics[d.metric]) * Math.sin((metrics.indexOf(d.metric) * 2 * Math.PI) / metrics.length - Math.PI / 2))
        .attr("r", 5) // Dot sizing / radius
        .style("fill", d => d.fighter.color)
        .style("opacity", 0.8);
    }

    // Dropdown menu for fighter selection
    const dropdown = d3.select("#fighterDropdown");
    dropdown // Placeholder option - First dropdown
      .append("option")
      .text("Select a fighter")
      .attr("value", "0");

    dropdown // First dropdown
      .selectAll("option:not(:first-child)") // Select all options except first
      .data(dataset.map(d => ({ id: d.id, name: d.name }))) // Creates an object with id and name properties
      .enter()
      .append("option")
      .text(d => d.name)
      .attr("value", d => d.id); // Assigns corresponding fighter ID to option value

    const dropdown2 = d3.select("#fighterDropdown2");
    dropdown2 // Placeholder option - Second dropdown
      .append("option")
      .text("Select a fighter")
      .attr("value", "0");

    dropdown2 // Second dropdown
      .selectAll("option:not(:first-child)")
      .data(dataset.map(d => ({ id: d.id, name: d.name })))
      .enter()
      .append("option")
      .text(d => d.name)
      .attr("value", d => d.id);

    // Event listener for dropdown changes
    dropdown.on("change", () => {
      const selectedFighter1 = dropdown.property("value");
      const selectedFighter2 = dropdown2.property("value");
      updateRadarChart(selectedFighter1, selectedFighter2);
      storeFighter1.set(selectedFighter1);
    });

    dropdown2.on("change", () => {
      const selectedFighter1 = dropdown.property("value");
      const selectedFighter2 = dropdown2.property("value");
      updateRadarChart(selectedFighter1, selectedFighter2);
      storeFighter2.set(selectedFighter2);
    });
  });
</script>

<!-- HTML -->
<section>
  <div id="fighter-selector">
    <div>
      <label for="fighterDropdown">Fighter 1:</label>
      <select id="fighterDropdown"></select>
    </div>
    <div>
      <label for="fighterDropdown2">Fighter 2:</label>
      <select id="fighterDropdown2"></select>
    </div>
  </div>
  <hr>
  <h3>Radarchart: Fighter Skills Accuracy</h3>
  <div id="radarchart"></div>
</section>

<!-- CSS -->
<style>
  section {
    display: flex;
    height: auto;
    width: 100%;
    flex-direction: column;
  }
  h3 {
    text-align: center;
  }
  #fighter-selector {
    display: flex;
    flex-direction: row;
    justify-content: space-around;
    margin: 1em 0em;
  }
  select {
    margin-top: 0.3em;
  }
  #fighter-selector > div {
    display: flex;
    flex-direction: column;
    justify-content: center;
    text-align: center;
  }
  hr {
    margin: 1em;
  }
  #radarchart {
    display: flex;
    justify-content: center;
  }
  #fighter-selector > div:first-of-type {
    color: blue;
  }
  #fighter-selector > div:last-of-type {
    color: red;
  }
</style>