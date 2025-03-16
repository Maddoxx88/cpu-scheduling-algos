<script>
    // Sample data for dynamic cards
    const cards = [
      { id: 1, title: "Emergency Room", description: "This is the first card", imageUrl: "https://via.placeholder.com/300x200" },
      { id: 2, title: "Earth Satellite Communication", description: "This is the second card", imageUrl: "https://via.placeholder.com/300x200" },
      { id: 3, title: "Space Mission Control", description: "This is the third card", imageUrl: "https://via.placeholder.com/300x200" },
      { id: 4, title: "Card 4", description: "This is the fourth card", imageUrl: "https://via.placeholder.com/300x200" }
    ];
  
    // Handle card click
    function handleCardClick(id) {
      alert(`Card ${id} clicked`);
    }
  </script>
  
  <main class="max-w-6xl mx-auto p-6">

      <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6 p-6">
        {#each cards as card}
          <div 
          aria-roledescription="card"
            class="group relative bg-gray-800 rounded-lg shadow-lg cursor-pointer overflow-hidden"
            on:click={() => handleCardClick(card.id)}>
            
            <!-- Background Image -->
            <div 
              class="absolute inset-0 bg-cover bg-center transition-all duration-300 group-hover:opacity-75"
              style="background-image: url('https://flowbite.com/docs/images/blog/image-1.jpg');">
            </div>
            
            <!-- Card Content -->
            <div class="relative p-6 space-y-4 z-10">
              <h2 class="text-4xl font-semibold text-white">{card.title}</h2>
              <p class="text-gray-300 text-xl">{card.description}</p>
            </div>
          </div>
        {/each}
      </div>
      
  </main>
  