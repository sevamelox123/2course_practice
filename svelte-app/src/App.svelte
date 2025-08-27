<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  
  let svgRef;
  let jsonData = [];
  let devicesHierarchy = new Map();
  let selectedUName = "";
  let selectedSerial = "";
  let selectedDid = "";
  let selectedValues = [];
  let isLoading = false;
  let useLocalData = true;
  let debug = false;

  let startDate = "2025-08-25";
  let startTime = "13:59:29";
  let endDate = "2025-08-25";
  let endTime = "15:59:29";


  let bgAnimation = {
    x: 0,
    y: 0,
    xDirection: 1,
    yDirection: 1,
    speed: 0.5
  };
  function toggleDataSource() {
    useLocalData = !useLocalData;
  }

  function animateBackground() {
    if (typeof window === 'undefined') return;
    
    bgAnimation.x += bgAnimation.speed * bgAnimation.xDirection;
    bgAnimation.y += bgAnimation.speed * bgAnimation.yDirection;
    
    if (bgAnimation.x > 50 || bgAnimation.x < -50) bgAnimation.xDirection *= -1;
    if (bgAnimation.y > 50 || bgAnimation.y < -50) bgAnimation.yDirection *= -1;
    
    const bgElement = document.querySelector('.animated-bg');
    if (bgElement) {
      bgElement.style.transform = `translate(${bgAnimation.x}px, ${bgAnimation.y}px)`;
    }
    
    requestAnimationFrame(animateBackground);
  }

  onMount(() => {
    animateBackground();
  });

  async function loadData() {
    try {
      isLoading = true;
      
      if (useLocalData) {
        const response = await fetch("./data.json");
        jsonData = await response.json();
      } else {
        if (!startDate || !endDate) {
          alert("Выберите диапазон дат для загрузки из API");
          return;
        }
        
        let sdate = new Date(...getDateTime(startDate, startTime));
        let edate = new Date(...getDateTime(endDate, endTime));
        
        if (sdate > edate) {
          alert("Начальная дата больше конечной! Измените для продолжения.");
          return;
        }

        const startYear = sdate.getFullYear();
        const startMonth = (sdate.getMonth() + 1).toString().padStart(2, "0");
        const startDay = sdate.getDate().toString().padStart(2, "0");
        const startHours = sdate.getHours().toString().padStart(2, "0");
        const startMinutes = sdate.getMinutes().toString().padStart(2, "0");
        const startSeconds = sdate.getSeconds().toString().padStart(2, "0");
        
        const endYear = edate.getFullYear();
        const endMonth = (edate.getMonth() + 1).toString().padStart(2, "0");
        const endDay = edate.getDate().toString().padStart(2, "0");
        const endHours = edate.getHours().toString().padStart(2, "0");
        const endMinutes = edate.getMinutes().toString().padStart(2, "0");
        const endSeconds = edate.getSeconds().toString().padStart(2, "0");

        const response = await fetch(
          `/api/not_calibr/log/${startYear}-${startMonth}-${startDay}%20${startHours}:${startMinutes}:${startSeconds}/${endYear}-${endMonth}-${endDay}%20${endHours}:${endMinutes}:${endSeconds}/`,
        );
        jsonData = await response.json();
      }

      buildHierarchy();
      
    } catch (error) {
      console.error("Ошибка загрузки:", error);
      alert("Ошибка загрузки данных: " + error.message);
    } finally {
      isLoading = false;
    }
  }

  function buildHierarchy() {
    devicesHierarchy = new Map();

    for (const id in jsonData) {
      const device = jsonData[id];
      const { uName, serial, data, Date: timestamp } = device;

      if (!devicesHierarchy.has(uName)) {
        devicesHierarchy.set(uName, new Map());
      }

      const serialsMap = devicesHierarchy.get(uName);

      if (!serialsMap.has(serial)) {
        serialsMap.set(serial, new Map());
      }

      const dataMap = serialsMap.get(serial);

      for (const did in data) {
        if (data.hasOwnProperty(did)) {
          if (!dataMap.has(did)) {
            dataMap.set(did, []);
          }

          dataMap.get(did).push({
            value: data[did],
            timestamp: new Date(timestamp),
            originalId: id,
          });
        }
      }
    }
  }

  function drawChart() {
    if (!svgRef || selectedValues.length === 0) return;

    d3.select(svgRef).selectAll("*").remove();

    const margin = { top: 20, right: 30, bottom: 50, left: 60 };
    const width = 1200 - margin.left - margin.right;
    const height = 400 - margin.top - margin.bottom;

    const svg = d3.select(svgRef)
      .attr("width", width + margin.left + margin.right)
      .attr("height", height + margin.top + margin.bottom)
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    const data = selectedValues.map(d => ({
      x: new Date(d.timestamp),
      y: d.value
    })).sort((a, b) => a.x - b.x);

    const x = d3.scaleTime()
      .domain(d3.extent(data, d => d.x))
      .range([0, width]);

    const y = d3.scaleLinear()
      .domain([d3.min(data, d => d.y) * 0.95, d3.max(data, d => d.y) * 1.05])
      .range([height, 0]);

    const line = d3.line()
      .x(d => x(d.x))
      .y(d => y(d.y));

    svg.append("path")
      .datum(data)
      .attr("fill", "none")
      .attr("stroke", "steelblue")
      .attr("stroke-width", 2)
      .attr("d", line);

    svg.selectAll(".dot")
      .data(data)
      .enter().append("circle")
      .attr("class", "dot")
      .attr("cx", d => x(d.x))
      .attr("cy", d => y(d.y))
      .attr("r", 3)
      .attr("fill", "steelblue");

    svg.append("g")
      .attr("transform", `translate(0,${height})`)
      .call(d3.axisBottom(x));

    svg.append("g")
      .call(d3.axisLeft(y));

    svg.append("text")
      .attr("transform", `translate(${width / 2},${height + margin.top + 20})`)
      .style("text-anchor", "middle")
      .text("Время");

    svg.append("text")
      .attr("transform", "rotate(-90)")
      .attr("y", -margin.left + 20)
      .attr("x", -height / 2)
      .style("text-anchor", "middle")
      .text(`${selectedDid} (${getUnit(selectedDid)})`);
  }

  $: uNames = Array.from(devicesHierarchy.keys()).sort();
  $: serials = selectedUName
    ? Array.from(devicesHierarchy.get(selectedUName).keys()).sort()
    : [];
  $: dids =
    selectedUName && selectedSerial
      ? Array.from(
          devicesHierarchy.get(selectedUName).get(selectedSerial).keys(),
        ).sort()
      : [];

  $: if (selectedUName && selectedSerial && selectedDid) {
    selectedValues =
      devicesHierarchy
        .get(selectedUName)
        .get(selectedSerial)
        .get(selectedDid) || [];
    setTimeout(drawChart, 100);
  } else {
    selectedValues = [];
  }

  function getUnit(did) {
    const lowerDid = did.toLowerCase();
    if (lowerDid.includes("temp")) return "°C";
    if (lowerDid.includes("humidity")) return "%";
    if (lowerDid.includes("pressure")) return "hPa";
    if (lowerDid.includes("rssi")) return "dBm";
    if (lowerDid.includes("upit")) return "V";
    if (lowerDid.includes("lux")) return "lx";
    if (lowerDid.includes("tvoc") || lowerDid.includes("eco2")) return "ppm";
    return "";
  }

  function getDateTime(date, time) {
    const partsArray = date.split("-");
    const year = partsArray[0];
    const month = partsArray[1] - 1;
    const day = partsArray[2];
    const partsArray2 = (time || "00:00:00").split(":");
    const hour = partsArray2[0] || "00";
    const minute = partsArray2[1] || "00";
    const second = partsArray2[2] || "00";
    return [ year, month, day, hour, minute, second ];
  }
