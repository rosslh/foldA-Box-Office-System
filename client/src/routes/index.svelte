<script>
  import { onMount } from "svelte";
  import DayPass from "../components/DayPass.svelte";
  import IndividualEvent from "../components/IndividualEvent.svelte";
  import { isAdmin } from "../stores.js";

  let purchasables = [];

  onMount(async () => {
    const result = await fetch(
      "https://folda-box-office-system.herokuapp.com/purchasables/",
      {
        mode: "cors",
        headers: {
          "Content-Type": "application/json"
        }
      }
    );
    purchasables = await result.json();
  });
</script>

<style>
  .events {
    display: block;
  }
  .heading {
    font-family: "Verdana", sans-serif;
    font-size: 40px;
    color: rgba(64, 69, 237);
    text-transform: uppercase;
  }
</style>

<svelte:head>
  <title>Events</title>
</svelte:head>
<h1 class="heading">Events</h1>
{#if $isAdmin == 'yes'}
  <div>
    <a class="button" href="NewIndividualEvent">New single event</a>
    <a class="button" href="NewDayPass">New day pass</a>
  </div>
{/if}
<div class="events">
  {#if purchasables.length}
    {#each purchasables as purchasable}
      {#if purchasable.type === 'individual'}
        <IndividualEvent {purchasable} />
      {:else}
        <DayPass {purchasable} />
      {/if}
    {/each}
  {:else}Loading...{/if}
</div>
