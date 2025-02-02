<script lang="ts">
    type QueueType = 'system' | 'interactive' | 'batch' | 'idle';
    type Algorithm = 'sjf' | 'ljf' | 'hrrn' | 'multilevel';
  
    interface Process {
      id: number;
      arrivalTime: number;
      burstTime: number;
      priority: number;
      queue: QueueType;
    }
  
    interface ScheduleSlot {
      id: number | 'idle';
      start: number;
      end: number;
      queue: QueueType;
    }
  
    interface ScheduleResults {
      schedule: ScheduleSlot[];
      avgWaitingTime: string;
      avgTurnaroundTime: string;
      avgResponseRatio: string;
    }
  
    interface QueuePriorities {
      [key: string]: number;
      system: number;
      interactive: number;
      batch: number;
    }
  
    interface QueueColors {
      [key: string]: string;
      system: string;
      interactive: string;
      batch: string;
      idle: string;
    }
  
    // Process queue data
    let processes: Process[] = [
      { id: 1, arrivalTime: 0, burstTime: 4, priority: 1, queue: 'system' },
      { id: 2, arrivalTime: 1, burstTime: 3, priority: 2, queue: 'interactive' },
      { id: 3, arrivalTime: 2, burstTime: 2, priority: 1, queue: 'batch' }
    ];
    
    let nextId = 4;
    let selectedAlgorithm: Algorithm = 'sjf';
    let currentTime = 0;
    
    let results: ScheduleResults = {
      schedule: [],
      avgWaitingTime: '0',
      avgTurnaroundTime: '0',
      avgResponseRatio: '0'
    };
  
    // Queue priorities for multilevel queue
    const queuePriorities: QueuePriorities = {
      system: 1,
      interactive: 2,
      batch: 3
    };
  
    function addProcess(): void {
      processes = [...processes, {
        id: nextId++,
        arrivalTime: 0,
        burstTime: 1,
        priority: 1,
        queue: 'batch'
      }];
    }
  
    function removeProcess(id: number): void {
      processes = processes.filter(p => p.id !== id);
    }
  
    function calculateResponseRatio(process: Process, currentTime: number): number {
      const waitingTime = Math.max(0, currentTime - process.arrivalTime);
      return (waitingTime + process.burstTime) / process.burstTime;
    }
  
    function findNextProcess(
      remainingProcesses: Process[], 
      currentTime: number, 
      algorithm: Algorithm
    ): Process | null {
      if (remainingProcesses.length === 0) return null;
      
      let availableProcesses = remainingProcesses.filter(p => p.arrivalTime <= currentTime);
      if (availableProcesses.length === 0) {
        return remainingProcesses.reduce((min, p) => 
          p.arrivalTime < min.arrivalTime ? p : min
        );
      }
  
      switch (algorithm) {
        case 'sjf':
          return availableProcesses.reduce((min, p) => 
            p.burstTime < min.burstTime ? p : min
          );
        
        case 'ljf':
          return availableProcesses.reduce((max, p) => 
            p.burstTime > max.burstTime ? p : max
          );
        
        case 'hrrn':
          return availableProcesses.reduce((max, p) => {
            const ratio1 = calculateResponseRatio(max, currentTime);
            const ratio2 = calculateResponseRatio(p, currentTime);
            return ratio2 > ratio1 ? p : max;
          });
        
        case 'multilevel':
          // Sort by queue priority first, then by arrival time
          return availableProcesses.reduce((selected, p) => {
            if (queuePriorities[p.queue] < queuePriorities[selected.queue]) return p;
            if (queuePriorities[p.queue] === queuePriorities[selected.queue]) {
              return p.arrivalTime < selected.arrivalTime ? p : selected;
            }
            return selected;
          });
  
        default:
          return availableProcesses[0];
      }
    }
  
    function calculateSchedule(): void {
      let remainingProcesses = [...processes].sort((a, b) => a.arrivalTime - b.arrivalTime);
      let schedule: ScheduleSlot[] = [];
      let currentTime = 0;
      let waitingTimes: number[] = [];
      let turnaroundTimes: number[] = [];
      let responseRatios: number[] = [];
  
      while (remainingProcesses.length > 0) {
        const nextProcess = findNextProcess(remainingProcesses, currentTime, selectedAlgorithm);
        
        if (!nextProcess) break;
  
        // Add idle time if there's a gap
        if (nextProcess.arrivalTime > currentTime) {
          schedule.push({
            id: 'idle',
            start: currentTime,
            end: nextProcess.arrivalTime,
            queue: 'idle'
          });
          currentTime = nextProcess.arrivalTime;
        }
  
        // Calculate metrics
        const waitingTime = currentTime - nextProcess.arrivalTime;
        waitingTimes.push(waitingTime);
        
        const turnaroundTime = waitingTime + nextProcess.burstTime;
        turnaroundTimes.push(turnaroundTime);
        
        const responseRatio = calculateResponseRatio(nextProcess, currentTime);
        responseRatios.push(responseRatio);
  
        // Add process to schedule
        schedule.push({
          id: nextProcess.id,
          start: currentTime,
          end: currentTime + nextProcess.burstTime,
          queue: nextProcess.queue
        });
  
        currentTime += nextProcess.burstTime;
        remainingProcesses = remainingProcesses.filter(p => p.id !== nextProcess.id);
      }
  
      results = {
        schedule,
        avgWaitingTime: (waitingTimes.reduce((a, b) => a + b, 0) / waitingTimes.length).toFixed(2),
        avgTurnaroundTime: (turnaroundTimes.reduce((a, b) => a + b, 0) / turnaroundTimes.length).toFixed(2),
        avgResponseRatio: (responseRatios.reduce((a, b) => a + b, 0) / responseRatios.length).toFixed(2)
      };
    }
  
    const queueColors: QueueColors = {
      system: 'bg-red-200',
      interactive: 'bg-blue-200',
      batch: 'bg-green-200',
      idle: 'bg-gray-200'
    };
  </script>
  
  <main class="p-4">
    <h1 class="text-2xl font-bold mb-4">CPU Scheduling Simulator</h1>
    
    <div class="mb-4 flex gap-4">
      <button 
        class="bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600"
        on:click={addProcess}
      >
        Add Process
      </button>
      
      <select 
        bind:value={selectedAlgorithm}
        class="border rounded px-2"
      >
        <option value="sjf">Shortest Job First</option>
        <option value="ljf">Longest Job First</option>
        <option value="hrrn">Highest Response Ratio Next</option>
        <option value="multilevel">Multilevel Queue</option>
      </select>
    </div>
    
    <div class="mb-6">
      <table class="w-full border-collapse border">
        <thead>
          <tr class="bg-gray-100">
            <th class="border p-2">Process ID</th>
            <th class="border p-2">Arrival Time</th>
            <th class="border p-2">Burst Time</th>
            <th class="border p-2">Queue Type</th>
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
                <select 
                  bind:value={process.queue}
                  class="p-1 border rounded"
                >
                  <option value="system">System</option>
                  <option value="interactive">Interactive</option>
                  <option value="batch">Batch</option>
                </select>
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
              class="{queueColors[slot.queue]} border p-2 text-center"
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
        <p>Average Response Ratio: {results.avgResponseRatio}</p>
      </div>
    {/if}
  </main>