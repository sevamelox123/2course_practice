<!-- local ip after running rollup http://localhost:8080 -->
<!-- <script>
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

      if (sdate > edate)
        alert("Начальная дата больше конечной! Измените для продолжения.");
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

      const response = await fetch("./data.json");
      const jsonData = await response.json();
      // const devicesMap = new Map();

      // for (const id in jsonData) {
      //   const device = jsonData[id];

      //   if (devicesMap.has(device.uName)) {
      //     devicesMap.get(device.uName).add(device.serial);
      //   } else {
      //     devicesMap.set(device.uName, new Set([device.serial]));
      //   }
      // }
      const devicesHierarchy = new Map();

      for (const id in jsonData) {
        const device = jsonData[id];
        const { uName, serial, data, Date: timestamp } = device;

        // 1. Уровень: uName
        if (!devicesHierarchy.has(uName)) {
          devicesHierarchy.set(uName, new Map());
        }

        const serialsMap = devicesHierarchy.get(uName);

        // 2. Уровень: serial
        if (!serialsMap.has(serial)) {
          serialsMap.set(serial, new Map());
        }

        const dataMap = serialsMap.get(serial);

        // 3. Уровень: данные (did -> значения)
        for (const did in data) {
          if (data.hasOwnProperty(did)) {
            if (!dataMap.has(did)) {
              dataMap.set(did, []);
            }

            // Добавляем само значение
            dataMap.get(did).push({
              value: data[did],
              timestamp: timestamp,
              originalId: id,
            });
          }
        }
      }

      // ВЫВОДИМ САМИ ЗНАЧЕНИЯ
      console.log("Все значения данных:");
      devicesHierarchy.forEach((serialsMap, uName) => {
        console.log(`\n=== ${uName} ===`);

        serialsMap.forEach((dataMap, serial) => {
          console.log(`\nSerial: ${serial}`);

          dataMap.forEach((values, did) => {
            console.log(`\n  ${did}:`);

            values.forEach((item, index) => {
              console.log(
                `    [${index + 1}] ${item.timestamp}: ${item.value}`,
              );
            });
          });
        });
      });

      // const uniqueDevices = Array.from(devicesMap, ([uName, serialsSet]) => ({
      //   uName,
      //   serials: Array.from(serialsSet),
      // }));

      // console.log("Уникальные серийные номера:");
      // uniqueDevices.forEach((device) => {
      //   console.log(device.uName, device.serials);
      // });

      // for (const key in jsonData) {
      //   if (jsonData.hasOwnProperty(key)) {
      //     const device = jsonData[key];
      //     console.log(`Устройство ${key}:`);
      //     console.log(`Дата: ${device.Date}`);
      //     console.log(`Серийный номер: ${device.serial}`);
      //     for (const did in device.data) {
      //       console.log(`name ${did} value ${device.data[key]} `);
      //     }
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
    let mounth = partsArray[1];
    let day = partsArray[2];
    var partsArray2 = time.split(":");
    let hour = partsArray2[0];
    let minute = partsArray2[1];
    let second = partsArray2[2];
    return { year, mounth, day, hour, minute, second };
  }
  // example
  const options = [
    { value: "", label: "Выберите устройство" },
    { value: "device1", label: "Hydra-Lite (01)" },
    { value: "device2", label: "Hydra-Lite (30)" },
    { value: "device3", label: "Тест Студии (schHome)" },
  ];
  // example
  let selectedValue = "";
