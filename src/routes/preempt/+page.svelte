<script lang="ts">
    type QueueType = 'system' | 'interactive' | 'batch' | 'idle';
    type Algorithm = 'round-robin' | 'srtf' | 'lrtf' | 'priority' | 'multilevel';
    
    interface Process {
      id: number;
      arrivalTime: number;
      burstTime: number;
      remainingTime: number;
      priority: number;
      queue: QueueType;
      quantumUsed?: number;
      lastExecutionTime?: number;
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
    
    const TIME_QUANTUM = 2;
    const QUEUE_QUANTUM = {
      system: 4,
      interactive: 3,
      batch: 2
    };
    
    let processes: Process[] = [
      { 
        id: 1, 
        arrivalTime: 0, 
        burstTime: 4, 
        remainingTime: 4, 
        priority: 1, 
        queue: 'system',
        quantumUsed: 0 
      },
      { 
        id: 2, 
        arrivalTime: 1, 
        burstTime: 3, 
        remainingTime: 3, 
        priority: 2, 
        queue: 'interactive',
        quantumUsed: 0 
      },
      { 
        id: 3, 
        arrivalTime: 2, 
        burstTime: 2, 
        remainingTime: 2, 
        priority: 3, 
        queue: 'batch',
        quantumUsed: 0 
      }
    ];
    
    let nextId = 4;
    let selectedAlgorithm: Algorithm = 'round-robin';
    let results: ScheduleResults = {
      schedule: [],
      avgWaitingTime: '0',
      avgTurnaroundTime: '0',
      avgResponseRatio: '0'
    };
    
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
        remainingTime: 1,
        priority: 1,
        queue: 'batch',
        quantumUsed: 0
      }];
    }
    
    function removeProcess(id: number): void {
      processes = processes.filter(p => p.id !== id);
    }
    
    function shouldPreempt(
      currentProcess: Process | null,
      newProcess: Process,
      algorithm: Algorithm
    ): boolean {
      if (!currentProcess) return true;
      
      switch (algorithm) {
        case 'srtf':
          return newProcess.remainingTime < currentProcess.remainingTime;
          
        case 'lrtf':
          return newProcess.remainingTime > currentProcess.remainingTime;
          
        case 'priority':
          return newProcess.priority < currentProcess.priority; // Lower number = higher priority
          
        case 'multilevel':
          return queuePriorities[newProcess.queue] < queuePriorities[currentProcess.queue];
          
        case 'round-robin':
          return currentProcess.quantumUsed! >= TIME_QUANTUM;
          
        default:
          return false;
      }
    }
    
    function findNextProcess(
      remainingProcesses: Process[],
      currentTime: number,
      algorithm: Algorithm,
      currentProcess: Process | null
    ): Process | null {
      if (remainingProcesses.length === 0) return null;
      
      let availableProcesses = remainingProcesses.filter(p => 
        p.arrivalTime <= currentTime && p.remainingTime > 0
      );
      
      if (availableProcesses.length === 0) {
        return remainingProcesses.reduce((min, p) => 
          p.arrivalTime < min.arrivalTime ? p : min
        );
      }
    
      switch (algorithm) {
        case 'round-robin':
          if (currentProcess && 
              currentProcess.quantumUsed! < TIME_QUANTUM && 
              currentProcess.remainingTime > 0) {
            return currentProcess;
          }
          return availableProcesses[0];
    
        case 'srtf':
          return availableProcesses.reduce((min, p) => 
            p.remainingTime < min.remainingTime ? p : min
          );
    
        case 'lrtf':
          return availableProcesses.reduce((max, p) => 
            p.remainingTime > max.remainingTime ? p : max
          );
    
        case 'priority':
          return availableProcesses.reduce((highest, p) => 
            p.priority < highest.priority ? p : highest
          );
    
        case 'multilevel':
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
      let remainingProcesses = processes.map(p => ({
        ...p,
        remainingTime: p.burstTime,
        quantumUsed: 0,
        lastExecutionTime: 0
      }));
      
      let schedule: ScheduleSlot[] = [];
      let currentTime = 0;
      let waitingTimes: number[] = [];
      let turnaroundTimes: number[] = [];
      let responseRatios: number[] = [];
      let currentProcess: Process | null = null;
      
      while (remainingProcesses.some(p => p.remainingTime > 0)) {
        const nextProcess = findNextProcess(
          remainingProcesses, 
          currentTime, 
          selectedAlgorithm,
          currentProcess
        );
        
        if (!nextProcess) {
          const nextArrival = Math.min(...remainingProcesses
            .filter(p => p.arrivalTime > currentTime)
            .map(p => p.arrivalTime)
          );
          
          schedule.push({
            id: 'idle',
            start: currentTime,
            end: nextArrival,
            queue: 'idle'
          });
          
          currentTime = nextArrival;
          continue;
        }
    
        let timeSlice = 1;  // Default for SRTF, LRTF, and Priority
        
        if (selectedAlgorithm === 'round-robin') {
          timeSlice = Math.min(
            TIME_QUANTUM - (nextProcess.quantumUsed || 0),
            nextProcess.remainingTime
          );
        } else if (selectedAlgorithm === 'multilevel') {
          timeSlice = Math.min(
            QUEUE_QUANTUM[nextProcess.queue] - (nextProcess.quantumUsed || 0),
            nextProcess.remainingTime
          );
        }
    
        const interruptingProcess = remainingProcesses.find(p => 
          p.arrivalTime > currentTime &&
          p.arrivalTime < currentTime + timeSlice &&
          shouldPreempt(nextProcess, p, selectedAlgorithm)
        );
    
        if (interruptingProcess) {
          timeSlice = interruptingProcess.arrivalTime - currentTime;
        }
    
        schedule.push({
          id: nextProcess.id,
          start: currentTime,
          end: currentTime + timeSlice,
          queue: nextProcess.queue
        });
    
        nextProcess.remainingTime -= timeSlice;
        nextProcess.quantumUsed = (nextProcess.quantumUsed || 0) + timeSlice;
        nextProcess.lastExecutionTime = currentTime + timeSlice;
    
        if (nextProcess.remainingTime === 0) {
          const waitingTime = (currentTime + timeSlice) - nextProcess.arrivalTime - nextProcess.burstTime;
          const turnaroundTime = (currentTime + timeSlice) - nextProcess.arrivalTime;
          const responseRatio = (waitingTime + nextProcess.burstTime) / nextProcess.burstTime;
          
          waitingTimes.push(waitingTime);
          turnaroundTimes.push(turnaroundTime);
          responseRatios.push(responseRatio);
        }
    
        currentTime += timeSlice;
        currentProcess = nextProcess;
    
        if (nextProcess.quantumUsed >= (selectedAlgorithm === 'multilevel' ? 
            QUEUE_QUANTUM[nextProcess.queue] : TIME_QUANTUM)) {
          nextProcess.quantumUsed = 0;
        }
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
      <h1 class="text-2xl font-bold mb-4">Preemptive CPU Scheduling Simulator</h1>
      
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
          <option value="round-robin">Round Robin</option>
          <option value="srtf">Shortest Remaining Time First</option>
          <option value="lrtf">Longest Remaining Time First</option>
          <option value="priority">Priority Scheduling</option>
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
              <th class="border p-2">Priority</th>
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
                  <input 
                    type="number" 
                    class="w-20 p-1 border rounded"
                    bind:value={process.priority}
                    min="1"
                  />
                </td>
                <td class="border p-2">
                  <select 
                    bind:value={process.queue}
                    class="w-1/2 p-1 border rounded"
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