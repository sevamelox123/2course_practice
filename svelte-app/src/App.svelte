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
      isLoading = true;
      let sdate = new Date(
        getDateTime(startDate, startTime).year,
        getDateTime(startDate, startTime).mounth,
        getDateTime(startDate, startTime).day,
        getDateTime(startDate, startTime).hour,
        getDateTime(startDate, startTime).minute,
        getDateTime(startDate, startTime).second,
      );

      let edate = new Date(
        getDateTime(endDate, endTime).year,
        getDateTime(endDate, endTime).mounth,
        getDateTime(endDate, endTime).day,
        getDateTime(endDate, endTime).hour,
        getDateTime(endDate, endTime).minute,
        getDateTime(endDate, endTime).second,
      );

      if (sdate > edate) alert("Начальная дата больше конечной! Измените для продолжения работы.");
      let startYear = sdate.getFullYear();
      let startMonth = sdate.getMonth();
      let startDay = sdate.getDate();
      let startHours = sdate.getHours();
      let startMinutes = sdate.getMinutes();
      let startSeconds = sdate.getSeconds();
      let endYear = edate.getFullYear();
      let endMonth = edate.getMonth();
      let endDay = edate.getDate();
      let endHours = edate.getHours();
      let endMinutes = edate.getMinutes();
      let endSeconds = edate.getSeconds();

      // let startYear = (2025).toString().padStart(4, "0");
      // let startMonth = (4).toString().padStart(2, "0");
      // let startDay = (22).toString().padStart(2, "0");
      // let startHours = (12).toString().padStart(2, "0");
      // let startMinutes = (7).toString().padStart(2, "0");
      // let startSeconds = (40).toString().padStart(2, "0");
      // let endYear = (2025).toString().padStart(4, "0");
      // let endMonth = (4).toString().padStart(2, "0");
      // let endDay = (22).toString().padStart(2, "0");
      // let endHours = (13).toString().padStart(2, "0");
      // let endMinutes = (7).toString().padStart(2, "0");
      // let endSeconds = (40).toString().padStart(2, "0");

      // http://dbrobo1.mf.bmstu.ru/db_api_REST/not_calibr/log/2025-07-20%2012:00:00/2025-08-01%2012:00:00/
      // const response = await fetch(
      //   `/api/not_calibr/log/${startYear}-${startMonth}-${startDay}%20${startHours}:${startMinutes}:${startSeconds}/${endYear}-${endMonth}-${endDay}%20${endHours}:${endMinutes}:${endSeconds}/`,
      // );
      // const response = await fetch("/api/not_calibr/log/2025-04-22%2012:07:40/2025-04-22%2013:07:40/");
         
      const response = await fetch('./data.json');
      const jsonData = await response.json();
      const devicesMap = new Map();
      
      for(const id in jsonData){
        const device = jsonData[id];
        
        if(devicesMap.has(device.uName)) {
          devicesMap.get(device.uName).add(device.serial);
        } 
        else {
          devicesMap.set(device.uName, new Set([device.serial]));
        }
      }
      
      // Преобразуем Set в массив для удобства
      const uniqueDevices = Array.from(devicesMap, ([uName, serialsSet]) => ({
        uName,
        serials: Array.from(serialsSet),
        count: serialsSet.size
      }));
      
      console.log("Уникальные серийные номера:");
      uniqueDevices.forEach(device => {
        console.log(device.uName, device.serials);
      });

      // for (const key in jsonData) {
      //   if (jsonData.hasOwnProperty(key)) {
      //     const device = jsonData[key];
      //     // console.log(`Устройство ${key}:`);
      //     // console.log(`Дата: ${device.Date}`);
      //     // console.log(`Серийный номер: ${device.serial}`);
      //     // console.log(`Температура: ${device.data.BME280_temp}°C`);
      //     // console.log(`Влажность: ${device.data.BME280_humidity}%`);
      //     // console.log("---------------------");
      //     console.log(device.hasOwnProperty);
      //   }
      // }

      // const jsonStr = JSON.stringify(jsonData, null, 2);

      // const blob = new Blob([jsonStr], { type: "application/json" });
      // const url = URL.createObjectURL(blob);
      // const a = document.createElement("a");
      // a.href = url;
      // a.download = "data-export.json";
      // document.body.appendChild(a);
      // a.click();
      // document.body.removeChild(a);
      // URL.revokeObjectURL(url);
    } catch (error) {
      console.error("Download failed:", error);
      alert("Download failed. See console for details.");
    } finally {
      isLoading = false;
    }
  }

  let startDate = "";
  let startTime = "";
  let endDate = "";
  let endTime = "";
  let isLoading = false;

  function getDateTime(date, time) {
    var partsArray = date.split("-");
    let year = partsArray[0];
    // console.log(` $$ ${partsArray[0]}`)
    let mounth = partsArray[1];
    let day = partsArray[2];
    var partsArray2 = time.split(":");
    let hour = partsArray2[0];
    let minute = partsArray2[1];
    let second = partsArray2[2];
    return { year, mounth, day, hour, minute, second };
  }
</script>

<div class="startDate">
  <label
    >Начальная дата и время:
    <input type="date" bind:value={startDate} />
    <input type="time" bind:value={startTime} step="1" />
  </label>
</div>

<div class="endDate">
  <label
    >Конечная дата и время:
    <input type="date" bind:value={endDate} />
    <input type="time" bind:value={endTime} step="1" />
  </label>
</div>

<p>Выбрано: с {startDate} | {startTime} по {endDate} | {endTime}</p>

<button on:click={downloadJson} disabled={isLoading}>
  {isLoading ? "Загрузка..." : "Скачать JSON"}
</button>

<main>
  <img class="teto" src="/favicon.jpg" alt="teto" />
  <p>svo</p>
</main>

<style>
  main {
    text-align: center;
    padding: 1em;
    max-width: 240px;
    margin: 0 auto;
  }

  p {
    color: #ff3e00;
    text-transform: uppercase;
    font-size: 2em;
    font-weight: 100;
    text-align: center;
  }
  button {
    padding: 10px 15px;
    background: #4caf50;
    color: white;
    border: none;
    cursor: pointer;
    text-align: center;
  }
  .startDate {
    text-align: center;
  }
  .endDate {
    text-align: center;
  }
  .error {
    color: red;
    margin: 10px 0;
  }
  button:disabled {
    background: #cccccc;
  }
</style>
