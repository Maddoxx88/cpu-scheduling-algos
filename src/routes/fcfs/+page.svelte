<script>
    let processes = [
      { id: 1, arrivalTime: 0, burstTime: 4 },
      { id: 2, arrivalTime: 1, burstTime: 3 },
      { id: 3, arrivalTime: 2, burstTime: 2 }
    ];
  
    let nextId = 4;
    let results = { 
      schedule: [],
      avgWaitingTime: 0,
      avgTurnaroundTime: 0
    };
  
    function addProcess() {
      processes = [...processes, { 
        id: nextId++, 
        arrivalTime: 0, 
        burstTime: 1 
      }];
    }
  
    function removeProcess(id) {
      processes = processes.filter(p => p.id !== id);
    }
  
    function calculateSchedule() {
      // Sort by arrival time
      let sortedProcesses = [...processes].sort((a, b) => a.arrivalTime - b.arrivalTime);
      
      let currentTime = 0;
      let schedule = [];
      let waitingTimes = [];
      let turnaroundTimes = [];
      
      sortedProcesses.forEach(process => {
        // If there's a gap between processes, add idle time
        if (process.arrivalTime > currentTime) {
          schedule.push({
            id: 'idle',
            start: currentTime,
            end: process.arrivalTime
          });
          currentTime = process.arrivalTime;
        }
        
        // Calculate waiting time
        let waitingTime = currentTime - process.arrivalTime;
        waitingTimes.push(waitingTime);
        
        // Add process to schedule
        schedule.push({
          id: process.id,
          start: currentTime,
          end: currentTime + process.burstTime
        });
        
        // Calculate turnaround time
        let turnaroundTime = waitingTime + process.burstTime;
        turnaroundTimes.push(turnaroundTime);
        
        currentTime += process.burstTime;
      });
      
      results = {
        schedule,
        avgWaitingTime: (waitingTimes.reduce((a, b) => a + b, 0) / waitingTimes.length).toFixed(2),
        avgTurnaroundTime: (turnaroundTimes.reduce((a, b) => a + b, 0) / turnaroundTimes.length).toFixed(2)
      };
    }

  </script>
  
  <main class="p-4">
    <div class="flex mb-8">
        <a href="/" class="text-blue-500 underline mr-4">Go Back</a>
        <h1 class="text-2xl font-bold">FCFS CPU Scheduler</h1>
    </div>
    <div class="mb-4">
      <button 
        class="bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600"
        on:click={addProcess}
      >
        Add Process
      </button>
    </div>
    
    <div class="mb-6">
      <table class="w-full border-collapse border">
        <thead>
          <tr class="bg-gray-100">
            <th class="border p-2">Process ID</th>
            <th class="border p-2">Arrival Time</th>
            <th class="border p-2">Burst Time</th>
            <th class="border p-2">Actions</th>
          </tr>
        </thead>
        <tbody>
          {#each processes as process (process.id)}
            <tr>
              <td class="border p-2">P{process.id}</td>
              <td class="border p-2">
                <input 
                  type="number" 
                  class="w-20 p-1 border rounded"
                  bind:value={process.arrivalTime}
                  min="0"
                />
              </td>
              <td class="border p-2">
                <input 
                  type="number" 
                  class="w-20 p-1 border rounded"
                  bind:value={process.burstTime}
                  min="1"
                />
              </td>
              <td class="border p-2">
                <button 
                  class="bg-red-500 text-white px-2 py-1 rounded hover:bg-red-600"
                  on:click={() => removeProcess(process.id)}
                >
                  Remove
                </button>
              </td>
            </tr>
          {/each}
        </tbody>
      </table>
    </div>
    
    <button 
      class="bg-green-500 text-white px-4 py-2 rounded hover:bg-green-600 mb-4"
      on:click={calculateSchedule}
    >
      Calculate Schedule
    </button>
    
    {#if results.schedule.length > 0}
      <div class="mb-4">
        <h2 class="text-xl font-bold mb-2">Gantt Chart</h2>
        <div class="flex">
          {#each results.schedule as slot}
            <div 
              class="border p-2 text-center"
              style="width: {slot.end - slot.start}00px"
            >
              {slot.id === 'idle' ? 'IDLE' : `P${slot.id}`}
              <div class="text-sm text-gray-500">
                {slot.start} - {slot.end}
              </div>
            </div>
          {/each}
        </div>
      </div>
      
      <div>
        <h2 class="text-xl font-bold mb-2">Results</h2>
        <p>Average Waiting Time: {results.avgWaitingTime}</p>
        <p>Average Turnaround Time: {results.avgTurnaroundTime}</p>
      </div>
    {/if}
  </main>