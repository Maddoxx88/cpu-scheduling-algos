<script>
    import { page } from '$app/state';
  </script>
  
  <div class="flex h-screen border justify-center items-center flex-col">
    <h1 class="text-4xl font-medium text-center">{page.status}: {page.error?.message}</h1>
    <a href="/" class="text-blue-500 underline my-2">Go home</a>
  </div>
