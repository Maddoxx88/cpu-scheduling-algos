<script lang="ts">
    import { writable } from 'svelte/store';
  
    interface Task {
      id: number;
      name: string;
      burstTime: number;
      arrivalTime: number;
      priority?: number;
    }
  
    const tasks = writable<Task[]>([]);
    let selectedScenario: string = 'Emergency Room (SJF - Preemptive)';
  
    const scenarios: Record<string, Task[]> = {
      'Emergency Room (SJF - Preemptive)': [
        { id: 1, name: 'Patient A', burstTime: 5, arrivalTime: 0 },
        { id: 2, name: 'Patient B', burstTime: 3, arrivalTime: 2 },
        { id: 3, name: 'Patient C', burstTime: 2, arrivalTime: 4 }
      ],
      'Earth Satellite Communication (Priority Scheduling - Preemptive)': [
        { id: 1, name: 'Critical Alert', burstTime: 2, arrivalTime: 0, priority: 1 },
        { id: 2, name: 'Routine Data Sync', burstTime: 4, arrivalTime: 1, priority: 3 },
        { id: 3, name: 'Mission Report', burstTime: 3, arrivalTime: 3, priority: 2 }
      ],
      'Space Mission Control (Round Robin - Preemptive)': [
        { id: 1, name: 'Navigation Update', burstTime: 4, arrivalTime: 0 },
        { id: 2, name: 'Life Support Check', burstTime: 3, arrivalTime: 1 },
        { id: 3, name: 'Communication Check', burstTime: 5, arrivalTime: 2 }
      ]
    };
  
    function loadScenario(): void {
      tasks.set(scenarios[selectedScenario] || []);
    }
  
    async function simulatePreemptiveSJF(): Promise<void> {
      let queue = [...scenarios[selectedScenario]].sort((a, b) => a.arrivalTime - b.arrivalTime);
      let result: Task[] = [];
      let currentTime = 0;
  
      while (queue.length) {
        const availableTasks = queue.filter(task => task.arrivalTime <= currentTime);
        if (availableTasks.length) {
          availableTasks.sort((a, b) => a.burstTime - b.burstTime);
          const currentTask = availableTasks[0];
          result.push({ ...currentTask, burstTime: currentTask.burstTime, arrivalTime: currentTime });
          tasks.set(result);
          await new Promise(resolve => setTimeout(resolve, 1000));
          currentTask.burstTime -= 1;
          if (currentTask.burstTime === 0) {
            queue = queue.filter(task => task.id !== currentTask.id);
          }
        } else {
          currentTime++; // Idle time when no task is available
        }
      }
      tasks.set(result);
    }
  
    async function simulateRoundRobin(timeQuantum = 2): Promise<void> {
      let queue = [...scenarios[selectedScenario]].sort((a, b) => a.arrivalTime - b.arrivalTime);
      let result: Task[] = [];
      let currentTime = 0;
  
      while (queue.length) {
        const currentTask = queue.shift()!;
        if (currentTask.burstTime > timeQuantum) {
          result.push({ ...currentTask, burstTime: timeQuantum, arrivalTime: currentTime });
          currentTime += timeQuantum;
          currentTask.burstTime -= timeQuantum;
          queue.push(currentTask);
        } else {
          result.push({ ...currentTask, burstTime: currentTask.burstTime, arrivalTime: currentTime });
          currentTime += currentTask.burstTime;
        }
        tasks.set(result);
        await new Promise(resolve => setTimeout(resolve, 1000));
      }
    }
  </script>
  
  <div class="max-w-md mx-auto p-6 bg-gray-800 text-white rounded-lg shadow-md">
    <h1 class="text-2xl font-bold mb-4">{selectedScenario}</h1>
  
    <label class="block text-sm font-medium mb-1">Choose Scenario:</label>
    <select bind:value={selectedScenario} class="w-full p-2 rounded bg-gray-700 border border-gray-600 mb-4">
      {#each Object.keys(scenarios) as scenario}
        <option value={scenario}>{scenario}</option>
      {/each}
    </select>
  
    <button
      class="bg-green-500 hover:bg-green-600 px-4 py-2 rounded w-full mb-4"
      on:click={loadScenario}
    >Load Scenario</button>
  
    <button
      class="bg-blue-500 hover:bg-blue-600 px-4 py-2 rounded w-full"
      on:click={selectedScenario.includes('Round Robin') ? simulateRoundRobin() : simulatePreemptiveSJF()}
    >Start Simulation</button>
  
    <div class="mt-4">
      <h2 class="text-lg font-semibold">Task Timeline:</h2>
      {#each $tasks as task (task.id)}
        <div class="bg-gray-700 rounded p-2 my-1">
          {task.name} - {task.burstTime > 0 ? `Remaining: ${task.burstTime}ms` : '✅ Completed'}
        </div>
      {/each}
    </div>
  
    <div class="mt-6 p-4 bg-gray-700 rounded">
      <h2 class="text-lg font-semibold">Explanation:</h2>
      <p><strong>Emergency Room (SJF - Preemptive):</strong> Shortest Job First prioritizes patients with shorter burst times to minimize critical delays.</p>
      <p><strong>Earth Satellite Communication (Priority Scheduling - Preemptive):</strong> Critical alerts are processed first to ensure the most urgent data is transmitted without delay.</p>
      <p><strong>Space Mission Control (Round Robin - Preemptive):</strong> Each system task is given an equal time slice to ensure fair execution and avoid starvation.</p>
    </div>
  </div>
  