</script>
\\<select bind:value={selectedValue}>
  {#each options as option}
    <option value={option.value}>{option.label}</option>
  {/each}
</select>

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
</main> -->

<!-- ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++ -->
<script>
  let jsonData = [];
  let devicesHierarchy = new Map();
  let selectedUName = "";
  let selectedSerial = "";
  let selectedDid = "";
  let selectedValues = [];
  let isLoading = false;

  async function loadData() {
    try {
      let startDate = "";
      let startTime = "";
      let endDate = "";
      let endTime = "";
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

      if (sdate > edate)
        alert("Начальная дата больше конечной! Измените для продолжения.");
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

      isLoading = true;
      // const response = await fetch("./data.json");
      const response = await fetch(
        `/api/not_calibr/log/${startYear}-${startMonth}-${startDay}%20${startHours}:${startMinutes}:${startSeconds}/${endYear}-${endMonth}-${endDay}%20${endHours}:${endMinutes}:${endSeconds}/`,
      );
      jsonData = await response.json();

      buildHierarchy();
    } catch (error) {
      console.error("Ошибка загрузки:", error);
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
            timestamp,
            originalId: id,
          });
        }
      }
    }
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
  } else {
    selectedValues = [];
  }

  // function getUnit(did) {
  //   const lowerDid = did.toLowerCase();
  //   if (lowerDid.includes("temp")) return "°C";
  //   if (lowerDid.includes("humidity")) return "%";
  //   if (lowerDid.includes("pressure")) return "hPa";
  //   if (lowerDid.includes("rssi")) return "dBm";
  //   if (lowerDid.includes("upit")) return "V";
  //   if (lowerDid.includes("lux")) return "lx";
  //   if (lowerDid.includes("tvoc") || lowerDid.includes("eco2")) return "ppm";
  //   return "";
  // } // мб хуйня

  let startDate = "";
  let startTime = "";
  let endDate = "";
  let endTime = "";

  function getDateTime(date, time) {
    var partsArray = date.split("-");
    let year = partsArray[0];
    let mounth = partsArray[1];
    let day = partsArray[2];
    var partsArray2 = time.split(":");
    let hour = partsArray2[0];
    let minute = partsArray2[1];
    let second = partsArray2[2];
    return { year, mounth, day, hour, minute, second };
  }
</script>

<div class="container">
  <button on:click={loadData} disabled={isLoading}>
    {isLoading ? "Загрузка..." : "Загрузить данные"}
  </button>

  {#if devicesHierarchy.size > 0}
    <div class="selectors">
      <!-- Выбор uName -->
      <div class="select-group">
        <label>Выберите устройство:</label>
        <select bind:value={selectedUName}>
          <option value="">-- Выберите uName --</option>
          {#each uNames as uName}
            <option value={uName}>{uName}</option>
          {/each}
        </select>
      </div>

      <!-- Выбор serial -->
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

      <!-- Выбор did -->
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
      <div class="results">
        <h3>
          {selectedUName} / {selectedSerial} / {selectedDid}
          <span class="count">({selectedValues.length} значений)</span>
        </h3>

        <div class="values-table">
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
                  <td class="timestamp">{item.timestamp}</td>
                  <td class="value">
                    {item.value}
                    {getUnit(selectedDid)}
                  </td>
                  <td class="id">{item.originalId}</td>
                </tr>
              {/each}
            </tbody>
          </table>
        </div>

        <div class="stats">
          <h4>Статистика:</h4>
          <p>Количество измерений: {selectedValues.length}</p>
          {#if selectedValues.length > 0}
            <p>
              Минимальное значение: {Math.min(
                ...selectedValues.map((v) => v.value),
              )}
              {getUnit(selectedDid)}
            </p>
            <p>
              Максимальное значение: {Math.max(
                ...selectedValues.map((v) => v.value),
              )}
              {getUnit(selectedDid)}
            </p>
            <p>
              Среднее значение: {(
                selectedValues.reduce((sum, v) => sum + v.value, 0) /
                selectedValues.length
              ).toFixed(2)}
              {getUnit(selectedDid)}
            </p>
          {/if}
        </div>
      </div>
    {:else if selectedDid}
      <div class="no-data">
        <p>Нет данных для выбранного параметра</p>
      </div>
    {/if}
  {/if}
</div>

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

<style>
  .container {
    max-width: 1000px;
    margin: 0 auto;
    padding: 20px;
  }

  button {
    padding: 10px 20px;
    background: #4caf50;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    margin-bottom: 20px;
  }

  button:disabled {
    background: #cccccc;
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

  select {
    padding: 10px;
    border: 2px solid #ddd;
    border-radius: 5px;
    font-size: 14px;
  }

  select:focus {
    outline: none;
    border-color: #4caf50;
  }

  .results {
    background: #f9f9f9;
    padding: 20px;
    border-radius: 8px;
    border: 1px solid #ddd;
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
  }

  table {
    width: 100%;
    border-collapse: collapse;
    background: white;
  }

  th,
  td {
    padding: 12px;
    text-align: left;
    border: 1px solid #ddd;
  }

  th {
    background: #f0f0f0;
    font-weight: bold;
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
    .selectors {
      flex-direction: column;
    }

    .select-group {
      width: 100%;
    }
  }
</style>
