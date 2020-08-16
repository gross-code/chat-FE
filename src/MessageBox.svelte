<script>
  export let messages;
</script>

<div>
  {#each messages as message}
    <p>{message}</p>
  {/each}
</div>
