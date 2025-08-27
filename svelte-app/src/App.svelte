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
  let processedData = [];
  let isLoading = false;
  let useLocalData = true;
  let debug = false;

  let startDate = "2025-08-25";
  let startTime = "13:59:29";
  let endDate = "2025-08-25";
  let endTime = "15:59:29";

  // Новые переменные для осреднения и типа графика
  let averagingType = "raw";
  let chartType = "line";

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

  // Функции для обработки данных
  function processData() {
    if (selectedValues.length === 0) {
      processedData = [];
      return;
    }
    
    const sortedData = [...selectedValues].sort((a, b) => a.timestamp - b.timestamp);
    
    switch (averagingType) {
      case "raw":
        processedData = sortedData;
        break;
        
      case "hourly":
        processedData = calculateAverages(sortedData, 60);
        break;
        
      case "3hourly":
        processedData = calculateAverages(sortedData, 180);
        break;
        
      case "daily":
        processedData = calculateAverages(sortedData, 1440);
        break;
        
      case "daily_minmax":
        processedData = calculateDailyMinMax(sortedData);
        break;
    }
    
    // Перерисовываем график после обработки данных
    setTimeout(drawChart, 100);
  }
  
  function calculateAverages(data, minutesInterval) {
    if (data.length === 0) return [];
    
    const result = [];
    let currentGroup = [];
    let groupStartTime = new Date(data[0].timestamp);
    
    // Округляем до начала интервала
    groupStartTime.setMinutes(Math.floor(groupStartTime.getMinutes() / minutesInterval) * minutesInterval);
    groupStartTime.setSeconds(0);
    groupStartTime.setMilliseconds(0);
    
    let groupEndTime = new Date(groupStartTime);
    groupEndTime.setMinutes(groupEndTime.getMinutes() + minutesInterval);
    
    data.forEach(item => {
      const itemTime = new Date(item.timestamp);
      
      if (itemTime < groupEndTime) {
        currentGroup.push(item);
      } else {
        if (currentGroup.length > 0) {
          const avgValue = currentGroup.reduce((sum, d) => sum + d.value, 0) / currentGroup.length;
          result.push({
            timestamp: new Date(groupStartTime.getTime() + (groupEndTime.getTime() - groupStartTime.getTime()) / 2),
            value: avgValue,
            count: currentGroup.length,
            min: Math.min(...currentGroup.map(d => d.value)),
            max: Math.max(...currentGroup.map(d => d.value))
          });
        }
        
        currentGroup = [item];
        groupStartTime = new Date(groupEndTime);
        groupEndTime = new Date(groupStartTime);
        groupEndTime.setMinutes(groupEndTime.getMinutes() + minutesInterval);
      }
    });
    
    // Добавляем последнюю группу
    if (currentGroup.length > 0) {
      const avgValue = currentGroup.reduce((sum, d) => sum + d.value, 0) / currentGroup.length;
      result.push({
        timestamp: new Date(groupStartTime.getTime() + (groupEndTime.getTime() - groupStartTime.getTime()) / 2),
        value: avgValue,
        count: currentGroup.length,
        min: Math.min(...currentGroup.map(d => d.value)),
        max: Math.max(...currentGroup.map(d => d.value))
      });
    }
    
    return result;
  }
  
  function calculateDailyMinMax(data) {
    if (data.length === 0) return [];
    
    const result = [];
    const groups = {};
    
    // Группируем по дням
    data.forEach(item => {
      const dateKey = new Date(item.timestamp);
      dateKey.setHours(0, 0, 0, 0);
      const key = dateKey.toISOString().split('T')[0];
      
      if (!groups[key]) {
        groups[key] = [];
      }
      
      groups[key].push(item);
    });
    
    // Для каждой группы находим min, max и среднее
    for (const dateKey in groups) {
      const group = groups[dateKey];
      const min = Math.min(...group.map(d => d.value));
      const max = Math.max(...group.map(d => d.value));
      const avg = group.reduce((sum, d) => sum + d.value, 0) / group.length;
      
      // Добавляем три точки: min, avg, max
      const date = new Date(dateKey);
      result.push({
        timestamp: new Date(date.setHours(6, 0, 0, 0)),
        value: min,
        type: "min"
      });
      
      result.push({
        timestamp: new Date(date.setHours(12, 0, 0, 0)),
        value: avg,
        type: "avg"
      });
      
      result.push({
        timestamp: new Date(date.setHours(18, 0, 0, 0)),
        value: max,
        type: "max"
      });
    }
    
    return result.sort((a, b) => a.timestamp - b.timestamp);
  }

  function drawChart() {
    if (!svgRef || processedData.length === 0) return;

    d3.select(svgRef).selectAll("*").remove();

    const margin = { top: 20, right: 30, bottom: 50, left: 60 };
    const width = 1200 - margin.left - margin.right;
    const height = 400 - margin.top - margin.bottom;

    const svg = d3.select(svgRef)
      .attr("width", width + margin.left + margin.right)
      .attr("height", height + margin.top + margin.bottom)
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    const data = processedData.map(d => ({
      x: new Date(d.timestamp),
      y: d.value,
      min: d.min,
      max: d.max,
      type: d.type,
      count: d.count
    })).sort((a, b) => a.x - b.x);

    const x = d3.scaleTime()
      .domain(d3.extent(data, d => d.x))
      .range([0, width]);

    const y = d3.scaleLinear()
      .domain([
        d3.min(data, d => (d.min !== undefined ? d.min : d.y)) * 0.95, 
        d3.max(data, d => (d.max !== undefined ? d.max : d.y)) * 1.05
      ])
      .range([height, 0]);

    // Рисуем график в зависимости от выбранного типа
    switch (chartType) {
      case "line":
        drawLineChart(svg, data, x, y, width, height);
        break;
      case "bar":
        drawBarChart(svg, data, x, y, width, height);
        break;
      case "area":
        drawAreaChart(svg, data, x, y, width, height);
        break;
    }

    // Оси
    svg.append("g")
      .attr("transform", `translate(0,${height})`)
      .call(d3.axisBottom(x));

    svg.append("g")
      .call(d3.axisLeft(y));

    // Подписи осей
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
  
  function drawLineChart(svg, data, x, y, width, height) {
    // Основная линия
    const line = d3.line()
      .x(d => x(d.x))
      .y(d => y(d.y));
    
    svg.append("path")
      .datum(data)
      .attr("fill", "none")
      .attr("stroke", "steelblue")
      .attr("stroke-width", 2)
      .attr("d", line);
    
    // Для daily_minmax рисуем дополнительные линии
    if (averagingType === "daily_minmax") {
      const minData = data.filter(d => d.type === "min");
      const maxData = data.filter(d => d.type === "max");
      const avgData = data.filter(d => d.type === "avg");
      
      if (minData.length > 0) {
        const minLine = d3.line()
          .x(d => x(d.x))
          .y(d => y(d.y));
        
        svg.append("path")
          .datum(minData)
          .attr("fill", "none")
          .attr("stroke", "red")
          .attr("stroke-width", 1)
          .attr("stroke-dasharray", "5,5")
          .attr("d", minLine);
      }
      
      if (maxData.length > 0) {
        const maxLine = d3.line()
          .x(d => x(d.x))
          .y(d => y(d.y));
        
        svg.append("path")
          .datum(maxData)
          .attr("fill", "none")
          .attr("stroke", "green")
          .attr("stroke-width", 1)
          .attr("stroke-dasharray", "5,5")
          .attr("d", maxLine);
      }
    }
    
    // Точки
    svg.selectAll(".dot")
      .data(data)
      .enter().append("circle")
      .attr("class", "dot")
      .attr("cx", d => x(d.x))
      .attr("cy", d => y(d.y))
      .attr("r", 3)
      .attr("fill", d => {
        if (averagingType === "daily_minmax") {
          if (d.type === "min") return "red";
          if (d.type === "max") return "green";
          return "steelblue";
        }
        return "steelblue";
      });
  }
  
  function drawBarChart(svg, data, x, y, width, height) {
    // Определяем ширину столбцов в зависимости от интервала
    let barWidth;
    if (averagingType === "raw") {
      // Для сырых данных используем очень узкие столбцы
      barWidth = 2;
    } else {
      // Для осредненных данных вычисляем ширину на основе интервала
      if (data.length > 1) {
        const interval = (x(data[1].x) - x(data[0].x)) * 0.7;
        barWidth = Math.max(2, Math.min(20, interval));
      } else {
        barWidth = 10;
      }
    }
    
    // Рисуем столбцы
    svg.selectAll(".bar")
      .data(data)
      .enter().append("rect")
      .attr("class", "bar")
      .attr("x", d => x(d.x) - barWidth/2)
      .attr("y", d => y(d.y))
      .attr("width", barWidth)
      .attr("height", d => height - y(d.y))
      .attr("fill", "steelblue")
      .attr("opacity", 0.7);
  }
  
  function drawAreaChart(svg, data, x, y, width, height) {
    // Область под кривой
    const area = d3.area()
      .x(d => x(d.x))
      .y0(height)
      .y1(d => y(d.y));
    
    svg.append("path")
      .datum(data)
      .attr("fill", "steelblue")
      .attr("opacity", 0.3)
      .attr("d", area);
    
    // Линия поверх области
    const line = d3.line()
      .x(d => x(d.x))
      .y(d => y(d.y));
    
    svg.append("path")
      .datum(data)
      .attr("fill", "none")
      .attr("stroke", "steelblue")
      .attr("stroke-width", 2)
      .attr("d", line);
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

  // Обработчики изменений для выпадающих списков
  function handleAveragingChange(e) {
    averagingType = e.target.value;
    processData();
  }
  
  function handleChartTypeChange(e) {
    chartType = e.target.value;
    setTimeout(drawChart, 100);
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
    processData();
  } else {
    selectedValues = [];
    processedData = [];
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

    {#if selectedUName && selectedSerial && selectedDid}
      <div class="processing-controls">
        <div class="select-group">
          <label>Тип осреднения данных:</label>
          <select value={averagingType} on:change={handleAveragingChange}>
            <option value="raw">Исходные данные</option>
            <option value="hourly">Осреднение за час</option>
            <option value="3hourly">Осреднение за 3 часа</option>
            <option value="daily">Осреднение за сутки</option>
            <option value="daily_minmax">Min/Max за сутки</option>
          </select>
        </div>

        <div class="select-group">
          <label>Тип графика:</label>
          <select value={chartType} on:change={handleChartTypeChange}>
            <option value="line">Линейный</option>
            <option value="bar">Столбчатый</option>
            <option value="area">Областной</option>
          </select>
        </div>
      </div>
    {/if}

    {#if processedData.length > 0}
      <div class="data-info">
        <p>
          Показано {processedData.length} {processedData.length === 1 ? 'значение' : 
          processedData.length < 5 ? 'значения' : 'значений'} 
          {averagingType === "raw" ? 'исходных данных' : 
           averagingType === "hourly" ? 'осредненных за час' :
           averagingType === "3hourly" ? 'осредненных за 3 часа' :
           averagingType === "daily" ? 'осредненных за сутки' :
           'минимальных/максимальных значений за сутки'}
        </p>
      </div>

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
      </div>
    
    {:else if selectedDid}
      <div class="no-data">
        <p>Нет данных для выбранного параметра</p>
      </div>
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

  .processing-controls {
    display: flex;
    gap: 20px;
    margin: 20px 0;
    flex-wrap: wrap;
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

  .data-info {
    margin: 10px 0;
    padding: 10px;
    background-color: #f0f8ff;
    border-left: 4px solid #4caf50;
    border-radius: 4px;
  }

  .data-info p {
    margin: 0;
    color: #2c5282;
    font-style: italic;
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

  .no-data {
    text-align: center;
    padding: 40px;
    color: #666;
    font-style: italic;
  }

  @media (max-width: 768px) {
    .selectors, .date-controls, .processing-controls {
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