<script context="module">
  export async function preload(page, session) {
    const { slug } = page.params;

    return { slug };
  }
</script>

<script>
  import { goto } from "@sapper/app";
  import { userId, jwt, isAdmin } from "../../stores.js";
  import { formatDate } from "../../utils.js";
  import Panel from "../../components/Panel.svelte";
  import { onMount } from "svelte";
  export let slug;
  const ticketSelection = {};
  let purchasable = {};
  let events = [];

  const getPurchasable = async () => {
    const res = await fetch(
      `https://folda-box-office-system.herokuapp.com/purchasables/${slug}/`,
      {
        mode: "cors",
        headers: {
          "Content-Type": "application/json"
        }
      }
    );
    purchasable = await res.json();
    events = purchasable.events;
    if (purchasable && purchasable.ticketClasses) {
      purchasable.ticketClasses.forEach(tc => {
        ticketSelection[tc.id] = 0;
      });
    }
  };

  onMount(() => {
    getPurchasable();
  });

  let selectedEvents = [];

  $: selectedEventIds = selectedEvents.map(e => e.id);

  $: conflictsWithSelection = (start, end) =>
    selectedEvents.some(
      event =>
        new Date(start) <= new Date(event.endTime) &&
        new Date(end) >= new Date(event.startTime)
    );

  $: addToCart = () => {
    Object.keys(ticketSelection)
      .filter(ticketClassId => ticketSelection[ticketClassId] > 0)
      .forEach(ticketClassId => {
        fetch(
          `https://folda-box-office-system.herokuapp.com/users/${$userId}/cart/`,
          {
            mode: "cors",
            method: "POST",
            headers: {
              "Content-Type": "application/json",
              Authorization: `Bearer ${$jwt}`
            },
            body: JSON.stringify({
              purchasableId: purchasable.id,
              ticketClassId: Number(ticketClassId),
              quantity: ticketSelection[ticketClassId],
              events: selectedEventIds
            })
          }
        ).then(r => {
          if (r.ok) {
            goto(`/Cart`);
          } else {
            alert("Something went wrong. Please try again in a moment.");
          }
        });
      });
  };

  $: deletePass = async () => {
    await fetch(
      `https://folda-box-office-system.herokuapp.com/purchasables/${slug}/`,
      {
        mode: "cors",
        method: "DELETE",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${$jwt}`
        }
      }
    );
    goto("/");
  };
</script>

<style>
  .heading {
    font-family: "Verdana", sans-serif;
    font-size: 1.6rem;
    color: rgba(64, 69, 237);
    font-weight: bolder;
  }

  table {
    background-color: white;
    border-radius: 4px;
    border: 1px solid #ddd;
    width: 100%;
  }

  table td,
  table th {
    padding: 0.2rem;
    text-align: center;
  }

  .description {
    padding: 1rem;
  }

  .deleteDayPass {
    background-color: red;
  }

  .headingWrapper > * {
    margin: 0;
  }

  .twoColumns {
    display: flex;
  }

  .dayEvent {
    display: flex;
    margin: 1rem 0;
    align-items: center;
  }

  .thumbnail {
    border-radius: 50%;
    overflow: hidden;
    width: 6rem;
    height: 6rem;
    margin-right: 1.5rem;
  }

  .thumbnail img {
    height: 100%;
    width: 100%;
  }

  .dayEventHeading {
    display: flex;
    align-items: center;
  }
  .dayEventHeading h4 {
    margin: 0;
    padding: 0;
    font-weight: bold;
    text-decoration: underline;
    font-size: 1.4rem;
  }

  .dayEventHeading time {
    margin: 0;
    padding: 0;
    margin-left: 2rem;
  }

  .artistName {
    font-style: italic;
  }
  .removeButtonWrapper {
    margin-left: 1.5rem;
    display: flex;
    justify-content: flex-end;
    flex-grow: 1;
  }

  button.addToCart {
    margin-top: 1rem;
  }

  .conflicts {
    color: red;
  }
</style>

<svelte:head>
  <title>Cart</title>
</svelte:head>
<h1 class="heading">{purchasable.name}</h1>
{#if $isAdmin == 'yes'}
  <a class="button" href={`/edit-day-pass/${purchasable.id}`}>Edit day pass</a>
  <button class="deleteDayPass" on:click={deletePass}>Delete day pass</button>
{/if}
<Panel title="Details">
  <div class="description">{purchasable.description}</div>
</Panel>
<Panel title="Select tickets">
  <table>
    <thead>
      <tr>
        <th class="tableheader">Type</th>
        <th class="tableheader">Price</th>
        <th class="tableheader">Quantity</th>
      </tr>
    </thead>
    <tbody>
      {#if purchasable && purchasable.ticketClasses}
        {#each purchasable.ticketClasses as tc}
          <tr>
            <td>{tc.description}</td>
            <td>${tc.price}</td>
            <td>
              <input
                type="number"
                bind:value={ticketSelection[tc.id]}
                on:change={() => {
                  ticketSelection = ticketSelection;
                }} />
            </td>
          </tr>
        {/each}
      {/if}
    </tbody>
  </table>
</Panel>
<div class="twoColumns">
  <Panel title="Select events">
    {#each events as event}
      <div class="dayEvent">
        <div class="thumbnail">
          <img src={`events/${event.imageUrl}`} alt={event.name} />
        </div>
        <div>
          <div class="dayEventHeading">
            <h4>{event.name}</h4>
            <time
              class={!selectedEventIds.includes(event.id) && conflictsWithSelection(event.startTime, event.endTime) ? 'conflicts' : ''}>
              {formatDate(event.startTime)}
            </time>
          </div>
          <div class="artistName">{event.artistName}</div>
          {#if event.embedMedia}
            <div class="video">
              <iframe
                title="video"
                width="280"
                height="157.5"
                src={event.embedMedia}
                frameborder="0"
                allow="accelerometer; autoplay; encrypted-media; gyroscope;
                picture-in-picture"
                allowfullscreen />
            </div>
          {/if}
        </div>
        <div class="removeButtonWrapper">
          {#if selectedEventIds.includes(event.id)}
            <button
              on:click={() => {
                selectedEvents = selectedEvents.filter(e => e.id !== event.id);
              }}>
              Remove
            </button>
          {:else}
            <button
              on:click={() => {
                if (!conflictsWithSelection(event.startTime, event.endTime) || confirm('This event conflicts with your current selection. Do you want to continue?')) {
                  selectedEvents = [...selectedEvents, event];
                }
              }}>
              Add
            </button>
          {/if}
        </div>
      </div>
    {/each}
  </Panel>
  <Panel title="Schedule">
    {#if selectedEvents.length}
      <table>
        <thead>
          <tr>
            <th>Time</th>
            <th>Event start</th>
            <th>Event end</th>
          </tr>
        </thead>
        <tbody>
          {#each selectedEvents as event}
            <tr>
              <td>{event.name}</td>
              <td>{formatDate(event.startTime)}</td>
              <td>{formatDate(event.endTime)}</td>
            </tr>
          {/each}
        </tbody>
      </table>
      <div>
        <button class="addToCart" on:click={addToCart}>Add to cart</button>
      </div>
    {:else}No events selected!{/if}
  </Panel>
</div>
