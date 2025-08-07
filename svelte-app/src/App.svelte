<!-- local ip after running rollup http://localhost:8080 -->

<!-- <script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';
  
    let svgRef;
  
    const data = [
      { x: 0, y: 5 },
      { x: 1, y: 9 },
      { x: 2, y: 7 },
      { x: 3, y: 12 },
      { x: 4, y: 6 }
    ];
  
    onMount(() => {
      const margin = { top: 20, right: 20, bottom: 30, left: 40 };
      const width = 600 - margin.left - margin.right;
      const height = 400 - margin.top - margin.bottom;
  
      const svg = d3.select(svgRef)
        .attr('width', width + margin.left + margin.right)
        .attr('height', height + margin.top + margin.bottom)
        .append('g')
        .attr('transform', `translate(${margin.left},${margin.top})`);
  
      const x = d3.scaleLinear()
        .domain([0, d3.max(data, d => d.x)])
        .range([0, width]);
  
      const y = d3.scaleLinear()
        .domain([0, d3.max(data, d => d.y)])
        .range([height, 0]);
  
      const line = d3.line()
        .x(d => x(d.x))
        .y(d => y(d.y));
  
      svg.append('path')
        .datum(data)
        .attr('fill', 'none')
        .attr('stroke', 'steelblue')
        .attr('stroke-width', 2)
        .attr('d', line);
  
      svg.append('g')
        .attr('transform', `translate(0,${height})`)
        .call(d3.axisBottom(x));
  
      svg.append('g')
        .call(d3.axisLeft(y));
    });
  </script> -->

<!-- <svg bind:this={svgRef}></svg> -->

<script>
  async function downloadJson() {
    try {
      let startYear = (2025).toString().padStart(4, "0");
      let startMonth = (4).toString().padStart(2, "0");
      let startDay = (22).toString().padStart(2, "0");
      let startHours = (12).toString().padStart(2, "0");
      let startMinutes = (7).toString().padStart(2, "0");
      let startSeconds = (40).toString().padStart(2, "0");
      let endYear = (2025).toString().padStart(4, "0");
      let endMonth = (4).toString().padStart(2, "0");
      let endDay = (22).toString().padStart(2, "0");
      let endHours = (13).toString().padStart(2, "0");
      let endMinutes = (7).toString().padStart(2, "0");
      let endSeconds = (40).toString().padStart(2, "0");

      // http://dbrobo1.mf.bmstu.ru/db_api_REST/not_calibr/log/2025-07-20%2012:00:00/2025-08-01%2012:00:00/
      const response = await fetch(
        `/api/not_calibr/log/${startYear}-${startMonth}-${startDay}%20${startHours}:${startMinutes}:${startSeconds}/${endYear}-${endMonth}-${endDay}%20${endHours}:${endMinutes}:${endSeconds}/`,
      );

      // const response = await fetch("/api/calibr/log/2025-04-22%2012:07:40/2025-04-22%2013:07:40/");
      const jsonData = await response.json();

      for (const key in jsonData) {
        if (jsonData.hasOwnProperty(key)) {
          const device = jsonData[key];
          console.log(`Устройство ${key}:`);
          console.log(`Дата: ${device.Date}`);
          console.log(`Серийный номер: ${device.serial}`);
          console.log(`Температура: ${device.data.BME280_temp}°C`);
          console.log(`Влажность: ${device.data.BME280_humidity}%`);
          console.log("---------------------");
        }
      }
      
      const jsonStr = JSON.stringify(jsonData, null, 2);

      const blob = new Blob([jsonStr], { type: "application/json" });
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = "data-export.json";
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      URL.revokeObjectURL(url);
    } catch (error) {
      console.error("Download failed:", error);
      alert("Download failed. See console for details.");
    }
  }
  let startDate = "";
  let endDate = "";
</script>

<div>
  <label
    >Начальная дата:
    <input type="date" bind:value={startDate} />
  </label>
</div>

<div>
  <label
    >Конечная дата:
    <input type="date" bind:value={endDate} />
  </label>
</div>

<p>Выбрано: с {startDate} по {endDate}</p>

<button on:click={downloadJson}>Download JSON</button>

<main>
  <img src="/favicon.jpg" alt="teto" />
  <p>svo</p>
</main>

<!-- <script>
	export let name;
    let count = 0

    function increment() {
        count+=1
  }
</script>

<main>
	<h1>Hello {name}!</h1>
	<p>SVo SVo</p>
    {#if count  < 5}
        <p> count is {count}</p>
    {/if}
    <button on:click={increment}>Click</button>
</main>

<style>
    main {
        text-align: center;
        padding: 1em;
        max-width: 240px;
        margin: 0 auto;
    }

    h1 {
        color: #ff3e00;
        text-transform: uppercase;
        font-size: 4em;
        font-weight: 100;
    }

    @media (min-width: 640px) {
        main {
            max-width: none;
        }
    }
</style> -->