</script>
{#if debug}
<div class="animated-bg"></div>
{/if}

<div class="container">
  <div class="data-source-toggle">
    <label>Источник данных:</label>
    <button on:click={toggleDataSource} class:active={useLocalData}>
      {useLocalData ? 'Тестовый набор данных' : 'Данные с удалённого сервера'}
    </button>
    <span class="source-info">
      {useLocalData ? 'Используется тестовый файл' : 'Используются данные полученные при помощи запроса с сервера'}
    </span>
  </div>

  {#if !useLocalData}
    <div class="date-controls">
      <div class="date-group">
        <label>Начальная дата и время:</label>
        <input type="date"  bind:value={startDate} />
        <input type="time" bind:value={startTime} step="1" />
      </div>
      
      <div class="date-group">
        <label>Конечная дата и время:</label>
        <input type="date" bind:value={endDate} />
        <input type="time" bind:value={endTime} step="1" />
      </div>
    </div>
  {/if}

  <button on:click={loadData} disabled={isLoading}>
    {isLoading ? "Загрузка..." : "Запросить данные"}
  </button>

  {#if devicesHierarchy.size > 0}
    <div class="selectors">
      <div class="select-group">
        <label>Выберите устройство:</label>
        <select bind:value={selectedUName}>
          <option value="">-- Выберите uName --</option>
          {#each uNames as uName}
            <option value={uName}>{uName}</option>
          {/each}
        </select>
      </div>

      {#if selectedUName}
        <div class="select-group">
          <label>Выберите serial:</label>
          <select bind:value={selectedSerial}>
            <option value="">-- Выберите serial --</option>
            {#each serials as serial}
              <option value={serial}>{serial}</option>
            {/each}
          </select>
        </div>
      {/if}

      {#if selectedSerial}
        <div class="select-group">
          <label>Выберите параметр:</label>
          <select bind:value={selectedDid}>
            <option value="">-- Выберите параметр --</option>
            {#each dids as did}
              <option value={did}>{did}</option>
            {/each}
          </select>
        </div>
      {/if}
    </div>

    {#if selectedValues.length > 0}
      <div class="chart-container">
        <h3>График: {selectedUName} / {selectedSerial} / {selectedDid}</h3>
        <svg bind:this={svgRef} class="chart"></svg>
      </div>
    {/if}

    {#if selectedValues.length > 0 && debug}
      <div class="results">
        <h3>
          {selectedUName} / {selectedSerial} / {selectedDid}
          <span class="count">({selectedValues.length} значений)</span>
        </h3>

        <!-- <div class="values-table">
          <table>
            <thead>
              <tr>
                <th>Время измерения</th>
                <th>Значение</th>
                <th>ID записи</th>
              </tr>
            </thead>
            <tbody>
              {#each selectedValues as item}
                <tr>
                  <td class="timestamp">{item.timestamp.toLocaleString()}</td>
                  <td class="value">
                    {item.value.toFixed(2)}
                    {getUnit(selectedDid)}
                  </td>
                  <td class="id">{item.originalId}</td>
                </tr>
              {/each}
            </tbody>
          </table>
        </div> -->
        {#if debug}
        <!-- <div class="stats">
          <h4>Статистика:</h4>
          <p>Количество измерений: {selectedValues.length}</p>
          {#if selectedValues.length > 0}
            <p>
              Минимальное значение: {Math.min(...selectedValues.map(v => v.value)).toFixed(2)}
              {getUnit(selectedDid)}
            </p>
            <p>
              Максимальное значение: {Math.max(...selectedValues.map(v => v.value)).toFixed(2)}
              {getUnit(selectedDid)}
            </p>
            <p>
              Среднее значение: {(selectedValues.reduce((sum, v) => sum + v.value, 0) / selectedValues.length).toFixed(2)}
              {getUnit(selectedDid)}
            </p>
          {/if}
        </div> -->
        {/if}
      </div>
    
    {:else if selectedDid}
      <!-- <div class="no-data">
        <p>Нет данных для выбранного параметра</p>
      </div> -->
    {/if}
  {/if}
</div>

<style>
  .animated-bg {
    position: fixed;
    top: -10%;
    left: -10%;
    width: 120%;
    height: 120%;
    z-index: -1;
    background: url('/favicon.gif') center/cover;
    opacity: 0.5; 
    pointer-events: none;
  }

  .container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
    position: relative;
    z-index: 1;
    background: rgba(255, 255, 255, 0.95);
    border-radius: 10px;
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
  }

  .data-source-toggle {
    display: flex;
    align-items: center;
    gap: 15px;
    margin-bottom: 20px;
    padding: 15px;
    background: #f5f5f5;
    border-radius: 8px;
  }

  .data-source-toggle button {
    padding: 10px 20px;
    border: 2px solid #4caf50;
    background: white;
    color: #4caf50;
    border-radius: 25px;
    cursor: pointer;
    transition: all 0.3s ease;
  }

  .data-source-toggle button.active {
    background: #4caf50;
    color: white;
  }

  .data-source-toggle button:hover {
    transform: translateY(-2px);
    box-shadow: 0 2px 8px rgba(76, 175, 80, 0.3);
  }

  .source-info {
    color: #666;
    font-style: italic;
  }

  .date-controls {
    display: flex;
    gap: 20px;
    margin-bottom: 20px;
    flex-wrap: wrap;
  }

  .date-group {
    display: flex;
    flex-direction: column;
    gap: 5px;
  }

  button {
    padding: 10px 20px;
    background: #4caf50;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    margin-bottom: 20px;
    font-size: 14px;
    transition: all 0.3s ease;
  }

  button:hover:not(:disabled) {
    background: #3d8b40;
    transform: translateY(-2px);
  }

  button:disabled {
    background: #cccccc;
    cursor: not-allowed;
    transform: none;
  }

  .selectors {
    display: flex;
    gap: 20px;
    margin-bottom: 30px;
    flex-wrap: wrap;
  }

  .select-group {
    display: flex;
    flex-direction: column;
    min-width: 200px;
  }

  label {
    font-weight: bold;
    margin-bottom: 5px;
    color: #333;
  }

  select, input[type="date"], input[type="time"] {
    padding: 10px;
    border: 2px solid #ddd;
    border-radius: 5px;
    font-size: 14px;
    transition: border-color 0.3s ease;
  }

  select:focus, input:focus {
    outline: none;
    border-color: #4caf50;
    box-shadow: 0 0 5px rgba(76, 175, 80, 0.3);
  }

  .chart-container {
    margin: 30px 0;
    border: 1px solid #ddd;
    border-radius: 8px;
    padding: 20px;
    background: white;
  }

  .chart {
    width: 100%;
    height: 400px;
    display: block;
  }

  .results {
    background: #f9f9f9;
    padding: 20px;
    border-radius: 8px;
    border: 1px solid #ddd;
    margin-top: 20px;
  }

  .results h3 {
    margin: 0 0 20px 0;
    color: #2c5282;
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .count {
    font-size: 0.8em;
    color: #666;
    font-weight: normal;
  }

  .values-table {
    overflow-x: auto;
    margin-bottom: 20px;
    max-height: 400px;
    overflow-y: auto;
  }

  table {
    width: 100%;
    border-collapse: collapse;
    background: white;
  }

  th, td {
    padding: 12px;
    text-align: left;
    border: 1px solid #ddd;
  }

  th {
    background: #f0f0f0;
    font-weight: bold;
    position: sticky;
    top: 0;
  }

  .timestamp {
    font-family: monospace;
    color: #666;
  }

  .value {
    font-weight: bold;
    color: #2c5282;
    font-size: 1.1em;
  }

  .id {
    font-family: monospace;
    color: #888;
    font-size: 0.9em;
  }

  .stats {
    background: white;
    padding: 15px;
    border-radius: 5px;
    border: 1px solid #eee;
  }

  .stats h4 {
    margin: 0 0 10px 0;
    color: #333;
  }

  .stats p {
    margin: 5px 0;
    color: #555;
  }

  .no-data {
    text-align: center;
    padding: 40px;
    color: #666;
    font-style: italic;
  }

  @media (max-width: 768px) {
    .selectors, .date-controls {
      flex-direction: column;
    }

    .select-group, .date-group {
      width: 100%;
    }
    
    .container {
      margin: 10px;
      padding: 15px;
    }
    
    .animated-bg {
      opacity: 0.05;
    }
  }

</style>